# 使用.pk8 和.pem签名生成.keystore 签名

将 platform.pk8 和 platform.x509.pem 格式的系统签名转换为 mykey.keystore 格式

需要系统中有openssl 和 jdk，windows 版openssl 可以在http://slproweb.com/products/Win32OpenSSL.html下载

```shell
openssl pkcs8 -inform DER -nocrypt -in platform.pk8 -out key.pem
openssl pkcs12 -export -in platform.x509.pem -inkey key.pem -out platform.p12 -password pass:android -name mykey
keytool -importkeystore -deststorepass password -destkeystore mykey.keystore -srckeystore platform.p12 -srcstoretype  PKCS12 -srcstorepass android
keytool -list -v -keystore mykey.keystore
```

第一步使用platform.pk8生成了key.pem 文件

第二步使用platform.x509.pem 和key.pem 生成了platform.p12 文件，其中签名的名字是mykey，密码是android

第三步使用platform.p12 生成了mykey.keystore 文件，keystore密码是password

第四步，查看签名文件信息



#### 1.`openssl pkcs8 -in platform.pk8 -inform DER -outform PEM -out shared.priv.pem -nocrypt`

#### 2`openssl pkcs12 -export -in platform.x509.pem -inkey shared.priv.pem -out shared.pk12 -name testalias`

其中testalias为keyAlias

#### 3.`keytool -importkeystore -deststorepass android -destkeypass android -destkeystore source.keystore -srckeystore shared.pk12 -srcstoretype PKCS12 -srcstorepass android -alias keyAlias`

生成的source.keystore即为所需要的keystore文件，storePassword和keyPassword为android
gradle配置：

