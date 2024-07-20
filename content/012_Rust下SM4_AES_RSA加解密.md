+++
title = "Rust下SM4/AES/RSA加解密"
date = 2024-07-20
[taxonomies]
tags = ["Rust", "RSA",  "SM4"]
+++

`aes`和`rsa`加密使用非常广泛, 而`sm4`在信创系统中比较常见, 以前用`Java`开发的时候, 有强大的[bc](https://github.com/bcgit/bc-java)库, 还有易用的[hutool](https://github.com/dromara/hutool), 现在转向了`Rust`, 也是找到了[RustCrypto](https://github.com/RustCrypto)这个项目.

`RustCrypto`有点类似于`bc`, 包含的东西非常多, 编码, 哈希, 签名, 加解密, 椭圆曲线等等. 常用的套件差不多准备全了, 这就来放手试试. 这个项目的文档还不那么完善, 而且代码库分得比较细, 我在使用中也是绕了一些弯路, 这篇文章呢就记录一下.

<!-- more -->

## 分块加密, 工作模式, 填充模式

**  分块加密**: 属于对称密钥算法, 加解密用的是同一个密钥. 分块的意思是, 在加解密过程中, 会把数据切分成等长的块来处理. RustCrypto的实现在[block-ciphers](https://github.com/RustCrypto/block-ciphers), `sm4`和`rsa`都属于分块加密算法    .

**工作模式**: 把初始向量, 密钥, 原文和密文这些分好块后, 要按什么顺序计算, 最终才得到加密的密文(或解密的原文). 工作模式和分块加密是组合关系, 一种加密算法能选用多种工作模式, 一种工作模式也能适用多种加密算法. RustCrypto的实现在[block-modes](https://github.com/RustCrypto/block-modes).

**填充模式**: 定义了怎么补全不完整的块, 上面提到分组加密需要把数据切分块处理, 比如16字节是一块, 而我们提供的数据并不总会是16的倍数, 这样最后一个块就可能会有"空白"部分, 而填充模式就决定了空白是什么内容. RustCrypto的填充模式实现在[block-padding](https://github.com/RustCrypto/utils/blob/master/block-padding/src/lib.rs).


## SM4加密/解密

```rust
[dependencies]
sm4 = "0.5"
cbc = { version = "0.1", features = ["alloc"] }

use sm4::cipher::block_padding::Pkcs7;
use sm4::cipher::{BlockEncryptMut, KeyIvInit};
```

上面是cbc+sm4加解密过程中要用到的依赖, 有些依赖项是通过`sm4`重新导出来的, 比如填充模式`Pkcs7`. 需要使用什么加密算法, 最好用这个库里导出的, 不然在编译时可能会出现晦涩的错误(一大串的泛型~).

```rust
//  CBC块模式加密
type Sm4CbcEnc = cbc::Encryptor<sm4::Sm4>;

fn sm4_encrypt_cbc_pkcs7(key: &[u8], iv: &[u8], data: &[u8]) -> Vec<u8> {
  let sm4 = Sm4CbcEnc::new(key.into(), iv.into());
  return sm4.encrypt_padded_vec_mut::<Pkcs7>(data);
}
```

这里给`cbc::Encryptor<sm4::Sm4>`起了个别名`Sm4CbcEnc`, 方法第一步是构造加密器, 参数是密钥`key`和初始向量`iv`. 虽然这两个参数的类型是切片, 但对长度是有一定要求的, `key`和`iv`得是16字节, 不然会`panic`. 

`data`参数是要加密的原数据, 如果采用的填充模式是`NoPadding`(不填充), 那数据的长度就得是16的倍数, 其他填充模式则不需要, 这里是用`pkcs7`模式来填充.

方法的第二步加密原文, 并返回密文字节. `encrypt_padded_vec_mut`方法会自动分配内存来放结果, 这需要开启`alloc`特性. 如果没有开启这个特性的话, 需要自己申请内存, 这样可以在没有`std`的环境中使用.


```rust
//  CBC块模式解密
type Sm4CbcDec = cbc::Decryptor<sm4::Sm4>;

fn sm4_decrypt_cbc_pkcs7(key: &[u8], iv: &[u8], data: &[u8]) -> Vec<u8> {
  let sm4 = crate::Sm4CbcDec::new(key.into(), iv.into());
  return sm4.encrypt_padded_vec_mut::<Pkcs7>(data);
}

```

解密代码和加密区别不大, 一个是改成了解密器`cbc::Decryptor<sm4::Sm4>`, 还一个是改成解密方法`encrypt_padded_vec_mut`. `key`, `iv`, 还有填充模式和数据长度, 和加密过程的要求一样.

## AES加密/解密

```rust
[dependencies]
aes = "0.8"
ofb = "0.6"

use aes::cipher::{KeyIvInit, StreamCipher};
```

某些工作模式可以让分块密码像流密码一样工作, 也就是不需要非得是完整的块, 也就不需要填充字节. `ctr`和`ofb`就是这样的模式, 接下来的`aes`加密我就搭配`ofb`的工作模式, 上面是我们需要用到的依赖. 

```rust
type Aes128OfbEnc = ofb::Ofb<aes::Aes128>;

fn aes_encrypt_ofb(key: &[u8], iv: &[u8], data: &mut [u8]) {
    let mut aes = Aes128OfbEnc::new(key.into(), iv.into());
    aes.apply_keystream(data);
}
```

`aes`的密钥长度可以是16, 24或32字节, 密钥的长度不同, 需要用到的类型也不同. 上面代码是16字节长度密钥的实例, 选用的类型是`Aes128`, 128b = 16B. 其他长度的密钥, 可以用`Aes192`, `Aes256`


```rust
type Aes128OfbDec = ofb::Ofb<aes::Aes128>;

fn aes_decrypt_ofb(key: &[u8], iv: &[u8], data: &mut [u8]) {
    let mut aes = Aes128OfbEnc::new(key.into(), iv.into());
    aes.apply_keystream(data);
}
```

加解密的代码是一样的, 别名类型是为了看上去有所区别. `apply_keystream`方法可以把密文数据存在原数据的位置, 所以这边的`data`传递的是可变引用, 这样在不再需要原文数据的情况下, 就省去了分配额外的内存的开销.

## RSA加密/解密

```rust
// [dependencies] 
// rsa = "0.9"
pub fn generate_keypair(length: usize) -> Result<(RsaPublicKey, RsaPrivateKey)> {
    let mut rng = rand::thread_rng();
    let pri_key = RsaPrivateKey::new(&mut rng, length)?;
    let pub_key = pri_key.to_public_key();
    return Ok((pub_key, pri_key));
}
```

RustCrypto的实现在仓库[RSA](https://github.com/RustCrypto/RSA), 这是一种非对称密钥算法, 有公钥`pub_key`和私钥`pri_key`两个密钥. 公钥用来加密, (公钥Φ原文)=密文; 私钥则用来解密, (私钥Φ密文)=原文. 上面是生成密钥的代码, 参数`length`指密钥长度为2<sup>length</sup>.

```rust
pub fn encrypt_byte(pub_der: &[u8], data: &[u8]) -> Result<Vec<u8>> {
    let mut rng = rand::thread_rng();
    // 从字节中解析公钥
    let pub_key = RsaPublicKey::from_public_key_der(pub_der)?;
    // 公钥加密
    return Ok(pub_key.encrypt(&mut rng, Pkcs1v15Encrypt, data)?);
}

pub fn decrypt_byte(pri_der: &[u8], data: &[u8]) -> Result<Vec<u8>> {
    // 从字节中解析私钥
    let pri_key = RsaPrivateKey::from_pkcs8_der(pri_der)?;
    // 私钥加密
    return Ok(pri_key.decrypt(Pkcs1v15Encrypt, data)?);
}
```
公私钥可以用`pem`(标头 + Base64数据), `der`(二进制数据)等方式编码, 私钥还有`pkcs1`和`pkcs8`格式的区别, 库里都提供了对应的解析方法. 上面实例代码中是解析 的`der`格式. `rsa`的加解密用起来说更简单一些, 因为没有了工作模式, 不过填充模式还是有, 这边用的  是`Pkcs1v15Encrypt`.
 

</br>

*-> 如果文章有不足之处或者有改进的建议，可以在[这边](https://github.com/dlzht/dlzht.github.io/discussions/12)告诉我，也可以发送给我的[邮箱](mailto:dlzht@protonmail.com)*