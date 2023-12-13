---
layout: post
title: "[WIP] Android & FIDO key attestation"
excerpt_separator: <!--more-->
date: 2023-12-16 00:00:00 +0800
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

- 根据 [Upgrading Android Attestation: Remote Provisioning](https://android-developers.googleblog.com/2022/03/upgrading-android-attestation-remote.html) 安卓12开始的验证机制改变了
- https://source.android.com/docs/security/features/keystore/attestation#expandable-1
- https://developer.android.com/privacy-and-security/keystore


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

- 在pixel 3A会生成如下的证书链(经过验证, 相同keyAlias返回的证书链一致, 与KeyProperties有关)
- cert0为attestation certificate, [attestation扩展字段的ASN.1 schema](https://source.android.com/docs/security/features/keystore/attestation#schema)
  
```
-----BEGIN CERTIFICATE-----
MIIClTCCAjugAwIBAgIBATAKBggqhkjOPQQDAjApMRkwFwYDVQQFExAxYzE3MmFlOWViMmMyNzg3MQwwCgYDVQQMDANURUUwIBcNNzAwMTAxMDAwMDAwWhgPMjEwNjAyMDcwNjI4MTVaMB8xHTAbBgNVBAMMFEFuZHJvaWQgS2V5c3RvcmUgS2V5MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEjdB2MWA/3v+MY3DYwA1mPjgBRRQAQTYrKF+8nKahGBlO3ElIQpEiCgPDZsv/gAVSB8w1uvBS7MEr34LQVhyf/aOCAVowggFWMA4GA1UdDwEB/wQEAwIHgDCCAUIGCisGAQQB1nkCAREEggEyMIIBLgIBAwoBAQIBBAoBAQQNMTcwMjQ1NTUzNjAxMQQAMFy/hT0IAgYBagODNqC/hUVMBEowSDEiMCAEG2NvbS5zaGVuMTk5MS5rZXlhdHRlc3RhdGlvbgIBATEiBCD8LIk3F02BeF5ShDgLPL3LVVk0KmnkAF2spAF2LqJygzCBsKEFMQMCAQKiAwIBA6MEAgIBAKULMQkCAQQCAQUCAQaqAwIBAb+DeAMCAQO/g3kEAgIBLL+FPgMCAQC/hUBMMEoEIIyomvGm2qdLAIEISTVt6SnPxEmO82r5ZHV73ooRO/RtAQH/CgEABCBNTud5A2eiWkUeg1kNiORXIjXApZX4U2bZgSblK994Qb+FQQUCAwHUwL+FQgUCAwMV3b+FTgYCBAE0ilm/hU8GAgQBNIpZMAoGCCqGSM49BAMCA0gAMEUCIAYGHK/afH+vEQ3FzyVuAHlDBuf0c/WjN18s/iyd9UjQAiEA3nK65HjMT/rl366WIgz+ftmUXAYOKkpABLFFckAzdAg=
-----END CERTIFICATE-----

-----BEGIN CERTIFICATE-----
MIICJjCCAaugAwIBAgIKBQEZiXBZcGSUkTAKBggqhkjOPQQDAjApMRkwFwYDVQQFExA0NDNkMjI4NGU5NmFiMjNiMQwwCgYDVQQMDANURUUwHhcNMTgxMjAzMjIzNDM1WhcNMjgxMTMwMjIzNDM1WjApMRkwFwYDVQQFExAxYzE3MmFlOWViMmMyNzg3MQwwCgYDVQQMDANURUUwWTATBgcqhkjOPQIBBggqhkjOPQMBBwNCAARUwB/D1Rb36KHnHZLegpyGlDdTlST1sf8qihs4afBp3Ph0Sbhhl/elouosSJppGpQ/eVHbMWV4NPX2EoiSKx4so4G6MIG3MB0GA1UdDgQWBBTF3ye+of2SeZJfghuQeyomYK+IPDAfBgNVHSMEGDAWgBTz+kTekgvidiuHITJW3HWiylgVkzAPBgNVHRMBAf8EBTADAQH/MA4GA1UdDwEB/wQEAwICBDBUBgNVHR8ETTBLMEmgR6BFhkNodHRwczovL2FuZHJvaWQuZ29vZ2xlYXBpcy5jb20vYXR0ZXN0YXRpb24vY3JsLzA1MDExOTg5NzA1OTcwNjQ5NDkxMAoGCCqGSM49BAMCA2kAMGYCMQDtW3tf/nIrThg6cbXyc8olLRV2t3/cvRp93zarRz6FDRrciKTcsimjwJ/8ZPMJNoUCMQCjEzXW6ZRIauWs3hy53eOtSzFHO5ZqZuDyWnyY6AEoX+/Us1wZgbUxT1U/Sr27F8M=
-----END CERTIFICATE-----

-----BEGIN CERTIFICATE-----
MIID0TCCAbmgAwIBAgIKA4gmZ2BliZaFzjANBgkqhkiG9w0BAQsFADAbMRkwFwYDVQQFExBmOTIwMDllODUzYjZiMDQ1MB4XDTE4MTIwMzIyMjQxNFoXDTI4MTEzMDIyMjQxNFowKTEZMBcGA1UEBRMQNDQzZDIyODRlOTZhYjIzYjEMMAoGA1UEDAwDVEVFMHYwEAYHKoZIzj0CAQYFK4EEACIDYgAE2tYXAmHqxDPYOcCe0QQ4/aMQ1kpkOFpiBsNnnfs0nsOR5ywGNHAbRNqWSKr8FYh/GuspoW2rT5JE3WLjAF9kHEQrOJ8HTThxJS9fl2VAfQS/2gJ64oHBabj6XB5qiqfAo4G2MIGzMB0GA1UdDgQWBBTz+kTekgvidiuHITJW3HWiylgVkzAfBgNVHSMEGDAWgBQ2YeEAfIgFCVGLRGxH/xpMyepPEjAPBgNVHRMBAf8EBTADAQH/MA4GA1UdDwEB/wQEAwICBDBQBgNVHR8ESTBHMEWgQ6BBhj9odHRwczovL2FuZHJvaWQuZ29vZ2xlYXBpcy5jb20vYXR0ZXN0YXRpb24vY3JsL0U4RkExOTYzMTREMkZBMTgwDQYJKoZIhvcNAQELBQADggIBAB9OWP309acpcELw3KHF5LrHsNLYfJsaAzem4gQplevmQlTfVQnok3h5zfmB99TgtlcOfTEO3SLZO4MwjK0oBVBL3AVxGN99+1WLjDEHm35hVqzFEZ93N41boFso+448hpl43BFJwRkAUg5NJ3vgrvGgPW7UhWSYE92ao+p5qmwzZlcIloX+tEVTR6yzowYlpYMacx5IKovoUX9DmATbufSt9iH65gtiJ4CmGM/U6qyMc7rPeA+eaW65NwRkRMOM3fKBSIdR1cde5nG62kkOchbJdGwyXM+Ux/zuCiXyyV0R7HMpmM3siOjDbP6lnGIt70iGVMaQ2JECy7GjAOaHOWZf1kUvZWDg6d15DsuHEDtk/oJIt6dtJ57kGJdhO0DtB3P5zW9VhjsUHrbdYCdzP0Fy5CNJFGo8c83f8yzQknLeGbAFFxHEhEff0F3rbneC26mU9UVaAcKo+o8HH1qnI0Twp9fieMUAwaiWfkjpkvzhl5nuFu4740eV1zC9zX9cGH9MMupoDrRvI42Djhd/7W6KJ1MQWu/7m/Xst7DeBTJSri++ZPcYtxuLpHiZEuvIzYBLGbLAjQ0OkDo/KDVtRrradd1P4746PW9Zfk6I3ALRjLrUT+q4UICLh+o+dtXTN0pNzlSqKWpV6yaFscwRQ4fdZ0grBzo0pbudDU26JmTm
-----END CERTIFICATE-----

-----BEGIN CERTIFICATE-----
MIIFYDCCA0igAwIBAgIJAOj6GWMU0voYMA0GCSqGSIb3DQEBCwUAMBsxGTAXBgNVBAUTEGY5MjAwOWU4NTNiNmIwNDUwHhcNMTYwNTI2MTYyODUyWhcNMjYwNTI0MTYyODUyWjAbMRkwFwYDVQQFExBmOTIwMDllODUzYjZiMDQ1MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAr7bHgiuxpwHsK7Qui8xUFmOr75gvMsd/dTEDDJdSSxtf6An7xyqpRR90PL2abxM1dEqlXnf2tqw1Ne4Xwl5jlRfdnJLmN0pTy/4lj4/7tv0Sk3iiKkypnEUtR6WfMgH0QZfKHM1+di+y9TFRtv6y//0rb+T+W8a9nsNL/ggjnar86461qO0rOs2cXjp3kOG1FEJ5MVmFmBGtnrKpa73XpXyTqRxB/M0n1n/W9nGqC4FSYa04T6N5RIZGBN2z2MT5IKGbFlbC8UrW0DxW7AYImQQcHtGl/m00QLVWutHQoVJYnFPlXTcHYvASLu+RhhsbDmxMgJJ0mcDpvsC4PjvB+TxywElgS70vE0XmLD+OJtvsBslHZvPBKCOdT0MS+tgSOIfga+z1Z1g7+DVagf7quvmag8jfPioyKvxnK/EgsTUVi2ghzq8wm27ud/mIM7AY2qEORR8Go3TVB4HzWQgpZrt3i5MIlCaY504LzSRiigHCzAPlHws+W0rB5N+er5/2pJKnfBSDiCiFAVtCLOZ7gLiMm0jhO2B6tUXHI/+MRPjy02i59lINMRRev56GKtcd9qO/0kUJWdZTdA2XoS82ixPvZtXQpUpuL12ab+9EaDK8Z4RHJYYfCT3Q5vNAXaiWQ+8PTWm2QgBR/bkwSWc+NpUFgNPN9PvQi8WEg5UmAGMCAwEAAaOBpjCBozAdBgNVHQ4EFgQUNmHhAHyIBQlRi0RsR/8aTMnqTxIwHwYDVR0jBBgwFoAUNmHhAHyIBQlRi0RsR/8aTMnqTxIwDwYDVR0TAQH/BAUwAwEB/zAOBgNVHQ8BAf8EBAMCAYYwQAYDVR0fBDkwNzA1oDOgMYYvaHR0cHM6Ly9hbmRyb2lkLmdvb2dsZWFwaXMuY29tL2F0dGVzdGF0aW9uL2NybC8wDQYJKoZIhvcNAQELBQADggIBACDIw41L3KlXG0aMiS//cqrG+EShHUGo8HNsw30W1kJtjn6UBwRM6jnmiwfBPb8VA91chb2vssAtX2zbTvqBJ9+LBPGCdw/E53Rbf86qhxKaiAHOjpvAy5Y3m00mqC0w/Zwvju1twb4vhLaJ5NkUJYsUS7rmJKHHBnETLi8GFqiEsqTWpG/6ibYCv7rYDBJDcR9W62BW9jfIoBQcxUCUJouMPH25lLNcDc1ssqvC2v7iUgI9LeoM1sNovqPmQUiG9rHli1vXxzCyaMTjwftkJLkf6724DFhuKug2jITV0QkXvaJWF4nUaHOTNA4uJU9WDvZLI1j83A+/xnAJUucIv/zGJ1AMH2boHqF8CY16LpsYgBt6tKxxWH00XcyDCdW2KlBCeqbQPcsFmWyWugxdcekhYsAWyoSf818NUsZdBWBaR/OukXrNLfkQ79IyZohZbvabO/X+MVT3rriAoKc8oE2Uws6DF+60PV7/WIPjNvXySdqspImSN78mflxDqwLqRBYkA3I75qppLGG9rp7UCdRjxMl8ZDBld+7yvHVgt1cVzJx9xnyGCC23UaicMDSXYrB4I4WHXPGjxhZuCuPBLTdOLU8YRvMYdEvYebWHMpvwGCF6bAx3JBpIeOQ1wDB5y0USicV3YgYGmi+NZfhA4URSh77Yd6uuJOJENRaNVTzk
-----END CERTIFICATE-----

```


```
Certificate 0:
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
                    04:8d:d0:76:31:60:3f:de:ff:8c:63:70:d8:c0:0d:
                    66:3e:38:01:45:14:00:41:36:2b:28:5f:bc:9c:a6:
                    a1:18:19:4e:dc:49:48:42:91:22:0a:03:c3:66:cb:
                    ff:80:05:52:07:cc:35:ba:f0:52:ec:c1:2b:df:82:
                    d0:56:1c:9f:fd
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature
            1.3.6.1.4.1.11129.2.1.17:
                0......
.....
1702455536011..0\..=....j..6...EL.J0H1"0 ..com.shen1991.keyattestation...1". .,.7.M.x^R.8.<..UY4*i..]...v..r.0....1.................1.................x......y....,..>......@L0J. .......K...I5m.)..I..j.du{...;.m...
..W"5....Sf..&.+.xA..A........B........N....4.Y..O....4.Y
    Signature Algorithm: ecdsa-with-SHA256



Certificate 1:
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


- [attestation扩展字段的ASN.1 schema](https://source.android.com/docs/security/features/keystore/attestation#schema)
```asn.1
30 82 01 2e 02 01 03 0a 01 01 02 01 04 0a 01 01 04 0d 31 37 30 32 34 35 35 35 33 36 30 31 31 04 00 30 5c bf 85 3d 08 02 06 01 6a 03 83 36 a0 bf 85 45 4c 04 4a 30 48 31 22 30 20 04 1b 63 6f 6d 2e 73 68 65 6e 31 39 39 31 2e 6b 65 79 61 74 74 65 73 74 61 74 69 6f 6e 02 01 01 31 22 04 20 fc 2c 89 37 17 4d 81 78 5e 52 84 38 0b 3c bd cb 55 59 34 2a 69 e4 00 5d ac a4 01 76 2e a2 72 83 30 81 b0 a1 05 31 03 02 01 02 a2 03 02 01 03 a3 04 02 02 01 00 a5 0b 31 09 02 01 04 02 01 05 02 01 06 aa 03 02 01 01 bf 83 78 03 02 01 03 bf 83 79 04 02 02 01 2c bf 85 3e 03 02 01 00 bf 85 40 4c 30 4a 04 20 8c a8 9a f1 a6 da a7 4b 00 81 08 49 35 6d e9 29 cf c4 49 8e f3 6a f9 64 75 7b de 8a 11 3b f4 6d 01 01 ff 0a 01 00 04 20 4d 4e e7 79 03 67 a2 5a 45 1e 83 59 0d 88 e4 57 22 35 c0 a5 95 f8 53 66 d9 81 26 e5 2b df 78 41 bf 85 41 05 02 03 01 d4 c0 bf 85 42 05 02 03 03 15 dd bf 85 4e 06 02 04 01 34 8a 59 bf 85 4f 06 02 04 01 34 8a 59

# https://lapo.it/asn1js/

SEQUENCE (8 elem)
  INTEGER 3
  ENUMERATED 1
  INTEGER 4
  ENUMERATED 1
  OCTET STRING (13 byte) 1702455536011
  OCTET STRING (0 byte)
  SEQUENCE (2 elem)
    [701] (1 elem)
      INTEGER (41 bit) 1554837092000
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
Offset: 130
Length: 2+5
(constructed)
Value:
(1 elem)
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