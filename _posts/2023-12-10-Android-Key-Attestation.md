---
layout: post
title: "Android key attestation"
excerpt_separator: <!--more-->
date: 2023-12-10 00:00:00 +0800
author: xiangxiang
categories: security
tags: [attestation security]
---
PKI的东西总是让人很头疼

<!--more-->
* auto-gen TOC:
{:toc}

## 0x00 refs
- https://developer.android.com/privacy-and-security/security-key-attestation
- https://fidoalliance.org/fido-technotes-the-truth-about-attestation/
 
- https://developer.android.com/privacy-and-security/keystore
- https://source.android.com/docs/security/features/keystore

- https://android-developers.googleblog.com/2022/03/upgrading-android-attestation-remote.html
- https://xdaforums.com/t/spoof-locked-bootloader-bypass-tee-check.4586251/
- https://github.com/vvb2060/KeyAttestation
- https://github.com/chiteroman/PlayIntegrityFix

- https://jbp.io/2014/04/07/android-keystore-leak.html


## 0x01 Attestation
- what is attestation? It is a key pair that is burned into the device during manufacturing time that is specific to a device model
- FIDO: Please note that attestation is supposed to be unique to a device model, not an individual device.

- During registration, a new public key is created and signed by an attestation private key that was created with the device


- Attestation accomplishes two things
1. if an attacker intercepts a registration message with their own, they would not be able to swap out the new public key with their own since the attestation signature wouldn’t match; （简单的替换而非完整的证书链）
2. it allows the service to trust that it knows the provenance of the authenticator being used.

## 0x02 Generate Android Key Pair with Attestation 

### 2.1 Code
- https://developer.android.com/privacy-and-security/keystore#GeneratingANewPrivateKey
- https://source.android.com/docs/security/features/keystore/attestation#expandable-1


- 主要参考[keystore开发文档](https://developer.android.com/privacy-and-security/keystore)
- 首先确定[使用keychain还是Android Keystore provider](https://developer.android.com/privacy-and-security/keystore#WhichShouldIUse)
- 对于[公私密钥对用于签名的场景](https://developer.android.com/privacy-and-security/keystore#GeneratingANewPrivateKey)先生成公私密钥对然后调用`KeyStore.setKeyEntry()`更换私钥对应的证书为CA签发的证
- 后续可以[通过alias在keystore中获取密钥然后使用](https://developer.android.com/privacy-and-security/keystore#WorkingWithKeyStoreEntries)
- 在[调用密钥时可以加入用户认证](https://developer.android.com/privacy-and-security/keystore#UserAuthentication)
- 注意生成的cert3(根证书)与[Google公开的](https://developer.android.com/privacy-and-security/security-key-attestation#root_certificate)一致


```java

    /**
     * generates a key pair and requests an attestation.
     * <a href="https://source.android.com/docs/security/features/keystore/attestation#expandable-1">Google官方说明</a>
     */
    public static KeyPair generateKeyPairWithAttestation(String keyAlias, byte[] challenge)
            throws NoSuchAlgorithmException, NoSuchProviderException, InvalidAlgorithmParameterException {
        KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance(
                KeyProperties.KEY_ALGORITHM_EC, "AndroidKeyStore");


        keyPairGenerator.initialize(
                new KeyGenParameterSpec.Builder(keyAlias, KeyProperties.PURPOSE_SIGN)
                        // using the NIST P-256 curve
                        .setAlgorithmParameterSpec(new ECGenParameterSpec("secp256r1"))
                        .setDigests(KeyProperties.DIGEST_SHA256,
                                KeyProperties.DIGEST_SHA384,
                                KeyProperties.DIGEST_SHA512)
                        // Only permit the private key to be used if the user
                        // authenticated within the last five minutes.
                        .setUserAuthenticationRequired(true)
                        .setUserAuthenticationValidityDurationSeconds(5 * 60)
                        // Request an attestation with challenge "hello world".
                        .setAttestationChallenge(challenge)
                        .build());

        // Generate the key pair. This will result in calls to both generate_key() and
        // attest_key() at the keymaster2 HAL.
        return keyPairGenerator.generateKeyPair();
    }

    // 以下为调用逻辑
    // 实践中, challenge需要服务端生成且必须为nonce
    byte[] challenge =  Long.toString(new Date().getTime()).getBytes();
    String keyAlias = "xx-demo";

    KeyPair kp = AttestationKeyHelper.generateKeyPairWithAttestation(keyAlias, challenge);
    binding.keyPairWithAttestationGenerated.setText("生成公私密钥成功: " + new String(Base64.getEncoder().encode(kp.getPublic().getEncoded())));
    // 这里无法获取私钥并打印

    // Get the certificate chain
    KeyStore keyStore = KeyStore.getInstance("AndroidKeyStore");
    keyStore.load(null);
    Certificate[] certs = keyStore.getCertificateChain(keyAlias);
    // 经过验证, 相同keyAlias返回的证书链一致

    // certs[0] is the attestation certificate. certs[1] signs certs[0], etc.,
    // up to certs[certs.length - 1].

```

### 2.2 Pixel 3A with locked BL
- 在pixel 3A会生成如下的证书链
- cert0为attestation certificate, [attestation扩展字段的ASN.1 schema](https://source.android.com/docs/security/features/keystore/attestation#schema)
  

#### 2.2.1 cert 0
- 使用[https://lapo.it/asn1js/](https://lapo.it/asn1js/)解析cert0的扩展字段
- [attestation扩展字段的ASN.1 schema](https://source.android.com/docs/security/features/keystore/attestation#schema)
  
```
-----BEGIN CERTIFICATE-----
MIICojCCAkmgAwIBAgIBATAKBggqhkjOPQQDAjApMRkwFwYDVQQFExAxYzE3MmFlOWViMmMyNzg3MQwwCgYDVQQMDANURUUwIBcNNzAwMTAxMDAwMDAwWhgPMjEwNjAyMDcwNjI4MTVaMB8xHTAbBgNVBAMMFEFuZHJvaWQgS2V5c3RvcmUgS2V5MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEr2XbCNUn04ZSHjZHDuGATFNHaatKYD64xVxq0hbXE/B/rVNN4mQbfMEehrZtD0clG9n0DTIBU7DhZJuk0z23RKOCAWgwggFkMA4GA1UdDwEB/wQEAwIHgDCCAVAGCisGAQQB1nkCAREEggFAMIIBPAIBAwoBAQIBBAoBAQQbY2hhbGxlbmdlLXNob3VsZC1iZS1hLW5vbmNlBAAwXL+FPQgCBgFqCA+sML+FRUwESjBIMSIwIAQbY29tLnNoZW4xOTkxLmtleWF0dGVzdGF0aW9uAgEBMSIEIPwsiTcXTYF4XlKEOAs8vctVWTQqaeQAXaykAXYuonKDMIGwoQUxAwIBAqIDAgEDowQCAgEApQsxCQIBBAIBBQIBBqoDAgEBv4N4AwIBA7+DeQQCAgEsv4U+AwIBAL+FQEwwSgQgjKia8abap0sAgQhJNW3pKc/ESY7zavlkdXveihE79G0BAf8KAQAEIE1O53kDZ6JaRR6DWQ2I5FciNcCllfhTZtmBJuUr33hBv4VBBQIDAdTAv4VCBQIDAxXdv4VOBgIEATSKWb+FTwYCBAE0ilkwCgYIKoZIzj0EAwIDRwAwRAIgF4eShdyYw0HHxfaFtOnVUrCP3+IuQsyIv74uPvK1h90CIBGv/4R2bDW53IRbt9j9yUpiR6esT6Ee3LvCRxps9klA
-----END CERTIFICATE-----

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1 (0x1)
        Signature Algorithm: ecdsa-with-SHA256
        Issuer: serialNumber = 1c172ae9eb2c2787, title = TEE
        Validity
            Not Before: Jan  1 00:00:00 1970 GMT
            Not After : Feb  7 06:28:15 2106 GMT
        Subject: CN = Android Keystore Key
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:af:65:db:08:d5:27:d3:86:52:1e:36:47:0e:e1:
                    80:4c:53:47:69:ab:4a:60:3e:b8:c5:5c:6a:d2:16:
                    d7:13:f0:7f:ad:53:4d:e2:64:1b:7c:c1:1e:86:b6:
                    6d:0f:47:25:1b:d9:f4:0d:32:01:53:b0:e1:64:9b:
                    a4:d3:3d:b7:44
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature
            1.3.6.1.4.1.11129.2.1.17:
                0..<...
.....
....challenge-should-be-a-nonce..0\..=....j...0..EL.J0H1"0 ..com.shen1991.keyattestation...1". .,.7.M.x^R.8.<..UY4*i..]...v..r.0....1.................1.................x......y....,..>......@L0J. .......K...I5m.)..I..j.du{...;.m...
..W"5....Sf..&.+.xA..A........B........N....4.Y..O....4.Y
    Signature Algorithm: ecdsa-with-SHA256
    Signature Value:
        30:44:02:20:17:87:92:85:dc:98:c3:41:c7:c5:f6:85:b4:e9:
        d5:52:b0:8f:df:e2:2e:42:cc:88:bf:be:2e:3e:f2:b5:87:dd:
        02:20:11:af:ff:84:76:6c:35:b9:dc:84:5b:b7:d8:fd:c9:4a:
        62:47:a7:ac:4f:a1:1e:dc:bb:c2:47:1a:6c:f6:49:40

PrivateKeyInfo SEQUENCE (8 elem)
  version Version INTEGER 3
  privateKeyAlgorithm AlgorithmIdentifier [?] ENUMERATED 1
  privateKey PrivateKey [?] INTEGER 4
  ENUMERATED 1
  OCTET STRING (27 byte) challenge-should-be-a-nonce
Offset: 16
Length: 2+27
Value:
(27 byte)
challenge-should-be-a-nonce
  OCTET STRING (0 byte)
  SEQUENCE (2 elem)
    [701] (1 elem)
      INTEGER (41 bit) 1554913406000
    [709] (1 elem)
      OCTET STRING (74 byte) 304831223020041B636F6D2E7368656E313939312E6B65796174746573746174696F6E…
        SEQUENCE (2 elem)
          SET (1 elem)
            SEQUENCE (2 elem)
              OCTET STRING (27 byte) com.shen1991.keyattestation
              INTEGER 1
          SET (1 elem)
            OCTET STRING (32 byte) FC2C8937174D81785E5284380B3CBDCB5559342A69E4005DACA401762EA27283
  SEQUENCE (13 elem)
    [1] (1 elem)
      SET (1 elem)
        INTEGER 2
    [2] (1 elem)
      INTEGER 3
    [3] (1 elem)
      INTEGER 256
    [5] (1 elem)
      SET (3 elem)
        INTEGER 4
        INTEGER 5
        INTEGER 6
    [10] (1 elem)
      INTEGER 1
    [504] (1 elem)
      INTEGER 3
    [505] (1 elem)
      INTEGER 300
    [702] (1 elem)
      INTEGER 0
    [704] (1 elem)
      SEQUENCE (4 elem)
        OCTET STRING (32 byte) 8CA89AF1A6DAA74B00810849356DE929CFC4498EF36AF964757BDE8A113BF46D
        BOOLEAN true
        ENUMERATED 0
        OCTET STRING (32 byte) 4D4EE7790367A25A451E83590D88E4572235C0A595F85366D98126E52BDF7841
    [705] (1 elem)
      INTEGER 120000
    [706] (1 elem)
      INTEGER 202205
    [718] (1 elem)
      INTEGER 20220505
    [719] (1 elem)
      INTEGER 20220505
```




#### 2.2.2 cert 1
```
-----BEGIN CERTIFICATE-----
MIICJjCCAaugAwIBAgIKBQEZiXBZcGSUkTAKBggqhkjOPQQDAjApMRkwFwYDVQQFExA0NDNkMjI4NGU5NmFiMjNiMQwwCgYDVQQMDANURUUwHhcNMTgxMjAzMjIzNDM1WhcNMjgxMTMwMjIzNDM1WjApMRkwFwYDVQQFExAxYzE3MmFlOWViMmMyNzg3MQwwCgYDVQQMDANURUUwWTATBgcqhkjOPQIBBggqhkjOPQMBBwNCAARUwB/D1Rb36KHnHZLegpyGlDdTlST1sf8qihs4afBp3Ph0Sbhhl/elouosSJppGpQ/eVHbMWV4NPX2EoiSKx4so4G6MIG3MB0GA1UdDgQWBBTF3ye+of2SeZJfghuQeyomYK+IPDAfBgNVHSMEGDAWgBTz+kTekgvidiuHITJW3HWiylgVkzAPBgNVHRMBAf8EBTADAQH/MA4GA1UdDwEB/wQEAwICBDBUBgNVHR8ETTBLMEmgR6BFhkNodHRwczovL2FuZHJvaWQuZ29vZ2xlYXBpcy5jb20vYXR0ZXN0YXRpb24vY3JsLzA1MDExOTg5NzA1OTcwNjQ5NDkxMAoGCCqGSM49BAMCA2kAMGYCMQDtW3tf/nIrThg6cbXyc8olLRV2t3/cvRp93zarRz6FDRrciKTcsimjwJ/8ZPMJNoUCMQCjEzXW6ZRIauWs3hy53eOtSzFHO5ZqZuDyWnyY6AEoX+/Us1wZgbUxT1U/Sr27F8M=
-----END CERTIFICATE-----

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            05:01:19:89:70:59:70:64:94:91
        Signature Algorithm: ecdsa-with-SHA256
        Issuer: serialNumber = 443d2284e96ab23b, title = TEE
        Validity
            Not Before: Dec  3 22:34:35 2018 GMT
            Not After : Nov 30 22:34:35 2028 GMT
        Subject: serialNumber = 1c172ae9eb2c2787, title = TEE
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:54:c0:1f:c3:d5:16:f7:e8:a1:e7:1d:92:de:82:
                    9c:86:94:37:53:95:24:f5:b1:ff:2a:8a:1b:38:69:
                    f0:69:dc:f8:74:49:b8:61:97:f7:a5:a2:ea:2c:48:
                    9a:69:1a:94:3f:79:51:db:31:65:78:34:f5:f6:12:
                    88:92:2b:1e:2c
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                C5:DF:27:BE:A1:FD:92:79:92:5F:82:1B:90:7B:2A:26:60:AF:88:3C
            X509v3 Authority Key Identifier:
                F3:FA:44:DE:92:0B:E2:76:2B:87:21:32:56:DC:75:A2:CA:58:15:93
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Certificate Sign
            X509v3 CRL Distribution Points:
                Full Name:
                  URI:https://android.googleapis.com/attestation/crl/05011989705970649491
    Signature Algorithm: ecdsa-with-SHA256
    Signature Value:
        30:66:02:31:00:ed:5b:7b:5f:fe:72:2b:4e:18:3a:71:b5:f2:
        73:ca:25:2d:15:76:b7:7f:dc:bd:1a:7d:df:36:ab:47:3e:85:
        0d:1a:dc:88:a4:dc:b2:29:a3:c0:9f:fc:64:f3:09:36:85:02:
        31:00:a3:13:35:d6:e9:94:48:6a:e5:ac:de:1c:b9:dd:e3:ad:
        4b:31:47:3b:96:6a:66:e0:f2:5a:7c:98:e8:01:28:5f:ef:d4:
        b3:5c:19:81:b5:31:4f:55:3f:4a:bd:bb:17:c3

```

#### 2.2.3 cert 2
```
-----BEGIN CERTIFICATE-----
MIID0TCCAbmgAwIBAgIKA4gmZ2BliZaFzjANBgkqhkiG9w0BAQsFADAbMRkwFwYDVQQFExBmOTIwMDllODUzYjZiMDQ1MB4XDTE4MTIwMzIyMjQxNFoXDTI4MTEzMDIyMjQxNFowKTEZMBcGA1UEBRMQNDQzZDIyODRlOTZhYjIzYjEMMAoGA1UEDAwDVEVFMHYwEAYHKoZIzj0CAQYFK4EEACIDYgAE2tYXAmHqxDPYOcCe0QQ4/aMQ1kpkOFpiBsNnnfs0nsOR5ywGNHAbRNqWSKr8FYh/GuspoW2rT5JE3WLjAF9kHEQrOJ8HTThxJS9fl2VAfQS/2gJ64oHBabj6XB5qiqfAo4G2MIGzMB0GA1UdDgQWBBTz+kTekgvidiuHITJW3HWiylgVkzAfBgNVHSMEGDAWgBQ2YeEAfIgFCVGLRGxH/xpMyepPEjAPBgNVHRMBAf8EBTADAQH/MA4GA1UdDwEB/wQEAwICBDBQBgNVHR8ESTBHMEWgQ6BBhj9odHRwczovL2FuZHJvaWQuZ29vZ2xlYXBpcy5jb20vYXR0ZXN0YXRpb24vY3JsL0U4RkExOTYzMTREMkZBMTgwDQYJKoZIhvcNAQELBQADggIBAB9OWP309acpcELw3KHF5LrHsNLYfJsaAzem4gQplevmQlTfVQnok3h5zfmB99TgtlcOfTEO3SLZO4MwjK0oBVBL3AVxGN99+1WLjDEHm35hVqzFEZ93N41boFso+448hpl43BFJwRkAUg5NJ3vgrvGgPW7UhWSYE92ao+p5qmwzZlcIloX+tEVTR6yzowYlpYMacx5IKovoUX9DmATbufSt9iH65gtiJ4CmGM/U6qyMc7rPeA+eaW65NwRkRMOM3fKBSIdR1cde5nG62kkOchbJdGwyXM+Ux/zuCiXyyV0R7HMpmM3siOjDbP6lnGIt70iGVMaQ2JECy7GjAOaHOWZf1kUvZWDg6d15DsuHEDtk/oJIt6dtJ57kGJdhO0DtB3P5zW9VhjsUHrbdYCdzP0Fy5CNJFGo8c83f8yzQknLeGbAFFxHEhEff0F3rbneC26mU9UVaAcKo+o8HH1qnI0Twp9fieMUAwaiWfkjpkvzhl5nuFu4740eV1zC9zX9cGH9MMupoDrRvI42Djhd/7W6KJ1MQWu/7m/Xst7DeBTJSri++ZPcYtxuLpHiZEuvIzYBLGbLAjQ0OkDo/KDVtRrradd1P4746PW9Zfk6I3ALRjLrUT+q4UICLh+o+dtXTN0pNzlSqKWpV6yaFscwRQ4fdZ0grBzo0pbudDU26JmTm
-----END CERTIFICATE-----


Certificate 2:
    Data:
        Version: 3 (0x2)
        Serial Number:
            03:88:26:67:60:65:89:96:85:ce
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: serialNumber = f92009e853b6b045
        Validity
            Not Before: Dec  3 22:24:14 2018 GMT
            Not After : Nov 30 22:24:14 2028 GMT
        Subject: serialNumber = 443d2284e96ab23b, title = TEE
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (384 bit)
                pub:
                    04:da:d6:17:02:61:ea:c4:33:d8:39:c0:9e:d1:04:
                    38:fd:a3:10:d6:4a:64:38:5a:62:06:c3:67:9d:fb:
                    34:9e:c3:91:e7:2c:06:34:70:1b:44:da:96:48:aa:
                    fc:15:88:7f:1a:eb:29:a1:6d:ab:4f:92:44:dd:62:
                    e3:00:5f:64:1c:44:2b:38:9f:07:4d:38:71:25:2f:
                    5f:97:65:40:7d:04:bf:da:02:7a:e2:81:c1:69:b8:
                    fa:5c:1e:6a:8a:a7:c0
                ASN1 OID: secp384r1
                NIST CURVE: P-384
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                F3:FA:44:DE:92:0B:E2:76:2B:87:21:32:56:DC:75:A2:CA:58:15:93
            X509v3 Authority Key Identifier:
                36:61:E1:00:7C:88:05:09:51:8B:44:6C:47:FF:1A:4C:C9:EA:4F:12
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Certificate Sign
            X509v3 CRL Distribution Points:
                Full Name:
                  URI:https://android.googleapis.com/attestation/crl/E8FA196314D2FA18
    Signature Algorithm: sha256WithRSAEncryption

```

#### 2.2.4 cert 3
```
-----BEGIN CERTIFICATE-----
MIIFYDCCA0igAwIBAgIJAOj6GWMU0voYMA0GCSqGSIb3DQEBCwUAMBsxGTAXBgNVBAUTEGY5MjAwOWU4NTNiNmIwNDUwHhcNMTYwNTI2MTYyODUyWhcNMjYwNTI0MTYyODUyWjAbMRkwFwYDVQQFExBmOTIwMDllODUzYjZiMDQ1MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAr7bHgiuxpwHsK7Qui8xUFmOr75gvMsd/dTEDDJdSSxtf6An7xyqpRR90PL2abxM1dEqlXnf2tqw1Ne4Xwl5jlRfdnJLmN0pTy/4lj4/7tv0Sk3iiKkypnEUtR6WfMgH0QZfKHM1+di+y9TFRtv6y//0rb+T+W8a9nsNL/ggjnar86461qO0rOs2cXjp3kOG1FEJ5MVmFmBGtnrKpa73XpXyTqRxB/M0n1n/W9nGqC4FSYa04T6N5RIZGBN2z2MT5IKGbFlbC8UrW0DxW7AYImQQcHtGl/m00QLVWutHQoVJYnFPlXTcHYvASLu+RhhsbDmxMgJJ0mcDpvsC4PjvB+TxywElgS70vE0XmLD+OJtvsBslHZvPBKCOdT0MS+tgSOIfga+z1Z1g7+DVagf7quvmag8jfPioyKvxnK/EgsTUVi2ghzq8wm27ud/mIM7AY2qEORR8Go3TVB4HzWQgpZrt3i5MIlCaY504LzSRiigHCzAPlHws+W0rB5N+er5/2pJKnfBSDiCiFAVtCLOZ7gLiMm0jhO2B6tUXHI/+MRPjy02i59lINMRRev56GKtcd9qO/0kUJWdZTdA2XoS82ixPvZtXQpUpuL12ab+9EaDK8Z4RHJYYfCT3Q5vNAXaiWQ+8PTWm2QgBR/bkwSWc+NpUFgNPN9PvQi8WEg5UmAGMCAwEAAaOBpjCBozAdBgNVHQ4EFgQUNmHhAHyIBQlRi0RsR/8aTMnqTxIwHwYDVR0jBBgwFoAUNmHhAHyIBQlRi0RsR/8aTMnqTxIwDwYDVR0TAQH/BAUwAwEB/zAOBgNVHQ8BAf8EBAMCAYYwQAYDVR0fBDkwNzA1oDOgMYYvaHR0cHM6Ly9hbmRyb2lkLmdvb2dsZWFwaXMuY29tL2F0dGVzdGF0aW9uL2NybC8wDQYJKoZIhvcNAQELBQADggIBACDIw41L3KlXG0aMiS//cqrG+EShHUGo8HNsw30W1kJtjn6UBwRM6jnmiwfBPb8VA91chb2vssAtX2zbTvqBJ9+LBPGCdw/E53Rbf86qhxKaiAHOjpvAy5Y3m00mqC0w/Zwvju1twb4vhLaJ5NkUJYsUS7rmJKHHBnETLi8GFqiEsqTWpG/6ibYCv7rYDBJDcR9W62BW9jfIoBQcxUCUJouMPH25lLNcDc1ssqvC2v7iUgI9LeoM1sNovqPmQUiG9rHli1vXxzCyaMTjwftkJLkf6724DFhuKug2jITV0QkXvaJWF4nUaHOTNA4uJU9WDvZLI1j83A+/xnAJUucIv/zGJ1AMH2boHqF8CY16LpsYgBt6tKxxWH00XcyDCdW2KlBCeqbQPcsFmWyWugxdcekhYsAWyoSf818NUsZdBWBaR/OukXrNLfkQ79IyZohZbvabO/X+MVT3rriAoKc8oE2Uws6DF+60PV7/WIPjNvXySdqspImSN78mflxDqwLqRBYkA3I75qppLGG9rp7UCdRjxMl8ZDBld+7yvHVgt1cVzJx9xnyGCC23UaicMDSXYrB4I4WHXPGjxhZuCuPBLTdOLU8YRvMYdEvYebWHMpvwGCF6bAx3JBpIeOQ1wDB5y0USicV3YgYGmi+NZfhA4URSh77Yd6uuJOJENRaNVTzk
-----END CERTIFICATE-----


Certificate 3:
    Data:
        Version: 3 (0x2)
        Serial Number:
            e8:fa:19:63:14:d2:fa:18
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: serialNumber = f92009e853b6b045
        Validity
            Not Before: May 26 16:28:52 2016 GMT
            Not After : May 24 16:28:52 2026 GMT
        Subject: serialNumber = f92009e853b6b045
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
                Modulus:
                    00:af:b6:c7:82:2b:b1:a7:01:ec:2b:b4:2e:8b:cc:
                    54:16:63:ab:ef:98:2f:32:c7:7f:75:31:03:0c:97:
                    52:4b:1b:5f:e8:09:fb:c7:2a:a9:45:1f:74:3c:bd:
                    9a:6f:13:35:74:4a:a5:5e:77:f6:b6:ac:35:35:ee:
                    17:c2:5e:63:95:17:dd:9c:92:e6:37:4a:53:cb:fe:
                    25:8f:8f:fb:b6:fd:12:93:78:a2:2a:4c:a9:9c:45:
                    2d:47:a5:9f:32:01:f4:41:97:ca:1c:cd:7e:76:2f:
                    b2:f5:31:51:b6:fe:b2:ff:fd:2b:6f:e4:fe:5b:c6:
                    bd:9e:c3:4b:fe:08:23:9d:aa:fc:eb:8e:b5:a8:ed:
                    2b:3a:cd:9c:5e:3a:77:90:e1:b5:14:42:79:31:59:
                    85:98:11:ad:9e:b2:a9:6b:bd:d7:a5:7c:93:a9:1c:
                    41:fc:cd:27:d6:7f:d6:f6:71:aa:0b:81:52:61:ad:
                    38:4f:a3:79:44:86:46:04:dd:b3:d8:c4:f9:20:a1:
                    9b:16:56:c2:f1:4a:d6:d0:3c:56:ec:06:08:99:04:
                    1c:1e:d1:a5:fe:6d:34:40:b5:56:ba:d1:d0:a1:52:
                    58:9c:53:e5:5d:37:07:62:f0:12:2e:ef:91:86:1b:
                    1b:0e:6c:4c:80:92:74:99:c0:e9:be:c0:b8:3e:3b:
                    c1:f9:3c:72:c0:49:60:4b:bd:2f:13:45:e6:2c:3f:
                    8e:26:db:ec:06:c9:47:66:f3:c1:28:23:9d:4f:43:
                    12:fa:d8:12:38:87:e0:6b:ec:f5:67:58:3b:f8:35:
                    5a:81:fe:ea:ba:f9:9a:83:c8:df:3e:2a:32:2a:fc:
                    67:2b:f1:20:b1:35:15:8b:68:21:ce:af:30:9b:6e:
                    ee:77:f9:88:33:b0:18:da:a1:0e:45:1f:06:a3:74:
                    d5:07:81:f3:59:08:29:66:bb:77:8b:93:08:94:26:
                    98:e7:4e:0b:cd:24:62:8a:01:c2:cc:03:e5:1f:0b:
                    3e:5b:4a:c1:e4:df:9e:af:9f:f6:a4:92:a7:7c:14:
                    83:88:28:85:01:5b:42:2c:e6:7b:80:b8:8c:9b:48:
                    e1:3b:60:7a:b5:45:c7:23:ff:8c:44:f8:f2:d3:68:
                    b9:f6:52:0d:31:14:5e:bf:9e:86:2a:d7:1d:f6:a3:
                    bf:d2:45:09:59:d6:53:74:0d:97:a1:2f:36:8b:13:
                    ef:66:d5:d0:a5:4a:6e:2f:5d:9a:6f:ef:44:68:32:
                    bc:67:84:47:25:86:1f:09:3d:d0:e6:f3:40:5d:a8:
                    96:43:ef:0f:4d:69:b6:42:00:51:fd:b9:30:49:67:
                    3e:36:95:05:80:d3:cd:f4:fb:d0:8b:c5:84:83:95:
                    26:00:63
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                36:61:E1:00:7C:88:05:09:51:8B:44:6C:47:FF:1A:4C:C9:EA:4F:12
            X509v3 Authority Key Identifier:
                36:61:E1:00:7C:88:05:09:51:8B:44:6C:47:FF:1A:4C:C9:EA:4F:12
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Digital Signature, Certificate Sign, CRL Sign
            X509v3 CRL Distribution Points:
                Full Name:
                  URI:https://android.googleapis.com/attestation/crl/
    Signature Algorithm: sha256WithRSAEncryption
```

### 2.3 XAGA with unlocked BL(TEE not Broken)

#### 2.3.1 cert 0
- 使用[https://lapo.it/asn1js/](https://lapo.it/asn1js/)解析cert0的扩展字段
- [attestation扩展字段的ASN.1 schema](https://source.android.com/docs/security/features/keystore/attestation#schema)

```
-----BEGIN CERTIFICATE-----
MIICkTCCAjegAwIBAgIBATAKBggqhkjOPQQDAjA5MQwwCgYDVQQMDANURUUxKTAnBgNVBAUTIDMyYzc4ZjNhYTE0NDM3YTM1YTRhOGQyZjJhMWJlOWVjMB4XDTcwMDEwMTAwMDAwMFoXDTQ4MDEwMTAwMDAwMFowHzEdMBsGA1UEAxMUQW5kcm9pZCBLZXlzdG9yZSBLZXkwWTATBgcqhkjOPQIBBggqhkjOPQMBBwNCAAS1JKLq7yJVPPEea74NxKRnBJQbWWMB9jsVdyKt7Qrw+TvxDRZSmyq2mnkLNKFN/ev6yZGF8mMTPzUPMW0yGe/Co4IBSDCCAUQwDgYDVR0PAQH/BAQDAgeAMIIBMAYKKwYBBAHWeQIBEQSCASAwggEcAgFkCgEBAgFkCgEBBBtjaGFsbGVuZ2Utc2hvdWxkLWJlLWEtbm9uY2UEADBQv4VFTARKMEgxIjAgBBtjb20uc2hlbjE5OTEua2V5YXR0ZXN0YXRpb24CAQExIgQg/CyJNxdNgXheUoQ4Czy9y1VZNCpp5ABdrKQBdi6icoMwgZyhBTEDAgECogMCAQOjBAICAQClCzEJAgEEAgEFAgEGqgMCAQG/g3gDAgEDv4N5BAICASy/hT4DAgEAv4VATDBKBCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEBAAoBAgQgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAC/hUEFAgMB1MC/hUIFAgMDFeIwCgYIKoZIzj0EAwIDSAAwRQIhAN5kZiQu/vmf+IYhcESEnfN2vVQtsdu9SCjV0O2x7vZIAiB85VmBJC72GGrNogRVGfjcQZuz5GeRsIwnZe1djQbcRg==
-----END CERTIFICATE-----


Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1 (0x1)
        Signature Algorithm: ecdsa-with-SHA256
        Issuer: title = TEE, serialNumber = 32c78f3aa14437a35a4a8d2f2a1be9ec
        Validity
            Not Before: Jan  1 00:00:00 1970 GMT
            Not After : Jan  1 00:00:00 2048 GMT
        Subject: CN = Android Keystore Key
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:b5:24:a2:ea:ef:22:55:3c:f1:1e:6b:be:0d:c4:
                    a4:67:04:94:1b:59:63:01:f6:3b:15:77:22:ad:ed:
                    0a:f0:f9:3b:f1:0d:16:52:9b:2a:b6:9a:79:0b:34:
                    a1:4d:fd:eb:fa:c9:91:85:f2:63:13:3f:35:0f:31:
                    6d:32:19:ef:c2
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature
            1.3.6.1.4.1.11129.2.1.17:
                0.....d
....d
....challenge-should-be-a-nonce..0P..EL.J0H1"0 ..com.shen1991.keyattestation...1". .,.7.M.x^R.8.<..UY4*i..]...v..r.0....1.................1.................x......y....,..>......@L0J. ...................................
... ..................................A........B......
    Signature Algorithm: ecdsa-with-SHA256
    Signature Value:
        30:45:02:21:00:de:64:66:24:2e:fe:f9:9f:f8:86:21:70:44:
        84:9d:f3:76:bd:54:2d:b1:db:bd:48:28:d5:d0:ed:b1:ee:f6:
        48:02:20:7c:e5:59:81:24:2e:f6:18:6a:cd:a2:04:55:19:f8:
        dc:41:9b:b3:e4:67:91:b0:8c:27:65:ed:5d:8d:06:dc:46


SEQUENCE (8 elem)
  INTEGER 100
  ENUMERATED 1
  INTEGER 100
  ENUMERATED 1
  OCTET STRING (27 byte) challenge-should-be-a-nonce
  OCTET STRING (0 byte)
Offset: 45
Length: 2+0
(encapsulates)
Value:
(0 byte)
  SEQUENCE (1 elem)
    [709] (1 elem)
      OCTET STRING (74 byte) 304831223020041B636F6D2E7368656E313939312E6B65796174746573746174696F6E…
        SEQUENCE (2 elem)
          SET (1 elem)
            SEQUENCE (2 elem)
              OCTET STRING (27 byte) com.shen1991.keyattestation
              INTEGER 1
          SET (1 elem)
            OCTET STRING (32 byte) FC2C8937174D81785E5284380B3CBDCB5559342A69E4005DACA401762EA27283
  SEQUENCE (11 elem)
    [1] (1 elem)
      SET (1 elem)
        INTEGER 2
    [2] (1 elem)
      INTEGER 3
    [3] (1 elem)
      INTEGER 256
    [5] (1 elem)
      SET (3 elem)
        INTEGER 4
        INTEGER 5
        INTEGER 6
    [10] (1 elem)
      INTEGER 1
    [504] (1 elem)
      INTEGER 3
    [505] (1 elem)
      INTEGER 300
    [702] (1 elem)
      INTEGER 0
    [704] (1 elem)
      SEQUENCE (4 elem)
        OCTET STRING (32 byte) 0000000000000000000000000000000000000000000000000000000000000000
        BOOLEAN false
        ENUMERATED 2
        OCTET STRING (32 byte) 0000000000000000000000000000000000000000000000000000000000000000
    [705] (1 elem)
      INTEGER 120000
    [706] (1 elem)
      INTEGER 202210
```

#### 2.3.2 cert 1

```
-----BEGIN CERTIFICATE-----
MIIB9DCCAXmgAwIBAgIQUN87SJSOda+SnVBeOKa8xTAKBggqhkjOPQQDAjA5MQwwCgYDVQQMDANURUUxKTAnBgNVBAUTIDhlYTAzZjlmNzcyNWFhNDhkNzgzNzk1MzNiMzAxYmE5MB4XDTIyMDkxNzE3MjY0MFoXDTMyMDkxNDE3MjY0MFowOTEMMAoGA1UEDAwDVEVFMSkwJwYDVQQFEyAzMmM3OGYzYWExNDQzN2EzNWE0YThkMmYyYTFiZTllYzBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABNcguWZ/jyXncReCB/YYQqNN/H+ALJ+4Qz93qfPJHfH9Mi/2EUa7mcfxIBtJ7gq4ePXxc8BltxpQT1HK2rZw/S2jYzBhMB0GA1UdDgQWBBQvPKrC11Cu2K3ss9xi7lPWzOpdgDAfBgNVHSMEGDAWgBROmIZs7FoUHqyCiL59cV5zcOZydjAPBgNVHRMBAf8EBTADAQH/MA4GA1UdDwEB/wQEAwICBDAKBggqhkjOPQQDAgNpADBmAjEAxBKLv1M9iT6Gy189UT/PU2NgKtdLCeVmzGJpoaUBD2yximnR0/fZ+1HRfxddtl5GAjEA2HtyYI0IKLvh9MmNLxpYBM353SyLZ9yFB1Gb587u+/Mn3T2fYudQS8Vbglo5fWrc
-----END CERTIFICATE-----


Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            50:df:3b:48:94:8e:75:af:92:9d:50:5e:38:a6:bc:c5
        Signature Algorithm: ecdsa-with-SHA256
        Issuer: title = TEE, serialNumber = 8ea03f9f7725aa48d78379533b301ba9
        Validity
            Not Before: Sep 17 17:26:40 2022 GMT
            Not After : Sep 14 17:26:40 2032 GMT
        Subject: title = TEE, serialNumber = 32c78f3aa14437a35a4a8d2f2a1be9ec
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:d7:20:b9:66:7f:8f:25:e7:71:17:82:07:f6:18:
                    42:a3:4d:fc:7f:80:2c:9f:b8:43:3f:77:a9:f3:c9:
                    1d:f1:fd:32:2f:f6:11:46:bb:99:c7:f1:20:1b:49:
                    ee:0a:b8:78:f5:f1:73:c0:65:b7:1a:50:4f:51:ca:
                    da:b6:70:fd:2d
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                2F:3C:AA:C2:D7:50:AE:D8:AD:EC:B3:DC:62:EE:53:D6:CC:EA:5D:80
            X509v3 Authority Key Identifier:
                4E:98:86:6C:EC:5A:14:1E:AC:82:88:BE:7D:71:5E:73:70:E6:72:76
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Certificate Sign
    Signature Algorithm: ecdsa-with-SHA256
    Signature Value:
        30:66:02:31:00:c4:12:8b:bf:53:3d:89:3e:86:cb:5f:3d:51:
        3f:cf:53:63:60:2a:d7:4b:09:e5:66:cc:62:69:a1:a5:01:0f:
        6c:b1:8a:69:d1:d3:f7:d9:fb:51:d1:7f:17:5d:b6:5e:46:02:
        31:00:d8:7b:72:60:8d:08:28:bb:e1:f4:c9:8d:2f:1a:58:04:
        cd:f9:dd:2c:8b:67:dc:85:07:51:9b:e7:ce:ee:fb:f3:27:dd:
        3d:9f:62:e7:50:4b:c5:5b:82:5a:39:7d:6a:dc
```


#### 2.3.3 cert 2
```

-----BEGIN CERTIFICATE-----
MIIDlDCCAXygAwIBAgIRAJsbJTM0rFYTusNOKGA6buwwDQYJKoZIhvcNAQELBQAwGzEZMBcGA1UEBRMQZjkyMDA5ZTg1M2I2YjA0NTAeFw0yMjA5MTcxNzIzMzhaFw0zMjA5MTQxNzIzMzhaMDkxDDAKBgNVBAwMA1RFRTEpMCcGA1UEBRMgOGVhMDNmOWY3NzI1YWE0OGQ3ODM3OTUzM2IzMDFiYTkwdjAQBgcqhkjOPQIBBgUrgQQAIgNiAARiTo/8v71cIEsKy224z2K7Hpn0woN4bM+3rLRCZ+pulfNmQjJbprsvdq9C2mBT1K50lKtwyASymKM6WsXZKaQ75hnFtAIFn0T1Bv/bNRqg3rSpVr1bDaZKrKjLrTQM0UOjYzBhMB0GA1UdDgQWBBROmIZs7FoUHqyCiL59cV5zcOZydjAfBgNVHSMEGDAWgBQ2YeEAfIgFCVGLRGxH/xpMyepPEjAPBgNVHRMBAf8EBTADAQH/MA4GA1UdDwEB/wQEAwICBDANBgkqhkiG9w0BAQsFAAOCAgEACJdpcp1R3EqCQTBa7y0Wf+HLU1DjEuHxXv9wBUrPBxjJl6wRMy32G9X9x0hNc0lqiiAKAuVi8UZOA7mKlM0nKOeu/Jkmpq+Boq/FHHgYtUSzGzvXZSfs0gtCJqD7LKLyTY2ScVsJgKUdbAyBOOjO2s97H/cbXX0zcL22x42KO/FoIrvsLWD57q/IOT44G3RA3nc6gQMYjGj3NdjDHdMfwA50x1x/C5JXiMqudWhVCilDPjkbxKgN9TZ2LqnmoVSLHi4M/JyxsjAEpMO2xGKvhJAGbMCBg8r8hiA33qMcKqryNbaYZyPwwTaTtvw0MhUq3KzZ1qjBvvJUOKk6xL9HxJH7Xt66fRJUJfkhz0qzNevRmTtANBaP8Wvj5eLk+m/n8cJlK6zHEXEH9tV+Oc5YQgh658fuFUGoFy72wGZWlRoA3eB8S1i1PI++uAlDLx2OdsOrbAx/ve9sl8yiy7xrZCKdpnogf85cu5BW4DmxUk9Is/FsE59D4GTIrIaIoYSJVE+zh8g6gZElfn0uey/BYGCsh3jGjOetwsG8HjNbeBxTh5Gksp2kfYPgXzFxUn8fXfqKxUip82646UsNcxBfMsPq6F8fy+zBoooJMtK5hDCN6KEp5tzyNxmPwhQL1TuGYROVrCAWVSey+YY8YHhCEuTBMw2RWgh+9v5osmHDtTE=
-----END CERTIFICATE-----



Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            9b:1b:25:33:34:ac:56:13:ba:c3:4e:28:60:3a:6e:ec
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: serialNumber = f92009e853b6b045
        Validity
            Not Before: Sep 17 17:23:38 2022 GMT
            Not After : Sep 14 17:23:38 2032 GMT
        Subject: title = TEE, serialNumber = 8ea03f9f7725aa48d78379533b301ba9
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (384 bit)
                pub:
                    04:62:4e:8f:fc:bf:bd:5c:20:4b:0a:cb:6d:b8:cf:
                    62:bb:1e:99:f4:c2:83:78:6c:cf:b7:ac:b4:42:67:
                    ea:6e:95:f3:66:42:32:5b:a6:bb:2f:76:af:42:da:
                    60:53:d4:ae:74:94:ab:70:c8:04:b2:98:a3:3a:5a:
                    c5:d9:29:a4:3b:e6:19:c5:b4:02:05:9f:44:f5:06:
                    ff:db:35:1a:a0:de:b4:a9:56:bd:5b:0d:a6:4a:ac:
                    a8:cb:ad:34:0c:d1:43
                ASN1 OID: secp384r1
                NIST CURVE: P-384
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                4E:98:86:6C:EC:5A:14:1E:AC:82:88:BE:7D:71:5E:73:70:E6:72:76
            X509v3 Authority Key Identifier:
                36:61:E1:00:7C:88:05:09:51:8B:44:6C:47:FF:1A:4C:C9:EA:4F:12
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Certificate Sign
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        08:97:69:72:9d:51:dc:4a:82:41:30:5a:ef:2d:16:7f:e1:cb:
        53:50:e3:12:e1:f1:5e:ff:70:05:4a:cf:07:18:c9:97:ac:11:
        33:2d:f6:1b:d5:fd:c7:48:4d:73:49:6a:8a:20:0a:02:e5:62:
        f1:46:4e:03:b9:8a:94:cd:27:28:e7:ae:fc:99:26:a6:af:81:
        a2:af:c5:1c:78:18:b5:44:b3:1b:3b:d7:65:27:ec:d2:0b:42:
        26:a0:fb:2c:a2:f2:4d:8d:92:71:5b:09:80:a5:1d:6c:0c:81:
        38:e8:ce:da:cf:7b:1f:f7:1b:5d:7d:33:70:bd:b6:c7:8d:8a:
        3b:f1:68:22:bb:ec:2d:60:f9:ee:af:c8:39:3e:38:1b:74:40:
        de:77:3a:81:03:18:8c:68:f7:35:d8:c3:1d:d3:1f:c0:0e:74:
        c7:5c:7f:0b:92:57:88:ca:ae:75:68:55:0a:29:43:3e:39:1b:
        c4:a8:0d:f5:36:76:2e:a9:e6:a1:54:8b:1e:2e:0c:fc:9c:b1:
        b2:30:04:a4:c3:b6:c4:62:af:84:90:06:6c:c0:81:83:ca:fc:
        86:20:37:de:a3:1c:2a:aa:f2:35:b6:98:67:23:f0:c1:36:93:
        b6:fc:34:32:15:2a:dc:ac:d9:d6:a8:c1:be:f2:54:38:a9:3a:
        c4:bf:47:c4:91:fb:5e:de:ba:7d:12:54:25:f9:21:cf:4a:b3:
        35:eb:d1:99:3b:40:34:16:8f:f1:6b:e3:e5:e2:e4:fa:6f:e7:
        f1:c2:65:2b:ac:c7:11:71:07:f6:d5:7e:39:ce:58:42:08:7a:
        e7:c7:ee:15:41:a8:17:2e:f6:c0:66:56:95:1a:00:dd:e0:7c:
        4b:58:b5:3c:8f:be:b8:09:43:2f:1d:8e:76:c3:ab:6c:0c:7f:
        bd:ef:6c:97:cc:a2:cb:bc:6b:64:22:9d:a6:7a:20:7f:ce:5c:
        bb:90:56:e0:39:b1:52:4f:48:b3:f1:6c:13:9f:43:e0:64:c8:
        ac:86:88:a1:84:89:54:4f:b3:87:c8:3a:81:91:25:7e:7d:2e:
        7b:2f:c1:60:60:ac:87:78:c6:8c:e7:ad:c2:c1:bc:1e:33:5b:
        78:1c:53:87:91:a4:b2:9d:a4:7d:83:e0:5f:31:71:52:7f:1f:
        5d:fa:8a:c5:48:a9:f3:6e:b8:e9:4b:0d:73:10:5f:32:c3:ea:
        e8:5f:1f:cb:ec:c1:a2:8a:09:32:d2:b9:84:30:8d:e8:a1:29:
        e6:dc:f2:37:19:8f:c2:14:0b:d5:3b:86:61:13:95:ac:20:16:
        55:27:b2:f9:86:3c:60:78:42:12:e4:c1:33:0d:91:5a:08:7e:
        f6:fe:68:b2:61:c3:b5:31
```

#### 2.3.4 cert 3

```
-----BEGIN CERTIFICATE-----
MIIFHDCCAwSgAwIBAgIJAPHBcqaZ6vUdMA0GCSqGSIb3DQEBCwUAMBsxGTAXBgNVBAUTEGY5MjAwOWU4NTNiNmIwNDUwHhcNMjIwMzIwMTgwNzQ4WhcNNDIwMzE1MTgwNzQ4WjAbMRkwFwYDVQQFExBmOTIwMDllODUzYjZiMDQ1MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAr7bHgiuxpwHsK7Qui8xUFmOr75gvMsd/dTEDDJdSSxtf6An7xyqpRR90PL2abxM1dEqlXnf2tqw1Ne4Xwl5jlRfdnJLmN0pTy/4lj4/7tv0Sk3iiKkypnEUtR6WfMgH0QZfKHM1+di+y9TFRtv6y//0rb+T+W8a9nsNL/ggjnar86461qO0rOs2cXjp3kOG1FEJ5MVmFmBGtnrKpa73XpXyTqRxB/M0n1n/W9nGqC4FSYa04T6N5RIZGBN2z2MT5IKGbFlbC8UrW0DxW7AYImQQcHtGl/m00QLVWutHQoVJYnFPlXTcHYvASLu+RhhsbDmxMgJJ0mcDpvsC4PjvB+TxywElgS70vE0XmLD+OJtvsBslHZvPBKCOdT0MS+tgSOIfga+z1Z1g7+DVagf7quvmag8jfPioyKvxnK/EgsTUVi2ghzq8wm27ud/mIM7AY2qEORR8Go3TVB4HzWQgpZrt3i5MIlCaY504LzSRiigHCzAPlHws+W0rB5N+er5/2pJKnfBSDiCiFAVtCLOZ7gLiMm0jhO2B6tUXHI/+MRPjy02i59lINMRRev56GKtcd9qO/0kUJWdZTdA2XoS82ixPvZtXQpUpuL12ab+9EaDK8Z4RHJYYfCT3Q5vNAXaiWQ+8PTWm2QgBR/bkwSWc+NpUFgNPN9PvQi8WEg5UmAGMCAwEAAaNjMGEwHQYDVR0OBBYEFDZh4QB8iAUJUYtEbEf/GkzJ6k8SMB8GA1UdIwQYMBaAFDZh4QB8iAUJUYtEbEf/GkzJ6k8SMA8GA1UdEwEB/wQFMAMBAf8wDgYDVR0PAQH/BAQDAgIEMA0GCSqGSIb3DQEBCwUAA4ICAQB8cMqTllHc8U+qCrOlg3H7174lmaCsbo/bJ0C17JEgMLb4kvrqsXZs01U3mB/qABg/1t5Pd5AORHARs1hhqGICW/nKMav574f9rZN4PC2ZlufGXb7sIdJpGiO9ctRhiLuYuly10JccUZGEHpHSYM2GtkgYbZba6lsCPYAAP83cyDV+1aOkTf1RCp/lM0PKvmxYN10RYsK631jrleGdcdkxoSK//mSQbgcWnmAEZrzHoF1/0gso1HZgIn0YLzVhLSA/iXCX4QT2h3J5z3znluKG1nv8NQdxei2DIIhASWfu804CA96cQKTTlaae2fweqXjdN1/v2nqOhngNyz1361mFmr4XmaKH/ItTwOe72NI9ZcwS1lVaCvsIkTDCEXdm9rCNPAY10iTunIHFXRh+7KPzlHGewCq/8TOohBRn0/NNfh7uRslOSZ/xKbN9tMBtw37Z8d2vvnXq/YWdsm1+JLVwn6yYD/yacNJBlwpddla8eaVMjsF6nBnIgQOf9zKSe06nSTqvgwUHosgOECZJZ1EuzbH4yswbt02tKtKEFhx+v+OTge/06V+jGsqTWLsfrOCNLuA8H++z+pUENmpqnnHovaI47gC+TNpkgYGkkBT6B/m/U01BuOBBTzhIlMEZq9qkDWuM2cA5kW5V3FJUcfHnw1IdYIg2Wxg7yHcQZemFQg==
-----END CERTIFICATE-----


Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            f1:c1:72:a6:99:ea:f5:1d
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: serialNumber = f92009e853b6b045
        Validity
            Not Before: Mar 20 18:07:48 2022 GMT
            Not After : Mar 15 18:07:48 2042 GMT
        Subject: serialNumber = f92009e853b6b045
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
                Modulus:
                    00:af:b6:c7:82:2b:b1:a7:01:ec:2b:b4:2e:8b:cc:
                    54:16:63:ab:ef:98:2f:32:c7:7f:75:31:03:0c:97:
                    52:4b:1b:5f:e8:09:fb:c7:2a:a9:45:1f:74:3c:bd:
                    9a:6f:13:35:74:4a:a5:5e:77:f6:b6:ac:35:35:ee:
                    17:c2:5e:63:95:17:dd:9c:92:e6:37:4a:53:cb:fe:
                    25:8f:8f:fb:b6:fd:12:93:78:a2:2a:4c:a9:9c:45:
                    2d:47:a5:9f:32:01:f4:41:97:ca:1c:cd:7e:76:2f:
                    b2:f5:31:51:b6:fe:b2:ff:fd:2b:6f:e4:fe:5b:c6:
                    bd:9e:c3:4b:fe:08:23:9d:aa:fc:eb:8e:b5:a8:ed:
                    2b:3a:cd:9c:5e:3a:77:90:e1:b5:14:42:79:31:59:
                    85:98:11:ad:9e:b2:a9:6b:bd:d7:a5:7c:93:a9:1c:
                    41:fc:cd:27:d6:7f:d6:f6:71:aa:0b:81:52:61:ad:
                    38:4f:a3:79:44:86:46:04:dd:b3:d8:c4:f9:20:a1:
                    9b:16:56:c2:f1:4a:d6:d0:3c:56:ec:06:08:99:04:
                    1c:1e:d1:a5:fe:6d:34:40:b5:56:ba:d1:d0:a1:52:
                    58:9c:53:e5:5d:37:07:62:f0:12:2e:ef:91:86:1b:
                    1b:0e:6c:4c:80:92:74:99:c0:e9:be:c0:b8:3e:3b:
                    c1:f9:3c:72:c0:49:60:4b:bd:2f:13:45:e6:2c:3f:
                    8e:26:db:ec:06:c9:47:66:f3:c1:28:23:9d:4f:43:
                    12:fa:d8:12:38:87:e0:6b:ec:f5:67:58:3b:f8:35:
                    5a:81:fe:ea:ba:f9:9a:83:c8:df:3e:2a:32:2a:fc:
                    67:2b:f1:20:b1:35:15:8b:68:21:ce:af:30:9b:6e:
                    ee:77:f9:88:33:b0:18:da:a1:0e:45:1f:06:a3:74:
                    d5:07:81:f3:59:08:29:66:bb:77:8b:93:08:94:26:
                    98:e7:4e:0b:cd:24:62:8a:01:c2:cc:03:e5:1f:0b:
                    3e:5b:4a:c1:e4:df:9e:af:9f:f6:a4:92:a7:7c:14:
                    83:88:28:85:01:5b:42:2c:e6:7b:80:b8:8c:9b:48:
                    e1:3b:60:7a:b5:45:c7:23:ff:8c:44:f8:f2:d3:68:
                    b9:f6:52:0d:31:14:5e:bf:9e:86:2a:d7:1d:f6:a3:
                    bf:d2:45:09:59:d6:53:74:0d:97:a1:2f:36:8b:13:
                    ef:66:d5:d0:a5:4a:6e:2f:5d:9a:6f:ef:44:68:32:
                    bc:67:84:47:25:86:1f:09:3d:d0:e6:f3:40:5d:a8:
                    96:43:ef:0f:4d:69:b6:42:00:51:fd:b9:30:49:67:
                    3e:36:95:05:80:d3:cd:f4:fb:d0:8b:c5:84:83:95:
                    26:00:63
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                36:61:E1:00:7C:88:05:09:51:8B:44:6C:47:FF:1A:4C:C9:EA:4F:12
            X509v3 Authority Key Identifier:
                36:61:E1:00:7C:88:05:09:51:8B:44:6C:47:FF:1A:4C:C9:EA:4F:12
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Certificate Sign
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        7c:70:ca:93:96:51:dc:f1:4f:aa:0a:b3:a5:83:71:fb:d7:be:
        25:99:a0:ac:6e:8f:db:27:40:b5:ec:91:20:30:b6:f8:92:fa:
        ea:b1:76:6c:d3:55:37:98:1f:ea:00:18:3f:d6:de:4f:77:90:
        0e:44:70:11:b3:58:61:a8:62:02:5b:f9:ca:31:ab:f9:ef:87:
        fd:ad:93:78:3c:2d:99:96:e7:c6:5d:be:ec:21:d2:69:1a:23:
        bd:72:d4:61:88:bb:98:ba:5c:b5:d0:97:1c:51:91:84:1e:91:
        d2:60:cd:86:b6:48:18:6d:96:da:ea:5b:02:3d:80:00:3f:cd:
        dc:c8:35:7e:d5:a3:a4:4d:fd:51:0a:9f:e5:33:43:ca:be:6c:
        58:37:5d:11:62:c2:ba:df:58:eb:95:e1:9d:71:d9:31:a1:22:
        bf:fe:64:90:6e:07:16:9e:60:04:66:bc:c7:a0:5d:7f:d2:0b:
        28:d4:76:60:22:7d:18:2f:35:61:2d:20:3f:89:70:97:e1:04:
        f6:87:72:79:cf:7c:e7:96:e2:86:d6:7b:fc:35:07:71:7a:2d:
        83:20:88:40:49:67:ee:f3:4e:02:03:de:9c:40:a4:d3:95:a6:
        9e:d9:fc:1e:a9:78:dd:37:5f:ef:da:7a:8e:86:78:0d:cb:3d:
        77:eb:59:85:9a:be:17:99:a2:87:fc:8b:53:c0:e7:bb:d8:d2:
        3d:65:cc:12:d6:55:5a:0a:fb:08:91:30:c2:11:77:66:f6:b0:
        8d:3c:06:35:d2:24:ee:9c:81:c5:5d:18:7e:ec:a3:f3:94:71:
        9e:c0:2a:bf:f1:33:a8:84:14:67:d3:f3:4d:7e:1e:ee:46:c9:
        4e:49:9f:f1:29:b3:7d:b4:c0:6d:c3:7e:d9:f1:dd:af:be:75:
        ea:fd:85:9d:b2:6d:7e:24:b5:70:9f:ac:98:0f:fc:9a:70:d2:
        41:97:0a:5d:76:56:bc:79:a5:4c:8e:c1:7a:9c:19:c8:81:03:
        9f:f7:32:92:7b:4e:a7:49:3a:af:83:05:07:a2:c8:0e:10:26:
        49:67:51:2e:cd:b1:f8:ca:cc:1b:b7:4d:ad:2a:d2:84:16:1c:
        7e:bf:e3:93:81:ef:f4:e9:5f:a3:1a:ca:93:58:bb:1f:ac:e0:
        8d:2e:e0:3c:1f:ef:b3:fa:95:04:36:6a:6a:9e:71:e8:bd:a2:
        38:ee:00:be:4c:da:64:81:81:a4:90:14:fa:07:f9:bf:53:4d:
        41:b8:e0:41:4f:38:48:94:c1:19:ab:da:a4:0d:6b:8c:d9:c0:
        39:91:6e:55:dc:52:54:71:f1:e7:c3:52:1d:60:88:36:5b:18:
        3b:c8:77:10:65:e9:85:42
```

### 2.4 XAGA with locked BL

#### 2.4.1 cert 0
- 使用[https://lapo.it/asn1js/](https://lapo.it/asn1js/)解析cert0的扩展字段
- [attestation扩展字段的ASN.1 schema](https://source.android.com/docs/security/features/keystore/attestation#schema)

```
-----BEGIN CERTIFICATE-----
MIICsjCCAlegAwIBAgIBATAKBggqhkjOPQQDAjA5MQwwCgYDVQQMDANURUUxKTAnBgNVBAUTIGQ1NWZhMWVmZTgwZjFlY2NmODM1NDQ5YzAzZDhjZGU0MB4XDTcwMDEwMTAwMDAwMFoXDTQ4MDEwMTAwMDAwMFowHzEdMBsGA1UEAxMUQW5kcm9pZCBLZXlzdG9yZSBLZXkwWTATBgcqhkjOPQIBBggqhkjOPQMBBwNCAARjNOoYzYIujs/OiJ7Tl3XNF5CVoevGt4mgEN+aRP/LrGJs8dwlZ0kPQxH1P7S5/ZGDeb8y1Nq6Q3EZFrGruF+Co4IBaDCCAWQwDgYDVR0PAQH/BAQDAgeAMIIBUAYKKwYBBAHWeQIBEQSCAUAwggE8AgFkCgEBAgFkCgEBBBtjaGFsbGVuZ2Utc2hvdWxkLWJlLWEtbm9uY2UEADBcv4U9CAIGAYxnZ5HQv4VFTARKMEgxIjAgBBtjb20uc2hlbjE5OTEua2V5YXR0ZXN0YXRpb24CAQExIgQg/CyJNxdNgXheUoQ4Czy9y1VZNCpp5ABdrKQBdi6icoMwgbChBTEDAgECogMCAQOjBAICAQClCzEJAgEEAgEFAgEGqgMCAQG/g3gDAgEDv4N5BAICASy/hT4DAgEAv4VATDBKBCAHrGWksyShKyYDh1pt6rID5/YqBpdIrguWTegi07S1VgEB/woBAAQg1IFxrl0udMEfKaH/FZjdH9agH67rUFAYHwlRpMqTJW2/hUEFAgMB+9C/hUIFAgMDFeS/hU4GAgQBNI0Rv4VPBgIEATSNETAKBggqhkjOPQQDAgNJADBGAiEAzCt+4nwOy9w4c2XyvFvee1+w3QtLA/EECrOBmI2WCAsCIQCigJ4qcBTRF2J36MXvYOzpF7yld2O+uKMGYyZwoJactA==
-----END CERTIFICATE-----


Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1 (0x1)
        Signature Algorithm: ecdsa-with-SHA256
        Issuer: title = TEE, serialNumber = d55fa1efe80f1eccf835449c03d8cde4
        Validity
            Not Before: Jan  1 00:00:00 1970 GMT
            Not After : Jan  1 00:00:00 2048 GMT
        Subject: CN = Android Keystore Key
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:63:34:ea:18:cd:82:2e:8e:cf:ce:88:9e:d3:97:
                    75:cd:17:90:95:a1:eb:c6:b7:89:a0:10:df:9a:44:
                    ff:cb:ac:62:6c:f1:dc:25:67:49:0f:43:11:f5:3f:
                    b4:b9:fd:91:83:79:bf:32:d4:da:ba:43:71:19:16:
                    b1:ab:b8:5f:82
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature
            1.3.6.1.4.1.11129.2.1.17:
                0..<..d
....d
....challenge-should-be-a-nonce..0\..=.....gg....EL.J0H1"0 ..com.shen1991.keyattestation...1". .,.7.M.x^R.8.<..UY4*i..]...v..r.0....1.................1.................x......y....,..>......@L0J. ..e..$.+&..Zm.....*..H...M."...V...
... ..q.].t..)...........PP...Q...%m..A........B........N....4....O....4..
    Signature Algorithm: ecdsa-with-SHA256
    Signature Value:
        30:46:02:21:00:cc:2b:7e:e2:7c:0e:cb:dc:38:73:65:f2:bc:
        5b:de:7b:5f:b0:dd:0b:4b:03:f1:04:0a:b3:81:98:8d:96:08:
        0b:02:21:00:a2:80:9e:2a:70:14:d1:17:62:77:e8:c5:ef:60:
        ec:e9:17:bc:a5:77:63:be:b8:a3:06:63:26:70:a0:96:9c:b4


SEQUENCE (8 elem)
  INTEGER 100
  ENUMERATED 1
  INTEGER 100
  ENUMERATED 1
  OCTET STRING (27 byte) challenge-should-be-a-nonce
Offset: 16
Length: 2+27
Value:
(27 byte)
challenge-should-be-a-nonce
  OCTET STRING (0 byte)
  SEQUENCE (2 elem)
    [701] (1 elem)
      INTEGER (41 bit) 1702541890000
    [709] (1 elem)
      OCTET STRING (74 byte) 304831223020041B636F6D2E7368656E313939312E6B65796174746573746174696F6E…
        SEQUENCE (2 elem)
          SET (1 elem)
            SEQUENCE (2 elem)
              OCTET STRING (27 byte) com.shen1991.keyattestation
              INTEGER 1
          SET (1 elem)
            OCTET STRING (32 byte) FC2C8937174D81785E5284380B3CBDCB5559342A69E4005DACA401762EA27283
  SEQUENCE (13 elem)
    [1] (1 elem)
      SET (1 elem)
        INTEGER 2
    [2] (1 elem)
      INTEGER 3
    [3] (1 elem)
      INTEGER 256
    [5] (1 elem)
      SET (3 elem)
        INTEGER 4
        INTEGER 5
        INTEGER 6
    [10] (1 elem)
      INTEGER 1
    [504] (1 elem)
      INTEGER 3
    [505] (1 elem)
      INTEGER 300
    [702] (1 elem)
      INTEGER 0
    [704] (1 elem)
      SEQUENCE (4 elem)
        OCTET STRING (32 byte) 07AC65A4B324A12B2603875A6DEAB203E7F62A069748AE0B964DE822D3B4B556
        BOOLEAN true
        ENUMERATED 0
        OCTET STRING (32 byte) D48171AE5D2E74C11F29A1FF1598DD1FD6A01FAEEB5050181F0951A4CA93256D
    [705] (1 elem)
      INTEGER 130000
    [706] (1 elem)
      INTEGER 202212
    [718] (1 elem)
      INTEGER 20221201
    [719] (1 elem)
      INTEGER 20221201

```

#### 2.4.2 cert 1
```
-----BEGIN CERTIFICATE-----
MIIB9DCCAXmgAwIBAgIQHIZ9phan3gKkaGygX/hMazAKBggqhkjOPQQDAjA5MQwwCgYDVQQMDANURUUxKTAnBgNVBAUTIGFmYjE1NDk5MDZjMjdhODRlZGE0NmYyNDgxM2ZiZmEwMB4XDTIyMDEyNTIzMjYxN1oXDTMyMDEyMzIzMjYxN1owOTEMMAoGA1UEDAwDVEVFMSkwJwYDVQQFEyBkNTVmYTFlZmU4MGYxZWNjZjgzNTQ0OWMwM2Q4Y2RlNDBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABIpbsUYv0MFsaajpCGoyvIaWnQIzVccK/UGXUoNN+EZz7lBKBNZtmkQ/Q2ebyZHt05oLwbQe+zuzmEzQFgDRyNCjYzBhMB0GA1UdDgQWBBR7imZZjjfMrcTEKG+UdkjTErLiWjAfBgNVHSMEGDAWgBQIT1dJ1ceCjitQ8mWbIeGpq4WBvDAPBgNVHRMBAf8EBTADAQH/MA4GA1UdDwEB/wQEAwICBDAKBggqhkjOPQQDAgNpADBmAjEA+gMugmp/OjHvxGxluu8cphvktmgZw2Ik2buyQWxbF8RhNM2V08ll6rj7ZeLK2qOmAjEAy4J/HuoglcR06VVyHHB3b/1eZFrSuuU8IYSRUSx9iiumYPwxme0bhzfxxbpIxwCF
-----END CERTIFICATE-----

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            1c:86:7d:a6:16:a7:de:02:a4:68:6c:a0:5f:f8:4c:6b
        Signature Algorithm: ecdsa-with-SHA256
        Issuer: title = TEE, serialNumber = afb1549906c27a84eda46f24813fbfa0
        Validity
            Not Before: Jan 25 23:26:17 2022 GMT
            Not After : Jan 23 23:26:17 2032 GMT
        Subject: title = TEE, serialNumber = d55fa1efe80f1eccf835449c03d8cde4
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:8a:5b:b1:46:2f:d0:c1:6c:69:a8:e9:08:6a:32:
                    bc:86:96:9d:02:33:55:c7:0a:fd:41:97:52:83:4d:
                    f8:46:73:ee:50:4a:04:d6:6d:9a:44:3f:43:67:9b:
                    c9:91:ed:d3:9a:0b:c1:b4:1e:fb:3b:b3:98:4c:d0:
                    16:00:d1:c8:d0
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                7B:8A:66:59:8E:37:CC:AD:C4:C4:28:6F:94:76:48:D3:12:B2:E2:5A
            X509v3 Authority Key Identifier:
                08:4F:57:49:D5:C7:82:8E:2B:50:F2:65:9B:21:E1:A9:AB:85:81:BC
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Certificate Sign
    Signature Algorithm: ecdsa-with-SHA256
    Signature Value:
        30:66:02:31:00:fa:03:2e:82:6a:7f:3a:31:ef:c4:6c:65:ba:
        ef:1c:a6:1b:e4:b6:68:19:c3:62:24:d9:bb:b2:41:6c:5b:17:
        c4:61:34:cd:95:d3:c9:65:ea:b8:fb:65:e2:ca:da:a3:a6:02:
        31:00:cb:82:7f:1e:ea:20:95:c4:74:e9:55:72:1c:70:77:6f:
        fd:5e:64:5a:d2:ba:e5:3c:21:84:91:51:2c:7d:8a:2b:a6:60:
        fc:31:99:ed:1b:87:37:f1:c5:ba:48:c7:00:85
```

#### 2.4.3 cert 2
```
-----BEGIN CERTIFICATE-----
MIIDkzCCAXugAwIBAgIQdf0ZJTfo+lrb+ci6uj5PtjANBgkqhkiG9w0BAQsFADAbMRkwFwYDVQQFExBmOTIwMDllODUzYjZiMDQ1MB4XDTIyMDEyNTIzMjUxNloXDTMyMDEyMzIzMjUxNlowOTEMMAoGA1UEDAwDVEVFMSkwJwYDVQQFEyBhZmIxNTQ5OTA2YzI3YTg0ZWRhNDZmMjQ4MTNmYmZhMDB2MBAGByqGSM49AgEGBSuBBAAiA2IABGAkq81HgjL+/1QZCD20OkhUaEwp/nvoFG8WocNgWiiwJOrkDfXcjQQXrrboa9w22O7Tr6d/BGzgnz3R0kd+W5QG8+S0XbtLyte1jek9/L8aBR4VnFstUCHcVi9jiBfsTqNjMGEwHQYDVR0OBBYEFAhPV0nVx4KOK1DyZZsh4amrhYG8MB8GA1UdIwQYMBaAFDZh4QB8iAUJUYtEbEf/GkzJ6k8SMA8GA1UdEwEB/wQFMAMBAf8wDgYDVR0PAQH/BAQDAgIEMA0GCSqGSIb3DQEBCwUAA4ICAQAM+sjCRY7aM8JItYZYK1TtypnE58A6VjusH+2kUPBAHe6gfHF0ofLajO2HG2caaPcuuYEP4CF2r+w0aBfDLRLL/LYbMuJ51eb8KpQm+TzUeUrA9QxEIRHUhriNptsx9ioAOK7c7SFOtIFTFd4fUTzcwYbWQy3BcIiLG/pjHNnI8lfSnHXBopFn2PfF6vs9Xho3efcSbxvEVvUB/eX+qXaVyGLKZIkDewwdL4mHC8UCmQdHMzEAerbb0imv0yTh5ItbH7s4EmZbghtZ3IEIDwU6gvvWiO8vCLjHpz9JCJDIqdLGLuGVmFn7MMlZBKJqDBr8NDQ+Wf36ag4SX3pT+kBDj8X89eoAQVuIZZZ0IyWp8cdXzPicsFQgtLKvxTULfT3jWW/yneUAM+qxQLAAAe9MnM/UtstospDQ/8u/oZLNRuT/0GmD3Yahm3pZaB4RH5qgm/Vf0tNt6bH3+K0NSFZfSskdxZagpvkJIGIdM5JjLAdGktxBpjFhrKpxG739pDVnVXbfWOQmHppf11Sh82vDpVlq6k9+W1fmVWExlIgTEO3/QTGB4rfFzlYCB6vOhjspaUjgr8mobMZfEic6hu7zMKxXmpipyRFSxyLNXUOvEXc/mSkGDErDzrLwPATFIuJeTe2QO/mwvWXD7amMGRo4qzSeU+N6AYM4fIKEE1LhNA==
-----END CERTIFICATE-----


Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            75:fd:19:25:37:e8:fa:5a:db:f9:c8:ba:ba:3e:4f:b6
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: serialNumber = f92009e853b6b045
        Validity
            Not Before: Jan 25 23:25:16 2022 GMT
            Not After : Jan 23 23:25:16 2032 GMT
        Subject: title = TEE, serialNumber = afb1549906c27a84eda46f24813fbfa0
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (384 bit)
                pub:
                    04:60:24:ab:cd:47:82:32:fe:ff:54:19:08:3d:b4:
                    3a:48:54:68:4c:29:fe:7b:e8:14:6f:16:a1:c3:60:
                    5a:28:b0:24:ea:e4:0d:f5:dc:8d:04:17:ae:b6:e8:
                    6b:dc:36:d8:ee:d3:af:a7:7f:04:6c:e0:9f:3d:d1:
                    d2:47:7e:5b:94:06:f3:e4:b4:5d:bb:4b:ca:d7:b5:
                    8d:e9:3d:fc:bf:1a:05:1e:15:9c:5b:2d:50:21:dc:
                    56:2f:63:88:17:ec:4e
                ASN1 OID: secp384r1
                NIST CURVE: P-384
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                08:4F:57:49:D5:C7:82:8E:2B:50:F2:65:9B:21:E1:A9:AB:85:81:BC
            X509v3 Authority Key Identifier:
                36:61:E1:00:7C:88:05:09:51:8B:44:6C:47:FF:1A:4C:C9:EA:4F:12
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Certificate Sign
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        0c:fa:c8:c2:45:8e:da:33:c2:48:b5:86:58:2b:54:ed:ca:99:
        c4:e7:c0:3a:56:3b:ac:1f:ed:a4:50:f0:40:1d:ee:a0:7c:71:
        74:a1:f2:da:8c:ed:87:1b:67:1a:68:f7:2e:b9:81:0f:e0:21:
        76:af:ec:34:68:17:c3:2d:12:cb:fc:b6:1b:32:e2:79:d5:e6:
        fc:2a:94:26:f9:3c:d4:79:4a:c0:f5:0c:44:21:11:d4:86:b8:
        8d:a6:db:31:f6:2a:00:38:ae:dc:ed:21:4e:b4:81:53:15:de:
        1f:51:3c:dc:c1:86:d6:43:2d:c1:70:88:8b:1b:fa:63:1c:d9:
        c8:f2:57:d2:9c:75:c1:a2:91:67:d8:f7:c5:ea:fb:3d:5e:1a:
        37:79:f7:12:6f:1b:c4:56:f5:01:fd:e5:fe:a9:76:95:c8:62:
        ca:64:89:03:7b:0c:1d:2f:89:87:0b:c5:02:99:07:47:33:31:
        00:7a:b6:db:d2:29:af:d3:24:e1:e4:8b:5b:1f:bb:38:12:66:
        5b:82:1b:59:dc:81:08:0f:05:3a:82:fb:d6:88:ef:2f:08:b8:
        c7:a7:3f:49:08:90:c8:a9:d2:c6:2e:e1:95:98:59:fb:30:c9:
        59:04:a2:6a:0c:1a:fc:34:34:3e:59:fd:fa:6a:0e:12:5f:7a:
        53:fa:40:43:8f:c5:fc:f5:ea:00:41:5b:88:65:96:74:23:25:
        a9:f1:c7:57:cc:f8:9c:b0:54:20:b4:b2:af:c5:35:0b:7d:3d:
        e3:59:6f:f2:9d:e5:00:33:ea:b1:40:b0:00:01:ef:4c:9c:cf:
        d4:b6:cb:68:b2:90:d0:ff:cb:bf:a1:92:cd:46:e4:ff:d0:69:
        83:dd:86:a1:9b:7a:59:68:1e:11:1f:9a:a0:9b:f5:5f:d2:d3:
        6d:e9:b1:f7:f8:ad:0d:48:56:5f:4a:c9:1d:c5:96:a0:a6:f9:
        09:20:62:1d:33:92:63:2c:07:46:92:dc:41:a6:31:61:ac:aa:
        71:1b:bd:fd:a4:35:67:55:76:df:58:e4:26:1e:9a:5f:d7:54:
        a1:f3:6b:c3:a5:59:6a:ea:4f:7e:5b:57:e6:55:61:31:94:88:
        13:10:ed:ff:41:31:81:e2:b7:c5:ce:56:02:07:ab:ce:86:3b:
        29:69:48:e0:af:c9:a8:6c:c6:5f:12:27:3a:86:ee:f3:30:ac:
        57:9a:98:a9:c9:11:52:c7:22:cd:5d:43:af:11:77:3f:99:29:
        06:0c:4a:c3:ce:b2:f0:3c:04:c5:22:e2:5e:4d:ed:90:3b:f9:
        b0:bd:65:c3:ed:a9:8c:19:1a:38:ab:34:9e:53:e3:7a:01:83:
        38:7c:82:84:13:52:e1:34
```

#### 2.4.4 cert 3
```
-----BEGIN CERTIFICATE-----
MIIFHDCCAwSgAwIBAgIJAMNrfES5rhgxMA0GCSqGSIb3DQEBCwUAMBsxGTAXBgNVBAUTEGY5MjAwOWU4NTNiNmIwNDUwHhcNMjExMTE3MjMxMDQyWhcNMzYxMTEzMjMxMDQyWjAbMRkwFwYDVQQFExBmOTIwMDllODUzYjZiMDQ1MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAr7bHgiuxpwHsK7Qui8xUFmOr75gvMsd/dTEDDJdSSxtf6An7xyqpRR90PL2abxM1dEqlXnf2tqw1Ne4Xwl5jlRfdnJLmN0pTy/4lj4/7tv0Sk3iiKkypnEUtR6WfMgH0QZfKHM1+di+y9TFRtv6y//0rb+T+W8a9nsNL/ggjnar86461qO0rOs2cXjp3kOG1FEJ5MVmFmBGtnrKpa73XpXyTqRxB/M0n1n/W9nGqC4FSYa04T6N5RIZGBN2z2MT5IKGbFlbC8UrW0DxW7AYImQQcHtGl/m00QLVWutHQoVJYnFPlXTcHYvASLu+RhhsbDmxMgJJ0mcDpvsC4PjvB+TxywElgS70vE0XmLD+OJtvsBslHZvPBKCOdT0MS+tgSOIfga+z1Z1g7+DVagf7quvmag8jfPioyKvxnK/EgsTUVi2ghzq8wm27ud/mIM7AY2qEORR8Go3TVB4HzWQgpZrt3i5MIlCaY504LzSRiigHCzAPlHws+W0rB5N+er5/2pJKnfBSDiCiFAVtCLOZ7gLiMm0jhO2B6tUXHI/+MRPjy02i59lINMRRev56GKtcd9qO/0kUJWdZTdA2XoS82ixPvZtXQpUpuL12ab+9EaDK8Z4RHJYYfCT3Q5vNAXaiWQ+8PTWm2QgBR/bkwSWc+NpUFgNPN9PvQi8WEg5UmAGMCAwEAAaNjMGEwHQYDVR0OBBYEFDZh4QB8iAUJUYtEbEf/GkzJ6k8SMB8GA1UdIwQYMBaAFDZh4QB8iAUJUYtEbEf/GkzJ6k8SMA8GA1UdEwEB/wQFMAMBAf8wDgYDVR0PAQH/BAQDAgIEMA0GCSqGSIb3DQEBCwUAA4ICAQBTNNZe5cuf8oiq+jV0itTGzWVhSTjOBEk2FQvh11J3o3lna0o7rd8RFHnN00q4hi6TapFhh4qaw/iG6Xg+xOan63niLWIC5GOPFgPeYXM9+nBb3zZzC8ABypYuCusWCmt6Tn3+Pjbz3MTVhRGXuT/TQH4KGFY4PhvzAyXwdjTOCXID+aHud4RLcSySr0Fq/L+R8TWalvM1wJJPhyRjqRCJerGtfBagiALzvhnmY7U1qFcS0NCnKjoO7oFedKdWlZz0YAfu3aGCJd4KHT0MsGiLZez9WP81xYSrKMNEsDK+zK5fVzw6jA7cxmpXcARTnmAuGUeI7VVDhDzKeVOctf3a0qQLwC+d0+xrETZ4r2fRGNw2YEs2W8Qj6oDcfPvq9JySe7pJ6wcHnl5EZ0lwc4xH7Y4Dx9RA1JlfooLMw3tOdJZH0enxPXaydfAD3YifeZpFaUzicHeLzVJLt9dvGB0bHQLE4+EqKFgOZv2EoP686DQqbVS1u+9k0p2xbMA105TBIk7npraa8VM0fnrRKi7wlZKwdH+aNAyhbXRW9xsnODJ+g8eF452zvbiKKngEKirK5LGieoXBX7tZ9D1GNBH2Ob3bKOwwIWdEFle/YF/h6zWgdeoaNGDqVBrLr2+0DtWoiB1aDEjLWl9FmyIUyUm7mD/vFDkzF+wm7cyWpQpCVQ==
-----END CERTIFICATE-----


Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            c3:6b:7c:44:b9:ae:18:31
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: serialNumber = f92009e853b6b045
        Validity
            Not Before: Nov 17 23:10:42 2021 GMT
            Not After : Nov 13 23:10:42 2036 GMT
        Subject: serialNumber = f92009e853b6b045
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
                Modulus:
                    00:af:b6:c7:82:2b:b1:a7:01:ec:2b:b4:2e:8b:cc:
                    54:16:63:ab:ef:98:2f:32:c7:7f:75:31:03:0c:97:
                    52:4b:1b:5f:e8:09:fb:c7:2a:a9:45:1f:74:3c:bd:
                    9a:6f:13:35:74:4a:a5:5e:77:f6:b6:ac:35:35:ee:
                    17:c2:5e:63:95:17:dd:9c:92:e6:37:4a:53:cb:fe:
                    25:8f:8f:fb:b6:fd:12:93:78:a2:2a:4c:a9:9c:45:
                    2d:47:a5:9f:32:01:f4:41:97:ca:1c:cd:7e:76:2f:
                    b2:f5:31:51:b6:fe:b2:ff:fd:2b:6f:e4:fe:5b:c6:
                    bd:9e:c3:4b:fe:08:23:9d:aa:fc:eb:8e:b5:a8:ed:
                    2b:3a:cd:9c:5e:3a:77:90:e1:b5:14:42:79:31:59:
                    85:98:11:ad:9e:b2:a9:6b:bd:d7:a5:7c:93:a9:1c:
                    41:fc:cd:27:d6:7f:d6:f6:71:aa:0b:81:52:61:ad:
                    38:4f:a3:79:44:86:46:04:dd:b3:d8:c4:f9:20:a1:
                    9b:16:56:c2:f1:4a:d6:d0:3c:56:ec:06:08:99:04:
                    1c:1e:d1:a5:fe:6d:34:40:b5:56:ba:d1:d0:a1:52:
                    58:9c:53:e5:5d:37:07:62:f0:12:2e:ef:91:86:1b:
                    1b:0e:6c:4c:80:92:74:99:c0:e9:be:c0:b8:3e:3b:
                    c1:f9:3c:72:c0:49:60:4b:bd:2f:13:45:e6:2c:3f:
                    8e:26:db:ec:06:c9:47:66:f3:c1:28:23:9d:4f:43:
                    12:fa:d8:12:38:87:e0:6b:ec:f5:67:58:3b:f8:35:
                    5a:81:fe:ea:ba:f9:9a:83:c8:df:3e:2a:32:2a:fc:
                    67:2b:f1:20:b1:35:15:8b:68:21:ce:af:30:9b:6e:
                    ee:77:f9:88:33:b0:18:da:a1:0e:45:1f:06:a3:74:
                    d5:07:81:f3:59:08:29:66:bb:77:8b:93:08:94:26:
                    98:e7:4e:0b:cd:24:62:8a:01:c2:cc:03:e5:1f:0b:
                    3e:5b:4a:c1:e4:df:9e:af:9f:f6:a4:92:a7:7c:14:
                    83:88:28:85:01:5b:42:2c:e6:7b:80:b8:8c:9b:48:
                    e1:3b:60:7a:b5:45:c7:23:ff:8c:44:f8:f2:d3:68:
                    b9:f6:52:0d:31:14:5e:bf:9e:86:2a:d7:1d:f6:a3:
                    bf:d2:45:09:59:d6:53:74:0d:97:a1:2f:36:8b:13:
                    ef:66:d5:d0:a5:4a:6e:2f:5d:9a:6f:ef:44:68:32:
                    bc:67:84:47:25:86:1f:09:3d:d0:e6:f3:40:5d:a8:
                    96:43:ef:0f:4d:69:b6:42:00:51:fd:b9:30:49:67:
                    3e:36:95:05:80:d3:cd:f4:fb:d0:8b:c5:84:83:95:
                    26:00:63
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                36:61:E1:00:7C:88:05:09:51:8B:44:6C:47:FF:1A:4C:C9:EA:4F:12
            X509v3 Authority Key Identifier:
                36:61:E1:00:7C:88:05:09:51:8B:44:6C:47:FF:1A:4C:C9:EA:4F:12
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Certificate Sign
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        53:34:d6:5e:e5:cb:9f:f2:88:aa:fa:35:74:8a:d4:c6:cd:65:
        61:49:38:ce:04:49:36:15:0b:e1:d7:52:77:a3:79:67:6b:4a:
        3b:ad:df:11:14:79:cd:d3:4a:b8:86:2e:93:6a:91:61:87:8a:
        9a:c3:f8:86:e9:78:3e:c4:e6:a7:eb:79:e2:2d:62:02:e4:63:
        8f:16:03:de:61:73:3d:fa:70:5b:df:36:73:0b:c0:01:ca:96:
        2e:0a:eb:16:0a:6b:7a:4e:7d:fe:3e:36:f3:dc:c4:d5:85:11:
        97:b9:3f:d3:40:7e:0a:18:56:38:3e:1b:f3:03:25:f0:76:34:
        ce:09:72:03:f9:a1:ee:77:84:4b:71:2c:92:af:41:6a:fc:bf:
        91:f1:35:9a:96:f3:35:c0:92:4f:87:24:63:a9:10:89:7a:b1:
        ad:7c:16:a0:88:02:f3:be:19:e6:63:b5:35:a8:57:12:d0:d0:
        a7:2a:3a:0e:ee:81:5e:74:a7:56:95:9c:f4:60:07:ee:dd:a1:
        82:25:de:0a:1d:3d:0c:b0:68:8b:65:ec:fd:58:ff:35:c5:84:
        ab:28:c3:44:b0:32:be:cc:ae:5f:57:3c:3a:8c:0e:dc:c6:6a:
        57:70:04:53:9e:60:2e:19:47:88:ed:55:43:84:3c:ca:79:53:
        9c:b5:fd:da:d2:a4:0b:c0:2f:9d:d3:ec:6b:11:36:78:af:67:
        d1:18:dc:36:60:4b:36:5b:c4:23:ea:80:dc:7c:fb:ea:f4:9c:
        92:7b:ba:49:eb:07:07:9e:5e:44:67:49:70:73:8c:47:ed:8e:
        03:c7:d4:40:d4:99:5f:a2:82:cc:c3:7b:4e:74:96:47:d1:e9:
        f1:3d:76:b2:75:f0:03:dd:88:9f:79:9a:45:69:4c:e2:70:77:
        8b:cd:52:4b:b7:d7:6f:18:1d:1b:1d:02:c4:e3:e1:2a:28:58:
        0e:66:fd:84:a0:fe:bc:e8:34:2a:6d:54:b5:bb:ef:64:d2:9d:
        b1:6c:c0:35:d3:94:c1:22:4e:e7:a6:b6:9a:f1:53:34:7e:7a:
        d1:2a:2e:f0:95:92:b0:74:7f:9a:34:0c:a1:6d:74:56:f7:1b:
        27:38:32:7e:83:c7:85:e3:9d:b3:bd:b8:8a:2a:78:04:2a:2a:
        ca:e4:b1:a2:7a:85:c1:5f:bb:59:f4:3d:46:34:11:f6:39:bd:
        db:28:ec:30:21:67:44:16:57:bf:60:5f:e1:eb:35:a0:75:ea:
        1a:34:60:ea:54:1a:cb:af:6f:b4:0e:d5:a8:88:1d:5a:0c:48:
        cb:5a:5f:45:9b:22:14:c9:49:bb:98:3f:ef:14:39:33:17:ec:
        26:ed:cc:96:a5:0a:42:55
```

## 0x03 Android Key Attestation Vulnerabilities?

- 假设严格根据Google文档进行使用, 没有已知风险
  1. 服务端生成challenge(注意新鲜数)
  2. 在**服务端**对证书链进行校验, 校验根证书及CRL列表
  3. 在**服务端**校验cert0的扩展字段, 分析BL状态、安卓及patch版本, 确保手机无已知漏洞
  4. 注意生成的cert3(根证书)与[Google公开的](https://developer.android.com/privacy-and-security/security-key-attestation#root_certificate)一致


那么实践中有哪些风险呢
1. 手机虽未解锁BL但有已知RCE漏洞 -> 破坏了条件3
2. 未校验设备BL解锁状态 -> 破坏了
3. 服务端challenge重放 -> 破坏了条件1, 实际利用方式暂未研究
4. 本地校验证书链 -> 破坏了条件2
5. 只校验证书链, 不校验根证书 -> 破坏了条件4

另外Google的文档中提到了这样一个细节

{% highlight text %}
If the root certificate doesn't contain the public key on this page, there are two likely reasons:

Most likely, the device launched with an Android version less than 7.0 and it doesn't support hardware attestation. In this case, Android has a software implementation of attestation that produces the same sort of attestation certificate, but signed with a key hardcoded in Android source code. Because this signing key isn't a secret, the attestation might have been created by an attacker pretending to provide secure hardware.
The other likely reason is that the device isn't a Google Play device. In that case, the device maker is free to create their own root and to make whatever claims they like about what the attestation means. Refer to the device maker's documentation. Note that as of this writing Google isn't aware of any device makers who have done this.
{% endhighlight %}


```cpp
static const uint8_t kEcAttestKey[] = {
    0x30, 0x77, 0x02, 0x01, 0x01, 0x04, 0x20, 0x21, 0xe0, 0x86, 0x43, 0x2a, 0x15, 0x19, 0x84, 0x59,
    0xcf, 0x36, 0x3a, 0x50, 0xfc, 0x14, 0xc9, 0xda, 0xad, 0xf9, 0x35, 0xf5, 0x27, 0xc2, 0xdf, 0xd7,
    0x1e, 0x4d, 0x6d, 0xbc, 0x42, 0xe5, 0x44, 0xa0, 0x0a, 0x06, 0x08, 0x2a, 0x86, 0x48, 0xce, 0x3d,
    0x03, 0x01, 0x07, 0xa1, 0x44, 0x03, 0x42, 0x00, 0x04, 0xeb, 0x9e, 0x79, 0xf8, 0x42, 0x63, 0x59,
    0xac, 0xcb, 0x2a, 0x91, 0x4c, 0x89, 0x86, 0xcc, 0x70, 0xad, 0x90, 0x66, 0x93, 0x82, 0xa9, 0x73,
    0x26, 0x13, 0xfe, 0xac, 0xcb, 0xf8, 0x21, 0x27, 0x4c, 0x21, 0x74, 0x97, 0x4a, 0x2a, 0xfe, 0xa5,
    0xb9, 0x4d, 0x7f, 0x66, 0xd4, 0xe0, 0x65, 0x10, 0x66, 0x35, 0xbc, 0x53, 0xb7, 0xa0, 0xa3, 0xa6,
    0x71, 0x58, 0x3e, 0xdb, 0x3e, 0x11, 0xae, 0x10, 0x14,
};
static const keymaster_key_blob_t kEcAttestKeyBlob = {
        (const uint8_t*)&kEcAttestKey, sizeof(kEcAttestKey)
};
static const uint8_t kEcAttestCert[] = {
    0x30, 0x82, 0x02, 0x78, 0x30, 0x82, 0x02, 0x1e, 0xa0, 0x03, 0x02, 0x01, 0x02, 0x02, 0x02, 0x10,
    0x01, 0x30, 0x0a, 0x06, 0x08, 0x2a, 0x86, 0x48, 0xce, 0x3d, 0x04, 0x03, 0x02, 0x30, 0x81, 0x98,
    0x31, 0x0b, 0x30, 0x09, 0x06, 0x03, 0x55, 0x04, 0x06, 0x13, 0x02, 0x55, 0x53, 0x31, 0x13, 0x30,
    0x11, 0x06, 0x03, 0x55, 0x04, 0x08, 0x0c, 0x0a, 0x43, 0x61, 0x6c, 0x69, 0x66, 0x6f, 0x72, 0x6e,
    0x69, 0x61, 0x31, 0x16, 0x30, 0x14, 0x06, 0x03, 0x55, 0x04, 0x07, 0x0c, 0x0d, 0x4d, 0x6f, 0x75,
    0x6e, 0x74, 0x61, 0x69, 0x6e, 0x20, 0x56, 0x69, 0x65, 0x77, 0x31, 0x15, 0x30, 0x13, 0x06, 0x03,
    0x55, 0x04, 0x0a, 0x0c, 0x0c, 0x47, 0x6f, 0x6f, 0x67, 0x6c, 0x65, 0x2c, 0x20, 0x49, 0x6e, 0x63,
    0x2e, 0x31, 0x10, 0x30, 0x0e, 0x06, 0x03, 0x55, 0x04, 0x0b, 0x0c, 0x07, 0x41, 0x6e, 0x64, 0x72,
    0x6f, 0x69, 0x64, 0x31, 0x33, 0x30, 0x31, 0x06, 0x03, 0x55, 0x04, 0x03, 0x0c, 0x2a, 0x41, 0x6e,
    0x64, 0x72, 0x6f, 0x69, 0x64, 0x20, 0x4b, 0x65, 0x79, 0x73, 0x74, 0x6f, 0x72, 0x65, 0x20, 0x53,
    0x6f, 0x66, 0x74, 0x77, 0x61, 0x72, 0x65, 0x20, 0x41, 0x74, 0x74, 0x65, 0x73, 0x74, 0x61, 0x74,
    0x69, 0x6f, 0x6e, 0x20, 0x52, 0x6f, 0x6f, 0x74, 0x30, 0x1e, 0x17, 0x0d, 0x31, 0x36, 0x30, 0x31,
    0x31, 0x31, 0x30, 0x30, 0x34, 0x36, 0x30, 0x39, 0x5a, 0x17, 0x0d, 0x32, 0x36, 0x30, 0x31, 0x30,
    0x38, 0x30, 0x30, 0x34, 0x36, 0x30, 0x39, 0x5a, 0x30, 0x81, 0x88, 0x31, 0x0b, 0x30, 0x09, 0x06,
    0x03, 0x55, 0x04, 0x06, 0x13, 0x02, 0x55, 0x53, 0x31, 0x13, 0x30, 0x11, 0x06, 0x03, 0x55, 0x04,
    0x08, 0x0c, 0x0a, 0x43, 0x61, 0x6c, 0x69, 0x66, 0x6f, 0x72, 0x6e, 0x69, 0x61, 0x31, 0x15, 0x30,
    0x13, 0x06, 0x03, 0x55, 0x04, 0x0a, 0x0c, 0x0c, 0x47, 0x6f, 0x6f, 0x67, 0x6c, 0x65, 0x2c, 0x20,
    0x49, 0x6e, 0x63, 0x2e, 0x31, 0x10, 0x30, 0x0e, 0x06, 0x03, 0x55, 0x04, 0x0b, 0x0c, 0x07, 0x41,
    0x6e, 0x64, 0x72, 0x6f, 0x69, 0x64, 0x31, 0x3b, 0x30, 0x39, 0x06, 0x03, 0x55, 0x04, 0x03, 0x0c,
    0x32, 0x41, 0x6e, 0x64, 0x72, 0x6f, 0x69, 0x64, 0x20, 0x4b, 0x65, 0x79, 0x73, 0x74, 0x6f, 0x72,
    0x65, 0x20, 0x53, 0x6f, 0x66, 0x74, 0x77, 0x61, 0x72, 0x65, 0x20, 0x41, 0x74, 0x74, 0x65, 0x73,
    0x74, 0x61, 0x74, 0x69, 0x6f, 0x6e, 0x20, 0x49, 0x6e, 0x74, 0x65, 0x72, 0x6d, 0x65, 0x64, 0x69,
    0x61, 0x74, 0x65, 0x30, 0x59, 0x30, 0x13, 0x06, 0x07, 0x2a, 0x86, 0x48, 0xce, 0x3d, 0x02, 0x01,
    0x06, 0x08, 0x2a, 0x86, 0x48, 0xce, 0x3d, 0x03, 0x01, 0x07, 0x03, 0x42, 0x00, 0x04, 0xeb, 0x9e,
    0x79, 0xf8, 0x42, 0x63, 0x59, 0xac, 0xcb, 0x2a, 0x91, 0x4c, 0x89, 0x86, 0xcc, 0x70, 0xad, 0x90,
    0x66, 0x93, 0x82, 0xa9, 0x73, 0x26, 0x13, 0xfe, 0xac, 0xcb, 0xf8, 0x21, 0x27, 0x4c, 0x21, 0x74,
    0x97, 0x4a, 0x2a, 0xfe, 0xa5, 0xb9, 0x4d, 0x7f, 0x66, 0xd4, 0xe0, 0x65, 0x10, 0x66, 0x35, 0xbc,
    0x53, 0xb7, 0xa0, 0xa3, 0xa6, 0x71, 0x58, 0x3e, 0xdb, 0x3e, 0x11, 0xae, 0x10, 0x14, 0xa3, 0x66,
    0x30, 0x64, 0x30, 0x1d, 0x06, 0x03, 0x55, 0x1d, 0x0e, 0x04, 0x16, 0x04, 0x14, 0x3f, 0xfc, 0xac,
    0xd6, 0x1a, 0xb1, 0x3a, 0x9e, 0x81, 0x20, 0xb8, 0xd5, 0x25, 0x1c, 0xc5, 0x65, 0xbb, 0x1e, 0x91,
    0xa9, 0x30, 0x1f, 0x06, 0x03, 0x55, 0x1d, 0x23, 0x04, 0x18, 0x30, 0x16, 0x80, 0x14, 0xc8, 0xad,
    0xe9, 0x77, 0x4c, 0x45, 0xc3, 0xa3, 0xcf, 0x0d, 0x16, 0x10, 0xe4, 0x79, 0x43, 0x3a, 0x21, 0x5a,
    0x30, 0xcf, 0x30, 0x12, 0x06, 0x03, 0x55, 0x1d, 0x13, 0x01, 0x01, 0xff, 0x04, 0x08, 0x30, 0x06,
    0x01, 0x01, 0xff, 0x02, 0x01, 0x00, 0x30, 0x0e, 0x06, 0x03, 0x55, 0x1d, 0x0f, 0x01, 0x01, 0xff,
    0x04, 0x04, 0x03, 0x02, 0x02, 0x84, 0x30, 0x0a, 0x06, 0x08, 0x2a, 0x86, 0x48, 0xce, 0x3d, 0x04,
    0x03, 0x02, 0x03, 0x48, 0x00, 0x30, 0x45, 0x02, 0x20, 0x4b, 0x8a, 0x9b, 0x7b, 0xee, 0x82, 0xbc,
    0xc0, 0x33, 0x87, 0xae, 0x2f, 0xc0, 0x89, 0x98, 0xb4, 0xdd, 0xc3, 0x8d, 0xab, 0x27, 0x2a, 0x45,
    0x9f, 0x69, 0x0c, 0xc7, 0xc3, 0x92, 0xd4, 0x0f, 0x8e, 0x02, 0x21, 0x00, 0xee, 0xda, 0x01, 0x5d,
    0xb6, 0xf4, 0x32, 0xe9, 0xd4, 0x84, 0x3b, 0x62, 0x4c, 0x94, 0x04, 0xef, 0x3a, 0x7c, 0xcc, 0xbd,
    0x5e, 0xfb, 0x22, 0xbb, 0xe7, 0xfe, 0xb9, 0x77, 0x3f, 0x59, 0x3f, 0xfb,
};
static const uint8_t kEcAttestRootCert[] = {
    0x30, 0x82, 0x02, 0x8b, 0x30, 0x82, 0x02, 0x32, 0xa0, 0x03, 0x02, 0x01, 0x02, 0x02, 0x09, 0x00,
    0xa2, 0x05, 0x9e, 0xd1, 0x0e, 0x43, 0x5b, 0x57, 0x30, 0x0a, 0x06, 0x08, 0x2a, 0x86, 0x48, 0xce,
    0x3d, 0x04, 0x03, 0x02, 0x30, 0x81, 0x98, 0x31, 0x0b, 0x30, 0x09, 0x06, 0x03, 0x55, 0x04, 0x06,
    0x13, 0x02, 0x55, 0x53, 0x31, 0x13, 0x30, 0x11, 0x06, 0x03, 0x55, 0x04, 0x08, 0x0c, 0x0a, 0x43,
    0x61, 0x6c, 0x69, 0x66, 0x6f, 0x72, 0x6e, 0x69, 0x61, 0x31, 0x16, 0x30, 0x14, 0x06, 0x03, 0x55,
    0x04, 0x07, 0x0c, 0x0d, 0x4d, 0x6f, 0x75, 0x6e, 0x74, 0x61, 0x69, 0x6e, 0x20, 0x56, 0x69, 0x65,
    0x77, 0x31, 0x15, 0x30, 0x13, 0x06, 0x03, 0x55, 0x04, 0x0a, 0x0c, 0x0c, 0x47, 0x6f, 0x6f, 0x67,
    0x6c, 0x65, 0x2c, 0x20, 0x49, 0x6e, 0x63, 0x2e, 0x31, 0x10, 0x30, 0x0e, 0x06, 0x03, 0x55, 0x04,
    0x0b, 0x0c, 0x07, 0x41, 0x6e, 0x64, 0x72, 0x6f, 0x69, 0x64, 0x31, 0x33, 0x30, 0x31, 0x06, 0x03,
    0x55, 0x04, 0x03, 0x0c, 0x2a, 0x41, 0x6e, 0x64, 0x72, 0x6f, 0x69, 0x64, 0x20, 0x4b, 0x65, 0x79,
    0x73, 0x74, 0x6f, 0x72, 0x65, 0x20, 0x53, 0x6f, 0x66, 0x74, 0x77, 0x61, 0x72, 0x65, 0x20, 0x41,
    0x74, 0x74, 0x65, 0x73, 0x74, 0x61, 0x74, 0x69, 0x6f, 0x6e, 0x20, 0x52, 0x6f, 0x6f, 0x74, 0x30,
    0x1e, 0x17, 0x0d, 0x31, 0x36, 0x30, 0x31, 0x31, 0x31, 0x30, 0x30, 0x34, 0x33, 0x35, 0x30, 0x5a,
    0x17, 0x0d, 0x33, 0x36, 0x30, 0x31, 0x30, 0x36, 0x30, 0x30, 0x34, 0x33, 0x35, 0x30, 0x5a, 0x30,
    0x81, 0x98, 0x31, 0x0b, 0x30, 0x09, 0x06, 0x03, 0x55, 0x04, 0x06, 0x13, 0x02, 0x55, 0x53, 0x31,
    0x13, 0x30, 0x11, 0x06, 0x03, 0x55, 0x04, 0x08, 0x0c, 0x0a, 0x43, 0x61, 0x6c, 0x69, 0x66, 0x6f,
    0x72, 0x6e, 0x69, 0x61, 0x31, 0x16, 0x30, 0x14, 0x06, 0x03, 0x55, 0x04, 0x07, 0x0c, 0x0d, 0x4d,
    0x6f, 0x75, 0x6e, 0x74, 0x61, 0x69, 0x6e, 0x20, 0x56, 0x69, 0x65, 0x77, 0x31, 0x15, 0x30, 0x13,
    0x06, 0x03, 0x55, 0x04, 0x0a, 0x0c, 0x0c, 0x47, 0x6f, 0x6f, 0x67, 0x6c, 0x65, 0x2c, 0x20, 0x49,
    0x6e, 0x63, 0x2e, 0x31, 0x10, 0x30, 0x0e, 0x06, 0x03, 0x55, 0x04, 0x0b, 0x0c, 0x07, 0x41, 0x6e,
    0x64, 0x72, 0x6f, 0x69, 0x64, 0x31, 0x33, 0x30, 0x31, 0x06, 0x03, 0x55, 0x04, 0x03, 0x0c, 0x2a,
    0x41, 0x6e, 0x64, 0x72, 0x6f, 0x69, 0x64, 0x20, 0x4b, 0x65, 0x79, 0x73, 0x74, 0x6f, 0x72, 0x65,
    0x20, 0x53, 0x6f, 0x66, 0x74, 0x77, 0x61, 0x72, 0x65, 0x20, 0x41, 0x74, 0x74, 0x65, 0x73, 0x74,
    0x61, 0x74, 0x69, 0x6f, 0x6e, 0x20, 0x52, 0x6f, 0x6f, 0x74, 0x30, 0x59, 0x30, 0x13, 0x06, 0x07,
    0x2a, 0x86, 0x48, 0xce, 0x3d, 0x02, 0x01, 0x06, 0x08, 0x2a, 0x86, 0x48, 0xce, 0x3d, 0x03, 0x01,
    0x07, 0x03, 0x42, 0x00, 0x04, 0xee, 0x5d, 0x5e, 0xc7, 0xe1, 0xc0, 0xdb, 0x6d, 0x03, 0xa6, 0x7e,
    0xe6, 0xb6, 0x1b, 0xec, 0x4d, 0x6a, 0x5d, 0x6a, 0x68, 0x2e, 0x0f, 0xff, 0x7f, 0x49, 0x0e, 0x7d,
    0x77, 0x1f, 0x44, 0x22, 0x6d, 0xbd, 0xb1, 0xaf, 0xfa, 0x16, 0xcb, 0xc7, 0xad, 0xc5, 0x77, 0xd2,
    0x56, 0x9c, 0xaa, 0xb7, 0xb0, 0x2d, 0x54, 0x01, 0x5d, 0x3e, 0x43, 0x2b, 0x2a, 0x8e, 0xd7, 0x4e,
    0xec, 0x48, 0x75, 0x41, 0xa4, 0xa3, 0x63, 0x30, 0x61, 0x30, 0x1d, 0x06, 0x03, 0x55, 0x1d, 0x0e,
    0x04, 0x16, 0x04, 0x14, 0xc8, 0xad, 0xe9, 0x77, 0x4c, 0x45, 0xc3, 0xa3, 0xcf, 0x0d, 0x16, 0x10,
    0xe4, 0x79, 0x43, 0x3a, 0x21, 0x5a, 0x30, 0xcf, 0x30, 0x1f, 0x06, 0x03, 0x55, 0x1d, 0x23, 0x04,
    0x18, 0x30, 0x16, 0x80, 0x14, 0xc8, 0xad, 0xe9, 0x77, 0x4c, 0x45, 0xc3, 0xa3, 0xcf, 0x0d, 0x16,
    0x10, 0xe4, 0x79, 0x43, 0x3a, 0x21, 0x5a, 0x30, 0xcf, 0x30, 0x0f, 0x06, 0x03, 0x55, 0x1d, 0x13,
    0x01, 0x01, 0xff, 0x04, 0x05, 0x30, 0x03, 0x01, 0x01, 0xff, 0x30, 0x0e, 0x06, 0x03, 0x55, 0x1d,
    0x0f, 0x01, 0x01, 0xff, 0x04, 0x04, 0x03, 0x02, 0x02, 0x84, 0x30, 0x0a, 0x06, 0x08, 0x2a, 0x86,
    0x48, 0xce, 0x3d, 0x04, 0x03, 0x02, 0x03, 0x47, 0x00, 0x30, 0x44, 0x02, 0x20, 0x35, 0x21, 0xa3,
    0xef, 0x8b, 0x34, 0x46, 0x1e, 0x9c, 0xd5, 0x60, 0xf3, 0x1d, 0x58, 0x89, 0x20, 0x6a, 0xdc, 0xa3,
    0x65, 0x41, 0xf6, 0x0d, 0x9e, 0xce, 0x8a, 0x19, 0x8c, 0x66, 0x48, 0x60, 0x7b, 0x02, 0x20, 0x4d,
    0x0b, 0xf3, 0x51, 0xd9, 0x30, 0x7c, 0x7d, 0x5b, 0xda, 0x35, 0x34, 0x1d, 0xa8, 0x47, 0x1b, 0x63,
    0xa5, 0x85, 0x65, 0x3c, 0xad, 0x4f, 0x24, 0xa7, 0xe7, 0x4d, 0xaf, 0x41, 0x7d, 0xf1, 0xbf,
};


```
