## 前言

我们不再对 [ja-netfilter](https://gitee.com/ja-netfilter) 和它对应的插件 [plugin-power](https://gitee.com/ja-netfilter/plugin-power) 的原理进行过多的介绍，请阅读参考资料进行了解。总的思路是我们需要生成一份自己的证书，然后使用自己的证书生成激活码，最后借助 plugin-power 在 JetBrains 校验激活码时替换校验结果，让 JetBrains 以为是自己的证书生成的激活码。

## Java 版本

在 Java 版本中我们使用的是 JDK 21，同时要借助一个第三方库 [Bouncy Castle](https://www.bouncycastle.org/) 的能力来完成证书和激活码的生成。因此我们首先要在项目中引入 Bouncy Castle

```xml
<dependency>
    <groupId>org.bouncycastle</groupId>
    <artifactId>bcpkix-jdk18on</artifactId>
    <version>1.77</version>
</dependency>
```

自定义 X509 证书，把证书和私钥的内容使用 PEM 格式分别保存到 `cert.pem` 和 `key.pem` 文件中

```java
import org.bouncycastle.asn1.x500.X500Name;
import org.bouncycastle.cert.X509CertificateHolder;
import org.bouncycastle.cert.X509v3CertificateBuilder;
import org.bouncycastle.cert.jcajce.JcaX509CertificateConverter;
import org.bouncycastle.cert.jcajce.JcaX509v3CertificateBuilder;
import org.bouncycastle.openssl.jcajce.JcaPEMWriter;
import org.bouncycastle.operator.ContentSigner;
import org.bouncycastle.operator.jcajce.JcaContentSignerBuilder;

import java.io.FileWriter;
import java.math.BigInteger;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.PrivateKey;
import java.security.Security;
import java.security.cert.X509Certificate;
import java.time.Duration;
import java.time.Instant;
import java.util.Date;
import java.util.Random;

public class CertificateGenerator {
    static {
        Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
    }

    public static void main(String[] args) throws Exception {
        // 生成密钥对
        KeyPair keyPair = generateKeyPair();

        // 创建 X.509 证书
        X509Certificate certificate = generateCertificate(keyPair);

        // 保存私钥
        savePrivateKey(keyPair.getPrivate());

        // 保存证书
        saveCertificate(certificate);
    }

    private static KeyPair generateKeyPair() throws Exception {
        KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
        keyPairGenerator.initialize(4096);
        return keyPairGenerator.generateKeyPair();
    }

    private static X509Certificate generateCertificate(KeyPair keyPair) throws Exception {
        Instant now = Instant.now();
        Instant notBefore = now.minus(Duration.ofDays(1));
        Instant notAfter = now.plus(Duration.ofDays(3650));

        X500Name issuer = new X500Name("CN=JetProfile CA");
        X500Name subject = new X500Name("CN=A Comma");

        X509v3CertificateBuilder certificateBuilder = new JcaX509v3CertificateBuilder(
                issuer,
                new BigInteger(64, new Random()),
                Date.from(notBefore),
                Date.from(notAfter),
                subject,
                keyPair.getPublic()
        );

        ContentSigner contentSigner = new JcaContentSignerBuilder("SHA256WithRSA").build(keyPair.getPrivate());
        X509CertificateHolder certificateHolder = certificateBuilder.build(contentSigner);
        return new JcaX509CertificateConverter().getCertificate(certificateHolder);
    }

    private static void savePrivateKey(PrivateKey privateKey) throws Exception {
        try (JcaPEMWriter writer = new JcaPEMWriter(new FileWriter("key.pem"))) {
            // 使用 PEM 格式保存私钥
            writer.writeObject(privateKey);
        }
    }

    private static void saveCertificate(X509Certificate certificate) throws Exception {
        try (JcaPEMWriter writer = new JcaPEMWriter(new FileWriter("cert.pem"))) {
            // 使用 PEM 格式保存证书
            writer.writeObject(certificate);
        }
    }
}
```

读取刚才生成的 cert.pem 和 key.pem 文件重建自定义证书和私钥，然后生成激活码和规则。把证书的生成和激活码的生成分开是为了重用证书生成一些列 JetBrains 软件的激活码，但是规则只有一条。

```java
import org.bouncycastle.openssl.PEMKeyPair;
import org.bouncycastle.openssl.PEMParser;
import org.bouncycastle.openssl.jcajce.JcaPEMKeyConverter;

import java.io.ByteArrayInputStream;
import java.io.FileInputStream;
import java.io.FileReader;
import java.io.IOException;
import java.math.BigInteger;
import java.nio.charset.StandardCharsets;
import java.security.InvalidKeyException;
import java.security.KeyPair;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.security.PrivateKey;
import java.security.Security;
import java.security.Signature;
import java.security.SignatureException;
import java.security.cert.CertificateEncodingException;
import java.security.cert.CertificateException;
import java.security.cert.CertificateFactory;
import java.security.cert.X509Certificate;
import java.security.interfaces.RSAPublicKey;
import java.util.Arrays;
import java.util.Base64;

public class ActivationCodeGenerator {
    static {
        Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
    }

    // 根证书来自 https://github.com/JetBrains/marketplace-makemecoffee-plugin/blob/master/src/main/java/com/company/license/CheckLicense.java
    private final static String ROOT_CERTIFICATE = """
            -----BEGIN CERTIFICATE-----
            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
            -----END CERTIFICATE-----
            """;

    public static void main(String[] args) throws CertificateException, IOException, NoSuchAlgorithmException, SignatureException, InvalidKeyException {
        CertificateFactory certificateFactory = CertificateFactory.getInstance("X.509");
        X509Certificate userCertificate = (X509Certificate) certificateFactory.generateCertificate(new FileInputStream("cert.pem"));
        X509Certificate rootCertificate = (X509Certificate) certificateFactory.generateCertificate(new ByteArrayInputStream(ROOT_CERTIFICATE.getBytes(StandardCharsets.UTF_8)));

        String activationCode = generateActivationCode(userCertificate);
        System.out.println("激活码：" + activationCode);

        String powerRule = generatePowerRule(rootCertificate, userCertificate);
        System.out.println("规则："+ powerRule);
    }

    private static String generateActivationCode(X509Certificate userCertificate) throws IOException, NoSuchAlgorithmException, InvalidKeyException, SignatureException, CertificateEncodingException {
        // 该 License 用于激活 IntelliJ IDEA
        String licensePart = """
                {"licenseId":"29VRVXKXEQ","licenseeName":"逗号","assigneeName":"","assigneeEmail":"","licenseRestriction":"","checkConcurrentUse":false,"products":[{"code":"II","fallbackDate":"2026-09-14","paidUpTo":"2026-09-14","extended":false},{"code":"PCWMP","fallbackDate":"2026-09-14","paidUpTo":"2026-09-14","extended":true},{"code":"PSI","fallbackDate":"2026-09-14","paidUpTo":"2026-09-14","extended":true},{"code":"PDB","fallbackDate":"2026-09-14","paidUpTo":"2026-09-14","extended":true}],"metadata":"0120230914PSAX000005","hash":"TRIAL:1649058719","gracePeriodDays":7,"autoProlongated":false,"isAutoProlongated":false}
                """;
        String licenseId = "29VRVXKXEQ";
        byte[] licensePartBytes = licensePart.getBytes(StandardCharsets.UTF_8);
        String licensePartBase64 = Base64.getEncoder().encodeToString(licensePartBytes);

        PrivateKey privateKey = getPrivateKey();
        Signature signature = Signature.getInstance("SHA1withRSA");
        signature.initSign(privateKey);
        signature.update(licensePartBytes);
        byte[] signatureBytes = signature.sign();
        String signatureBase64 = Base64.getEncoder().encodeToString(signatureBytes);

        String certificateBase64 = Base64.getEncoder().encodeToString(userCertificate.getEncoded());

        return licenseId + "-" + licensePartBase64 + "-" + signatureBase64 + "-" + certificateBase64;
    }

    private static String generatePowerRule(X509Certificate rootCertificate, X509Certificate userCertificate) throws CertificateEncodingException, NoSuchAlgorithmException {
        RSAPublicKey publicKey = (RSAPublicKey) rootCertificate.getPublicKey();

        BigInteger z = publicKey.getModulus();
        BigInteger x = new BigInteger(1, userCertificate.getSignature());
        BigInteger y = new BigInteger("65537");
        BigInteger r = calculateR(userCertificate);

        return "EQUAL," + x + "," + y + "," + z + "->" + r;
    }

    /**
     * r：对 DER 编码的证书信息（即来自该证书的 TBSCertificate）进行 SHA-256 摘要计算，计算的结果转换为 ASN1 格式数据，ASN1格式数据再进行填充得到的。
     * <p>参考 <a href="https://linux.do/t/topic/469">Idea agent 激活原理</a>
     */
    private static BigInteger calculateR(X509Certificate userCertificate) throws NoSuchAlgorithmException, CertificateEncodingException {
        int modBits = ((RSAPublicKey) userCertificate.getPublicKey()).getModulus().bitLength();
        int emLen = (modBits + 7) / 8;

        // 使用 SHA-256 进行摘要计算
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte[] tbsCertificateBytes = userCertificate.getTBSCertificate();
        byte[] digestBytes = digest.digest(tbsCertificateBytes);

        // DER-encoded，digestAlgo 的值为 SHA-256 算法的对象标识符的 16 进制表示
        byte[] digestAlgo = new byte[]{0x30, 0x31, 0x30, 0x0d, 0x06, 0x09, 0x60, (byte) 0x86, 0x48, 0x01, (byte) 0x65, 0x03, 0x04, 0x02, 0x01, 0x05, 0x00, 0x04, 0x20};
        byte[] digestInfo = new byte[digestAlgo.length + digestBytes.length];
        System.arraycopy(digestAlgo, 0, digestInfo, 0, digestAlgo.length);
        System.arraycopy(digestBytes, 0, digestInfo, digestAlgo.length, digestBytes.length);

        // 补齐
        byte[] ps = new byte[emLen - digestInfo.length - 3];
        Arrays.fill(ps, (byte) 0xFF);

        // 构造最终结果
        byte[] encoded = new byte[emLen];
        encoded[0] = 0x00;
        encoded[1] = 0x01;
        System.arraycopy(ps, 0, encoded, 2, ps.length);
        encoded[ps.length + 2] = 0x00;
        System.arraycopy(digestInfo, 0, encoded, ps.length + 3, digestInfo.length);

        return new BigInteger(encoded);
    }

    private static PrivateKey getPrivateKey() throws IOException {
        PEMParser parser = new PEMParser(new FileReader("key.pem"));
        JcaPEMKeyConverter converter = new JcaPEMKeyConverter().setProvider("BC");
        Object object = parser.readObject();
        KeyPair keyPair = converter.getKeyPair((PEMKeyPair) object);
        return keyPair.getPrivate();
    }
}
```

代码基本都是自解释的，只在必要的地方添加了适当的注释。

## Python 版本

在 Python 版本中我们使用的 Python 版本是 `3.8.2`，同时使用第三方库 [Cryptography](https://cryptography.io/) 来完成证书和激活码的生成。因此，我们需要安装一下 Cryptography

```shell
pip install cryptograph
```

生成自签名的证书和私钥 `create_certificate.py`，并使用 PEM 格式保存证书和私钥的内容。生成自签名证书主要参考 [Creating a self-signed certificate](https://cryptography.io/en/latest/x509/tutorial/#creating-a-self-signed-certificate)

```python
import datetime

from cryptography import x509
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.x509.oid import NameOID

private_key = rsa.generate_private_key(public_exponent=65537, key_size=4096, backend=default_backend())

builder = x509.CertificateBuilder()
builder = builder.subject_name(x509.Name([x509.NameAttribute(NameOID.COMMON_NAME, 'A Comma')]))
builder = builder.issuer_name(x509.Name([x509.NameAttribute(NameOID.COMMON_NAME, 'JetProfile CA')]))
builder = builder.public_key(private_key.public_key())
builder = builder.serial_number(x509.random_serial_number())
builder = builder.not_valid_before(datetime.datetime.now(datetime.timezone.utc) - datetime.timedelta(days=1))
builder = builder.not_valid_after(datetime.datetime.now(datetime.timezone.utc) + datetime.timedelta(days=3650))

certificate = builder.sign(private_key=private_key, algorithm=hashes.SHA256(), backend=default_backend())

with open("key.pem", "wb") as f:
    f.write(private_key.private_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PrivateFormat.TraditionalOpenSSL,
        encryption_algorithm=serialization.NoEncryption()
    ))

with open("cert.pem","wb") as f:
    f.write(certificate.public_bytes(encoding=serialization.Encoding.PEM))
```

读取刚才生成的 cert.pem 和 key.pem 文件重建自定义证书和私钥，然后生成激活码和规则 `generate_activation_code.py`

```python
import base64

from cryptography import x509
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding

with open("key.pem", "rb") as f:
    private_key = serialization.load_pem_private_key(data=f.read(), password=None, backend=default_backend())

with open("cert.pem", "rb") as f:
    certificate = x509.load_pem_x509_certificate(data=f.read())

# 激活 DataGrip 的 License
licensePart = '{"licenseId":"VPQ9LWBJ0Z","licenseeName":"逗号","assigneeName":"","assigneeEmail":"","licenseRestriction":"","checkConcurrentUse":false,"products":[{"code":"PSI","fallbackDate":"2025-08-01","paidUpTo":"2025-08-01","extended":true},{"code":"PDB","fallbackDate":"2025-08-01","paidUpTo":"2025-08-01","extended":true},{"code":"DB","fallbackDate":"2025-08-01","paidUpTo":"2025-08-01","extended":false},{"code":"PWS","fallbackDate":"2025-08-01","paidUpTo":"2025-08-01","extended":true}],"metadata":"0120220902PSAN000005","hash":"TRIAL:-22891717","gracePeriodDays":7,"autoProlongated":false,"isAutoProlongated":false}'
licenseId = 'VPQ9LWBJ0Z'
licensePartBytes = licensePart.encode('utf-8')
licensePartBase64 = base64.b64encode(licensePartBytes).decode('utf-8')

signatureBytes = private_key.sign(data=licensePart.encode('utf-8'), padding=padding.PKCS1v15(), algorithm=hashes.SHA1())
signatureBase64 = base64.b64encode(signatureBytes).decode('utf-8')

certificateBytes = certificate.public_bytes(serialization.Encoding.DER)
certificateBase64 = base64.b64encode(certificateBytes).decode('utf-8')

activation_code = licenseId + '-' + licensePartBase64 + '-' + signatureBase64 + '-' + certificateBase64

print(f'activation_code:\n{activation_code}')

x = int.from_bytes(certificate.signature, 'big')
fakeResult = pow(x, 65537, private_key.public_key().public_numbers().n)

# EQUAL,x,y,z->fakeResult，y 和 z 是固定的，z 也可以从 JetBrains 根证书获取
power_rule_template = 'EQUAL,{x},65537,860106576952879101192782278876319243486072481962999610484027161162448933268423045647258145695082284265933019120714643752088997312766689988016808929265129401027490891810902278465065056686129972085119605237470899952751915070244375173428976413406363879128531449407795115913715863867259163957682164040613505040314747660800424242248055421184038777878268502955477482203711835548014501087778959157112423823275878824729132393281517778742463067583320091009916141454657614089600126948087954465055321987012989937065785013284988096504657892738536613208311013047138019418152103262155848541574327484510025594166239784429845180875774012229784878903603491426732347994359380330103328705981064044872334790365894924494923595382470094461546336020961505275530597716457288511366082299255537762891238136381924520749228412559219346777184174219999640906007205260040707839706131662149325151230558316068068139406816080119906833578907759960298749494098180107991752250725928647349597506532778539709852254478061194098069801549845163358315116260915270480057699929968468068015735162890213859113563672040630687357054902747438421559817252127187138838514773245413540030800888215961904267348727206110582505606182944023582459006406137831940959195566364811905585377246353->{fakeResult}'
power_rule = power_rule_template.format(x=x, fakeResult=fakeResult)

print(f'power_rule:\n{power_rule}')
```

`fakeResult` 的计算没有采用参考资料 1 或者参考资料 2 中的方式，而是采用了参考资料 5 的方式，避免去操作 ASN1 格式。

## 激活码内容获取方式

在 Java 版本或者 Python 版本中我们直接给出了 `licensePart` 的内容。我们可以打开 [https://3.jetbra.in/](https://3.jetbra.in/) 网站，随便进入一个可用的网站，下载 jetbra.zip 并按照文档配置。然后选择你需要的产品的激活码，比如 IntelliJ IDEA

```text
29VRVXKXEQ-eyJsaWNlbnNlSWQiOiIyOVZSVlhLWEVRIiwibGljZW5zZWVOYW1lIjoiZ3VyZ2xlcyB0dW1ibGVzIiwiYXNzaWduZWVOYW1lIjoiIiwiYXNzaWduZWVFbWFpbCI6IiIsImxpY2Vuc2VSZXN0cmljdGlvbiI6IiIsImNoZWNrQ29uY3VycmVudFVzZSI6ZmFsc2UsInByb2R1Y3RzIjpbeyJjb2RlIjoiSUkiLCJmYWxsYmFja0RhdGUiOiIyMDI2LTA5LTE0IiwicGFpZFVwVG8iOiIyMDI2LTA5LTE0IiwiZXh0ZW5kZWQiOmZhbHNlfSx7ImNvZGUiOiJQQ1dNUCIsImZhbGxiYWNrRGF0ZSI6IjIwMjYtMDktMTQiLCJwYWlkVXBUbyI6IjIwMjYtMDktMTQiLCJleHRlbmRlZCI6dHJ1ZX0seyJjb2RlIjoiUFNJIiwiZmFsbGJhY2tEYXRlIjoiMjAyNi0wOS0xNCIsInBhaWRVcFRvIjoiMjAyNi0wOS0xNCIsImV4dGVuZGVkIjp0cnVlfSx7ImNvZGUiOiJQREIiLCJmYWxsYmFja0RhdGUiOiIyMDI2LTA5LTE0IiwicGFpZFVwVG8iOiIyMDI2LTA5LTE0IiwiZXh0ZW5kZWQiOnRydWV9XSwibWV0YWRhdGEiOiIwMTIwMjMwOTE0UFNBWDAwMDAwNSIsImhhc2giOiJUUklBTDoxNjQ5MDU4NzE5IiwiZ3JhY2VQZXJpb2REYXlzIjo3LCJhdXRvUHJvbG9uZ2F0ZWQiOmZhbHNlLCJpc0F1dG9Qcm9sb25nYXRlZCI6ZmFsc2V9-YKRuMTrLQcfyWisYF1q6RhCN+Ub13VOCayGGc6tklGA97oxRM1HCIR0oI5yfTjL7UQYDbNMokT0U0ZQ2obYaUx+MMf7+3FfUYp5dYzP7G9YrEehrGWQ4O8ENrDLDAClB8o8jud9cafW9WTx9hDNd9j2FfjwSaRibClwGBRdO5fSkWlKGhx4tV0K9IyotNYDQzT1QCDRWSxHYGqfDAQI2k+ZAqzNEHValupSM3TKw813kFGKIQndMfw57B6uMzgN6PvuuLpBlghdO3imrgKYj0Q59JYbuXRUpHhPnNLY1XmewdlfcJkvTiRwueCPMNEW/CQEh8X/Als92WCr2H3uFRA==-MIIETDCCAjSgAwIBAgIBDTANBgkqhkiG9w0BAQsFADAYMRYwFAYDVQQDDA1KZXRQcm9maWxlIENBMB4XDTIwMTAxOTA5MDU1M1oXDTIyMTAyMTA5MDU1M1owHzEdMBsGA1UEAwwUcHJvZDJ5LWZyb20tMjAyMDEwMTkwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCUlaUFc1wf+CfY9wzFWEL2euKQ5nswqb57V8QZG7d7RoR6rwYUIXseTOAFq210oMEe++LCjzKDuqwDfsyhgDNTgZBPAaC4vUU2oy+XR+Fq8nBixWIsH668HeOnRK6RRhsr0rJzRB95aZ3EAPzBuQ2qPaNGm17pAX0Rd6MPRgjp75IWwI9eA6aMEdPQEVN7uyOtM5zSsjoj79Lbu1fjShOnQZuJcsV8tqnayeFkNzv2LTOlofU/Tbx502Ro073gGjoeRzNvrynAP03pL486P3KCAyiNPhDs2z8/COMrxRlZW5mfzo0xsK0dQGNH3UoG/9RVwHG4eS8LFpMTR9oetHZBAgMBAAGjgZkwgZYwCQYDVR0TBAIwADAdBgNVHQ4EFgQUJNoRIpb1hUHAk0foMSNM9MCEAv8wSAYDVR0jBEEwP4AUo562SGdCEjZBvW3gubSgUouX8bOhHKQaMBgxFjAUBgNVBAMMDUpldFByb2ZpbGUgQ0GCCQDSbLGDsoN54TATBgNVHSUEDDAKBggrBgEFBQcDATALBgNVHQ8EBAMCBaAwDQYJKoZIhvcNAQELBQADggIBABKaDfYJk51mtYwUFK8xqhiZaYPd30TlmCmSAaGJ0eBpvkVeqA2jGYhAQRqFiAlFC63JKvWvRZO1iRuWCEfUMkdqQ9VQPXziE/BlsOIgrL6RlJfuFcEZ8TK3syIfIGQZNCxYhLLUuet2HE6LJYPQ5c0jH4kDooRpcVZ4rBxNwddpctUO2te9UU5/FjhioZQsPvd92qOTsV+8Cyl2fvNhNKD1Uu9ff5AkVIQn4JU23ozdB/R5oUlebwaTE6WZNBs+TA/qPj+5/we9NH71WRB0hqUoLI2AKKyiPw++FtN4Su1vsdDlrAzDj9ILjpjJKA1ImuVcG329/WTYIKysZ1CWK3zATg9BeCUPAV1pQy8ToXOq+RSYen6winZ2OO93eyHv2Iw5kbn1dqfBw1BuTE29V2FJKicJSu8iEOpfoafwJISXmz1wnnWL3V/0NxTulfWsXugOoLfv0ZIBP1xH9kmf22jjQ2JiHhQZP7ZDsreRrOeIQ/c4yR8IQvMLfC0WKQqrHu5ZzXTH4NO3CwGWSlTY74kE91zXB5mwWAx1jig+UXYc2w4RkVhy0//lOmVya/PEepuuTTI4+UJwC7qbVlh5zfhj8oTNUXgN0AOc+Q0/WFPl1aw5VV/VrO8FCoB15lFVlpKaQ1Yh+DVU8ke+rt9Th0BCHXe0uZOEmH0nOnH/0onD
```

然后使用 `-` 分割后获取第二部分内容

```text
eyJsaWNlbnNlSWQiOiIyOVZSVlhLWEVRIiwibGljZW5zZWVOYW1lIjoiZ3VyZ2xlcyB0dW1ibGVzIiwiYXNzaWduZWVOYW1lIjoiIiwiYXNzaWduZWVFbWFpbCI6IiIsImxpY2Vuc2VSZXN0cmljdGlvbiI6IiIsImNoZWNrQ29uY3VycmVudFVzZSI6ZmFsc2UsInByb2R1Y3RzIjpbeyJjb2RlIjoiSUkiLCJmYWxsYmFja0RhdGUiOiIyMDI2LTA5LTE0IiwicGFpZFVwVG8iOiIyMDI2LTA5LTE0IiwiZXh0ZW5kZWQiOmZhbHNlfSx7ImNvZGUiOiJQQ1dNUCIsImZhbGxiYWNrRGF0ZSI6IjIwMjYtMDktMTQiLCJwYWlkVXBUbyI6IjIwMjYtMDktMTQiLCJleHRlbmRlZCI6dHJ1ZX0seyJjb2RlIjoiUFNJIiwiZmFsbGJhY2tEYXRlIjoiMjAyNi0wOS0xNCIsInBhaWRVcFRvIjoiMjAyNi0wOS0xNCIsImV4dGVuZGVkIjp0cnVlfSx7ImNvZGUiOiJQREIiLCJmYWxsYmFja0RhdGUiOiIyMDI2LTA5LTE0IiwicGFpZFVwVG8iOiIyMDI2LTA5LTE0IiwiZXh0ZW5kZWQiOnRydWV9XSwibWV0YWRhdGEiOiIwMTIwMjMwOTE0UFNBWDAwMDAwNSIsImhhc2giOiJUUklBTDoxNjQ5MDU4NzE5IiwiZ3JhY2VQZXJpb2REYXlzIjo3LCJhdXRvUHJvbG9uZ2F0ZWQiOmZhbHNlLCJpc0F1dG9Qcm9sb25nYXRlZCI6ZmFsc2V9
```

它是 BASE64 格式的，随便找一个能解码 BASE64 的网站解码

```json
{"licenseId":"29VRVXKXEQ","licenseeName":"gurgles tumbles","assigneeName":"","assigneeEmail":"","licenseRestriction":"","checkConcurrentUse":false,"products":[{"code":"II","fallbackDate":"2026-09-14","paidUpTo":"2026-09-14","extended":false},{"code":"PCWMP","fallbackDate":"2026-09-14","paidUpTo":"2026-09-14","extended":true},{"code":"PSI","fallbackDate":"2026-09-14","paidUpTo":"2026-09-14","extended":true},{"code":"PDB","fallbackDate":"2026-09-14","paidUpTo":"2026-09-14","extended":true}],"metadata":"0120230914PSAX000005","hash":"TRIAL:1649058719","gracePeriodDays":7,"autoProlongated":false,"isAutoProlongated":false}
```

这样我们就得到了 `licensePart` 的内容。可以根据需要修改 `licenseeName`、`paidUpTo` 字段的值。

## 参考资料

1. [Idea agent 激活原理](https://linux.do/t/topic/469)
2. [ja-netfilter power插件原理](https://www.xuzhengtong.com/2022/07/25/ja-netfilter/ja-netfilter-plugins-power/)，[Github](https://github.com/intxzt/intxzt.github.io/blob/master/2022/07/25/ja-netfilter/ja-netfilter-plugins-power/index.html)
3. [[原创]注册码证书验证过程](https://bbs.kanxue.com/thread-271052.htm)
4. [[讨论] ja-netfilter 代理框架](https://bbs.kanxue.com/thread-271578.htm)
5. [wmymz / jetbrains_tool](https://gitee.com/qy6174/jetbrains_tool)
