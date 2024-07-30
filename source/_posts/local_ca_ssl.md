---
title: 制作CA签发验证SSL证书
tags: []
categories: []
date: '2024-01-12T17:05:48.000Z'
abbrlink: 54c1e4fe
---
# 背景

**为什么要用SSL？**

对于我来说，最重要的就是**数据加密**。在互联网访问中，HTTPS相比HTTP，提供了TLS加密数据。别人只能看到你访问了某个域名和IP，但是不知道我们之间传输了什么内容。

**TLS解决了什么？**


1. 如何保证数据传输的保密性；
2. 如何保证数据传输的完整性；
3. 如何保证通信主体的真实性；

其一，它用对称加密方法加密中间来回传输的流量，使得中间节点只能看到密文，并且中间节点一般拿不到解密密钥把密文恢复成明文。

其二，它提供了端到端的安全性，就好像是在两个进程之间建立了一条坚实可靠的「管道」保护着管道里边传输的报文，使得数据在进程到进程之间的传播是完整的，就算被篡改了也是能检验得出的；

其三，它通过公钥加密的技术，结合现有的 PKI（公钥密码学基础设施），提供了通信双方验证对端身份真实性的一种机制，

**为什么不用免费的SSL证书？**

实际上我用了Cloudflare的服务后，自动提供了SSL的证书，且不需要关心有效期的问题。

但是 [在公司无感访问家里的服务](https://blog.lianglianglee.com/2024/01/11/office_visit_home_server/) 中，办公环境下，全程都是HTTP，数据完全透明，同样需要HTTPS为数据安全提供保障。

如果使用免费的证书，则每三个月需要申请一次，比较麻烦，因为都是内网访问，所以更简单一点，直接用自己的根证书，后续也能用于内网的各种HTTPS访问

**计算机怎么知道这个证书是可信的呢？**

操作系统通常会预装一组根证书（Root Certificates）作为信任锚点，用于验证数字证书的真实性。这些根证书是由大型、受信任的CA签发的，因此用户可以信任这些根证书，从而信任由这些根证书签发的其他数字证书。这种信任链建立了对整个公共PKI的信任，确保了数字证书的可信度。新的根证书可能会在操作系统更新中加入，以适应不断变化的证书颁发机构和安全需求。

**获取TSL的流程**

获取TLS（Transport Layer Security）证书通常涉及以下步骤。因为TLS的前身是SSL（Secure Sockets Layer），所以经常会说SSL证书


1. 选择证书颁发机构
2. 生成证书签发请求（CSR）
3. 提交CSR到CA
4. 身份验证
5. 接受证书
6. 安装证书

主要流程如上，实际过程中，提交CSR，身份验证，接受证书都会受CA机构的影响，会有所不同。

# 创建根证书

根据前面的介绍，我们只需要安装自己生成的根证书，那这个证书颁发的所有证书都是可信的。所以我们需要先生成一个根证书，然后让私有设备信任这个根证书。

> 对于这篇文章中讲到的根证书、普通证书，学名都是 X.509v3 证书，区别只在于：
>
> 
> 1. 根证书是自己给自己签发的
> 2. 根证书和普通证书的扩展参数有些不一样
>
> \
> CA 证书也好普通证书也好，都可以统一地用 openssl 的那一套命令行工具来生成、签发、验证和查看，非常的方便。

接下来不涉及到 CRL 和 OCSP，感兴趣的朋友可以自行查阅 Wikipedia 和 [rfc5280](https://datatracker.ietf.org/doc/html/rfc5280).

## 生成私钥

最好在一个单独的目录操作，先生成一个CA私钥

```bash
openssl genrsa -out ca.key 2048
```

我们会得到一个 [PEM](https://en.wikipedia.org/wiki/Privacy-Enhanced_Mail) 格式的私钥，它的默认 permissions 也很有意思：

```bash
ls -all ca.key
---
-rw------- 1 root root 1704 Dec 10 15:18 ca.key
```

接下来要开始签发自签证书，按照 openssl 的惯例，我们可以先把证书的一些信息写到一个配置文件中：包括这个证书对应的主体 (subject) 是谁、这个证书是做什么用的、这个证书的唯一标识和签发者的唯一标识等获取方式等：

```ini
[ req ]
distinguished_name  = req_distinguished_name
prompt              = no
x509_extensions     = v3_ca

[ req_distinguished_name ]
countryName                     = CN
stateOrProvinceName             = Beijing
localityName                    = Beijing
0.organizationName              = LocalOrganizationName              
organizationalUnitName          = IT
commonName                      = Homelab Root CA
emailAddress                    = admin@example.com

[ v3_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:2
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
```

把它保存为当前目录的 `ca.root.ini​​` 文件。

> 希望进一步了解配置文件的格式以及其中各个字段含义的读者可以参阅 man 5 config​​ 和 man 5 x509v3_config​​.

## 生成证书

```bash
openssl req -new -x509 -key ca.key -config ca.root.ini​​ -out ca.crt
```

得到一个 PEM 格式的 `ca.crt`​​ 文件位于当前文件夹。

我们可以通过 `openssl x509`​​ 命令查看此证书的详细信息：

```bash
openssl x509 -in ca.crt -noout -text
---
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            60:9a:7a:ba:8c:98:9b:f8:1a:de:46:c4:0c:05:0c:7f:b2:b8:c9:ab
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = CN, ST = Beijing, L = Beijing, O = LocalOrganizationName, OU = IT, CN = Homelab Root CA, emailAddress = admin@example.com
        Validity
            Not Before: Jan 12 09:31:08 2024 GMT
            Not After : Feb 11 09:31:08 2024 GMT
        Subject: C = CN, ST = Beijing, L = Beijing, O = LocalOrganizationName, OU = IT, CN = Homelab Root CA, emailAddress = admin@example.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:bf:c3:ee:37:75:2d:15:9c:1a:dd:d7:77:5c:78:
                    .....略
                    dd:a7:5e:09:36:d4:87:96:0b:42:12:a3:fd:99:95:
                    8c:63
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier: 
                51:27:75:BE:3E:24:AE:A5:93:C7:1C:DE:F8:23:FB:7F:4D:BF:53:34
            X509v3 Authority Key Identifier: 
                51:27:75:BE:3E:24:AE:A5:93:C7:1C:DE:F8:23:FB:7F:4D:BF:53:34
            X509v3 Basic Constraints: critical
                CA:TRUE, pathlen:2
            X509v3 Key Usage: critical
                Digital Signature, Certificate Sign, CRL Sign
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        8a:c9:ab:63:2d:a2:aa:72:7b:d9:41:27:d2:5d:00:14:f0:77:
        ...略
        76:08:94:3c:b5:24:12:0a:71:8a:46:60:5e:83:78:1d:09:68:
        f3:80:87:0e
```

我们发现 X509v3 Subject Key Identifier 和 X509v3 Authority Key Identifier 有相同的值，前者是它自己的唯一编号，后者是签发此证书的证书的唯一编号，再结合 X509v3 Basic Constraints 的 CA:TRUE 字样，使得验证者可以判断它是一个自签的根 CA 证书。

这个 CA 证书的 pathlen 是 2，这意味着它还可以再签发一个 pathlen 等于 1 的中级 CA 证书，然后这个 pathlen 等于 1 的中级 CA 证书自己又可以再签发一个 pathlen 等于 0 的中级 CA 证书，pathlen 等于 0 的 CA 证书不能再签发 CA 证书，只能签发最终用户证书 (end-entity certificate).

我们可以用这行命令验证它：

```bash
openssl verify --show_chain -CAfile ca.crt ca.crt
---
ca.crt: OK
Chain:
depth=0: C = CN, ST = Beijing, L = Beijing, O = LocalOrganizationName, OU = IT, CN = Homelab Root CA, emailAddress = admin@example.com
```

接下来我们选择直接用它签发 end-entity certificate

# 证书签发

## 生成私钥

```bash
openssl genrsa -out x.com.key 2048
```

准备一个 `x.com.ini​​` 配置文件，填写主体信息：

```ini
[ req ]
distinguished_name     = req_distinguished_name
prompt                 = no

[ req_distinguished_name ]
countryName                     = CN
stateOrProvinceName             = Beijing
localityName                    = Beijing
0.organizationName              = LocalOrganizationName              
organizationalUnitName          = IT
commonName                      = Homelab Root CA
emailAddress                    = admin@example.com

subjectAltName=@SubjectAlternativeName

[ SubjectAlternativeName ]
DNS.1 = *.x.com
```

> 请注意SubjectAlternativeName模块，这个地方是证书用于什么域名，多个可以用
>
> ```ini
> DNS.1 = *.x.com
> DNS.2 = *.x.com
> ```
>
> 也可以指定IP
>
> ```ini
> DNS.1 = *.x.com
> DNS.2 = *.x.com
> IP.1 = xxx.xxx.xxx.xxx
> IP.2 = xxx.xxx.xxx.xxx
> ```

## 创建CSR (certificate signing request):

```bash
openssl req -new -key x.com.key -config x.com.ini -out x.com.csr
```

可以用下列命令来查看 CSR 信息，确保它已经有了对应的私钥和主体信息：

```ini
openssl req -in x.com.csr -noout -text
---
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: C = CN, ST = Beijing, L = Beijing, O = LocalOrganizationName, OU = IT, CN = Homelab Root CA, emailAddress = admin@example.com, subjectAltName = @SubjectAlternativeName
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:d7:ef:3f:0f:50:b6:1a:43:2c:90:77:82:01:2a:
                    ...略
                    5b:67:11:4c:b2:f2:51:3a:7d:77:1a:9d:cf:0e:86:
                    da:f7
                Exponent: 65537 (0x10001)
        Attributes:
            (none)
            Requested Extensions:
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        3d:3a:d4:78:81:1f:fa:29:15:6a:67:2d:6c:92:42:7f:4c:fe:
        ...略
        01:c0:cc:f4:16:5f:d8:2e:60:e4:3e:94:e1:6e:18:8f:b3:a8:
        a8:93:71:06
```

然后我们再创建一个 `x.com.crt.ini` 文件，保存以下内容，**其中SubjectAlternativeName跟上面的保持一致**

```ini
[ server_cert ]
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth

subjectAltName=@SubjectAlternativeName
[ SubjectAlternativeName ]
DNS.1 = *.x.com
```

> 同样也可以多个

## 签发证书

```ini
openssl x509 \
    -in x.com.csr \
    -req \
    -CA ca.crt \
    -CAkey ca.key \
    -CAserial ca.srl \
    -CAcreateserial \
    -extfile x.com.crt.ini \
    -extensions server_cert \
    -copy_extensions copyall \
    -days 360 -out x.com.crt
    
---
Certificate request self-signature ok
subject=C = CN, ST = Beijing, L = Beijing, O = LocalOrganizationName, OU = IT, CN = Homelab Root CA, emailAddress = admin@example.com, subjectAltName = @SubjectAlternativeName   
```

> `extensions server_cert` 与`x.com.crt.ini`的标签保持一致

在以上过程中，`x.com.ini​​` 和 `x.com.csr​​` 是申请证书的客户（比如网站站长、应用开发者）生成，`x.com.crt.ini`​​ 是签发证书的 CA 生成，`x.com.crt`​​ 也是 CA 生成。

## 查看证书

```ini
openssl x509 -in x.com.crt -noout -text
---
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            5b:3e:9f:d1:7d:8f:f2:ab:18:ba:fa:1d:d1:65:ac:4e:40:f2:e3:ac
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = CN, ST = Beijing, L = Beijing, O = LocalOrganizationName, OU = IT, CN = Homelab Root CA, emailAddress = admin@example.com
        Validity
            Not Before: Jan 12 09:49:16 2024 GMT
            Not After : Jan  6 09:49:16 2025 GMT
        Subject: C = CN, ST = Beijing, L = Beijing, O = LocalOrganizationName, OU = IT, CN = Homelab Root CA, emailAddress = admin@example.com, subjectAltName = @SubjectAlternativeName
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:d7:ef:3f:0f:50:b6:1a:43:2c:90:77:82:01:2a:
                    ...略
                    da:f7
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints: 
                CA:FALSE
            X509v3 Subject Key Identifier: 
                AC:41:8A:AD:CB:6C:D5:68:E9:B7:59:F2:4B:E5:EF:F5:DA:50:49:C4
            X509v3 Authority Key Identifier: 
                keyid:51:27:75:BE:3E:24:AE:A5:93:C7:1C:DE:F8:23:FB:7F:4D:BF:53:34
                DirName:/C=CN/ST=Beijing/L=Beijing/O=LocalOrganizationName/OU=IT/CN=Homelab Root CA/emailAddress=admin@example.com
                serial:60:9A:7A:BA:8C:98:9B:F8:1A:DE:46:C4:0C:05:0C:7F:B2:B8:C9:AB
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment, Data Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication
            X509v3 Subject Alternative Name: 
                DNS:*.x.com
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        05:8e:3f:dc:a5:a8:f8:7b:ac:74:6a:c3:c7:b3:f0:33:31:ca:
        ...略
        91:f6:40:0e
```

会看到这里已经有了签名值，它的 X509v3 Authority Key Identifier 刚好就是 CA 的 X509v3 Subject Key Identifier，它的 Public Key Modulus 和 CSR 里面的也一样。

## 验证证书

```ini
openssl verify --show_chain -CAfile ca.crt x.com.crt
---
x.com.crt: OK
Chain:
depth=0: C = CN, ST = Beijing, L = Beijing, O = LocalOrganizationName, OU = IT, CN = Homelab Root CA, emailAddress = admin@example.com, subjectAltName = @SubjectAlternativeName (untrusted)
depth=1: C = CN, ST = Beijing, L = Beijing, O = LocalOrganizationName, OU = IT, CN = Homelab Root CA, emailAddress = admin@example.com
```

至此证书生成及颁发已经完成

# 安装根证书

在整个文章中，根证书是`ca.crt`下载到设备上在安装即可。

以windows为例，直接双击打开，即可安装

![](https://static.lianglianglee.com/2024/01/a134652335648d6ae317be397ddcca45.png)

 这里可以选择任意存储

# 配置域名证书

以nginx为例

```ini
server {
  listen [::]:1234 ssl http2;

  gzip off;
  server_name x.com;

  ssl_certificate /etc/nginx/certs/x.com.crt;
  ssl_certificate_key /etc/nginx/certs/x.com.key;

  location / {
    add_header Content-Type "text/html; charset=UTF-8";
    return 200 "Hello, World!";
  }
}
```

把 `ssl_certificate` 和 `ssl_certificate_key` 的值改为证书和私钥的实际路径。

执行 `sudo nginx -t` 测试配置有效性，执行 `sudo nginx -s reload` 更新配置。

在本地 hosts 配置域名解析，把 `x.com` 解析到服务器的IP地址，这个在不同操作系统上做法不同。

## 验证

以 https scheme 加端口的方式在浏览器访问，`https://x.com`

![](https://static.lianglianglee.com/2024/01/2ca5b19e09cad661ba0578ebd1ed610e.png)

# 总结和后续

我们介绍了一种创建自签 CA 证书的方式，并且介绍了如何使用 openssl 命令行工具创建和签发自己的证书。自签证书的好处是时效性和低成本，我们不需要有自己的服务器或者 Web 虚拟主机空间，也不需要购买域名，也不需要花时间等待 CA 做验证和恢复。

这种自签证书适用于内网 (intranet)，尤其是部署安装根 CA 证书的成本很低的情形，安装了根证书后，后续所有由该根证书签发的证书都可以被系统自动信任。

TLS 可以自动使得上层的应用层服务自动地获得端到端的安全性：**1）数据完整性；2）保密性；3）真实性**，因此许多应用软件都已提供了对 TLS 的支持，包括最常见的浏览器、数据库命令行工具、远程桌面软件、常驻程序的后台管理界面等。我们可以给每一个这样的服务都签发 TLS 证书，而在终端设备上只需添加信任一次根证书，从而以低成本的方式获得信息安全性提高以及更好的隐私保障。