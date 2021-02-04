本文继续进行dapr sentry源码阅读，本次是对dapr sentry服务启动，以及为外部请求签发证书的过程

```golang
dapr sentry提供grpc 50001端口服务
路由：/dapr.proto.sentry.v1.CA/SignCertificate
```

## dapr sentry服务启动

如果dapr sentry服务部署在k8s中，则该服务默认在dapr-system命名空间下，且该服务deployment的名称为dapr-sentry, 下面这个CAServer接口，用于启动grpc 50001端口服务，且带有一个关闭服务的API

```golang
type CAServer interface {
    Run(port int, trustBundle ca.TrustRootBundler) error
    Shutdown()
}
```

在dapr sentry源码中，server为CAServer的默认实现。如下所示：

```golang
// server为CAServer接口的实现，并提供SignedCertificate服务，用于给请求的CSR签署证书
type server struct {
    certificate *tls.Certificate
    certAuth    ca.CertificateAuthority
    srv         *grpc.Server
    validator   identity.Validator
}

// NewCAServer创建一个CAServer的实例对象
// 参数CertificateAuthority， 表示传入defaultCA对象，用于获取或者创建CA根证书，以及Issuer证书颁布者的证书和密钥
// 以及为CSR签署证书和验证CSR请求的合法性
func NewCAServer(ca ca.CertificateAuthority, validator identity.Validator) CAServer {
    return &server{
        certAuth:  ca,
        validator: validator,
    }
}

// Run 启动dapr sentry服务，对外CSR(Certificate Signed Request)提供签署证书的服务
// TrustRootBunder接口，用于获取CA根证书和Issuer证书颁布者的证书和私钥
func (s *server) Run(port int, trustBundler ca.TrustRootBundler) error {
    // dapr sentry服务的提供的grpc默认端口是50001
    addr := fmt.Sprintf(":%v", port)
    lis, err := net.Listen("tcp", addr)
    if err != nil {
        return errors.Wrapf(err, "could not listen on %s", addr)
    }

    tlsOpt := s.tlsServerOption(trustBundler)
    s.srv = grpc.NewServer(tlsOpt)
    sentryv1pb.RegisterCAServer(s.srv, s)

    if err := s.srv.Serve(lis); err != nil {
        return errors.Wrap(err, "grpc serve error")
    }
    return nil
}

// TrustRootBundler 在上文中我们已经通过defaultCA实例，创建好CA根证书和Issuer证书颁布者的公钥和私钥
// 并把这些数据存储到trustRootBundle对象中，该对象实现了TrustRootBundler接口
type TrustRootBundler interface {
    // 获取Issuer证书颁布者的公钥封装好的x509 v3标准协议数据包
    GetIssuerCertPem() []byte
    // 获取CA根证书的公钥封装好的x509 v3标准协议数据包
    GetRootCertPem() []byte
    // 获取Issuer证书颁布者的自己公钥证书的有效期
    GetIssuerCertExpiry() time.Time
    // 获取CA根证书的公钥列表，构建成证书池
    GetTrustAnchors() *x509.CertPool
    // 获取dapr sentry服务的可信域，也就是来源
    GetTrustDomain() string
}

// tlsServerOption 根据传入的TrustRootBundle构建一个创建grpc服务且提供tls证书的的参数 grpc.ServerOption
func (s *server) tlsServerOption(trustBundler ca.TrustRootBundler) grpc.ServerOption {
    // 通过传入的TrustRootBundle，获取CA根证书的x509.CertPool公钥池
    cp := trustBundler.GetTrustAnchors()

    // nolint:gosec
    // 创建tls配置实例, 用于为grpc添加tls安全证书
    config := &tls.Config{
        // CA根证书公钥池
        ClientCAs: cp,
        // 要求客户端访问dapr sentry grpc服务，必须带证书且验证证书
        ClientAuth: tls.RequireAndVerifyClientCert,
        // GetCertificate 这个地方不太理解
        // 看代码好像是，获取grpc服务需要的TLS证书
        GetCertificate: func(*tls.ClientHelloInfo) (*tls.Certificate, error) {
            // 如果dapr sentry初始化时，其certificate成员为nil，或者证书过期
            //
            // 注意这里的证书过期，是指马上过期。范围15分钟内就过期，则需要动态生成一个新的证书
            if s.certificate == nil || needsRefresh(s.certificate, serverCertExpiryBuffer) {
                cert, err := s.getServerCertificate()
                if err != nil {
                    monitoring.ServerCertIssueFailed("server_cert")
                    log.Error(err)
                    return nil, errors.Wrap(err, "failed to get TLS server certificate"  )
                }
                s.certificate = cert
            }
            return s.certificate, nil
        },
    }
    // 最后把TLS证书作为grpc的ServerOptions参数，则grpc服务启动则使用了TLS安全证书，且任何客户端访问dapr sentry都必须带上证书进行校验。否则拒绝建立连接
    return grpc.Creds(credentials.NewTLS(config))
}

// getServerCertificate 为dapr sentry提供grpc服务生成一个TLS证书
func (s *server) getServerCertificate() (*tls.Certificate, error) {
    // 生成一个x509 v3标准协议数据的公钥证书和私钥, 且该公钥还只是一个无效，没有经过任何CA机构认证的公钥
    // 相当于是一个CSR请求公钥证书, 而私钥不需要认证
    csrPem, pkPem, err := csr.GenerateCSR("", false)
    if err != nil {
        return nil, err
    }

    // 获取Issuer证书颁布者的过期时间
    now := time.Now().UTC()
    issuerExp := s.certAuth.GetCACertBundle().GetIssuerCertExpiry()
    // 表示为CSR请求签署的证书有效期
    // 这里注意：Issuer证书颁布者自身证书的有效期为1年，那么该Issuer证书颁布者为其他CSR签署的证书有效期肯定是小于1年
    serverCertTTL := issuerExp.Sub(now)

    // 需要使用CA机构对刚刚，自己创建的CSR证书，进行签署公钥和认证
    resp, err := s.certAuth.SignCSR(csrPem, s.certAuth.GetCACertBundle().GetTrustDomain(), nil, serverCertTTL, false)
    if err != nil {
        return nil, err
    }

    // 把CA机构为CSR证书签署好的x509 v3标准协议数据流，也就是公钥证书数据流
    // 以及CA根证书和Issuer证书颁布者公钥，全部写入到certPem, 并返回给dapr sentry服务，作为自身的公钥列表。
    certPem := resp.CertPEM
    certPem = append(certPem, s.certAuth.GetCACertBundle().GetIssuerCertPem()...)
    certPem = append(certPem, s.certAuth.GetCACertBundle().GetRootCertPem()...)

    // 最后再把签发好的公钥证书和私钥，构建成TLS证书，并返回给grpc的ServerOptions参数中
    cert, err := tls.X509KeyPair(certPem, pkPem)
    if err != nil {
        return nil, err
    }

    return &cert, nil
}

// GenerateCSR 根据传入参数创建一个CSR请求，申请公钥和私钥，并返回x509 v3标准协议数据的证书和私钥
// 这个公钥证书其实是一个CERTIFICATE REQUEST类型的x509 v3标准协议数据，它没有经过CA认证。
// 所以其实是一个CSR证书, 需要使用TrustRootBundle的SignCSR API对CSR公钥请求进行签署
func GenerateCSR(org string, pkcs8 bool) ([]byte, []byte, error) {
    // 生成一个随机的私钥, 密钥生成的方式EC Private Key
    key, err := certs.GenerateECPrivateKey()
    if err != nil {
        return nil, nil, errors.Wrap(err, "unable to generate private keys")
    }

    // 根据传入参数，也就是生成公钥所需要的CSR请求
    templ, err := genCSRTemplate(org)
    if err != nil {
        return nil, nil, errors.Wrap(err, "error generating csr template")
    }

    // 根据CSR(Certificate Signed Request)请求，来生成一个公钥CSR请求证书.
    csrBytes, err := x509.CreateCertificateRequest(rand.Reader, templ, crypto.PrivateKey(key))
    if err != nil {
        return nil, nil, errors.Wrap(err, "failed to create CSR")
    }

    // 然后把CSR请求公钥证书和私钥证书，分别构建成x509 v3标准协议的数据流
    crtPem, keyPem, err := encode(true, csrBytes, key, pkcs8)
    return crtPem, keyPem, err
}

// genCSRTemplate 构建成一个CSR签署证书的请求， org表示证书的subject。也就是哪个机构帮忙签署的
func genCSRTemplate(org string) (*x509.CertificateRequest, error) {
    return &x509.CertificateRequest{
        Subject: pkix.Name{
            Organization: []string{org},
        },
    }, nil
}

// encode 把公钥和私钥证书，分别编码成x509 v3标准协议数据流
func encode(csr bool, csrOrCert []byte, privKey *ecdsa.PrivateKey, pkcs8 bool) ([]byte, []byte, error) {
    // CERTIFICATE 表示公钥类型为CERTIFICATE
    // CERTIFICATE REQUEST 表示公钥类型为CSR类型
    // encodeMsgCert: CERTIFICATE
    encodeMsg := encodeMsgCert
    if csr {
        // encodeMsgCSR: CERTIFICATE REQUEST
        encodeMsg = encodeMsgCSR
    }
    // 把公钥数据流编码为x509 v3标准协议数据流
    csrOrCertPem := pem.EncodeToMemory(&pem.Block{Type: encodeMsg, Bytes: csrOrCert})

    var encodedKey, privPem []byte
    var err error

    // PKCS#8 表示需要把私钥通过PKCS#8方式，进行编码， 然后再构建成x509 v3标准协议，编码类型PRIVATE KEY
    // 因为PKCS#8特定的编码方式，会进行case查找，它可以对rsa.PrivateKey(RSA PRIVATE KEY), ecdsa.PrivateKey(ECC PRIVATE KEY)，ed25519.PrivateKey等密钥进行判断，然后再根据各自加密方式进行解码
    if pkcs8 {
        if encodedKey, err = x509.MarshalPKCS8PrivateKey(privKey); err != nil {
            return nil, nil, err
        }
        // 那么这里x509 v3标准协议数据中的编码类型：PRIVATE KEY
        privPem = pem.EncodeToMemory(&pem.Block{Type: blockTypePrivateKey, Bytes: encodedKey})
    } else {
        // 如果不是PKCS#8封装EC编码处理，则直接使用EC编码方式，进行编码获取数据流
        encodedKey, err = x509.MarshalECPrivateKey(privKey)
        if err != nil {
            return nil, nil, err
        }
        // 然后再构建成x509 v3标准协议数据，编码协议类型：EC PRIVATE KEY
        privPem = pem.EncodeToMemory(&pem.Block{Type: blockTypeECPrivateKey, Bytes: encodedKey})
    }
    // 最后把创建好的x509 v3标准协议数据，包括公钥证书和私钥都返回
    return csrOrCertPem, privPem, nil
}

// SignCSR，也就是CA机构为编码好的x509 v3协议数据流CSR请求，签署和颁发公钥证书
func (c *defaultCA) SignCSR(csrPem []byte, subject string, identity *identity.Bundle, ttl time.Duration, isCA bool) (*SignedCertificate, error) {
    c.issuerLock.RLock()
    defer c.issuerLock.RUnlock()

    // 校验如果Issuer证书颁布者的有效期过短，则使用dapr sentry的SentryConfig中的证书有效期
    certLifetime := ttl
    if certLifetime.Seconds() < 0 {
        certLifetime = c.config.WorkloadCertTTL
    }

    // 并且允许一定的证书冗余有效期
    certLifetime += c.config.AllowedClockSkew

    // Issuer证书颁布者为CSR请求，签署和颁布证书
    signingCert := c.bundle.issuerCreds.Certificate
    signingKey := c.bundle.issuerCreds.PrivateKey

    // 解析使用x509 v3标准协议封装好的CSR请求，转换成x509 CertificateRequest协议数据
    cert, err := certs.ParsePemCSR(csrPem)
    if err != nil {
        return nil, errors.Wrap(err, "error parsing csr pem")
    }

    // 使用Issuer证书颁布者，为CSR请求签署证书
    crtb, err := csr.GenerateCSRCertificate(cert, subject, identity, signingCert, cert.PublicKey, signingKey.Key, certLifetime, isCA)
    if err != nil {
        return nil, errors.Wrap(err, "error signing csr")
    }

    // 把上一步获取的公钥，解析成x509 Certificate证书
    csrCert, err := x509.ParseCertificate(crtb)
    if err != nil {
        return nil, errors.Wrap(err, "error parsing cert")
    }

    // 然后再把Issuer证书颁布者签署的公钥数据流，构建成x509 v3标准协议数据流
    certPem := pem.EncodeToMemory(&pem.Block{
        Type:  certs.Certificate,
        Bytes: crtb,
    })

    // 把签署和构建好的证书和x509 v3标准协议数据流返回给上游
    return &SignedCertificate{
        Certificate: csrCert,
        CertPEM:     certPem,
    }, nil
}
```

从dapr sentry grpc 50001端口服务创建过程中，我们可以看到主要是有两个步骤：
1. 为grpc 50001端口服务添加TLS安全证书；
2. grpc服务注册，主要为第三方CSR请求，提供SignCertificate API服务，签署和颁布公钥证书

前面的源码分析，主要是是针对dapr sentry grpc 50001端口服务，创建TLS证书，包括几个步骤：
1. 创建一个随机的EC Private Key加密算法私钥;
2. 构建一个CSR请求参数，并创建CertificateRequest请求；
3. 再根据2得到的CertificateRequest请求，编码成x509 v3标准协议数据;
4. 再根据x509 v3标准协议封装好的CSR请求，让Issuer证书颁布者签署和颁发公钥证书；
5. 最后再把Issuer证书颁布者颁布的公钥证书和自己生成的随机私钥，构建成TLS证书并返回给grpc server作为TLS参数

这里面有个GenerateCSRCertificate方法没有源码分析，这个放到grpc server接收外部请求，并通过SignCertificate API处理时，进行详细分析，因为这个里面提及里一个[SPIFEE标准](https://github.com/spiffe/spiffe), 它用于微服务内部调用的安全性，防止内部服务之间的攻击。

SPIFEE标准在dapr runtime中使用，形如：**spiffe://trustDomain/ns/namespace/appID**

## dapr sentry服务签发证书

在dapr-system命名空间下的dapr sentry服务，为dapr runtime所有需要使用grpc协议的服务，提供CA公钥证书认证, 下面主要是对dapr sentry提供grpc 50001端口服务的SignCertificate API服务，进行详细分析

```golang
// SignCertificate 对所有dapr runtime容器提供CA证书签署和颁发
func (s *server) SignCertificate(ctx context.Context, req *sentryv1pb.SignCertificateRequest) (*sentryv1pb.SignCertificateResponse, error) {
    // 根据请求，获取CSR CertificateRequest请求数据流
    csrPem := req.GetCertificateSigningRequest()

    // 并把x509 v3标准协议数据封装的CertificateRequest，解析为CSR CertificateRequest实例对象
    csr, err := certs.ParsePemCSR(csrPem)
    if err != nil {
        err = errors.Wrap(err, "cannot parse certificate signing request pem")
        return nil, err
    }

    // 并通过TrustRootBundle来验证CertificateRequest请求参数的合法性
    // 目前主要是验证Subject
    err = s.certAuth.ValidateCSR(csr)
    if err != nil {
        err = errors.Wrap(err, "error validating csr")
        return nil, err
    }

    // 然后再验证id、token和namespace的合法性和正确性
    err = s.validator.Validate(req.GetId(), req.GetToken(), req.GetNamespace())
    if err != nil {
        err = errors.Wrap(err, "error validating requester identity")
        return nil, err
    }

    // 把CSR请求中的参数，构建成bundle对象, 相当于一个CSR请求签署公钥的身份证
    identity := identity.NewBundle(csr.Subject.CommonName, req.GetNamespace(), req.GetTrustDomain())
    // 为CSR请求签署和颁发公钥证书
    signed, err := s.certAuth.SignCSR(csrPem, csr.Subject.CommonName, identity, -1, false)
    if err != nil {
        err = errors.Wrap(err, "error signing csr")
        return nil, err
    }

    // 把为CSR请求颁发的公钥证书、以及CA根证书和Issuer证书颁布者的公钥证书，构建成一个证书列表数据流
    certPem := signed.CertPEM
    issuerCert := s.certAuth.GetCACertBundle().GetIssuerCertPem()
    rootCert := s.certAuth.GetCACertBundle().GetRootCertPem()

    certPem = append(certPem, issuerCert...)
    certPem = append(certPem, rootCert...)

    if len(certPem) == 0 {
        err = errors.New("insufficient data in certificate signing request, no certs signed")
        return nil, err
    }

    // 校验时间是否是合法的
    expiry := timestamppb.New(signed.Certificate.NotAfter)
    if err = expiry.CheckValid(); err != nil {
        return nil, errors.Wrap(err, "could not validate certificate validity")
    }

    // 返回公钥证书列表数据流和证书链，以及证书的有效期
    resp := &sentryv1pb.SignCertificateResponse{
        WorkloadCertificate:    certPem,
        TrustChainCertificates: [][]byte{issuerCert, rootCert},
        ValidUntil:             expiry,
    }

    return
}

// GenerateCSRCertificate 为x509 CertificateRequest请求签署和颁布一个公钥证书
func GenerateCSRCertificate(csr *x509.CertificateRequest, subject string, identityBundle *identity.Bundle, signingCert *x509.Certificate, publicKey interface{}, signingKey crypto.PrivateKey,
  ttl time.Duration, isCA bool) ([]byte, error) {
  // 创建一个x509.Certificate实例对象, 并填充数据到证书中
  cert, err := generateBaseCert(ttl, publicKey)
  if err != nil {
      return nil, errors.Wrap(err, "error generating csr certificate")
  }
  if isCA {
      cert.KeyUsage = x509.KeyUsageCertSign | x509.KeyUsageCRLSign
  } else {
      cert.KeyUsage = x509.KeyUsageDigitalSignature | x509.KeyUsageKeyEncipherment
      cert.ExtKeyUsage = append(cert.ExtKeyUsage, x509.ExtKeyUsageServerAuth, x509.ExtKeyUsageClientAuth)
  }

  if subject == "cluster.local" {
      cert.Subject = pkix.Name{
          CommonName: subject,
      }
      cert.DNSNames = []string{subject}
  }

  cert.Issuer = signingCert.Issuer
  cert.IsCA = isCA
  cert.IPAddresses = csr.IPAddresses
  cert.Extensions = csr.Extensions
  cert.BasicConstraintsValid = true
  cert.SignatureAlgorithm = csr.SignatureAlgorithm

  // 重点：如果是外部CSR请求，则需要对请求的服务进行保护，防止dapr runtime之间的恶意调用
  // 这里使用SPIFFE标准，相当于k8s service account。限定业务之间的保护调用。并把这个参数添加到证书的扩展字段中
  //
  // 证书中的扩展字段：X509v3 extensions
  if identityBundle != nil {
      // 创建一个spiffe id，ID协议结构：
      // spiffe://trustDomain/ns/namespace/appID
      // 如：spiffe://dc1/ns/default/demo
      //
      // 在k8s中
      // spiffe://cluster.local/ns/default/sa/default
      spiffeID, err := identity.CreateSPIFFEID(identityBundle.TrustDomain, identityBundle.Namespace, identityBundle.ID)
      if err != nil {
          return nil, errors.Wrap(err, "error generating spiffe id")
      }

      // 然后通过x509 v3标准协议中的扩展字段，进行SPIFFE标准数据填充
      // 但是具体如何使用，目前我还不太清楚，后面清楚了，我再补充进去
      // ::TODO
      rv := []asn1.RawValue{
          {
              Bytes: []byte(spiffeID),
              Class: asn1.ClassContextSpecific,
              Tag:   asn1.TagOID,
          },
          {
              // 比如dns: dapr-placement-server.dapr-system.svc.cluster.local
              Bytes: []byte(fmt.Sprintf("%s.%s.svc.cluster.local", subject,identityBundle.Namespace)),
              Class: asn1.ClassContextSpecific,
              Tag:   2,
          },
      }

      b, err := asn1.Marshal(rv)
      if err != nil {
          return nil, errors.Wrap(err, "failed to marshal asn1 raw value for spiffe id")
      }

      cert.ExtraExtensions = append(cert.ExtraExtensions, pkix.Extension{
          Id:       oidSubjectAlternativeName,
          Value:    b,
          Critical: true, // According to x509 and SPIFFE specs, a SubjAltName extension must be critical if subject name and DNS are not present.
      })
  }

  // 把x509.Certificate编码为加密数据流
  return x509.CreateCertificate(rand.Reader, cert, signingCert, publicKey, signingKey)
}
```

通过对外部grpc client访问dapr sentry服务，可以理解CSR请求证书签署和颁布公钥证书(数字签名)的整个过程，并且进行数字签名的过程中，还在证书带上了SPIFFE标准，用于保护内部微服务之间，也就是dapr runtime容器服务之间的访问保护，进行身份校验。

目前还不知道具体是如何在访问过程中进行证书扩展字段SPIFFE标准校验的, 后面理解后再补充**::TODO**

## dapr sentry grpc服务的封装

要把dapr sentry服务CA根证书和Issuer证书颁布者的证书和私钥创建过程，以及为dapr sentry grpc服务添加TLS证书的创建过程联系起来，则需要一个封装过程，这个是用sentry.CertificateAuthority接口来实现的。其默认实现类是sentry类

```golang
// CertificateAuthority 接口表示dapr sentry服务的运行和重启
type CertificateAuthority interface {
    Run(context.Context, config.SentryConfig, chan bool)
    Restart(ctx context.Context, conf config.SentryConfig)
}

// sentry表示CertificateAuthority接口的实现之一
type sentry struct {
    server    server.CAServer
    reloading bool
}

// NewSentryCA 创建一个CertificateAuthority接口实例sentry对象
func NewSentryCA() CertificateAuthority {
    return &sentry{}
}

// Run 启动dapr sentry服务，对于k8s 则为dapr-system命名空间下的dapr-sentry POD.
func (s *sentry) Run(ctx context.Context, conf config.SentryConfig, readyCh chan bool)   {
    // 创建CA接口实例，也就是defaultCA实例对象
    certAuth, err := ca.NewCertificateAuthority(conf)
    if err != nil {
        log.Fatalf("error getting certificate authority: %s", err)
    }
    log.Info("certificate authority loaded")

    // 加载或者创建TrustRootBundle对象，包括CA根证书、Issuer证书颁布者的证书和私钥
    err = certAuth.LoadOrStoreTrustBundle()
    if err != nil {
        log.Fatalf("error loading trust root bundle: %s", err)
    }
    log.Infof("trust root bundle loaded. issuer cert expiry: %s", certAuth.GetCACertBundle().GetIssuerCertExpiry().String())
    monitoring.IssuerCertExpiry(certAuth.GetCACertBundle().GetIssuerCertExpiry())

    // Create 创建Validator，用于校验CSR请求的身份identity。包括：ID、Namespace和token
    v, err := createValidator()
    if err != nil {
        log.Fatalf("error creating validator: %s", err)
    }
    log.Info("validator created")

    // 创建grpc 50001端口服务，并通过CA机构创建证书，并最后填充到grpc ServerOptions的TLS参数中
    s.server = server.NewCAServer(certAuth, v)

    go func() {
        <-ctx.Done()
        log.Info("sentry certificate authority is shutting down")
        s.server.Shutdown() // nolint: errcheck
    }()
    if readyCh != nil {
      readyCh <- true
      s.reloading = false
    }

    log.Infof("sentry certificate authority is running, protecting ya'll")
    err = s.server.Run(conf.Port, certAuth.GetCACertBundle())
    if err != nil {
        log.Fatalf("error starting gRPC server: %s", err)
    }
}

// createValidator 获取dapr k8s包创建的Validator验证实例, 用于校验identity的身份，包括ID、token和命名空间
// 如果是host，则不用任何校验
func createValidator() (identity.Validator, error) {
    if config.IsKubernetesHosted() {
        // we're in Kubernetes, create client and init a new serviceaccount token valid  ator
        kubeClient, err := k8s.GetClient()
        if err != nil {
            return nil, errors.Wrap(err, "failed to create kubernetes client")
        }
        return kubernetes.NewValidator(kubeClient), nil
    }
    return selfhosted.NewValidator(), nil
}

// Restart 停止grpc server服务，并重启端口服务
func (s *sentry) Restart(ctx context.Context, conf config.SentryConfig) {
    if s.reloading {
        return
    }
    s.reloading = true

    s.server.Shutdown()
    go s.Run(ctx, conf, nil)
}
```

## 总结

通过下篇dapr sentry服务的整个创建过程，以及为自身grpc服务提供TLS证书，和处理外部grpc client的CSR签署和颁布数字证书请求， 使得我们可以基本上了解CA机构和Issuer证书颁布者，以及如何为服务签署数字证书。
