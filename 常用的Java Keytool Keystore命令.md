```
使用安装的jdk中的bin下面的keytool工具
```


# 一. Keytool创建和导入命令

## 创建keystore和密钥对

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
