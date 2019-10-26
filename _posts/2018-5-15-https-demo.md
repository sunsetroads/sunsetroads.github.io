---
layout: post
title: 使用自签名证书搭建 HTTPS 服务
categories: iOS
description: 使用自签名证书搭建 HTTPS 服务
keywords: HTTPS, iOS, node.js
---

简单来说，HTTPS 就是 HTTP 协议上再加一层加密处理的 SSL/TLS 协议。相比 HTTP，HTTPS 可以保证内容在传输过程中不会被第三方查看、及时发现被第三方篡改的内容、防止身份冒充等，从而更有效的保证网络数据的安全。

### HTTPS 客户端与服务器交互过程：
1. 客户端第一次请求时，服务器会返回一个包含公钥的数字证书给客户端；
2. 客户端生成对称加密密钥并用其得到的公钥对其加密后返回给服务器；
3. 服务器使用自己私钥对收到的加密数据解密，得到对称加密密钥并保存；
4. 然后双方通过对称加密的数据进行传输。

### 证书的内容

- 颁发证书的机构的名字（CA）
- 证书内容本身的数字签名（用 CA 私钥加密）
- 证书持有者的公钥
- 证书签名用到的 hash 算法

### 数字签名
```
明文 --> hash运算 --> 摘要 --> CA 私钥加密 --> 数字签名
```

### 证书校验流程
客户端握手阶段拿到证书后开始下面的校验
1. 证书颁发的机构是伪造的，客户端不认识，直接认为是危险证书
2. 证书颁发的机构是确实存在的，于是根据 CA 名，找到对应内置的CA根证书、CA 的公钥
3. 用 CA 的公钥，对伪造的证书的摘要进行解密，发现解不了，认为是危险证书
4. CA 的公钥，对证书的数字签名进行解密，得到对应的证书摘要 AA
5. 根据证书签名使用的 hash 算法，计算出当前证书的摘要 BB
6. 对比 AA 跟 BB，发现不一致，判定是危险证书

### 自签名证书
向 CA 机构申请证书是要花钱的，也可以借助 openssl 自制证书，但使用自签名证书时，在证书校验第一步就会出问题，客户端不认识自签名证书的。如果想用，那就需要在客户端代码中将该证书配置为信任证书。

### openssl 生成证书

使用 openssl 生成证书
```
# 1.生成私钥
$ openssl genrsa -out server.key 2048

# 2.生成 CSR (Certificate Signing Request)
$ openssl req -subj "/C=CN/ST=GuangDong/L=ShenZhen/O=xlcw/OU=xlcw Software" -new -key server.key -out server.csr

# 3.生成自签名证书
$ openssl x509 -req -days 3650 -in server.csr -signkey server.key -out server.crt
```
### 服务端配置
引入系统的 https 模块即可（其它语言类似），也可以通过 Nginx 配置实现。以 node.js 为例：
```
const express = require('express')
const path = require('path')
const app = express()

//使用nodejs自带的http、https模块
var https = require('https');
var http = require('http');
var fs = require('fs');

//根据项目的路径导入生成的证书文件
var privateKey  = fs.readFileSync(path.join(__dirname, './certificate/private.pem'), 'utf8');
var certificate = fs.readFileSync(path.join(__dirname, './certificate/ca.cer'), 'utf8');
var credentials = {key: privateKey, cert: certificate};

//创建http与HTTPS服务器
var httpServer = http.createServer(app);
var httpsServer = https.createServer(credentials, app);

//分别设置http、https的访问端口号, https默认端口443，这样设置可以让接口同时支持http和https。
//不使用默认端口时可以通过nginx反向代理实现
var PORT = 80;
var SSLPORT = 443;

//创建http服务器
httpServer.listen(PORT, function() {
    console.log('HTTP Server is running on: http://localhost:%s', PORT);
});

//创建https服务器
httpsServer.listen(SSLPORT, function() {
    console.log('HTTPS Server is running on: https://localhost:%s', SSLPORT);
});
  
//根据请求判断是http还是https
app.get('/', function (req, res) {
    if(req.protocol === 'https') {
        res.status(200).send({
            message: 'This is https visit!',
        });
    }
    else {
        res.status(200).send({
            message: 'This is http visit!',
        });
    }
});
```
### 5. 客户端配置
客户端需要将证书放在客户端内，与请求下来的服务端证书比对，防止类似于 Charles 类的软件抓包。
这里iOS 使用 AFNetworking 来发起请求，下面是配置信任自签名证书的过程
```
NSString *cerPath = [[NSBundle mainBundle] pathForResource:@"ca" ofType:@"cer"];
NSData* caCert = [NSData dataWithContentsOfFile:cerPath];
NSSet * certSet = [[NSSet alloc]initWithObjects:certData, nil];

/*
    *AFSecurityPolicy分三种验证模式：
    *AFSSLPinningModeNone:只是验证证书是否在信任列表中
    *AFSSLPinningModeCertificate：该模式会验证证书是否在信任列表中，然后再对比服务端证书和客户端证书是否一致
    *AFSSLPinningModePublicKey：只验证服务端证书与客户端证书的公钥是否一致
    */
AFSecurityPolicy *security = [AFSecurityPolicy policyWithPinningMode:AFSSLPinningModeCertificate withPinnedCertificates:certSet];
security.allowInvalidCertificates = YES;
security.validatesDomainName = NO;

NSURL *url = [NSURL URLWithString:@"https://127.0.0.1"];
AFHTTPSessionManager *manager = [[AFHTTPSessionManager manager] initWithBaseURL:url];
if ([url.scheme isEqualToString: @"https"]) {
    manager.securityPolicy = security;
}

[manager GET:@"/" parameters:nil headers:nil progress:^(NSProgress * _Nonnull downloadProgress) {
    
} success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
    
    NSLog(@"\n*********\n请求成功___%@\n************\n",responseObject);
} failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
    
    NSLog(@"\n*********\n请求失败___%@\n************\n",error.localizedDescription);
}];
```