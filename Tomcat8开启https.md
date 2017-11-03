
## 1.使用jdk自带的keytool生成keystore

```
jdk/bin/keytool -genkey -alias mydomain -keyalg RSA -keystore /opt/dev/tomcat8/conf/tomcat.keystore -storepass mydomain
```

## 2.修改tomcat下面server.xml增加https的配置

```XML
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
               maxThreads="150" SSLEnabled="true">
        <SSLHostConfig>
            <Certificate certificateKeystoreFile="/opt/dev/tomcat8/conf/tomcat.keystore"
                         certificateKeystoreType="JKS" certificateKeystorePassword="mydomain" /> <!--certificateKeystorePassword的值为第一步的storepass选项指定的值 -->
        </SSLHostConfig>
 </Connector>
```
