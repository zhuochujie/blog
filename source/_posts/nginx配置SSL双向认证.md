---
title: nginx配置SSL双向认证
date: 2024-02-15 15:54:43
tags: [nginx,ssl]
---

## 背景
有一些网站我们需要限制用户访问,比如说管理后台,企业内部系统......仅仅使用管理员账号登陆是不安全的(作者本人吃过大亏),后面改为了防火墙限制IP访问,非常安全,但是当时没有固定IP,IP刷新后需要重新设置IP白名单,后面我便想有没有一种方式能像Web3钱包一样的方式,通过签名和验签的方式进行授权访问.当然,我能想到的肯定早就有人想到过了(Too young, too simple!).那就是通过SSL进行客户端认证.
## 流程
先生成一个CA证书,后面通过CA证书去给客户端生成证书
### 自签CA证书
```
openssl genrsa -out ca.key 2048
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt
```

### 生成客户端证书
```
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr
openssl x509 -req -sha256 -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 3650 -out client.crt
openssl pkcs12 -export -legacy -in client.crt -inkey client.key -out client.p12
```
生成p12证书切记要带上 -legacy 参数,由于我电脑上的openssl版本是3.x,与设备需要的证书不兼容,会提示密码不正确(即使输入了正确的密码).
网友给出的解决方案是将openssl降级到1.x再生成p12证书,或者使用 -legacy 参数.
### 配置nginx
在server区域内配置
ssl_client_certificate ./ca.crt;
ssl_verify_client on;
### 测试
直接打开该网站,提示 No required SSL certificate was sent
安装好client.p12证书,关闭并重新打开浏览器,进入该网站提示选择证书进行验证,成功访问!