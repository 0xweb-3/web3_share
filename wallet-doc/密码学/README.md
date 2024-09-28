# 密码学

## 对称加密和非对称加密

* 对称加密：单密钥加解密算法
  * DES：已经不安全
  * 3DES：具备安全性，但效率低
  * ✅AES：钱包中保存密钥用这个
* 非对称加密：使用公私钥对，使用公钥加密，私钥解密，在加解密阶段。在身份认证中，使用私钥签名，公钥验签。
  * RSA
  * GPG
  * ECC
    * ECDSA
      * Secp256k1（BTC、ETH、cosmos、）
      * secp256R1 （EOS）
    * ✅EDDSA
      * ED25519：效率和安全性都比较高（Dot、sui、Aptos、sol）

## 单向散列函数

常见的就是Hash、Md5、sha1、blake2s

## schnorr和BLS

schnorr：短密钥、节省手续费

BLS：聚合签名、短签名、安全性高 , 常用在DA中

## 门限共享秘密算法

## MPC算法



