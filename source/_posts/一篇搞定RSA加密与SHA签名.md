---
title: 一篇搞定RSA加密与SHA签名
date: 2016-01-12 17:19:25
categories: iOS
tags: [RSA加密, SHA签名]
---
看到这篇文章的同学可幸福了，当时在做RSA加密与签名的时候网上的资料简直不要太老，做完后实在是忍受不下去了，这篇文章我会详细讲解iOS如何实现RSA加密与签名，并且与Java完全同步。

<!--more-->

# 基础知识 

 <p>

1. **什么是RSA？**<p>
答：RSA是一种非对称加密算法，常用来对传输数据进行加密，配合上数字摘要算法，也可以进行文字签名。

2. **RSA加密中padding？**<p>
答：padding即填充方式，由于RSA加密算法中要加密的明文是要比模数小的，padding就是通过一些填充方式来限制明文的长度。后面会详细介绍padding的几种模式以及分段加密。

3. **加密和加签有什么区别？**<p>
答：加密：公钥放在客户端，并使用公钥对数据进行加密，服务端拿到数据后用私钥进行解密； 
      加签：私钥放在客户端，并使用私钥对数据进行加签，服务端拿到数据后用公钥进行验签。       
前者完全为了加密；后者主要是为了防恶意攻击，防止别人模拟我们的客户端对我们的服务器进行攻击，导致服务器瘫痪。

# 基本原理<p>

RSA使用“密钥对”对数据进行加密解密，在加密解密前需要先生存公钥（Public Key）和私钥（Private Key）。<p>
**公钥(Public key):** 用于加密数据. 用于公开, 一般存放在数据提供方, 例如iOS客户端。<p>
**私钥(Private key):** 用于解密数据. 必须保密, 私钥泄露会造成安全问题。<p>
iOS中的Security.framework提供了对RSA算法的支持，这种方式需要对密匙对进行处理, 根据public key生成证书, 通过private key生成p12格式的密匙。想想jave直接用字符串进行加密解密简单多了。(⊙o⊙)…

# 实战
<p>
## 证书生成
RSA加密这块公钥、私钥必不可少的。**Apple是不支持直接使用字符串进行加密解密的，推荐使用p12文件**。这边教大家去生成在加密中使用到的所有文件，并提供给Java使用，想当年这个公钥私钥搞了半天了。 %>_<%

>* 生成模长为1024bit的私钥
openssl genrsa -out private_key.pem 1024
* 生成certification require file
openssl req -new -key private_key.pem -out rsaCertReq.csr 
* 生成certification 并指定过期时间
openssl x509 -req -days3650-in rsaCertReq.csr -signkey private_key.pem -out rsaCert.crt
* 生成公钥供iOS使用
openssl x509 -outform der -in rsaCert.crt -out public_key.der
* 生成私钥供iOS使用 这边会让你输入密码，后期用到在生成secKeyRef的时候会用到这个密码
openssl pkcs12 -export -out private_key.p12 -inkey private_key.pem -in rsaCert.crt
* 生成pem结尾的公钥供Java使用
openssl rsa -in private_key.pem -out rsa_public_key.pem -pubout
* 生成pem结尾的私钥供Java使用openssl pkcs8 -topk8 -in private_key.pem -out pkcs8_private_key.pem -nocrypt

**以上所有的步骤都是在终端下完成的哦  (*^__^*)**

## 生成公钥和私钥的secKeyRef<p>
   ```
   //根据你的p12文件生成私钥对应的SecKeyRef 这边返回若是nil 请检查你p12文件的生成步骤
- (SecKeyRef)getPrivateKeyRefrenceFromData:(NSData*)p12Data password:(NSString*)password {

SecKeyRef privateKeyRef = NULL;
NSMutableDictionary * options = [[NSMutableDictionary alloc] init];
[options setObject: password forKey:(__bridge id)kSecImportExportPassphrase];
CFArrayRef items = CFArrayCreate(NULL, 0, 0, NULL);
OSStatus securityError = SecPKCS12Import((__bridge CFDataRef) p12Data, (__bridge CFDictionaryRef)options, &items);
if (securityError == noErr && CFArrayGetCount(items) > 0) {
    CFDictionaryRef identityDict = CFArrayGetValueAtIndex(items, 0);
    SecIdentityRef identityApp = (SecIdentityRef)CFDictionaryGetValue(identityDict, kSecImportItemIdentity);
    securityError = SecIdentityCopyPrivateKey(identityApp, &privateKeyRef);
    if (securityError != noErr) {
        privateKeyRef = NULL;
    }
}
CFRelease(items);

return privateKeyRef;
}  
   ```
   
   
   ```
    //根据你的der文件公钥对应的SecKeyRef
 - (SecKeyRef)getPublicKeyRefrenceFromeData:    (NSData*)derData {

SecCertificateRef myCertificate = SecCertificateCreateWithData(kCFAllocatorDefault, (__bridge CFDataRef)derData);
SecPolicyRef myPolicy = SecPolicyCreateBasicX509();
SecTrustRef myTrust;
OSStatus status = SecTrustCreateWithCertificates(myCertificate,myPolicy,&myTrust);
SecTrustResultType trustResult;
if (status == noErr) {
    status = SecTrustEvaluate(myTrust, &trustResult);
}
SecKeyRef securityKey = SecTrustCopyPublicKey(myTrust);
CFRelease(myCertificate);
CFRelease(myPolicy);
CFRelease(myTrust);

return securityKey;
}
   ```
   
## 加密与解密 <p>
 
 ```
 - (NSData*)rsaEncryptData:(NSData*)data {
    SecKeyRef key = [self getPublicKey];
    size_t cipherBufferSize = SecKeyGetBlockSize(key);
    uint8_t *cipherBuffer = malloc(cipherBufferSize * sizeof(uint8_t));
    size_t blockSize = cipherBufferSize - 11;
      size_t blockCount = (size_t)ceil([data length] / (double)blockSize);
      NSMutableData *encryptedData = [[NSMutableData alloc] init];
    for (int i=0; i<blockCount; i++) {
    unsigned long bufferSize = MIN(blockSize , [data length] - i * blockSize);
    NSData *buffer = [data subdataWithRange:NSMakeRange(i * blockSize, bufferSize)];
    OSStatus status = SecKeyEncrypt(key, kSecPaddingPKCS1, (const uint8_t *)[buffer bytes], [buffer length], cipherBuffer, &cipherBufferSize);

    if (status != noErr) {
        return nil;
    }

    NSData *encryptedBytes = [[NSData alloc] initWithBytes:(const void *)cipherBuffer length:cipherBufferSize];
    [encryptedData appendData:encryptedBytes];
    }

  if (cipherBuffer){
    free(cipherBuffer);
  }

  return encryptedData;
  }
 ```
 
 ```
 - (NSData*)rsaDecryptData:(NSData*)data {
SecKeyRef key = [self getPrivatKey];

size_t cipherBufferSize = SecKeyGetBlockSize(key);
size_t blockSize = cipherBufferSize;
size_t blockCount = (size_t)ceil([data length] / (double)blockSize);

NSMutableData *decryptedData = [[NSMutableData alloc] init];

for (int i = 0; i < blockCount; i++) {
    unsigned long bufferSize = MIN(blockSize , [data length] - i * blockSize);
    NSData *buffer = [data subdataWithRange:NSMakeRange(i * blockSize, bufferSize)];

    size_t cipherLen = [buffer length];
    void *cipher = malloc(cipherLen);
    [buffer getBytes:cipher length:cipherLen];
    size_t plainLen = SecKeyGetBlockSize(key);
    void *plain = malloc(plainLen);

    OSStatus status = SecKeyDecrypt(key, kSecPaddingPKCS1, cipher, cipherLen, plain, &plainLen);

    if (status != noErr) {
        return nil;
    }

    NSData *decryptedBytes = [[NSData alloc] initWithBytes:(const void *)plain length:plainLen];
    [decryptedData appendData:decryptedBytes];
}

return decryptedData;
}
 ```
 
### RSA加密中的Padding 

* RSA_PKCS1_PADDING 填充模式，最常用的模式<p>
要求: 输入：必须 比 RSA 钥模长(modulus) 短至少11个字节, 也就是　RSA_size(rsa) – 11 如果输入的明文过长，必须切割，然后填充。<p>
输出：和modulus一样长<p>
根据这个要求，对于1024bit的密钥，block length = 1024/8 – 11 = 117 字节<p>

* RSA_PKCS1_OAEP_PADDING<p>
输入：RSA_size(rsa) – 41<p>
输出：和modulus一样长<p>

* RSA_NO_PADDING　　不填充<p>
输入：可以和RSA钥模长一样长，如果输入的明文过长，必须切割，　然后填充<p>
输出：和modulus一样长<p>

## 签名与验证

```
 //对数据进行sha256签名
- (NSData *)rsaSHA256SignData:(NSData *)plainData {
      SecKeyRef key = [self getPrivatKey];
    
      size_t signedHashBytesSize = SecKeyGetBlockSize(key);
      uint8_t* signedHashBytes = malloc(signedHashBytesSize);
      memset(signedHashBytes, 0x0, signedHashBytesSize);
    
      size_t hashBytesSize = CC_SHA256_DIGEST_LENGTH;
      uint8_t* hashBytes = malloc(hashBytesSize);
      if (!CC_SHA256([plainData bytes], (CC_LONG)[plainData length], hashBytes)) {
        return nil;
    }
    
           SecKeyRawSign(key,
                  kSecPaddingPKCS1SHA256,
                  hashBytes,
                  hashBytesSize,
                  signedHashBytes,
                  &signedHashBytesSize);
    
        NSData* signedHash = [NSData dataWithBytes:signedHashBytes
                                        length:(NSUInteger)signedHashBytesSize];
    
        if (hashBytes)
        free(hashBytes);
    if (signedHashBytes)
        free(signedHashBytes);
    
        return signedHash;
}
        
```


```

//这边对签名的数据进行验证 验签成功，则返回YES
- (BOOL)rsaSHA256VerifyData:(NSData *)plainData     withSignature:(NSData *)signature {
        SecKeyRef key = [self getPublicKey];
  
        size_t signedHashBytesSize = SecKeyGetBlockSize(key);
        const void* signedHashBytes = [signature bytes];
    
        size_t hashBytesSize = CC_SHA256_DIGEST_LENGTH;
        uint8_t* hashBytes = malloc(hashBytesSize);
        if (!CC_SHA256([plainData bytes], (CC_LONG)[plainData length], hashBytes)) {
           return NO;
        }
    
          OSStatus status = SecKeyRawVerify(key,
                                      kSecPaddingPKCS1SHA256,
                                      hashBytes,
                                      hashBytesSize,
                                      signedHashBytes,
                                      signedHashBytesSize);
    
        return status == errSecSuccess;
}
```
<p>
**文章到此就结束了，希望大家能够喜欢。请点击[Git](https://github.com/PanXianyue/XYCryption)获取相关demo**