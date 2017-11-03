```
使用安装的jdk中的bin下面的keytool工具
```

Keytool 是一个Java数据证书的管理工具 ,Keytool将密钥（key）和证书（certificates）存在一个称为keystore的文件中在keystore里，包含两种数据:密钥实体（Key entity）-密钥（secret key）或者是私钥和配对公钥（采用非对称加密）可信任的证书实体（trusted certificate entries）-只包含公钥.


# 一. Keytool创建和导入命令

## 创建keystore和密钥对(创建了一个自签证书)

```Bash
bin/keytool -genkey -alias tomcat -keyalg RSA -keystore ~/tomcat.keystore -validity 36500
```

```
  -alias 域名或别名
  -genkey:创建一个public-private key pair
  -keyalg：使用的算法，建议使用RSA获得更好的兼容
  -keystore：指定生成的keystore存储的位置
  -validity: 有效期
  
```
>> keytool -genkey -alias mydomain -keyalg RSA -keystore keystore.jks -keysize 2048


## 为存在的keystore生成证书请求文件CSR

```Bash
keytool -certreq -alias mydomain -keystore keystore.jks -file mydomain.csr
```
## 导入根证书或中级证书到keystore

```Bash
keytool -import -trustcacerts -alias root -file mydomain.crt -keystore keystore.jks
```
>> [crt生成教程](http://blog.csdn.net/u014410763/article/details/50555902)

## 导入SSL服务器证书到keystore

```Bash
keytool -import -trustcacerts -alias mydomain -file mydomain.crt -keystore keystore.jks
```

## 为存在的keystore生成自签名证书

```
keytool -genkey -keyalg RSA -alias selfsigned -keystore keystore.jks -storepass password -validity 360 -keysize 2048
```

# 二、Keytool查看命令

## 查看单个证书
```Bash
keytool -printcert -v -file mydomain.crt
```

## 列出keystore存在的所有证书

```Bash
keytool -list -v -keystore keystore.jks
```

## 使用别名查看keystore特定条目

```Bash
keytool -list -v -keystore keystore.jks -alias mydomain
```

# 三、其他Keytool命令

## 删除keystore里面指定证书
```Bash
keytool -delete -alias mydomain -keystore keystore.jks
```
## 更改keysore密码
```Bash
keytool -storepasswd -new new_storepass -keystore keystore.jks
```
## 删除keystore里面指定证书
```Bash
keytool -delete -alias mydomain -keystore keystore.jks
```
## 导出keystore里面的指定证书
```Bash
keytool -export -alias mydomain -file mydomain.crt -keystore keystore.jks
```
## 列出信任的CA证书
```Bash
keytool -list -v -keystore $JAVA_HOME/jre/lib/security/cacerts
```
## 导入新的CA到信任证书
```Bash
keytool -import -trustcacerts -file /path/to/ca/ca.pem -alias CA_ALIAS -keystore $JAVA_HOME/jre/lib/security/cacerts
```

