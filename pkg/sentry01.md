sentry服务的作用和源码分析

可以参考三篇通俗易懂的文章：

1. [数字签名是什么？](https://www.ruanyifeng.com/blog/2011/08/what_is_a_digital_signature.html)
2. [HTTPS 精读之 TLS 证书校验](https://zhuanlan.zhihu.com/p/30655259)
3. [证书链-Digital Certificates](https://www.jianshu.com/p/46e48bc517d0)

CA：Certificate Authority(证书签发机构)， 用来签发公钥
公钥：public key
私钥：private key

### CA机构
 **需要CA机构的理由**：

 比如：鲍勃有一把公钥和私钥，苏姗写信给鲍勃。公钥是公开，大家都可以拿到的；私钥只能鲍勃有，且不能泄漏

 **正常情况下**:苏珊把写信的内容通过公钥进行加密，鲍勃接收到苏珊加密的信件后，用私钥解密，就得到了信的可见内容；

 **异常情况下**：
1. 如果第三方窃取了鲍勃的私钥，则可以打开任何人写给鲍勃的信件。(这个需要鲍勃自己防窃取，暂时找不到合适的方式);
2. 如果第三方把苏珊的公钥给换成第三方自己的公钥，则第三方可以解析苏珊写给鲍勃的任何信件；(这个就可以通过CA机构解决);

针对第二种异常情况，需要一个证明该公钥是鲍勃身份的权威机构，如果第三方偷换了苏珊手上的公钥，则苏珊写信之前找这个权威机构查下自己手上的公钥是不是鲍勃的，如果不是，则表示该公钥被篡改了。而这个权威机构就是CA机构。它用来为想要使用证书的单位签署和颁布公钥证书。这样任何公钥都是安全的，且不会被偷换了。

那么在dapr-system命名空间下的dapr sentry服务，就是为所有想要CSR(certificate signing request, 证书签名请求), 获取数字证书，也即是证书公钥文件。这样经过认证的公钥，都是在CA机构进行备案且能查找公钥的企业或者个人的真实身份。且是权威的。

因为dapr sentry服务作为CA机构，它主要用于k8s集群内部的dapr runtime sidecar grpc的http2请求认证，内部认证且不会和外部有访问。所以自己内部建立一个CA机构就OK了。

### 私钥和公钥

公钥是指CERTIFICATE，该证书内容通过解码后，可以获取**协议版本号**， **签名算法**(sha256对数据进行哈希，然后与CA机构私钥进行RSA加密), **证书有效期范围**, **主体**(地址，机构名等), **公钥**等信息

公钥示例如下：
```shell
-----BEGIN CERTIFICATE-----
MIIDgzCCAmugAwIBAgIEckecVjANBgkqhkiG9w0BAQsFADByMQswCQYDVQQGEwJD
TjESMBAGA1UECBMJZ3Vhbmdkb25nMREwDwYDVQQHEwhzaGVuemhlbjEQMA4GA1UE
ChMHdGVuY2VudDEQMA4GA1UECxMHdGVuY2VudDEYMBYGA1UEAxMPc3NvLnRlbmNl
bnQuY29tMB4XDTE1MTEzMDAyMDcxNloXDTE2MDIyODAyMDcxNlowcjELMAkGA1UE
BhMCQ04xEjAQBgNVBAgTCWd1YW5nZG9uZzERMA8GA1UEBxMIc2hlbnpoZW4xEDAO
BgNVBAoTB3RlbmNlbnQxEDAOBgNVBAsTB3RlbmNlbnQxGDAWBgNVBAMTD3Nzby50
ZW5jZW50LmNvbTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAJvvN6AU
eq01dFk+47xRF8OBIyKCxQUPvaGRJJKbh1F58jmyLSUfbN/HG99F+KbKxSr8zvkF
FvJ5gRjqJjEX+Ed6KUBYIYm/OXOu6ylMGBdJElNQSgTJ88iwWeKVn1Ohq5MBVD6q
mUz67R5XU8yaO/IIwAn2e0E0FLdJOr8m4T5BFMWXGT2hZbKLaIbK2QUrhwI34qVg
4wsFR2n6zaQBWLcyihBhYrTMzcionvFL7riH+NZflrTB55BH1b1S5y0jnpumvRHY
fwOGGzJ7oKbUF1R39piBCfOk3x1I25uBMFmslpMhKFnRmwM+tz5BORjSQ0Vtx4X+
cX5GJswMslaJ0UMCAwEAAaMhMB8wHQYDVR0OBBYEFN1yU7xGDyNQB+9+BUofztoO
bXMHMA0GCSqGSIb3DQEBCwUAA4IBAQBFq5Rjkz9+83J8XKwXfMqmRG9rqliBs26H
/O6ONyHuHYuJ00MxAnMUOjXJFpPoGdItN/qTm3xHHYYEtFfO0TyJyJyviRixAdby
foUvIWCBbVhbsbttkus+Aseg+nej8oaIM8Win6sxhmvaQMvQSHgaINgIJuxi9/AN
6mtrFWsQ59DtGLlV9YYPzXRyMmJQYsNkDdWr/RlFozuHPa7CTlTjk8iCaGa6nOQK
eZhxvuhZG3iCoeuHVYAymJboZyTzSpNQnECh/KcUy7q15/aZvHd4UNq8gjWrYQVv
CCU1V6S4KwyQXIr9kEiiE1UvNGB/KLJOxza4+lAhsN/3KpzYojhF
-----END CERTIFICATE-----
```

上面的公钥经过下面的代码解析后，公钥里面的数据如下所示：

```golang
package main

import (
    "crypto/x509"
    "encoding/pem"
    "errors"
    "fmt"
    "io/ioutil"
)

const (
    Certificate   = "CERTIFICATE"
    ECPrivateKey  = "EC PRIVATE KEY"
    RSAPrivateKey = "RSA PRIVATE KEY"
)

func decodeCertificatePEM(crtb []byte) (*x509.Certificate, []byte, error) {
    block, crtb := pem.Decode(crtb)
    if block == nil {
        return nil, crtb, errors.New("invalid PEM certificate")
    }
    if block.Type != Certificate {
        return nil, nil, nil
    }
    c, err := x509.ParseCertificate(block.Bytes)
    return c, crtb, err
}

func main() {
    var (
        bts  []byte
        cert *x509.Certificate
        err  error
    )
    bts, _ = ioutil.ReadFile("ca.pem")
    cert, bts, err = decodeCertificatePEM(bts)
    if err != nil {
        panic(err)
    }
    fmt.Println(*cert)
    fmt.Println(string(bts))
}
```

公钥解析后的数据如下：

```shell
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1917295702 (0x72479c56)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=CN, ST=guangdong, L=shenzhen, O=tencent, OU=tencent, CN=sso.tencent.com
        Validity
            Not Before: Nov 30 02:07:16 2015 GMT
            Not After : Feb 28 02:07:16 2016 GMT
        Subject: C=CN, ST=guangdong, L=shenzhen, O=tencent, OU=tencent, CN=sso.tencent.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:9b:ef:37:a0:14:7a:ad:35:74:59:3e:e3:bc:51:
                    17:c3:81:23:22:82:c5:05:0f:bd:a1:91:24:92:9b:
                    87:51:79:f2:39:b2:2d:25:1f:6c:df:c7:1b:df:45:
                    f8:a6:ca:c5:2a:fc:ce:f9:05:16:f2:79:81:18:ea:
                    26:31:17:f8:47:7a:29:40:58:21:89:bf:39:73:ae:
                    eb:29:4c:18:17:49:12:53:50:4a:04:c9:f3:c8:b0:
                    59:e2:95:9f:53:a1:ab:93:01:54:3e:aa:99:4c:fa:
                    ed:1e:57:53:cc:9a:3b:f2:08:c0:09:f6:7b:41:34:
                    14:b7:49:3a:bf:26:e1:3e:41:14:c5:97:19:3d:a1:
                    65:b2:8b:68:86:ca:d9:05:2b:87:02:37:e2:a5:60:
                    e3:0b:05:47:69:fa:cd:a4:01:58:b7:32:8a:10:61:
                    62:b4:cc:cd:c8:a8:9e:f1:4b:ee:b8:87:f8:d6:5f:
                    96:b4:c1:e7:90:47:d5:bd:52:e7:2d:23:9e:9b:a6:
                    bd:11:d8:7f:03:86:1b:32:7b:a0:a6:d4:17:54:77:
                    f6:98:81:09:f3:a4:df:1d:48:db:9b:81:30:59:ac:
                    96:93:21:28:59:d1:9b:03:3e:b7:3e:41:39:18:d2:
                    43:45:6d:c7:85:fe:71:7e:46:26:cc:0c:b2:56:89:
                    d1:43
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                DD:72:53:BC:46:0F:23:50:07:EF:7E:05:4A:1F:CE:DA:0E:6D:73:07
    Signature Algorithm: sha256WithRSAEncryption
         45:ab:94:63:93:3f:7e:f3:72:7c:5c:ac:17:7c:ca:a6:44:6f:
         6b:aa:58:81:b3:6e:87:fc:ee:8e:37:21:ee:1d:8b:89:d3:43:
         31:02:73:14:3a:35:c9:16:93:e8:19:d2:2d:37:fa:93:9b:7c:
         47:1d:86:04:b4:57:ce:d1:3c:89:c8:9c:af:89:18:b1:01:d6:
         f2:7e:85:2f:21:60:81:6d:58:5b:b1:bb:6d:92:eb:3e:02:c7:
         a0:fa:77:a3:f2:86:88:33:c5:a2:9f:ab:31:86:6b:da:40:cb:
         d0:48:78:1a:20:d8:08:26:ec:62:f7:f0:0d:ea:6b:6b:15:6b:
         10:e7:d0:ed:18:b9:55:f5:86:0f:cd:74:72:32:62:50:62:c3:
         64:0d:d5:ab:fd:19:45:a3:3b:87:3d:ae:c2:4e:54:e3:93:c8:
         82:68:66:ba:9c:e4:0a:79:98:71:be:e8:59:1b:78:82:a1:eb:
         87:55:80:32:98:96:e8:67:24:f3:4a:93:50:9c:40:a1:fc:a7:
         14:cb:ba:b5:e7:f6:99:bc:77:78:50:da:bc:82:35:ab:61:05:
         6f:08:25:35:57:a4:b8:2b:0c:90:5c:8a:fd:90:48:a2:13:55:
         2f:34:60:7f:28:b2:4e:c7:36:b8:fa:50:21:b0:df:f7:2a:9c:
         d8:a2:38:45
```

我们可以看到sso.tencent.com机构颁发的这个公钥，实际上已经过期了，表示这个证书已经失效。

## 证书协议

### x509标准

[X.509是密码学里公钥证书的格式标准](https://zh.wikipedia.org/wiki/X.509)

证书组成结构标准用ASN.1（一种标准的语言）来进行描述. X.509 v3 数字证书结构如下：

```x509v3
证书
  版本号
  序列号
  签名算法
  颁发者
  证书有效期
    此日期前无效
    此日期后无效
  主题
  主题公钥信息
    公钥算法
    主题公钥
  颁发者唯一身份信息（可选项）
  主题唯一身份信息（可选项）
  扩展信息（可选项）
  ...
证书签名算法
数字签名
```

将上面x509 v3版本协议数据，通过ASN1序列化为比特流，也就是序列化后的字符串数据。最后再在数据上标识该字符串数据的数据类型, 比如：CERTIFICATE、RSA PRIVATE KEY、EC PRIVATE KEY或者PRIVATE KEY等等类型

最终的PEM协议文件内容，标准格式如下所示:

```shell
1. CERTIFICATE类型
-----BEGIN CERTIFICATE-----
......
-----END CERTIFICATE-----

2. RSA PRIVATE KEY类型
-----BEGIN RSA PRIVATE KEY-----
......
-----END RSA PRIVATE KEY-----

3. EC PRIVATE KEY类型
-----BEGIN EC PRIVATE KEY-----
.....
-----END EC PRIVATE KEY-----

4. PRIVATE KEY类型
-----BEGIN PRIVATE KEY-----
......
-----END PRIVATE KEY-----
```

我们可以看到最终的协议证书文件的标准格式，如下：

```shell
-----BEGIN TYPE-----
......
-----END TYPE-----
```

那么我们在创建或者解析证书时，就是按照这个流程，进行封装或者解析。


## CA根证书和Issuer证书的解析

dapr sentry内容非常多，但是具体也就做了一件事情，就是dapr runtime sidecar服务需要使用grpc服务的，为它颁布公钥和证书链.

服务提供者：dapr sentry

请求协议：grpc

默认端口：50001

路由：通过**/dapr.proto.sentry.v1.CA/SignCertificate**路由提供CSR请求服务

目的：颁布证书公钥文件

### CA机构

因为dapr sentry是一个内部集群的私有CA机构，只为内部的grpc提供证书公钥文件签署服务，首先需要创建一个CA机构，它主要做以下几件事情：

1. 为自己创建根公钥和私钥证书；
2. 再创建一个颁布者issuer公钥和私钥证书;
3. 最后再为通过grpc 50001端口请求的CSR，来颁布公钥证书文件。

再dapr sentry服务中，存在一个CA接口，它的定义如下所示：

```golang
type CertificateAuthority interface {
    // 加载或者创建根级公钥和私钥证书
    LoadOrStoreTrustBundle() error
    // 获取CA证书Bundle
    GetCACertBundle() TrustRootBundler
    // 为CSR请求，签署公钥证书文件
    SignCSR(csrPem []byte, subject string, identity *identity.Bundle, ttl time.Duration, isCA bool) (*SignedCertificate, error)
    // 验证CSR请求是否合法
    ValidateCSR(csr *x509.CertificateRequest) error
}
```

dapr sentry服务提供的默认CA机构，是自己内部私有的。但是该架构也预留了通过component-contrib公共接口提供的第三方CA机构服务。接下来，先讲解私有CA机构的实现

```golang
// defaultCA dapr sentry服务提供的私有CA机构，它作为为CSR签署公钥证书文件
type defaultCA struct {
    // bundle 是指CA根证书和中间证书机构链
    bundle     *trustRootBundle
    // dapr sentry服务启动需要的参数，包括根证书路径、中间证书路径, 服务端口、证书有效期等
    config     config.SentryConfig
    issuerLock *sync.RWMutex
}

// NewCertificateAuthority 创建一个CA对象，目前只支持defaultCA，也就是自身实现的私有CA机构
func NewCertificateAuthority(config config.SentryConfig) (CertificateAuthority, error)   {
    // Load future external CAs from components-contrib.
    switch config.CAStore {
    default:
        return &defaultCA{
            config:     config,
            issuerLock: &sync.RWMutex{},
        }, nil
    }
}

// LoadOrStoreTrustBundle 根据dapr sentry加载证书或者创建CA机构证书
func (c *defaultCA) LoadOrStoreTrustBundle() error {
    // 加载或者创建CA机构和中间机构的各个证书
    bundle, err := c.validateAndBuildTrustBundle()
    if err != nil {
        return err
    }

    c.bundle = bundle
    return nil
}

// validateAndBuildTrustBundle 验证CA机构证书或者创建CA机构证书
func (c *defaultCA) validateAndBuildTrustBundle() (*trustRootBundle, error) {
    var issuerCreds *certs.Credentials
    var rootCertBytes []byte
    var issuerCertBytes []byte

    // 校验k8s或者hosted来判断是否应该创建TrustBundle
    // 也就是CA根证书和Issuer证书
    if !shouldCreateCerts(c.config) {
        // 进来，表示证书已经存在, 则从defaultCA对象的SentryConfig配置中获取路径
        // 并进行证书解析，构建成trustRootBundle证书链

        // 检测根证书路径是否存在，不存在.
        // 一般情况下，如果dapr sentry跑在k8s中，且dapr-trust-bundle secret对象已存在，这个RootCertPath路径可能不存在
        // 如果是运行在host上，走入到这个逻辑，RootCertPath是一定存在的。
        //
        // detectCertificates在30s内，每秒检测一次，查看RootCertPath根证书文件是否存在
        // 30s内如果检测都不存在，则返回异常
        err := detectCertificates(c.config.RootCertPath)
        if err != nil {
            return nil, err
        }

        // 从磁盘上读取三个证书路径：
        // 1. CA根证书文件数据；(公钥)
        // 2. 证书颁布者证书文件数据；(公钥)
        // 3. 证书颁布者密钥文件数据；
        // 这三个数据存储在证书链中, 协议格式：{RootCA: 根证书公钥, Cert: 证书颁布者公钥, Key： 证书颁布者私钥}
        certChain, err := credentials.LoadFromDisk(c.config.RootCertPath, c.config.IssuerCertPath, c.config.IssuerKeyPath)
        if err != nil {
            return nil, errors.Wrap(err, "error loading cert chain from disk")
        }

        // 根据证书颁布者的公钥证书和私钥数据流，解析成Credentials={PrivateKey, x509.Certificate}
        issuerCreds, err = certs.PEMCredentialsFromFiles(certChain.Cert, certChain.Key)
        if err != nil {
            return nil, errors.Wrap(err, "error reading PEM credentials")
        }

        // 把解析到的CA根证书和证书颁布者的公钥传入，并返回
        rootCertBytes = certChain.RootCA
		    issuerCertBytes = certChain.Cert
    } else {
        // 否则，需要dapr sentry服务在初始化阶段去创建CA根证书、证书颁布者的公钥证书和私钥
        log.Info("root and issuer certs not found: generating self signed CA")
        var err error
        issuerCreds, rootCertBytes, issuerCertBytes, err = c.generateRootAndIssuerCerts()
        if err != nil {
            return nil, errors.Wrap(err, "error generating trust root bundle")
        }

        log.Info("self signed certs generated and persisted successfully")
    }

    // load trust anchors
    trustAnchors, err := certs.CertPoolFromPEM(rootCertBytes)
    if err != nil {
        return nil, errors.Wrap(err, "error parsing cert pool for trust anchors")
    }

    return &trustRootBundle{
        issuerCreds:   issuerCreds,
        trustAnchors:  trustAnchors,
        trustDomain:   c.config.TrustDomain,
        rootCertPem:   rootCertBytes,
        issuerCertPem: issuerCertBytes,
    }, nil
}

// shouldCreateCerts 校验证书是否存在
func shouldCreateCerts(conf config.SentryConfig) bool {
    // 检查如果dapr sentry是以k8s方式启动运行，则校验k8s secret是否存在dapr-trust-bundle实例
    // 比如：
    /*
    root:~$ kubectl describe secret dapr-trust-bundle -n dapr-system
            Name:         dapr-trust-bundle
            Namespace:    dapr-system
            Labels:       <none>
            Annotations:  <none>

            Type:  Opaque

            Data
            ====
            ca.crt:      704 bytes
            issuer.crt:  676 bytes
            issuer.key:  227 bytes
    */
    exists, err := certs.CredentialsExist(conf)
    if err != nil {
        log.Errorf("error chcecking if credetials exist: %s", err)
    }
    // 如果存在表示不用再创建根证书和issuer证书
    if exists {
        return false
    }

    // 否则，校验CA根证书和issuer证书路径是否存在
    //
    // 如果不存在，则返回true，表示需要创建证书
    if _, err = os.Stat(conf.RootCertPath); os.IsNotExist(err) {
        return true
    }
    b, err := ioutil.ReadFile(conf.IssuerCertPath)
    if err != nil {
        return true
    }
    return len(b) == 0
}

// PEMCredentialsFromFiles 根据公钥和私钥文件数据流，构建成Credentials={PrivateKey, x509.Certificate}对象
func PEMCredentialsFromFiles(certPem, keyPem []byte) (*Credentials, error) {
    // 先解析私钥数据流, 也就是解析下面这个x509 v3标准协议格式
    /*
       -----BEGIN TYPE-----
       ......
       -----END TYPE-----
    */
    // 这里我们只解析所有证书文件中的第一个x509 v3标准协议数据
    // 并返回自定义的PrivateKey={Type, 加密数据}
    pk, err := DecodePEMKey(keyPem)
    if err != nil {
        return nil, err
    }

    // 再解析公钥数据流, 解码成x509.Certificate列表
    crts, err := DecodePEMCertificates(certPem)
    if err != nil {
        return nil, err
    }

    // 如果公钥文件数据流中没有有效证书，则直接返回
    if len(crts) == 0 {
        return nil, errors.New("no certificates found")
    }

    // 校验私钥和公钥是否匹配
    // 匹配方式：私钥信息中的公钥是否和公钥信息相同
    match := matchCertificateAndKey(pk, crts[0])
    if !match {
        return nil, errors.New("error validating credentials: public and private key pa  ir do not match")
    }

    // 如果匹配， 则表示传入的私钥和公钥证书是一对
    // 并返回自定义证书的私钥和公钥数据流中的第一个公钥证书
    creds := &Credentials{
        PrivateKey:  pk,
        Certificate: crts[0],
    }

    return creds, nil
}

// DecodePEMKey 解析私钥数据流，反序列化为PrivateKey={Type: 私钥类型, key: 表示具体协议对象}
//
// 目前支持两种私钥加密算法格式：
/*
RSA：由 RSA 公司发明，是一个支持变长密钥的公共密钥算法，需要加密的文件块的长度也是可变的；
ECC（Elliptic Curves Cryptography）：椭圆曲线密码编码学。

ECC相比RSA优势：
抗攻击性强
CPU 占用少
内容使用少
网络消耗低
加密速度快
*/
func DecodePEMKey(key []byte) (*PrivateKey, error) {
    // 把密钥数据流解析成pem.Block={Type, Headers, Bytes};
    // 比如：TYPE: RSA PRIVATE KEY, Headers: nil, Bytes则是加密后的密钥, 也就是BEGIN和END之间的数据
    // 下文我们可以看看具体的解析
    // 因为一个证书文件中，可能存在多个BEGIN和END对，所以第二个参数是解析一对BEGIN和END后，剩余的BEGIN和END还没有解析
    block, _ := pem.Decode(key)
    if block == nil {
        return nil, errors.New("key is not PEM encoded")
    }
    // 我们这里只支持两种加密方式，且默认为ECC加密方式
    // ECC和RSA
    switch block.Type {
    case ECPrivateKey:
        // 如果加密的Type是EC PRIVATE KEY, 则通过x509包中的EC密钥解析方法获取ecdsa.PrivateKey对象
        // 注意，每个密钥中都带有公钥信息, 但是公钥信息不带密钥信息
        k, err := x509.ParseECPrivateKey(block.Bytes)
        if err != nil {
            return nil, err
        }
        // 返回自定义的PrivateKey对象
        return &PrivateKey{Type: ECPrivateKey, Key: k}, nil
    case RSAPrivateKey:
        // 如果加密的Type是RSA PRIVATE KEY，则通过x509包中的RSA密钥解析方法获取rsa.PrivateKey对象
        // 注意，这里的RSA反序列化方式为PCKS1
        k, err := x509.ParsePKCS1PrivateKey(block.Bytes)
        if err != nil {
            return nil, err
        }
        // 返回自定义的PrivateKey对象
        return &PrivateKey{Type: RSAPrivateKey, Key: k}, nil
    default:
        return nil, errors.Errorf("unsupported block type %s", block.Type)
    }
}

// DecodePEMCertificates 根据传入的数据流解析公钥证书列表
func DecodePEMCertificates(crtb []byte) ([]*x509.Certificate, error) {
    certs := []*x509.Certificate{}
    // 因为一个证书文件中可能存在多个x509 v3标准协议数据
    // 所以这里需要一个个解析, 并存储到x509.Ceritificate证书列表中
    for len(crtb) > 0 {
        var err error
        var cert *x509.Certificate

        // 根据传入尚未解析的证书数据流，进行x509 v3标准协议数据解析
        // 并返回一个证书和剩余未解析的数据流
        cert, crtb, err = decodeCertificatePEM(crtb)
        if err != nil {
            return nil, err
        }
        // 如果cert不为空，表示解析到了一个x509 v3证书
        if cert != nil {
            // it's a cert, add to pool
            certs = append(certs, cert)
        }
    }
    return certs, nil
}

// decodeCertificatePEM 最终也是采用x509 v3标准协议进行解码
func decodeCertificatePEM(crtb []byte) (*x509.Certificate, []byte, error) {
    // 解码x509 v3标准协议
    block, crtb := pem.Decode(crtb)
    if block == nil {
        return nil, crtb, errors.New("invalid PEM certificate")
    }
    // 如果x509 v3标准协议数据的类型不是CERTIFICATE类型， 则表示该协议不是公钥证书，直接返回全部nil
    if block.Type != Certificate {
        return nil, nil, nil
    }
    // 然后通过asn1解析公钥数据，并反序列化为x509.Certificate协议数据
    c, err := x509.ParseCertificate(block.Bytes)
    return c, crtb, err
}

// matchCertificateAndKey 对私钥和公钥进行匹配，看看是不是一对
func matchCertificateAndKey(pk *PrivateKey, cert *x509.Certificate) bool {
    // 首先对私钥的加密类型进行断言
    switch pk.Type {
    case ECPrivateKey:
        // 如果加密类型为EC PRIVATE KEY, 则直接断言为ecdsa.PrivateKey
        key := pk.Key.(*ecdsa.PrivateKey)
        pub, ok := cert.PublicKey.(*ecdsa.PublicKey)
        // 并比较公钥信息是否相同
        return ok && pub.X.Cmp(key.X) == 0 && pub.Y.Cmp(key.Y) == 0
    case RSAPrivateKey:
        // 如果加密类型为RSA PRIVATE KEY，则直接断言为rsa.PublicKey
        key := pk.Key.(*rsa.PrivateKey)
        pub, ok := cert.PublicKey.(*rsa.PublicKey)
        // 并比较公钥信息是否相同
        return ok && pub.N.Cmp(key.N) == 0 && pub.E == key.E
    default:
        return false
    }
}
```

## CA根证书和Issuer证书的创建

在前面了解到，如果传入的CA根证书和Issuer证书文件都不存在，或者k8s dapr-system命名空间下dapr-trust-bundle secret不存在，则需要创建CA根证书和Issuer证书，包括公钥和私钥，并存储到secret资源方式k8s dapr-system命名空间下，或者存储到指定的证书文件中。

```golang
// generateRootAndIssuerCerts 创建CA根证书和Issuer证书
/*
   这里分三个步骤：
   1. 创建CA根证书的私钥和公钥;
   2. 创建Issuer颁布者的私钥和公钥.
   3. 把创建好的CA根证书和Issuer颁布者的私钥和公钥写入到SentryConfig或者k8s dapr-system命名空间下类型为token，名为dapr-trust-bundle的资源对象中存储.
*/
func (c *defaultCA) generateRootAndIssuerCerts() (*certs.Credentials, []byte, []byte, error) {
    // 1. 创建CA根证书的私钥和公钥
    // 目前dapr对加密密钥的方式都是采用EC PRIVATE KEY, 创建随机密钥
    rootKey, err := certs.GenerateECPrivateKey()
    if err != nil {
        return nil, nil, nil, err
    }
    // 然后根据Certificate Signed Request证书签名请求，创建一个x509 v3的证书请求x509.Certifiate
    rootCsr, err := csr.GenerateRootCertCSR(caOrg, caCommonName, &rootKey.PublicKey, selfSignedRootCertLifetime)
    if err != nil {
        return nil, nil, nil, err
    }

    // 根据CSR证书签名请求，创建经过base64和asn1序列化的公钥数据流
    //
    // 这里注意后面四个参数：template, parent *Certificate, pub, priv interface{}
    // 对于前面两个：如果template与parent相同，表示自签名请求；否则，是对template作为公钥对CSR进行签名
    // 对于后面两个：public表示被签发的公钥（CSR请求的公钥）；priv表示受签发的私钥(表示是哪个给你签)
    rootCertBytes, err := x509.CreateCertificate(rand.Reader, rootCsr, rootCsr, &rootKey.PublicKey, rootKey)
    if err != nil {
        return nil, nil, nil, err
    }

    // 把加密后的密钥数据流和加密类型，编码成x509 v3标准协议数据.
    /*
      -----BEGIN TYPE-----
      ......
      -----END TYPE-----
    */
    rootCertPem := pem.EncodeToMemory(&pem.Block{Type: certs.Certificate, Bytes: rootCertBytes})

    // 把前面通过CSR请求获得的公钥数据流，解码成x509.Certificate协议数据对象
    rootCert, err := x509.ParseCertificate(rootCertBytes)
    if err != nil {
        return nil, nil, nil, err
    }

    // 2. 创建Issuer颁布者的公钥和私钥
    issuerKey, err := certs.GenerateECPrivateKey()
    if err != nil {
        return nil, nil, nil, err
    }

    // 根据CSR请求，创建一个x509 v3的证书请求x509.Certifiate
    issuerCsr, err := csr.GenerateIssuerCertCSR(caCommonName, &issuerKey.PublicKey, selfSignedRootCertLifetime)
    if err != nil {
        return nil, nil, nil, err
    }

    // 然后根据x509 v3的证书请求，创建经过base64和asn1序列化的公钥数据流
    //
    // 这里再注意后面四个参数：
    // 前面两个：分别是issuerCsr和rootCert。这次是对Issuer证书颁布者的CSR进行签名， 后者表示根证书
    // 后面两个：分别是Issuer公钥和CA根证书。前者是被签署的公钥，后者是帮CSR签署公钥证书的私钥
    issuerCertBytes, err := x509.CreateCertificate(rand.Reader, issuerCsr, rootCert, &issuerKey.PublicKey, rootKey)
    if err != nil {
        return nil, nil, nil, err
    }

    // 把签发的公钥数据流，构建成x509 v3标准协议数据流
    issuerCertPem := pem.EncodeToMemory(&pem.Block{Type: certs.Certificate, Bytes: issuerCertBytes})

    // 把未加密的私钥通过EC PRIVATE KEY方式加密成数据流
    encodedKey, err := x509.MarshalECPrivateKey(issuerKey)
    if err != nil {
        return nil, nil, nil, err
    }
    // 然后最后把加密后的数据流构建成x509 v3标准协议数据
    issuerKeyPem := pem.EncodeToMemory(&pem.Block{Type: certs.ECPrivateKey, Bytes: encodedKey})

    // 并把公钥的数据流，解码为x509.Certificate对象
    issuerCert, err := x509.ParseCertificate(issuerCertBytes)
    if err != nil {
        return nil, nil, nil, err
    }

    // 3. 把CA根证书和Issuer的公钥和私钥写入到k8s token或者文件中
    err = certs.StoreCredentials(c.config, rootCertPem, issuerCertPem, issuerKeyPem)
    if err != nil {
        return nil, nil, nil, err
    }

    // 最后，我们就得到了Issuer证书颁布者的私钥证书和公钥证书
    // 以及封装成x509 v3的CA根公钥和Issuer证书颁布者的公钥
    return &certs.Credentials{
        PrivateKey: &certs.PrivateKey{
            Type: certs.ECPrivateKey,
            Key:  issuerKey,
        },
        Certificate: issuerCert,
   }, rootCertPem, issuerCertPem, nil
}

// 我们可以从创建CA根证书和Issuer证书颁布者的证书和私钥，可以看到创建完成后，会进行证书的存储操作
//
// 两种存储情况：1. 以k8s seret资源的方式存储到k8s dapr-system命名空间下;
// 2. 存储到SentryConfig配置指定的证书路径下。
func StoreCredentials(conf config.SentryConfig, rootCertPem, issuerCertPem, issuerKeyPem []byte) error {
    // 判断当前服务是否运行k8s环境中
    if config.IsKubernetesHosted() {
        return storeKubernetes(rootCertPem, issuerCertPem, issuerKeyPem)
    }
    // 否则就是运行在host上
    return storeSelfhosted(rootCertPem, issuerCertPem, issuerKeyPem, conf.RootCertPath, conf.IssuerCertPath, conf.IssuerKeyPath)
}

// storeKubernetes 把CA根证书、Issuer证书颁布者的公钥和私钥，以secret资源的方式存储到k8s中
func storeKubernetes(rootCertPem, issuerCertPem, issuerCertKey []byte) error {
    // 从.kube/config加载配置，获取k8s client
    kubeClient, err := kubernetes.GetClient()
    if err != nil {
        return err
    }

    // 构建成secret资源对象，并通过k8s api server client写入到etcd集群中
    // 这个secret前面已经了解过了。
    // NAMESPACE=dapr-system; Name=dapr-trust-bundle
    namespace := getNamespace()
    secret := &v1.Secret{
        Data: map[string][]byte{
            credentials.RootCertFilename:   rootCertPem,
            credentials.IssuerCertFilename: issuerCertPem,
            credentials.IssuerKeyFilename:  issuerCertKey,
        },
        ObjectMeta: metav1.ObjectMeta{
            Name:      KubeScrtName,
            Namespace: namespace,
        },
        Type: v1.SecretTypeOpaque,
    }

    // 通过k8s api server client写入到etcd集群中
    _, err = kubeClient.CoreV1().Secrets(namespace).Update(context.TODO(), secret, metav1.UpdateOptions{})
    if err != nil {
        return errors.Wrap(err, "failed saving secret to kubernetes")
    }
    return nil
}

func getNamespace() string {
    // 这个Namespace为dapr-system
    namespace := os.Getenv("NAMESPACE")
    if namespace == "" {
        // 默认命名空间为default。
        namespace = defaultSecretNamespace
    }
    return namespace
}

// storeSelfhosted 把各个证书写入到SentryConfig配置指定的文件中
func storeSelfhosted(rootCertPem, issuerCertPem, issuerKeyPem []byte, rootCertPath, issuerCertPath, issuerKeyPath string) error {
    err := ioutil.WriteFile(rootCertPath, rootCertPem, 0644)
    if err != nil {
        return errors.Wrapf(err, "failed saving file to %s", rootCertPath)
    }

    err = ioutil.WriteFile(issuerCertPath, issuerCertPem, 0644)
    if err != nil {
        return errors.Wrapf(err, "failed saving file to %s", issuerCertPath)
    }

    err = ioutil.WriteFile(issuerKeyPath, issuerKeyPem, 0644)
    if err != nil {
        return errors.Wrapf(err, "failed saving file to %s", issuerKeyPath)
    }
    return nil
}
```

通过CA根证书的公钥和私钥的创建，以及Issuer证书颁布者的公钥和私钥的创建过程，我们可以了解整个证书的创建过程，以后再也不会对这些概念既熟悉又陌生了。

## 介绍x509协议的解析

x509 v3标准协议的解析过程，是针对下面的结构解析出pem.Block结构

```shell
-----BEGIN TYPE-----
......
-----END TYPE-----
```

那么所有证书文件中的数据流，都是先解析该结构，然后再针对证书的内容进行解码。

比如：
1. 公钥通过ASN.1进行解析；
2. 私钥则通过RSA或者ECC加密方式进行解析.

针对x509 v3标准的解码过程，encoding/pem.Decode的解码方法，获取到pem.Block和剩余数据流.

```golang
var pemStart = []byte("\n-----BEGIN ")
var pemEnd = []byte("\n-----END ")
var pemEndOfLine = []byte("-----")

// Decode 是encoding/pem包的Decode方法，用于解析x509 v3标准协议
func Decode(data []byte) (p *Block, rest []byte) {
    rest = data
    // 校验数据流是否是以-----BEGIN 开头
    // 如果是，则剔除这个头，目的是只获取有效数据，包括类型和类型数据
    if bytes.HasPrefix(data, pemStart[1:]) {
        rest = rest[len(pemStart)-1 : len(data)]
    // 如果不是，则寻找以\n-----BEGIN 开头的第一条数据流, 并剔除前面的无效数据
    } else if i := bytes.Index(data, pemStart); i >= 0 {
        rest = rest[i+len(pemStart) : len(data)]
    // 否则，表示数据流中没有符合x509 v3标准的协议数据, 直接返回
    } else {
        return nil, data
    }

    // 获取TYPE------行数据
    typeLine, rest := getLine(rest)
    // 校验typeLine是否以------结尾，如果不是，表示不符合x509 v3标准
    if !bytes.HasSuffix(typeLine, pemEndOfLine) {
        return decodeError(data, rest)
    }
    // 所以第一行typeLine剔除-----后，剩下的就只有类型值了：
    // 比如：-----BEGIN EC PRIVATE KEY-----
    // 则解析后的typeLine值为EC PRIVATE KEY
    typeLine = typeLine[0 : len(typeLine)-len(pemEndOfLine)]

    // 然后再解析密钥类型数据
    p = &Block{
        Headers: make(map[string]string),
        Type:    string(typeLine),
    }
    ......
    // 最后获取的第一个x509 v3标准协议数据后，通过base64解析数据后，把有效数据存储到pem.Block的Bytes数据流中
    // 并返回pem.Block和rest，分别表示第一个x509 v3标准协议数据对象和剩余数据流
    base64Data := removeSpacesAndTabs(rest[:endIndex])
    p.Bytes = make([]byte, base64.StdEncoding.DecodedLen(len(base64Data)))
    n, err := base64.StdEncoding.Decode(p.Bytes, base64Data)
    if err != nil {
        return decodeError(data, rest)
    }
    p.Bytes = p.Bytes[:n]
    ......
    return
}
```


## 总结

通过本文，我们应该掌握了下面知识点：
1. 对通信的数据加解密，包括：公钥、私钥和CA根证书，有了比较清楚的理解；
2. 对证书和私钥的签发过程，包括x509 v3标准协议，及其编解码、ASN.1、以及密钥加密算法的加解密（EC PRIVATE KEY和RSA PRIVATE KEY），有了清楚的理解；
3. 对dapr sentry服务的CA机构自签署、Issuer证书颁布者的公钥和私钥的签署，有了清楚的理解和掌握；
4. 对dapr sentry服务的CA根证书、Issuer证书和私钥的存储，包括k8s secret资源存储和host方式文件存储，也有了很好的账号

其中重点在于第一点和第二点

1. [加密算法RSA与ECC对比，以及Android、java中使用](https://blog.csdn.net/Jocerly/article/details/83339826)
