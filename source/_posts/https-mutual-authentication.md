---
title: HTTPS 双向认证
date: 2020-08-15 14:04:21
tags:
  - tls
  - ssl
  - https
  - mutual authentication
---

## TLS 协议

传输层安全性协议 `(TLS: Transport Layer Security)` 及其前身安全套接层 `(SSL: Secure Sockets Layer)` 是一种安全协议，目的是为互联网通信提供安全及数据完整性保障。

`SSL` 包含记录层 `(Record Layer)` 和传输层，记录层协议确定传输层数据的封装格式。传输层安全协议使用 `X.509` 认证，然后利用非对称加密演算来对通信方做身份认证，之后交换对称密钥作为会谈密钥 `(Session key)`。这个会谈密钥是用来将通信两方交换的数据做加密，保证两个应用间通信的保密性和可靠性，使客户与服务器应用之间的通信不被攻击者窃听。

## HTTPS 协议

超文本传输安全协议 `(HTTPS: HyperText Transfer Protocol Secure，常称为HTTP over TLS、HTTP over SSL或HTTP Secure)` 是一种通过计算机网络进行安全通信的传输协议。`HTTPS` 经由 `HTTP` 进行通信，但利用 `SSL/TLS` 来加密数据包。`HTTPS` 开发的主要目的，是提供对网站服务器的身份认证，保护交换数据的隐私与完整性。

## 认证过程

### 单向认证

在访问大多数网站 `(google, facebook)` 时，会使用单向认证的方式。客户端(浏览器)会验证服务端证书的合法性，过程如下：

![01-one-way-authentication](/images/https-mutual-authentication/01-one-way-authentication.png)

1. 客户端发起建立 `HTTPS` 连接请求，将 `SSL` 协议版本的信息发送给服务器端
1. 服务器端将本机的公钥证书 `(server.crt)` 发送给客户端
1. 客户端读取公钥证书 `(server.crt)` 取出服务端公钥
1. 客户端生成随机密钥 `R`，用服务器公钥加密密钥`R`，将密文发送给服务端
1. 服务端用私钥 `(server.key)` 解密密文，得到了密钥 `R`
1. 双方使用随机密钥 `R` 通信的对称加密密钥

### 双向认证

而在某些有较高安全性要求，或需要验证访问者身份的场景，则可能会需要用到双向认证的方式：

![02-mutual-anthentication](/images/https-mutual-authentication/02-mutual-anthentication.png)

1. 客户端发起建立 `HTTPS` 连接请求，将 `SSL` 协议版本的信息发送给服务端；
1. 服务器端将本机的公钥证书 `(server.crt)` 发送给客户端
1. 客户端读取公钥证书 `(server.crt)` 取出服务端公钥
1. 客户端将客户端公钥证书 `(client.crt)` 发送给服务器端
1. 服务器端使用根证书 `(root.crt)` 解密客户端公钥证书，得到客户端公钥
1. 客户端发送自己支持的加密方案给服务器端
1. 服务器端根据自己和客户端的能力，选择一个双方都能接受的加密方案，使用客户端的公钥加密目标方案. 后发送给客户端；
1. 客户端使用自己的私钥解密加密方案，生成随机密钥 `R`，使用服务器公钥加密后传给服务器端；
1. 服务端用自己的私钥去解密这个密文，得到了密钥 `R`
1. 双方使用随机密钥 `R` 通信的对称加密密钥

## 生成自签名证书

生成这一些列证书之前，我们需要先生成一个 `CA` 根证书，然后由这个 `CA` 根证书颁发服务器公钥证书和客户端公钥证书。为了验证根证书颁发与验证客户端证书这个逻辑，我们使用根证书生成两套不同的客户端证书，然后同时用两个客户端证书来发送请求，看服务器端是否都能识别。下面是证书生成的内在逻辑示意图：

![03-self-signed-sertificate](/images/https-mutual-authentication/03-self-signed-sertificate.png)

### 生成根证书

``` bash
# 创建根证书私钥：
$ openssl genrsa -out root.key 1024

# 创建根证书请求文件：
$ openssl req -new -out root.csr -key root.key
# 后续参数请自行填写，下面是一个例子：
# Country Name (2 letter code) [XX]:cn
# State or Province Name (full name) []:bj
# Locality Name (eg, city) [Default City]:bj
# Organization Name (eg, company) [Default Company Ltd]:alibaba
# Organizational Unit Name (eg, section) []:test
# Common Name (eg, your name or your servers hostname) []:root
# Email Address []: a.divinerapier.cn
# A challenge password []:
# An optional company name []:

# 创建根证书：
$ openssl x509 -req -in root.csr -out root.crt -signkey root.key -CAcreateserial -days 3650
```

可以得到

* `root.crt`: 有效期为 `10` 年的根证书

### 生成自签名服务器端证书

``` bash
# 生成服务器端证书私钥：
$ openssl genrsa -out server.key 1024

# 生成服务器证书请求文件，过程和注意事项参考根证书，本节不详述：
$ openssl req -new -out server.csr -key server.key

# 生成服务器端公钥证书
$ openssl x509 -req -in server.csr -out server.crt -signkey server.key -CA root.crt -CAkey root.key -CAcreateserial -days 3650
```

可以得到

* `server.key`: 服务端私钥文件
* `server.crt`: 有效期为 `10` 年的服务端公钥文件

### 生成自签名客户端证书

``` bash
# 生成客户端证书秘钥：
$ openssl genrsa -out client.key 1024

# 生成客户端证书请求文件，过程和注意事项参考根证书，本节不详述：
$ openssl req -new -out client.csr -key client.key

# 生客户端证书
$ openssl x509 -req -in client.csr -out client.crt -signkey client.key -CA root.crt -CAkey root.key -CAcreateserial -days 3650

# 生客户端p12格式证书，输入一个好记的密码，比如 123456
$ openssl pkcs12 -export -clcerts -in client.crt -inkey client.key -out client.p12
```

可以得到

* `client.key`: 客户端私钥文件
* `client.crt`: 有效期为 `10` 年的客户端公钥文件
* `client.p12`: 同时包含公钥与私钥的客户端 `p12` 证书文件

### 生成证书注意事项

在创建证书请求文件的时候需要注意

* 根证书的 `Common Name` 填写 `root` 就可以
* 所有客户端和服务器端的 `Common Name` 需要填写域名
* 根证书的 `Common Name` 与客户端证书、服务端证书的 `Common Name` 不能相同
* 其他所有字段的填写，根证书、服务器端证书、客户端证书需保持一致 最后的密码可以直接回车跳过

## 使用 golang 构建双向认证通信

### 服务端代码

``` go
package main

import (
    "crypto/tls"
    "crypto/x509"
    "flag"
    "fmt"
    "io/ioutil"
    "net/http"
)

type handler struct {
}

func (h *handler) ServeHTTP(w http.ResponseWriter,
    r *http.Request) {
    fmt.Fprintf(w,
        "Hi, This is an example of https service in golang!\n")
}

func main() {
    var capath, certpath, keypath, bind string
    flag.StringVar(&capath, "ca", "./root.crt", "path of ca file")
    flag.StringVar(&certpath, "cert", "./server.crt", "path of cert file")
    flag.StringVar(&keypath, "key", "./server.key", "path of key file")
    flag.StringVar(&bind, "bind", ":443", "local address to bind")
    flag.Parse()

    pool := x509.NewCertPool()

    caCrt, err := ioutil.ReadFile(capath)
    if err != nil {
        fmt.Println("ReadFile err:", err)
        return
    }
    pool.AppendCertsFromPEM(caCrt)
    s := &http.Server{
        Addr:    bind,
        Handler: &handler{},
        TLSConfig: &tls.Config{
            ClientCAs:  pool,
            ClientAuth: tls.RequireAndVerifyClientCert,
        },
    }

    err = s.ListenAndServeTLS(certpath, keypath)

    if err != nil {
        fmt.Println("ListenAndServeTLS err:", err)
    }
}
```

### 客户端代码

``` go
package main

import (
    "crypto/tls"
    "crypto/x509"
    "flag"
    "fmt"
    "io/ioutil"
    "net/http"
)

func main() {
    var host, capath, certpath, keypath string
    flag.StringVar(&capath, "ca", "./root.crt", "path of ca file")
    flag.StringVar(&certpath, "cert", "./client.crt", "path of cert file")
    flag.StringVar(&keypath, "key", "./client.key", "path of key file")
    flag.StringVar(&host, "host", "https://localhost", "host of https server")
    flag.Parse()

    pool := x509.NewCertPool()

    caCrt, err := ioutil.ReadFile(capath)
    if err != nil {
        fmt.Println("ReadFile err:", err)
        return
    }
    pool.AppendCertsFromPEM(caCrt)

    cliCrt, err := tls.LoadX509KeyPair(certpath, keypath)
    if err != nil {
        fmt.Println("Loadx509keypair err:", err)
        return
    }

    tr := &http.Transport{
        TLSClientConfig: &tls.Config{
            RootCAs:      pool,
            Certificates: []tls.Certificate{cliCrt},
        },
    }
    client := &http.Client{Transport: tr}
    resp, err := client.Get(host)
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()
    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        panic(err)
    }
    fmt.Println(string(body) + "\n")
}
```
