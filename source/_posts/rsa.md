---
title: 证书和签名学习汇总
date: 2017-7-14 19:36:59
tags:
 - 证书
 - 签名
categories:
 - 后端
---

证书相关的玩意，各种文件格式一大堆，大体可以分为：
1. 公私钥文件(.key ...)
2. 证书文件(.pem ...)
3. 证书签名申请文件(.csr ...)

具体的格式又分成两种
1. PEM - Privacy Enhanced Mail,打开看文本格式,以"-----BEGIN..."开头, "-----END..."结尾,内容是BASE64编码
2. DER - Distinguished Encoding Rules,打开看是二进制格式,不可读.

参考<http://www.cnblogs.com/guogangj/p/4118605.html>

>查看PEM格式证书的信息:openssl x509 -in certificate.pem -text -noout
Apache和*NIX服务器偏向于使用这种编码格式.

>查看DER格式证书的信息:openssl x509 -in certificate.der -inform der -text -noout
Java和Windows服务器偏向于使用这种编码格式.

关于DER格式，使用了偏门的编码**[ASN.1](https://en.wikipedia.org/wiki/ASN.1)**，参见[这里](https://tls.mbed.org/kb/cryptography/asn1-key-structures-in-der-and-pem)

# 常用格式
## key/pub
公钥私钥文件
下面的代码生成私钥
``` bash
# 后面的数字可以成比率缩放
openssl genrsa -out server.key 2048
```
实际上生成的key文件包含了公钥和私钥，下面的命令导出公钥
``` bash
openssl rsa -in server.key -pubout > server.pub
```
``` bash
king@king:/tmp/a$ cat server.pub
-----BEGIN PUBLIC KEY-----
king@king:/tmp/a$ cat server.key
-----BEGIN RSA PRIVATE KEY-----
```

前面提到，pem格式都是已BEGIN开头，那么中间又区分若干类型，关于公私钥，有
2. -----BEGIN RSA PUBLIC KEY-----  #PKCS#1
3. -----BEGIN RSA PRIVATE KEY----- #PKCS#1
3. -----BEGIN PUBLIC KEY----- #PKCS#8
4. -----BEGIN PRIVATE KEY----- #PKCS#8
5. -----BEGIN ENCRYPTED PRIVATE KEY----- #encrypted PKCS#8

参见[这里](https://tls.mbed.org/kb/cryptography/asn1-key-structures-in-der-and-pem)，具体而言，设计到PKCS#8 PKCS#1规范的差异

>BEGIN RSA PRIVATE KEY is PKCS#1 and is just an RSA key. It is essentially just the key object from PKCS#8, but without the version or algorithm identifier in front. BEGIN PRIVATE KEY is PKCS#8 and indicates that the key type is included in the key data itself. From the link:

## PKCS#8 PKCS#1转换
``` bash
# 转pkcs8
openssl pkcs8 -topk8 -inform PEM -in server.key -outform pem -nocrypt -out server8.key
# 导出公钥
openssl rsa -in server8.key -pubout > server8.pub
# 转pkcs1
openssl rsa -in server8.key -out server1.key
```
``` bash
king@king:/tmp/a$ cat server8.key
-----BEGIN PRIVATE KEY-----
king@king:/tmp/a$ cat server8.pub
-----BEGIN PUBLIC KEY-----
king@king:/tmp/a$ cat server1.key
-----BEGIN RSA PRIVATE KEY-----
```
参考[这里](http://www.voidcn.com/blog/duan19056/article/p-6137597.html)

关于PKCS，参考
1. <https://zh.wikipedia.org/wiki/%E5%85%AC%E9%92%A5%E5%AF%86%E7%A0%81%E5%AD%A6%E6%A0%87%E5%87%86>
2. <http://falchion.iteye.com/blog/1472453>

## 查看Key
``` bash
openssl rsa -in server8.key -text -noout
openssl rsa -in server1.key -text -noout
# Der格式
openssl rsa -in mykey.key -text -noout -inform der
```

## 其他
1. pem 证书
2. crt certificate的三个字母 常见于*NIX系统,有可能是PEM编码,也有可能是DER编码,大多数应该是PEM编码
3. cer 还是certificate,常见于Windows系统,同样的,可能是PEM编码,也可能是DER编码,大多数应该是DER编码.
4. csr ,即证书签名请求,这个并不是证书,而是向权威证书颁发机构获得签名证书的申请,其核心内容是一个公钥(当然还附带了一些别的信息),在生成这个申请的时候,同时也会生成一个私钥,私钥要自己保管好.
5. pfx/p12 - predecessor of PKCS#12,对*nix服务器来说,一般CRT和KEY是分开存放在不同文件中的,但Windows的IIS则将它们存在一个PFX文件中,(因此这个文件包含了证书及私钥)这样会不会不安全？应该不会,PFX通常会有一个"提取密码",你想把里面的东西读取出来的话,它就要求你提供提取密码
6. jks 即Java Key Storage,这是Java的专利,跟OpenSSL关系不大,利用Java的一个叫"keytool"的工具,可以将PFX转为JKS

参考[这里](http://www.cnblogs.com/guogangj/p/4118605.html)

# 证书编码的转换
1. PEM转为DER 
``` bash
openssl x509 -in cert.crt -outform der -out cert.der
```
1. DER转为PEM 
``` bash
openssl x509 -in cert.crt -inform der -outform pem -out cert.pem
```

>(提示:要转换KEY文件也类似,只不过把x509换成rsa,要转CSR的话,把x509换成req...)

# TLS
``` bash
# 生成服务器端的私钥
openssl genrsa -out server.key 2048
# 生成服务器端证书
openssl req -new -x509 -key server.key -out server.pem -days 3650
```

## 查看证书
``` bash
openssl x509 -noout -text -in server.pem
```

## 证书内容
X.509 数字证书标准，定义证书文件的结构和内容，详情参考 [RFC5280](https://www.ietf.org/rfc/rfc5280.txt),一般由用户公共密钥和用户标识符组成，此外还包括版本号、证书序列号、CA 标识符、签名算法标识、签发者名称、证书有效期等信息。

## Golang [TLS](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)
1. 客户端向服务器端索要并验证公钥。
2. 双方协商生成"对话密钥"。
3. 双方采用"对话密钥"进行加密通信。

Golang基础库对tls支持非常好，Go Package tls部分实现了 tls 1.2的功能，可以满足我们日常的应用。Package crypto/x509提供了证书管理的相关操作。参考<http://colobu.com/2016/06/07/simple-golang-tls-examples/>

server.go
``` go
package main
import (
	"bufio"
	"crypto/tls"
	"log"
	"net"
)
func main() {
	cert, err := tls.LoadX509KeyPair("server.pem", "server.key")
	if err != nil {
		log.Println(err)
		return
	}
	config := &tls.Config{Certificates: []tls.Certificate{cert}}
	ln, err := tls.Listen("tcp", ":4433", config)
	if err != nil {
		log.Println(err)
		return
	}
	defer ln.Close()
	for {
		conn, err := ln.Accept()
		if err != nil {
			log.Println(err)
			continue
		}
		go handleConn(conn)
	}
}
func handleConn(conn net.Conn) {
	defer conn.Close()
	r := bufio.NewReader(conn)
	for {
		msg, err := r.ReadString('\n')
		if err != nil {
			log.Println(err)
			return
		}
		println(msg)
		n, err := conn.Write([]byte("world\n"))
		if err != nil {
			log.Println(n, err)
			return
		}
	}
}
```

cli.go
``` go
package main
import (
	"crypto/tls"
	"log"
)
func main() {
	conf := &tls.Config{
		InsecureSkipVerify: true,
	}
	conn, err := tls.Dial("tcp", "127.0.0.1:4433", conf)
	if err != nil {
		log.Println(err)
		return
	}
	defer conn.Close()
	n, err := conn.Write([]byte("hello\n"))
	if err != nil {
		log.Println(n, err)
		return
	}
	buf := make([]byte, 100)
	n, err = conn.Read(buf)
	if err != nil {
		log.Println(n, err)
		return
	}
	println(string(buf[:n]))
}
```

InsecureSkipVerify用来控制客户端是否证书和服务器主机名。如果设置为true,则不会校验证书以及证书中的主机名和服务器主机名是否一致。

## 验证客户端身份

在有的情况下，需要双向认证，服务器也需要验证客户端的真实性。在这种情况下，我们需要服务器和客户端进行一点额外的配置。

``` go
package main
import (
	"bufio"
	"crypto/tls"
	"crypto/x509"
	"io/ioutil"
	"log"
	"net"
)
func main() {
	cert, err := tls.LoadX509KeyPair("server.pem", "server.key")
	if err != nil {
		log.Println(err)
		return
	}
	certBytes, err := ioutil.ReadFile("client.pem")
	if err != nil {
		panic("Unable to read cert.pem")
	}
	clientCertPool := x509.NewCertPool()
	ok := clientCertPool.AppendCertsFromPEM(certBytes)
	if !ok {
		panic("failed to parse root certificate")
	}
	config := &tls.Config{
		Certificates: []tls.Certificate{cert},
		ClientAuth:   tls.RequireAndVerifyClientCert,
		ClientCAs:    clientCertPool,
	}
	ln, err := tls.Listen("tcp", ":443", config)
	if err != nil {
		log.Println(err)
		return
	}
	defer ln.Close()
	for {
		conn, err := ln.Accept()
		if err != nil {
			log.Println(err)
			continue
		}
		go handleConn(conn)
	}
}
func handleConn(conn net.Conn) {
	defer conn.Close()
	r := bufio.NewReader(conn)
	for {
		msg, err := r.ReadString('\n')
		if err != nil {
			log.Println(err)
			return
		}
		println(msg)
		n, err := conn.Write([]byte("world\n"))
		if err != nil {
			log.Println(n, err)
			return
		}
	}
}
```

client.go
``` go
package main
import (
	"crypto/tls"
	"crypto/x509"
	"io/ioutil"
	"log"
)
func main() {
	cert, err := tls.LoadX509KeyPair("client.pem", "client.key")
	if err != nil {
		log.Println(err)
		return
	}
	certBytes, err := ioutil.ReadFile("client.pem")
	if err != nil {
		panic("Unable to read cert.pem")
	}
	clientCertPool := x509.NewCertPool()
	ok := clientCertPool.AppendCertsFromPEM(certBytes)
	if !ok {
		panic("failed to parse root certificate")
	}
	conf := &tls.Config{
		RootCAs:            clientCertPool,
		Certificates:       []tls.Certificate{cert},
		InsecureSkipVerify: true,
	}
	conn, err := tls.Dial("tcp", "127.0.0.1:443", conf)
	if err != nil {
		log.Println(err)
		return
	}
	defer conn.Close()
	n, err := conn.Write([]byte("hello\n"))
	if err != nil {
		log.Println(n, err)
		return
	}
	buf := make([]byte, 100)
	n, err = conn.Read(buf)
	if err != nil {
		log.Println(n, err)
		return
	}
	println(string(buf[:n]))
}
```

# 证书
参考
1. <http://www.cnblogs.com/JeffreySun/archive/2010/06/24/1627247.html>
2. <http://www.barretlee.com/blog/2016/04/24/detail-about-ca-and-certs/>
3. <http://www.ruanyifeng.com/blog/2011/08/what_is_a_digital_signature.html>
4. <https://en.wikipedia.org/wiki/Public_key_certificate>


证书部分典型字段

1. 证书的发布机构
1. 证书的有效期
1. 公钥
1. 证书所有者（Subject）
1. 签名所使用的算法
1. 指纹以及指纹算法

◆Issuer (证书的发布机构)

指出是什么机构发布的这个证书，也就是指明这个证书是哪个公司创建的(只是创建证书，不是指证书的使用者)。例如"SecureTrust CA"


◆Valid from , Valid to (证书的有效期)

也就是证书的有效时间，或者说证书的使用期限。 过了有效期限，证书就会作废，不能使用了。

◆Public key (公钥)

这个我们在前面介绍公钥密码体制时介绍过，公钥是用来对消息进行加密的，第2章的例子中经常用到的。这个数字证书的公钥是2048位的，它的值可以在图的中间的那个对话框中看得到，是很长的一串数字。

◆Subject (主题)

这个证书是发布给谁的，或者说证书的所有者，一般是某个人或者某个公司名称、机构的名称、公司网站的网址等。 对于这里的证书来说，证书的所有者是Trustwave这个公司。

◆Signature algorithm (签名所使用的算法)

就是指的这个数字证书的数字签名所使用的加密算法，这样就可以使用证书发布机构的证书里面的公钥，根据这个算法对指纹进行解密。指纹的加密结果就是数字签名(第1.5节中解释过数字签名)。

◆Thumbprint, Thumbprint algorithm (指纹以及指纹算法)

这个是用来保证证书的完整性的，也就是说确保证书没有被修改过，这东西的作用和2.7中说到的第3个问题类似。 其原理就是在发布证书时，发布者根据指纹算法(一个hash算法)计算整个证书的hash值(指纹)并和证书放在一起，使用者在打开证书时，自己也根据指纹算法计算一下证书的hash值(指纹)，如果和刚开始的值对得上，就说明证书没有被修改过，因为证书的内容被修改后，根据证书的内容计算的出的hash值(指纹)是会变化的。 注意，这个指纹会使用"SecureTrust CA"这个证书机构的私钥用签名算法(Signature algorithm)加密后和证书放在一起。

就是指的这个数字证书的数字签名所使用的加密算法，这样就可以使用证书发布机构的证书里面的公钥，根据这个算法对指纹进行解密。指纹的加密结果就是数字签名(第1.5节中解释过数字签名)。

## 如何向证书的发布机构去申请证书

举个例子方便大家理解，假设我们公司"ABC Company"花了1000块钱，向一个证书发布机构"SecureTrust CA"为我们自己的公司"ABC Company"申请了一张证书，注意，这个证书发布机构"SecureTrust CA"是一个大家公认并被一些权威机构接受的证书发布机构，我们的操作系统里面已经安装了"SecureTrust CA"的证书。"SecureTrust CA"在给我们发布证书时，把Issuer,Public key,Subject,Valid from,Valid to等信息以明文的形式写到证书里面，然后用一个指纹算法计算出这些数字证书内容的一个指纹，并把指纹和指纹算法用自己的私钥进行加密，然后和证书的内容一起发布。

## 证书的吊销

CA 证书的吊销存在两种机制，一种是在线检查，client 端向 CA 机构发送请求检查 server 公钥的靠谱性；第二种是 client 端储存一份 CA 提供的证书吊销列表，定期更新。前者要求查询服务器具备良好性能，后者要求每次更新提供下次更新的时间，一般时差在几天。安全性要求高的网站建议采用第一种方案。

大部分 CA 并不会提供吊销机制（CRL/OCSP），靠谱的方案是为根证书提供中间证书，一旦中间证书的私钥泄漏或者证书过期，可以直接吊销中间证书并给用户颁发新的证书。中间证书的签证原理于上上条提到的原理一样，中间证书还可以产生下一级中间证书，多级证书可以减少根证书的管理负担。

## [证书链](http://www.jianshu.com/p/46e48bc517d0)
除了end-user之外，证书被分为root Certificates和intermediates Certificates。相应地，CA也分了两种类型：root CAs 和 intermediates CAs。首先，CA的组织结构是一个树结构，一个root CAs下面包含多个intermediates CAs，而intermediates又可以包含多个intermediates CAs。root CAs 和 intermediates CAs都可以颁发证书给用户，颁发的证书分别是root Certificates和intermediates Certificates，最终用户用来认证公钥的证书则被称为end-user Certificates。

我们使用end-user certificates来确保加密传输数据的公钥(public key)不被篡改，而又如何确保end-user certificates的合法性呢？这个认证过程跟公钥的认证过程类似，首先获取颁布end-user certificates的CA的证书，然后验证end-user certificates的signature。一般来说，root CAs不会直接颁布end-user certificates的，而是授权给多个二级CA，而二级CA又可以授权给多个三级CA，这些中间的CA就是intermediates CAs，它们才会颁布end-user certificates。

但是intermediates certificates的可靠性又如何保证呢？这就是涉及到证书链，Certificate Chain ，链式向上验证证书，直到Root Certificates

那Root Certificates又是如何来的呢？根据 <https://support.dnsimple.com/articles/what-is-ssl-certificate-chain/> 这篇文章的说法，除了可以下载安装之外，device（例如浏览器，操作系统）都会内置一些root certificates，称之为trusted root certificates，<https://support.apple.com/en-us/HT202858> ，在Apple的官网上可以看到这个列表，有各个操作版本直接内置的Root Certificates。

最后一个问题，为什么需要证书链这么麻烦的流程？Root CA为什么不直接版本证书，而是要搞那么多中间层级呢？找了一下，godaddy官方给了一个答案，为了确保root certificates的绝对安全性，<https://sg.godaddy.com/en/help/what-is-an-intermediate-certificate-868> ，将根证书隔离地越严格越好。

## 证书请求(certificate sign request)
``` bash
openssl genrsa -out ssl.key 2048
openssl req -new -key ssl.key -out ssl.csr
```

证书(certificate)和证书请求(certificate sign request)

1. 证书是自签名或CA签名过的凭据，用来进行身份认证
1. 证书请求是对签名的请求，需要使用私钥进行签名

x509命令可以将证书和证书请求相互转换

## 自签名证书
自签名的原理是用私钥对该私钥生成的证书请求进行签名，生成证书文件。该证书的签发者就是自己，所以验证方必须有该证书的私钥才能对签发信息进行验证，所以要么验证方信任一切证书，面临冒名顶替的风险，要么被验证方的私钥（或加密过的私钥）需要发送到验证方手中，面临私钥泄露的风险。
``` bash
king@king:/tmp/a$ openssl rsa -in ssl.key -des3 -out encrypted.key
king@king:/tmp/a$ openssl req -new -key ssl.key -out ssl.csr
king@king:/tmp/a$ openssl x509 -req -in ssl.csr -signkey ssl.key -out ssl.crt
```

# Exe签名
导出p12格式的证书，易被Windows下的工具支持
``` bash
openssl pkcs12 -export -clcerts -in ssl.crt -inkey ssl.key -out ssl.p12
# 用Windows sdk的工具signtool实现exe文件签名
signtool sign /f ssl.pfx /p mypassword abc.exe
```

# Https
1. 服务器把证书信息发送给浏览器
2. 浏览器现在开始验证证书的合法性，证书验证之后，随机一个对称加密的秘钥，并加密一段握手信息，并用公钥加密秘钥之后发给服务器
3. 服务器用私钥解密之后得到秘钥，解密密文验证无误后，加密一段握手信息发送给浏览器
4. 浏览器验证密文无误，开始SSL安全连接

## 问题
1. 直接用rsa加密，效率太低
2. 直接用rsa加密，回程无安全性
