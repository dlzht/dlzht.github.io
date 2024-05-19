+++
title = "从redis源码讲事件循环"
date = 2024-05-18
[taxonomies]
tags = ["redis", "异步"]
+++

redis是用C实现的, 事件循环部分简洁而优雅, 代码也紧凑而独立, 不仅能比较流畅地阅读, 而且能很方便地引用到项目中, 压测工具[wrk](https://github.com/wg/wrk/blob/master/src/ae.c)就使用了这部分代码. 下面我们就开始介绍redis中事件循环的实现.

<!-- more -->

## 数据结构

#### 1. 时间事件
```c
// 时间事件
typedef struct aeTimeEvent {
  long long id; //递增id
  monotime when; //触发时刻
  aeTimeProc *timeProc; //触发时执行
  aeEventFinalizerProc *finalizerProc; //删除时执行
  struct aeTimeEvent *prev, *next; //链表指针
  ...
} aeTimeEvent;
```

时间事件是到了某个时间点触发. 因为对redis来说时间事件的数量不会很多, 所以这边用链表来管理, `perv`和`next`两个指针前后相连, 每轮事件处理会遍历整个链表. `id`字段类似于数据库里的自增字段, 当事件不再需要了, 就设置成`AE_DELETED_EVENT_ID`(-1), 下次遍历时会从链表里删除.

`when`是在什么时间触发, `timeProc`是触发时执行什么操作. 遍历链表处理事件时, 就比较`when`和当前时间`now`, 如果触发时间早于当前时间, 就执行`timeProc`函数. `aeEventFinalizerProc`是删除后置操作, 把事件从链表里删除后触发.

```c
// 周期任务serverCron
aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL);
// 即时任务evictionTimeProc
aeCreateTimeEvent(server.el, 0, evictionTimeProc, NULL, NULL);
```

redis里的时间事件主要有:

1. 周期执行的`serverCron`, 1毫秒执行一次, 每次执行完会设置下次执行的时间. 很大后台任务都会在这里面执行, 配合上`hz`参数可以调整任务的执行频率.
2. 即时执行的`evictionTimeProc`, 当内存使用量达到`maxmemory`时, 会创建这个时间事件用来释放内存, 触发时间设置成当前时间, 意味着需要立即执行.

#### 2. 文件事件 
```c
// 文件事件
typedef struct aeFileEvent {
  int mask; //读写标记         
  aeFileProc *rfileProc; //读事件操作
  aeFileProc *wfileProc; //写事件操作
  ...
} aeFileEvent;
```

文件事件是某个文件可读或者可写时触发. 和操作系统用数组维护进程的打开文件一样, redis也用数组维护了打开的文件(网络套接字, 管道文件等), fd也是aeEventLoop里的events数组的下标.

*fd文件操作符是什么? 对于每个进程, Linux系统会维护一个数组, 保存进程打开的每个文件(文件, 套接字, 管道等待)的信息. 配置服务器时经常要用ulimit调整最大连接数, 调整的其实就是这个数组的长度. 而fd就是这个数组的下标, 程序把fd给到系统, 系统找到对应的文件信息项, 就知道程序要操作的是哪个文件.*

就像上面说的, fd和数组的下标是相同的, 所以用来表示哪个文件的fd没有再出现在结构体里. `mask`字段是读写标记, 表示关注的是可读还是可写事件. `rfileProc`和`wfileProc`分别是可读和可写事件发生时要执行的操作.

```c
// 服务监听
aeCreateFileEvent(server.el, sfd->fd[j], AE_READABLE, accept_handler,NULL);
// 连接事件
aeCreateFileEvent(server.el, conn->fd, AE_WRITABLE, conn->type->ae_handler, conn);
...
```

redis里的文件事件主要有:

1. 服务监听端口(比如6379)产生的网络套接字, 每当有客户端来连接时, 这个文件就会变成可读
2. 接受客户端连接后会产生另一个套接字, 有读取请求也有返回结果, 会陆续注册可读和可写事件
3. aof和rdb文件的读写, 哨兵之间的通信

#### 3. 就绪事件
```c
// 就绪事件
typedef struct aeFiredEvent {
  int fd; //文件描述符
  int mask; //读写标记
  ...
} aeFiredEvent;
```

就绪事件不是另外一种事件类型, 只是用来记录哪个文件事件就绪了. `fd`就是系统返回的文件描述符, 也是文件事件在events数组里的下标, `mask`用来标记就绪的事件是可读还是可写. 就绪事件也是用数组来保存的, 数组的大小和维护文件事件的数组一样(最多是所有文件都就绪).

#### 4. 事件循环

{{ image(src="/image/009_01.png", alt="image 404", position="center") }}

`aeEventLoop`是事件循环的主角, `timeEventNextId`保存当前时间事件的id, 每次创建一个时间事件时会自增. `aeFileEvent`数组存储文件事件, `aeFiredEvent`存储就绪的事件, `timeEventHead`链表存储时间事件, `beforesleep`和`aftersleep`则是在每次poll的前置和后置操作.

## 代码实现

事件循环相关的代码([6.2.14](https://github.com/redis/redis/tree/6.2.14/))主要集中在[ae.h](https://github.com/redis/redis/blob/6.2.14/src/ae.h), [ae.c](https://github.com/redis/redis/blob/6.2.14/src/ae.c)和[ae_epoll.c](https://github.com/redis/redis/blob/6.2.14/src/ae_epoll.c). ae.h里是数据结构和函数方法的定义, ae.c里是相关函数的具体实现, 而ae_epoll.cli里则是对[epoll](https://man7.org/linux/man-pages/man7/epoll.7.html)的封装. 


#### 1. 创建和启动事件循环

```c
int main(int argc, char **argv) { 
  initServer();
  // 启动事件循环
  aeMain(server.el);
}

void initServer(void) { 
  // 创建事件循环
  server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);
}

aeEventLoop *aeCreateEventLoop(int setsize) {
  if ((eventLoop = zmalloc(sizeof(*eventLoop))) == NULL) goto err;
  // 申请文件事件数组的内存
  eventLoop->events = zmalloc(sizeof(aeFileEvent)*setsize);
  // 申请就绪事件数组的内存
  eventLoop->fired = zmalloc(sizeof(aeFiredEvent)*setsize);
  eventLoop->timeEventNextId = 0;
  for (i = 0; i < setsize; i++)
    // 清理事件的读写就绪标记
    eventLoop->events[i].mask = AE_NONE;
  return eventLoop;
}
```

先来看[server.c](https://github.com/redis/redis/blob/6.2.14/src/server.c#L3216)里创建`aeEventLoop`的代码. `main`函数会在调用`initServer`时创建事件循环`eventLoop`, 然后调用`aeMain`启动事件循环.

```c
void aeMain(aeEventLoop *eventLoop) {
  eventLoop->stop = 0;
  while (!eventLoop->stop) {
    aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_BEFORE_SLEEP|AE_CALL_AFTER_SLEEP);
  }
}
```

`aeMain`执行事件循环的代码很简单, 只要服务不暂停, 就一直执行`aeProcessEvents`这个函数, 而`aeProcessEvents`里会处理所有的文件事件和时间事件.

#### 2. 处理文件事件和时间事件

```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags) {
  if (eventLoop->maxfd != -1 || ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
    // 遍历时间事件链表, 找到最早事件的触发时间
    if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
      usUntilTimer = usUntilEarliestTimer(eventLoop);
    if (usUntilTimer >= 0) {
      tv.tv_sec = usUntilTimer / 1000000;
      tv.tv_usec = usUntilTimer % 1000000;
      tvp = &tv;
    }
    // 执行poll前置操作
    if (eventLoop->beforesleep != NULL && flags & AE_CALL_BEFORE_SLEEP)
      eventLoop->beforesleep(eventLoop);
    // epoll调用的封装
    numevents = aeApiPoll(eventLoop, tvp);
    // 执行poll后置操作
    if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
      eventLoop->aftersleep(eventLoop);
    // 遍历处理就绪的事件
    for (j = 0; j < numevents; j++) {
      int fd = eventLoop->fired[j].fd;
      aeFileEvent *fe = &eventLoop->events[fd];
      int mask = eventLoop->fired[j].mask;
      // 执行读就绪操作
      if (!invert && fe->mask & mask & AE_READABLE) {
        fe->rfileProc(eventLoop,fd,fe->clientData,mask);
      }
      // 执行写就绪操作
      if (fe->mask & mask & AE_WRITABLE) {
        fe->wfileProc(eventLoop,fd,fe->clientData,mask);
      }
      processed++;
    }
  }
  // 处理时间事件
  if (flags & AE_TIME_EVENTS)
    processed += processTimeEvents(eventLoop);
  return processed;
}
```

`aeProcessEvents`会先遍历时间事件链表, 找到最早时间事件的触发时间, 然后先执行poll前置操作`beforesleep`. 

接着是调用epoll(Linux下)的封装`aeApiPoll`, 第二个入参`tvp`是超时时间, 传入的值是上面的最早的时间事件的触发时间. 调用`aeApiPoll`时线程会阻塞, 直到有文件事件就绪或者超时时间到达, 这边利用了"超时时间"这个参数, 实现了时间和文件事件都能唤醒事件循环.

`aeApiPoll`返回后就执行poll后置操作`aftersleep`, 然后遍历就绪事件, 根据是读事件就绪还是写事件就绪, 调用`rfileProc`或者`wfileProc`. 处理完了文件事件后, 再调用`processTimeEvents`处理时间事件.

```c
static int processTimeEvents(aeEventLoop *eventLoop) {
  te = eventLoop->timeEventHead;
  // 遍历时间事件链表
  while(te) {
    // 删除id为-1的事件
    if (te->id == AE_DELETED_EVENT_ID) {
      zfree(te); continue;
    }
    // 判断触发时间
    if (te->when <= now) {
      // 事件就绪执行操作
      retval = te->timeProc(eventLoop, id, te->clientData);
      // 返回非0代表下次执行的间隔时间
      if (retval != AE_NOMORE) {
        te->when = now + retval * 1000;
      } else {
        // 设置id为-1, 下次会清除事件
        te->id = AE_DELETED_EVENT_ID;
      }
    }
    te = te->next;
  }
  return processed;
}
```

`processTimeEvents`处理时间事件, 遍历时间事件链表, 如果id被标记为删除(-1)了, 就从链表里删除这个事件. 接着比较触发时间和当前时间, 如果满足触发条件, 就调用事件的处理函数`timeProc`. 处理函数的返回值如果是0, 表示这个事件不需要了, 就会把id置为-1下次删除; 如果不是0, 表示这个事件需要保留, 返回的值就是下次触发的间隔时间. 周期任务`serverCron`的返回值就是`1000/server.hz`.

#### 3. epoll调用的封装

```c
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
  int retval, numevents = 0;
  // 调用系统调用epoll_wait
  retval = epoll_wait(state->epfd, state->events, eventLoop->setsize, tvp ? (tvp->tv_sec*1000 + (tvp->tv_usec + 999)/1000) : -1);
  if (retval > 0) {
    // 遍历系统返回的就绪事件
    for (j = 0; j < numevents; j++) {
      struct epoll_event *e = state->events+j;
      if (e->events & EPOLLIN) mask |= AE_READABLE;
      if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
      if (e->events & EPOLLERR) mask |= AE_WRITABLE|AE_READABLE;
      if (e->events & EPOLLHUP) mask |= AE_WRITABLE|AE_READABLE;
      // 设置就绪事件fd
      eventLoop->fired[j].fd = e->data.fd;
      // 设置就绪事件标记
      eventLoop->fired[j].mask = mask;
  }
  return numevents;
}
```
`aeApiPoll`是对系统提供的多路复用IO的封装, 不同的操作系统下有不同的实现, 在Linux下主要是epoll. `aeApiPoll`的作用就是询问操作系统哪些事件就绪了, 然后把这些就绪事件写入到就绪事件数组`fired`中,接着`aeProcessEvents`就可以处理就绪的事件.

#### 4. 创建文件事件和事件事件

```c
long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds, aeTimeProc *proc, void *clientData, aeEventFinalizerProc *finalizerProc) {
  // 事件id自增
  long long id = eventLoop->timeEventNextId++;
  te = zmalloc(sizeof(*te));
  te->id = id;
  // 计算触发时间
  te->when = getMonotonicUs() + milliseconds * 1000;
  te->timeProc = proc;
  // 调整链表指针
  te->next = eventLoop->timeEventHead;
  te->refcount = 0;
  if (te->next)
    te->next->prev = te;
  eventLoop->timeEventHead = te;
  return id;
}

int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask, aeFileProc *proc, void *clientData) {
  // 超过最大文件数
  if (fd >= eventLoop->setsize) 
    return AE_ERR;
  aeFileEvent *fe = &eventLoop->events[fd];
  // 注册到事件循环中
  if (aeApiAddEvent(eventLoop, fd, mask) == -1)
    return AE_ERR;
  // 设置就绪事件标记
  fe->mask |= mask;
  if (mask & AE_READABLE) fe->rfileProc = proc;
  if (mask & AE_WRITABLE) fe->wfileProc = proc;
  return AE_OK;
}
```

创建时间事件的`aeCreateTimeEvent`和文件事件的`aeCreateFileEvent`, 代码逻辑清晰易懂, 会创建事件的情况在上面也提到了, 比如客户端连接, 处理请求, 返回结果, 后台定时任务等等.

## 总结下redis的事件循环

1. 主函数初始化服务时创建事件循环然后启动, redis服务的核心就是事件循环的执行
1. 线程阻塞在poll调用上, 等待有文件事件就绪, 或者最早的时间事件就绪(超时参数)
2. 首先处理就绪的文件事件, 这过程中也可能会产生新的事件, 新事件也注册到poll中
3. 然后遍历时间事件链表, 清理掉不需要的事件, 再判断触发时间, 执行对应的操作
4. 判断是否需要停止事件循环, 否则再次回到第2步, 继续阻塞在poll上等待事件就绪

</br>

*-> 如果文章有不足之处或者有改进的建议，可以在[这边](https://github.com/dlzht/dlzht.github.io/discussions/10)告诉我，也可以发送给我的[邮箱](mailto:dlzht@protonmail.com)*