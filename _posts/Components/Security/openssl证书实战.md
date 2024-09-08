---
title: openssl证书实战
date: 2020-07-17 00:09:13
tags: [Security]
---

# 生成证书步骤 #

1. 自签发Root Key, 生成Root CA, 并将Root CA加入到系统或浏览器信任证书列表中。
2. 由Root CA 生成Itermidiate CAs(可跳过, Ps: 通常Root CA不会自己签发证书而是签发Intermidate CAs, 由它们来签发证书，从而保护根证书的绝对安全)
3. 由1或2生成的 CA签发证书， 把该证书用于应用环境。

OpenSSl是一个免费开源的库，提供了构建数字证书的命令行工具，可以用于构建证书。

# OpenSSL 查看证书 # 

查看KEY信息

&gt; openssl rsa -noout -text -in myserver.key

查看CSR信息

&gt; openssl req -noout -text -in myserver.csr

查看证书信息

&gt; openssl x509 -noout -text -in ca.crt

# OpenSSL 生成证书 #

## 自签发CA Authority证书 ##

### 1. 选择CA 路径 ###

首先选择一个目录，作为工作目录。系统自带的OpenSSl工作目录如下：

&gt; cd /etc/pki/CA

当然也可以自定义一个目录进行配置，本文使用默认路径。

选择CA路径后，需要进行如下配置

```shell
 mkdir certs  crl  newcerts  private
 chmod 700 private
 touch index.txt
 touch serial
 echo 01 &gt; serial
 # 如果不是默认路径，则需要下载openssl.cnf文件
 # 默认路径下，该文件路径为/etc/pki/tls/openssl.cnf
```

* private，CA的私钥；
* newcerts， 保存CA新签发的证书；
* crl ， 被吊销的证书列表；
* index.txt，保存签发的证书信息；
* serial，保存证书签发的序列号；


### 2. 生成CA Root Pair (Root Key + Root Certificate) ###

作为一个CA Authority，需要管理大量的 pair 对，而原始的一对 pair 对叫做 root pair，它包含了 root key（cakey.pem）和 root certificate（cacert.pem）。

* cakey.pem: 一对RSA公钥密钥, 
* cacert.pem: 证书

创建Root key

```shell
# 创建root key, 选择一种加密方式生成密钥 并设置密钥长度
&gt; openssl genrsa -des3 -out private/cakey.pem 2048 

# 修改root key的访问权限为只读
&gt; chmod 400 private/cakey.pem

# 查看公钥
&gt; cat private/cakey.pem

# 查看私钥
&gt; openssl rsa -in private/cakey.pem -out user_ca_unencrypted.pem -outform PEM

# ps : key和.pem的格式并没有什么区别, 只是一个后缀名
```

创建Root Certification

```shell
# 自定义目录 直接生成Root Crt
# 若使用openSSL 默认目录, 则无需指定OpenSSL Config路径
openssl req -config openssl.cnf -key private/cakey.pem -new -x509 -days 7300 -sha256 -extensions v3_ca -out cacert.pem

Enter pass phrase for ca.key.pem: secretpassword
You are about to be asked to enter information that will be incorporated
into your certificate request.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name []:Zhejiang
Locality Name []: HangZhou
Organization Name []: 
Organizational Unit Name []:AMA Certificate Authority
Common Name []: AMA Certificate Authority
Email Address []: test@lab.com

```

通常情况下，root CA 不会直接为服务器或者客户端签发证书，它们会先为自己生成几个中间 CA（intermediate CAs），这几个中间 CA 作为 root CA 的代表为服务器和客户端签证。

如下图所示，DigCert生成中间CA为zhihu.com颁发了证书

![](../../files/images/certificate_zhihu.png)

---

## 为用户签发证书 ##

### 1. 生成用户的RSA密钥 ###

&gt; openssl genrsa -des3 -out user_ca.key 2048

查看公钥

&gt; cat user_ca.key

查看私钥

&gt; openssl rsa -in user_ca.key -out user_ca_unencrypted.pem -outform PEM

### 2. 生成用户CA Request ###

&gt; openssl req -new -days 365 -key user_ca.key -out user_ca.csr 

自定义目录需指定config 路径

&gt; openssl req -new -days 365 -key user_ca.key -out user_ca.csr -config openssl.cnf

### 3. 使用CA Authority证书签发用户CA 证书 ###

&gt; openssl ca -in user_ca.csr -out user_ca.pem -extensions v3_req

---

# CN：Common Name 如何填写 #

当我们生成一个CA证书，需要指定一个Common Name，客户端使用HTTPS访问某个网址时, 会获得并校验网站的CA证书是否合法，其中包括域名合法性检测，匹配当前URL是否匹配CN或DNS或IP Address。

在证书不支持SubjectAltName情况下，默认匹配Common Name.

因此我们需要绑定CN到某个具体的Host Name，这里需要引入通配符。

&gt; SSL通配符证书是在一个单一的证书中，在通用名（域名）中包含一个“*”通配符字段。这使得该证书可以保护无限数量的多个子域名（主机）。

* *.domain.com 可以匹配 www.domain.com ， mail.domain.com ，pay.domain.com .
* 通配符只能匹配当前一级且只能设置一个，当设置多个通配符无效。
* 通配符后面的域名必须有两级或以上。如*.domain.com 有效，*.com无效
* 通配符不可匹配空。 如*.domain.com 不可匹配 domain.com

---

# 如何签发多域名证书 #

## 多域名机制 ## 

使用Common Name,即便使用通配符，也只能匹配一类的域名。

SubjectAlternativeName（简称：san）是X509 Version 3 (RFC 2459)的扩展，允许ssl证书指定多个可以匹配的名称。

![](../../files/images/google_san.png)

## 生成证书请求文件 ##

生成一个通用的Csr文件时，会交互式的键入Subject所属的内容，但是SAN则不会要求输入。我们需要修改openssl.cnf的x509 V3扩展的配置和san的参数。


SubjectAltName 可以包含email 地址，ip地址，正则匹配DNS主机名，等等。 
ssl这样的一个特性叫做：SubjectAlternativeName（简称：san）

```shell
vi  /etc/pki/tls/openssl.cnf

# 设置启动req_extensions， 并使用v3_req的配置
[ req ]
...
req_extensions = v3_req 

[ v3_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
# 添加subjectAltName， 并使用alt_names的配置
subjectAltName = @alt_names

# 配置DNS 和 IP 
[ alt_names ]
DNS.1 = *.statestr.com
DNS.2 = *.it.statestr.com
IP.1 = 10.248.66.59

```

在配置完openssl.cnf后，后续生成crt操作参照本文对应章节。


如图所示，IP Address 通过了HTTPS认证

![](../../files/images/san.png)


# 使用NodeJS 校验SSL 证书 #

```js
// https-server.js
var https = require(&#39;https&#39;);
var fs = require(&#39;fs&#39;);

var options = {
  key: fs.readFileSync(&quot;C:\\Users\\e662491\\Desktop\\https\\user_ca.key&quot;),
  cert: fs.readFileSync(&#39;C:\\Users\\e662491\\Desktop\\https\\user_ca.pem&#39;),
  passphrase: &#39;123456789&#39;
};

https.createServer(options, function(req, res) {
  res.writeHead(200);
  res.end(&#39;hello world&#39;);
}).listen(8000, function(){
  console.log(&#39;Right URL: https://10.248.66.59:8000&#39;);
  console.log(&#39;Error URL: https://localhost:8000&#39;);
});
```

启动 HTTPS Server

&gt; node https-server.js

# CA 证书链 #

通过CA 证书加密HTTPS请求，数据实现了加密，那么CA证书又是如何被确认为真实的证书的呢。
这里同样需要一套机制来确保CA 证书的正确性。
详情可见下文

[证书链-Digital Certificates](https://www.jianshu.com/p/46e48bc517d0)

# 参考资料 #

[使用OpenSSL生成多域名自签名证书进行HTTPS开发调试](https://zhuanlan.zhihu.com/p/26646377)

[关于SSL证书通用名（CN）通配符的实验](https://webcache.googleusercontent.com/search?q=cache:rJFd3sHjYcMJ:https://blog.creke.net/777.html+&cd=7&hl=en&ct=clnk&gl=us)

[细说 CA 和证书](https://www.barretlee.com/blog/2016/04/24/detail-about-ca-and-certs/)