---
layout: post
title:  "OpenVPN On Chromebook"
date:   2015-07-14 13:30:00
categories: openvpn
---

前些天用openvpnas搭建了一个openvpn服务器，openvpnas全称`OpenVPN Access Server`，是openvpn的商业化解决方案，不需要复杂的配置，安装完成后就能使用。通过web来进行管理，修改配置和添加用户，用户登录后下载用户的配置文件给对应平台的客户端。没有许可证时同时连接的客户端不超过2个，个人在此范围内可自由使用。所以它主要面向的是企业用户。

Chromebook原生支持L2TP和OpenVPN, openvpn的配置比较灵活，chromeos支持的并不友好，它只支持用户名和密码的方式。直接使用ovpn配置文件的方式不支持，我并不想打开开发者模式，希望在用户层面就能够得到解决。chromeos有一个内部地址可以导入ONC配置文件，ONC配置文件是chromeos内部使用的网络配置文件,json格式，全称`Open Network Configuration`。通过它可以定义多种网络配置，用户可方便地在不同配置间切换。目前还没有工具生成或者转换成onc配置，只能手工将ovpn配置转换成onc配置文件，这需要对两个配置都要有足够的理解。

# 参考资源

* [OpenVPN on ChromeOS Documentation][openvpn_on_chromeos] 描述配置步骤
* [Open Network Configuration Format][onc_format] 描述配置文件的json格式
* [Online UUID Generator][uuid_generator] 在线UUID生成器

[openvpn_on_chromeos]: https://docs.google.com/document/d/18TU22gueH5OKYHZVJ5nXuqHnk2GN6nDvfu2Hbrb4YLE/pub#h.buf7fpkgt9c8
[onc_format]: http://src.chromium.org/chrome/trunk/src/components/onc/docs/onc_spec.html#sec_OpenVPN_connections_and_types
[uuid_generator]: https://www.uuidgenerator.net/

# 配置步骤

## 准备证书文件

ovpn文件支持内嵌证书，`ca`标签之间的内容认证机构证书，提取并保存为ca.crt。`cert`和`key`分别是用户证书和私钥，保存为client.crt和client.key。chromeos仅支持PKCS12格式的用户证书，要将client.crt和client.key格式转换成pkcs12。
使用openssl命令来生成pkcs12格式的用户证书，client.p12。

    openssl pkcs12 -export -in client.crt -inkey client.key -certfile ca.crt -name MyClient -out client.p12
    
`tls-auth`标签之间是tls授权使用的静态key，保存为ta.key。

## 导入证书文件

打开地址`chrome://settings/certificates`，选择授权中心，导入ca.crt证书。选择您的证书，使用`导入并绑定到设备`导入client.p12文件。

## 准备ONC文件

根据`OpenVPN on ChromeOS Documentation`参考资源所提供的文件实例，结合自己的ovpn文件进行修改。我在配置文件及注意事项会在下面进行介绍。

## 导入ONC文件

打开地址chrome://net-internals， 选择ChromeOS，在`Import ONC file`标题下，执行导入操作。

# ONC配置文件

`RemoteCertTLS`的默认值是`server`，我在开始配置的过程中，总是配置失败，查看日志一直是`ssl3_get_server_certificate:certificate verify failed`错误。后来仔细比较内部配置输出和ovpn配置的差异，将`RemoteCertTLS`设为`none`，取得配置成功。

证书的格式是`X509`，内容占一行，要将ca.crt文件中的多行合并成一行。

    grep -v '-' ca.crt | perl -p -e 's/\n/\\n/' -
    
tls授权的静态key和json key是`TLSAuthContents`, 它的内容也是占一行，但不同于证书的格式。

    grep -v '-' ca.crt | perl -p -e 's/\n/\\n/' -
    
GUID字段是一个全局唯一的字符串，可以用UUID生成器来生成。
    
## 调试方法

内部地址: chrome://system，用来查看系统信息。`network-services`记录服务启动信息。`netlog`记录了网络工作的日志。如果出现错误可以在这两个地址进行错误信息的查找。

## 文件内容

它所对应的ovpn文件在下面给出。

    {
      "Type":"UnencryptedConfiguration",
      "Certificates": [ 
        {
          "GUID": "{ecff0b6e-5c6c-4d17-95e3-813ba2352eb2}",
          "Type": "Authority",
          "X509": "MIICuDCCAaCgAwIBAgIEVZSkUzAxxxxxxxxG9w0BAQsFADAVMRMwEQYDVQQDDApPcGVuVlBOIENBMB4XDTE1MDYyNTAyMzkxNVoXDTI1MDYyOTAyMzkxNVowFTETMBEGA1UEAwwKT3BlblZQTiBDQTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBALq161pCqQIXOTGtFJ79CNTcVlQKt6GwPvVrrEd/Cuiu5I/ZijaGyKYLxVywEGkTudJD13yldW+h5ILmAHFu2l7ODq+rLsIPxT4kYvrcI+kClUhcNqFEh+Qd5tPKAhJKZYlwAS/0yj58kNlhxQnQimgIg84BiOR7Pg4QM5ag9nsKehnIfVWS0VkuPS3T/WMaJXXXi/zoCAUVXOSaIo2QtFMBnVIywMbcKqAHBz/ZUxhWuixhkG2vVf6TCS1LI560xxxxxxxxkIZDDtnbapm9jw7ImouxPePYLSDoZEduoqg9ZqbDZNfFxch4r+aU90IXqjUjuxXZlVNBerePPM1Ex/UCAwEAAaMQMA4wDAYDVR0TBAUwAwEB/zANBgkqhkiG9w0BAQsFAAOCAQEASggdKwfjfWxv12Z+rqm11IU8nQTnwnTSYX8oSUV2uoqFmGFjir196U1UA3zIjksLwUlu/vsJVG2Z63S2VgEcq1Nq9jaL+6/7zIxh1AYk885mijBSDMrK9PfwnWUwF5fykbC0sy2TRQoUXP42NPQOCz+kSyZsmYcF0gi/8uwzgbRErVaI5tuH6Ov/Qv1jTFPKfxo4DHyZMcQrxtaQv6iADpNLnmFPMWl1eKwXgkF3HbmDK0rnpQ4K+VHXeiKKGHKxxxxxxxxxCqIPVyRoyk/9GNjeMS2Kg396iH5wPsjgmZefZ7ZW1vtPKNkFlnCPpFBDs5tGlbbDI+u8Ls3/iJyx6Q=="
         } 
       ],
      "NetworkConfigurations": [
        {
          "GUID": "{e05b15b4-8da9-4c53-9cb7-3ca963dec70b}",
          "Name": "ConfName",
          "Type": "VPN",
          "VPN": {
            "Type": "OpenVPN",
            "Host": "50.116.0.220",
            "OpenVPN": {
              "ServerCARef": "{ecff0b6e-5c6c-4d17-95e3-813ba2352eb2}",
              "AuthRetry": "interact",
              "ClientCertType": "Pattern",
              "ClientCertPattern": {              
                "IssuerCARef": [ "{ecff0b6e-5c6c-4d17-95e3-813ba2352eb2}" ]
              },
              "CompLZO": "true",
              "Port": 1194,
              "Proto": "udp",
              "RemoteCertTLS": "none",
              "NsCertType": "server",
              "SaveCredentials": false,
              "ServerPollTimeout": 10,
              "Username": "insprt",
              "KeyDirection":"1",                    
              "TLSAuthContents":"-----BEGIN OpenVPN Static key V1-----\n8c74c79410xxxxxxxxb911a2d3e7be36\na658f9a89f06d0aeb11f1b1b0081c3f3\n8ca7fbfc1a005228bf790a0b238a3f17\n7a3f3581a3ea0118af34d987106f7456\nae623d169062b4c7ed1111b410c252fd\n27cd88d0e52f424b4b02a51fbfdd8854\n57b5881c8ac973fbbfecfbdfbe12e004\n0b09d26bdf107ee818f721dbf54048bc\nc59661108bde2b5c84af8f6fc81e4e6e\n04dc0af3e5f04707da293b1cee7df018\n00ef14c77e481441xxxxxxxx37a37195\n14d24dd77c61c55d2f338641395ba0c4\n9cc2d9bbfa33ced6c4c6ca698b47fe62\n5cec47a04cde25e73294ec8ee1826bfb\nae2ccd5437c76cd29c99c665d08f95fd\n40d58d7bd25bed82bb36c2c2d6eda34d\n-----END OpenVPN Static key V1-----\n"
            }
          }
        }
      ]
    }

# ovpn配置文件

    setenv FORWARD_COMPATIBLE 1
    client
    server-poll-timeout 4
    nobind
    remote xx.xx.xx.xx 1194 udp
    remote xx.xx.xx.xx 1194 udp
    remote xx.xx.xx.xx 443 tcp
    remote xx.xx.xx.xx 1194 udp
    remote xx.xx.xx.xx 1194 udp
    remote xx.xx.xx.xx 1194 udp
    dev tun
    dev-type tun
    ns-cert-type server
    reneg-sec 604800
    sndbuf 100000
    rcvbuf 100000
    auth-user-pass
    comp-lzo no
    verb 3
    setenv PUSH_PEER_INFO

    <ca>
    -----BEGIN CERTIFICATE-----
    MIICuDCCAaCgAwIBAgIExxxxxxxxxxxxxxxxxxxxAQsFADAVMRMwEQYDVQQDDApP
    cGVuVlBOIENBMB4XDTE1MDYyNTAyMzkxNVoXDTI1MDYyOTAyMzkxNVowFTETMBEG
    A1UEAwwKT3BlblZQTiBDQTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEB
    ALq161pCqQIXOTGtFJ79CNTcVlQKt6GwPvVrrEd/Cuiu5I/ZijaGyKYLxVywEGkT
    udJD13yldW+h5ILmAHFu2l7ODq+rLsIPxT4kYvrcI+kClUhcNqFEh+Qd5tPKAhJK
    ZYlwAS/0yj58kNlhxQnQimxxxxxxxxxxxxxxxxag9nsKehnIfVWS0VkuPS3T/WMa
    JXXXi/zoCAUVXOSaIo2QtFMBnVIywMbcKqAHBz/ZUxhWuixhkG2vVf6TCS1LI560
    ffSzZ172kIZDDtnbapm9jw7ImouxPePYLSDoZEduoqg9ZqbDZNfFxch4r+aU90IX
    qjUjuxXZlVNBerePPM1Ex/UCAwEAAaMQMA4wDAYxxxxxxxxxxxxxxxANBgkqhkiG
    9w0BAQsFAAOCAQEASggdKwfjfWxv12Z+rqm11IU8nQTnwnTSYX8oSUV2uoqFmGFj
    ir196U1UA3zIjksLwUlu/vsJVG2Z63S2VgEcq1Nq9jaL+6/7zIxh1AYk885mijBS
    DMrK9PfwnWUwF5fykbC0sy2TRQoUXP42NPQOCz+kSyZsmYcF0gi/8uwzgbRErVaI
    5tuH6Ov/Qv1jTFPKfxo4DHyZMcQrxtaQv6iADpNLnmFPMWl1eKwXgkF3HbmDK0rn
    pQ4K+VHXeiKKGHKN3ZDhVTQxCqIPVyRoyk/9GNjeMS2Kg396iH5wPsjgmZefZ7ZW
    1vtPKNkFlnCPpFBDs5tGlbbDI+u8Ls3/iJyx6Q==
    -----END CERTIFICATE-----
    </ca>

    <cert>
    -----BEGIN CERTIFICATE-----
    MIICwTCCAamgAwIBAgIBAzANBgkqhkixxxxxxxxxxxxxxxxxxxxxxxxxDApPcGVu
    VlBOIENBMB4XDTE1MDYyNTAzNDkzNVoXDTI1MDYyOTAzNDkzNVowETEPMA0GA1UE
    AwwGaW5zcHJ0MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAsjeIji54
    AF+LCmLxth63sNrXhxxxxxxxxxxxxxxxxxxxxSAUM+fuq60hGUfJt75n0msST/sy
    sJn0+T86Qlav9GW80sUApVmnm2Y27+eaUoWEhav3xxq/hwHvvNIwjGsAA3+vyBib
    UGaZS6HHSJBHVfuKm7TIxxxxxxxxxxxxxxxxxxxxxxxT1mWq9JFvHOAMvcnEYPtb
    P+eqzikR3zTG0NthzGWcuadBalC/s3+kM8nZ6z/5sTX7Xw0MjK7LEzvu0H3GZcBn
    Zl9ZiNSA5yoN278gLdO7Qvqw/SUSBq3kfDBk4htglllffg7omdTVKMwuyJoeeTUZ
    Mu2ZVBgVNDE3WQIDAQABoyAwHjAJBgNVHRMEAjAAMBEGCWCGSAGG+EIBAQQEAwIH
    gDANBgkqhkiG9w0BAQsFAAOCAQEADrxFw8Dn5nqOKLLr1RFndq97rZZPMdyscSxF
    6mctlrc+QL6NY1JiBvXomJq8QsntxqRiiIbanb5PVMPjeUhoiNXNVzihvE5Bzvw1
    Xxm+qIrcu7hFOWg1LB3gHJaGyBk6rVMKxPcgF0KPD4caLljAIRo7n022CTbBJUsi
    Zh7o2FVB8eWFYqAjx+vAscvlf6aq4xq81IgR3kYhpFSYKjh9SU3uahqowNfYyR98
    k9IcHLlZRSliqO+yEX3DVpf24CC3Iv/szTnoxv60Mxxxxxxxxxxx6Exm9JVr4/Kk
    LRfqO08nXuFM0H8YwCyvyIA2ZDlmmlT0x/RAoxnrgqIDTCLbgw==
    -----END CERTIFICATE-----
    </cert>

    <key>
    -----BEGIN PRIVATE KEY-----
    MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQCyN4iOLngAX4sK
    YvG2Hrew2teFTj5bKsVL06oZ/QOecfM1IBQz5+6rrSEZR8m3vmfSaxJP+zKwmfT5
    PzpCVq/0ZbzSxQClWaebZjbv55pShYSFq/fHGr+HAe+80jCMawADf6/IGJtQZplL
    ocdIkEdV+4qbtMioN5276hkivyOmTvwxxxxxxxxxxxxxxxxx4Ay9ycRg+1s/56rO
    KRHfNMbQ22HMZZy5p0FqUL+zf6QzydnrP/mxNftfDQyMrssTO+7QfcZlwGdmX1mI
    1IDnKg3bvyAt07tC+rD9JRIGreR8MGTiG2CWWV9+DuiZ1NUozC7Imh55NRky7ZlU
    GBU0MTdZAgMBAAECggEABvxoTPKDX7hfEewo/3OazcL2WdJkXVyC2WMVsukZIDfl
    SbrVL+eykmY5+uy2eo5rMXNxxxxxxxxxxxxxxx3GnfTy/uwcB19JU60hECxq/zse
    o8LG9rYUte0cgbFXl9mF6Z0yvcxBIlizP6S61Bxbv4IZv9rJVta/RyN5EsSdWCKF
    t7/d4yUpf5ZopdqiZBKR9n91bOTa+pensUz+2h4SbzdthApJmbuKK+BsP32EYEm+
    I40yLrt2fUhAOI3G3KO1uGjpeidSJRT9FtFUr0EwWij5Ar1zonjuQfpDZgBfab4x
    lXWu3gSqPOnS7kgAwzaBpuTahlwSv1rtr0tjWq2jgQKBgQDibuqLvTCgcEDQmSKS
    GT6KgCeJJ7jiLVg9hLdikPaLPSR/6ZqbfZVMFEPwq0CmdUr9OLrKZBTq5NIp1Gjp
    m0OSPtRdIBG+2gOMzYx0Q4nym+nAsexgFwlHWH24mbA5oQ7CJQ2eQtxz9ptpYWj7
    ER471mkZTNgfxhw2gTkyEWihMQKBgQDJfN5bNHo29iWgbzw3hnmLs2CrZODUNRNV
    rKBnvvRWWqD8chvbRCC3UAMH11drLcraLUFZj1T1snVUcIaDDxTs2Yu6FN/BhSum
    4k1hYfgJYJX6kUhHJk5ppv5OVpevOqKMwB3FJWWTlJlHmvs+UYyUcDKsKjYAKJcW
    RULe8aguqQKBgH4yDNv2g9xW03iub/r2wMlV5TLmhX7ggLZAeigf3Jf7apUzb2xL
    UGLHRJokB3L+Gd4IuOnFX3cOMicH77SKSN1/0MFZ9ynjvWjCwg2l+oLQ7DTttGxV
    SmGN6vtwBCwKG/yNxAo4/z5N6Y2QsX6DqtL0izyDfEwxEFY8LNE/rI1xAoGBAMBQ
    UAfrsc8t6EIWifpRf0fpQZa2JaZGtpqqtzvu1lZqEIiD/bSudS+izhG453akcZ8H
    XP23whb1a+nZsXn8djOPfT9yVxPmIQEbtVIC6XVB3EUaUEug820CeG6bVhJpu+bu
    JDwc8rQHPLpM4gvcWHsCEEulyn8iPvuBxk73h1hpAoGAJo5YuS3Z64TXcCgMSmIk
    iJyolElsa4aczmK8yDfL7AfYUXHmumwNw5UNYxE3yygyEEktlfqQijfkCEi2ytxB
    9OIwXDW9b2F+35JbpzYmGQwSO4egBh5JQ3LMyzs9LwM6RSwknBUdyNrXOIj4kv0m
    ENghnEwYGS4kiGHbXxQqimk=
    -----END PRIVATE KEY-----
    </key>

    key-direction 1
    <tls-auth>
    #
    # 2048 bit OpenVPN static key (Server Agent)
    #
    -----BEGIN OpenVPN Static key V1-----
    8c74c7941xxxxxxxx4b911a2d3e7be36
    a658f9a89f06d0aeb11f1b1b0081c3f3
    8ca7fbfc1a005228bf790a0b238a3f17
    7a3f3581a3ea011xxxxxxxx7106f7456
    ae623d169062b4c7ed1111b410c252fd
    27cd88d0e52f424b4b02a51fbfdd8854
    57b5881c8ac973fbbfecfbdfbe12e004
    0b09d26bdf107ee818f721dbf54048bc
    c59661108bde2b5c84af8f6fc81e4e6e
    04dc0af3e5f04707da293b1cee7df018
    00ef14c77e481441376afe2937a37195
    14d24dd77c61c55d2f338641395ba0c4
    9cc2d9bbfa33ced6c4c6ca698b47fe62
    5cec47a04cde25e73294ec8ee1826bfb
    ae2ccd5437c76cd29c99c665d08f95fd
    40d58d7bd25bed82bb36c2c2d6eda34d
    -----END OpenVPN Static key V1-----
    </tls-auth>
