+++
title = "Tokio的时间轮结构"
date = 2024-08-05
[taxonomies]
tags = ["Rust", "Tokio", "lib"]
+++

时间轮是`tokio`里处理时间事件的"相关部门", 这篇文章的内容主要是时间事件是什么样的, 时间轮是怎么运作的, 以及, 时间轮和运行时是怎么协作. 

<!-- more -->

## 1. 时间事件和时间轮

### 1.1 有哪些时间事件

粗略点说, 时间事件就是到某个时刻要执行什么操作, `tokio::time`下提供了三种主要类型, 休眠事件`Sleep`, 超时事件`Timeout`和周期事件`Interval`.

休眠事件就是, 到某个时刻, 继续做之前的事; 超时事件就是, 到某个时刻, 如果有件事还没做完, 就取消不做了; 周期事件就是, 到某个时刻, 开始做事, 并且设定在下一个时刻, 还做这件事.

```rust
// 休眠事件
pub struct Sleep {
  entry: TimerEntry,
}

// 超时事件
pub struct Timeout<T> {
  value: T,
  delay: Sleep,
}

// 周期事件
pub struct Interval {
  delay: Pin<Box<Sleep>>,
  period: Duration,
}
```

可以看到, 超时`Timeout`和周期`Interval`都是借助休眠事件`Sleep`实现的, 而`Sleep`里的`TimeEntry`差不多就是时间轮里处理的时间事件了.

```rust
// 休眠Sleep里的数据类型
pub(crate) struct TimerEntry {
  inner: StdUnsafeCell<Option<TimerShared>>,
  // 触发时间
  deadline: Instant,
  registered: bool,
}

// 时间轮操作的数据类型
pub(crate) struct TimerHandle {
  inner: NonNull<TimerShared>,
}

// 两者共同的部分
pub(crate) struct TimerShared {
  shard_id: u32,
  cached_when: AtomicU64,
  state: StateCell,
}

pub(super) struct StateCell {
  state: AtomicU64,
  result: UnsafeCell<TimerResult>,
  // 唤醒Future
  waker: AtomicWaker,
}
```

把上面的代码"小事化了"下, 时间事件差不多可以写成`(when, waker)`, `when`表示在哪个时刻触发, 而`waker`用来唤醒`Future`, 表示要执行的操作是什么.

### 1.2 时间轮是什么

系统中可能存在很多的时间事件, 比如web服务器, 对TCP连接加一个读超时的话, n个连接就至少有n个时间事件, 而管理这些时间事件会面临下面几点挑战:

+ 事件的创建和终止都无法预测, 需要有新增和删除的能力
+ 最近的事件总是会被最先触发, 所以还要提供排序的能力
+ 事件的触发时刻是不确定的, 不能简单预设在某一范围内
+ 一共有多少事件, 事件是的分布如何, 这些也无法预测  

时间轮就是管理时间事件的一种数据结构(算法), 之所以叫这个名称, 大概是因为时间轮的实现中, 通常会把时间按不同的单位(跨度)划分, 有点类似于钟表(时分秒), 或者咬合滚动的齿轮.

{{ image(src="/image/016_01.png", alt="image 404", position="center") }}

上图是仿着钟表画的时间轮示意图, 如果事件在2触发, 就放到第二个绿色格里; 如果事件在65触发, 就放到第一个黄色格里. 1个黄色格, 等于60个绿色格, 进制是60.

为了后面方便说明, 这里我们暂且把格叫做槽位, 而把相同颜色的一组格叫做槽级, 把格的区间长度叫做槽级的精度, 把槽级的区间长度叫做跨度(如果有更好的名称, 还请麻烦告知我~).

## 2. 时间轮的数据结构

```rust
pub(crate) struct Wheel {
  // 时间轮从启动开始经历的毫秒数
  elapsed: u64,
  // 6个槽级, 每级64个槽位
  levels: Box<[Level; 6]>,
  // 就绪事件链表
  pending: EntryList,
}

pub(crate) struct Level {
  // 当前槽级是什么, 0~5
  level: usize,
  // 位图, 表示槽位里是否有事件
  occupied: u64,
  // 槽位数组,  槽位里是事件链表
  slot: [EntryList; 64],
}
```

`tokio`时间轮的实现在[Wheel](https://github.com/tokio-rs/tokio/blob/tokio-1.38.x/tokio/src/runtime/time/wheel/mod.rs), 结构本身不复杂只有三个字段, `elapsed`表示从时间轮启动开始, 一共经历的毫秒数, 在时间轮推动(poll)时会被更新; `levels`表示不同时间跨度的槽级, 这里一共有6级, 相邻槽级的时间跨度是64倍关系; `pending`是就绪事件的链表, 在时间轮被推动时, 就绪的事件先会放到这个链表中.

`Level`是表示槽级的数据结构, 其中字段`level`表示当前的槽是哪一级(0~5), 可以用来计算这一级的精度(64<sup>level</sup>)和跨度(64<sup>level+1</sup>); `solt`是槽位数组, 每个槽级都有的64个槽位; `slot`是位图, 如果第n位是1, 表示第n个槽位里有事件, 用来优化查找.

{{ image(src="/image/016_02.png", alt="image 404", position="center") }}

第一个槽级的时间精度是1(64<sup>0</sup>)ms, 跨度是0~63, 触发时间在[0, 64)的事件会放到第一个槽级. 触发时间在5的事件, 会放到第6(数组下标5)个槽位里. 第二个槽级的精度是64(64<sup>1</sup>)ms, 跨度是[~~0~~64, 64<sup>2</sup>), 触发时间在100的事件, 会放到第二槽级的第2个槽位.

上面的1毫秒, 64倍, 6个级这些数字, 是目前代码里的"合理设定". 1毫秒这个精度是和事件通知机制(比如Linux下epoll的timeout参数)有关, [mio](https://docs.rs/mio/1.0.1/mio/struct.Poll.html#method.poll)的时间精度通常是1毫秒, 进一步把时间细分下去意义也不大. 

64倍是为了更快的位运算, 因为64 = 2<sup>6</sup>, 下面的方法里会看见相关的代码. 而6个级这个数, 因为64<sup>6</sup>毫秒大约是2年, 达到年这个单位, 也差不多"够意思"了. 这也是槽位数量和"拥挤"程度之间的一个权衡, 当然, 超过64<sup>6</sup>这种事件, 会放在最后一个槽位里.

## 3. 时间轮的对外接口

`Wheel`对外的接口主要有4个, 新增事件, 删除事件, 推动时间轮和查询最近的时间. 下面的代码忽略了"不那么重要"的部分, 比如`unsafe`, `assert`, 一些分支判断等, 只展示主要的逻辑.

### 3.1 新增事件

```rust
pub(crate) fn insert(&mut self, item: TimerHandle) -> Result<u64, (TimerHandle, InsertError)> {
  // when 是事件触发的时刻
  let when = item.sync_when();
  // 计算放到哪个槽级
  let level = self.level_for(when);
  // 放到对应的槽级里
  self.levels[level].add_entry(item);
  Ok(when)
}
```

新增事件的方法是`insert`, 首先根据事件要触发的时刻, 计算要放到哪个槽级, 然后再调用对应槽级的`add_entry`方法放到槽位里.

```rust
// ...  000000 000000 000001 000001   65
//      \____/ \____/ \____/ \____/
//         |      |      |      |
//       第3组   第2组   第1组   第0组
fn level_for(elapsed: u64, when: u64) -> usize {
  const SLOT_MASK: u64 = (1 << 6) - 1;
  let mut masked = elapsed ^ when | SLOT_MASK;
  // 计算最右侧的1在什么位置
  let leading_zeros = masked.leading_zeros() as usize;
  let significant = 63 - leading_zeros;
  // 最右侧的1在第几个组里
  significant / 6 
}
```

计算放到哪个槽级的方法是`level_for`, 这里就能看到64倍这个"设定"大显身手了. 入参`elapsed`是时间轮上的那个毫秒, 入参`when`是事件触发的时间, `elapsed ^ when`是`when - elapsed`的"简化", 两者之差的最右侧1会保留下来.

因为槽级之间是64(2<sup>6</sup>)倍的关系, 所以决定事件放到哪个槽级的, 就是最右侧的1在什么位置. 数下前置0的数量, 反推最右侧1的位置, 然后从左边开始6位一组, 这个1在第n组, 事件就放在第n级. 因为64(2的次方)的关系, 槽级的查找能直接用位运算来完成(方便计算机运算, 却不一定方便人理解-^-).

```rust
pub(crate) fn add_entry(&mut self, item: TimerHandle) {
  // 计算对应的槽位
  let slot = slot_for(item.cached_when(), self.level);
  // 放入事件链表
  self.slot[slot].push_front(item);
  // 位图上置为1
  self.occupied |= occupied_bit(slot);
}

// ...  000000 000000 000001 000001   65
//      \____/ \____/ \____/ \____/
//                       |       
//                      槽位
fn slot_for(duration: u64, level: usize) -> usize {
    ((duration >> (level * 6)) % 64) as usize
}
```

接下来是计算放到哪个槽位的方法`slot_for`, 如果放入的是槽级1(从0开始), 那最后6位就可以忽略; 如果放入的是槽级2, 那最后12位就可以忽略. 这样类推到最后, 其实最终剩下的那组, 也就是最右侧1所在的那组, 就是需要放入的槽位的下标了.

槽位的计算也是得益于2次方的倍数关系, 如上找到需要放入的槽位后, 就把事件放入到这个槽位的事件链表里, 位图上的对应位也置为1, 到此, 新增事件的过程就全部完成了.

### 3.2 删除事件

```rust
pub(crate) fn remove(&mut self, item: NonNull<TimerShared>) {
  let when = item.as_ref().cached_when();
  // 计算到哪个槽级里找
  let level = self.level_for(when);
  // 从对应的槽级里删除
  self.levels[level].remove_entry(item);
}

pub(crate) fn remove_entry(&mut self, item: NonNull<TimerShared>) {
  // 计算对应的槽位
  let slot = slot_for(unsafe { item.as_ref().cached_when() }, self.level);
  // 从事件链表里删除
  unsafe { self.slot[slot].remove(item) };
  if self.slot[slot].is_empty() {
      // 槽位里如果已经没有事件了, 置0  
      self.occupied ^= occupied_bit(slot);
  }
}
```

删除事件的方法是`remove`, 对于给定的事件, 触发时刻是`when`, "寻址"的过程和`insert`方法里是一样的. 找到对应槽位的事件链表后, 再调用链表的`remove`方法删除事件.

### 3.3 推动时间轮

```rust
pub(crate) fn poll(&mut self, now: u64) -> Option<TimerHandle> {
  loop {
    // 先返回就绪事件链表里的事件
    if let Some(handle) = self.pending.pop_back() {
        return Some(handle);
    }
    // 找到最近的槽位
    match self.next_expiration() {
        Some(ref expiration) if expiration.deadline <= now => {
            // 重新处理这个槽位里的事件
            self.process_expiration(expiration);
            // 更新时间轮上的毫秒数
            self.set_elapsed(expiration.deadline);
        }
        // 没有就绪的事件
        _ => {
            self.set_elapsed(now);
            break;
        }
    }
  }
  self.pending.pop_back()
}
```

推动时间轮的方法是`poll`, 如果没有就绪的事件, 就返回`None`, 否则返回一个就绪的事件, 所以如果有多个事件就绪, 需要调用多次`poll`方法来处理.

就绪的事件会先放到`pending`就绪链表里, 所以如果就绪链表里有事件, 就先返回链表里的事件. 如果没有"直接"就绪的事件, 再尝试处理最近(时间上最早)槽位里的事件.

```rust
pub(crate) fn process_expiration(&mut self, expiration: &Expiration) {
  // 拿到槽位里的事件链表
  let mut entries = self.take_entries(expiration);
  while let Some(item) = entries.pop_back() {
    match unsafe { item.mark_pending(expiration.deadline) } {
      // 已到触发时间, 放入就绪事件链表
      Ok(()) => { self.pending.push_front(item); }
      Err(expiration_tick) => {
        // 还没到时间的, 重新计算后放入时间轮
        let level = level_for(expiration.deadline, expiration_tick);
        self.levels[level].add_entry(item);
      }
    }
  }
}
```

`process_expiration`方法遍历槽位里的所有事件, 如果事件到了触发时间, 就放到就绪链表, 否则重新计算后放入时间轮, 再次计算的过程就是事件"降级", 时间轮"转动"的过程.

比如在0时刻, 新增一个在100触发的事件, 此时事件被放入槽级1的第1(下标)个槽位. 当时刻来到72时再调用`poll`, 这个事件就会被下放到槽级0里了. 这时候的第0槽级代表的时刻不再是[0, 63), 而是[64, 128). 第0槽级转动了1圈, 第1槽级转动了1格.

### 3.4 最近就绪时间

```rust
pub(crate) fn poll_at(&self) -> Option<u64> {
  self.next_expiration().map(|expiration| expiration.deadline)
}

fn next_expiration(&self) -> Option<Expiration> {
  // 遍历所有的槽级
  for (level_num, level) in self.levels.iter().enumerate() {
    if let Some(expiration) = level.next_expiration(self.elapsed) {
      return Some(expiration);
    }
  }
  None
}
```

查询最近就绪时间方法是`poll_at`, 这个方法会遍历所有的槽级, 直到找到第一个非空的槽级, 找到里面第一个非空的槽位, 然后算出这个槽位对应的时间.

```rust
pub(crate) fn next_expiration(&self, now: u64) -> Option<Expiration> {
  // 利用位图找到最近(低位)的非空的槽位
  let slot = self.next_occupied_slot(now)?;
  // 槽级的时间跨度, 比如第0级是64
  let level_range = level_range(self.level);
  // 槽位的时间跨度, 比如第0级是1
  let slot_range = slot_range(self.level);
  // 计算上时间轮的转动, 当前槽级的起始时间
  let level_start = now & !(level_range - 1);
  // 槽位的起始时间
  let mut deadline = level_start + slot as u64 * slot_range;
  if deadline <= now {
      deadline += level_range;
  }
  Some(Expiration { level: self.level, slot, deadline, })
}
``` 

`next_expiration`计算最近的非空槽位的起始时间, 比较绕的是`now & !(level_range - 1)`, 这一步里计算了时间轮转动后的时间. 

还是上面推动时间轮里的例子, 72时刻调用的话, 第0槽级的起始时间`72 & !(64 - 1)`是64. 因为此时第0槽级已经转过了一圈, 表示的时间区间已经是[64, 128). 

`poll_at`找到下次推动时间轮的时间, 这个时间对应的是某个非空槽位的起始时间. 在下次调用`poll`推动时间轮时, 这个槽位里的事件就会被处理, 加入就绪链表, 或被"降级". 

## 4. 时间轮和运行时的关系

```rust
// https://github.com/tokio-rs/tokio/blob/tokio-1.38.x/tokio/src                       
// /runtime/scheduler/current_thread/mod.rs#L687 
fn Runtime::block_on() {
  // 和poll_at基本相同
  // /runtime/time/mod.rs#L197
  let timeout = wheel.next_expiration_time();
  // 推动事件循环, 比如epoll
  // /runtime/io/driver.rs#L149
  io.poll(events, timeout);
  // 遍历就绪事件, 逐个处理
  // /runtime/io/driver.rs#L162
  for event in events.iter() {
    ...
  }
}
```

运行时和时间轮怎么协作起来的, 贴源代码的话是在是太多了, 只能发挥下抽象大法了L^L, "神"应该还是八九不离十的. 这边给出的是"单线程"运行时的例子, 对于想要点进源码, 追踪具体实现的同学, 大概"路径"我也贴在注释里了.

主逻辑并不复杂, 查询时间轮里最近的时间, 然后作为超时参数传给事件循环, 事件循环阻塞在poll上, 最后取出就绪事件, 遍历处理, 准备开始下次循环. 这一套大差不差都这样, [redis的eventloop](https://github.com/redis/redis/blob/unstable/src/ae.c#L380)也是如此.

## 小结

`tokio`的代码也是之前看的了, 还向[ihciah](https://github.com/ihciah)同学请教过问题(字节也有运行时的实现[monoio](https://github.com/bytedance/monoio)). 如果有什么地方我理解有误, 或者有哪些可以改进的, 也请大家不吝指出, 感谢!

最近接触"时间"的内容比较多, 想起[pingora](https://github.com/cloudflare/pingora)里的超时有`FastTimeout`和`TokioTimeout`两套, 也是蛮有意思的, 就想着写一下. 一不留神, 发现光是`tokio`的时间轮可写的也不少, 索性放到后面再写一篇了.



</br>

*-> 如果文章有不足之处或者有改进的建议，可以在[这边](https://github.com/dlzht/dlzht.github.io/discussions/12)告诉我，也可以发送给我的[邮箱](mailto:dlzht@protonmail.com)*