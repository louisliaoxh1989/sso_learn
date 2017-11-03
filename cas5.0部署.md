**第一步：开启Tomcat8的https**

[请参照Tomcat8开启https](Tomcat8开启https.md)


**第二步：下载cas官网提供的cas-overlay-template**

```Bash
https://github.com/apereo/cas-overlay-template.git
```

**第三步：编译生成war并拷贝到tomcat8的webapps目录中**

```Bash
# 生成需要部署的web文件 war
./build.sh package
# 修改相应配置application.properties
cd cas-overlay-template/target/cas/WEB-INF/classes/
vi application.properties
# 修改其中的 
server.context-path=/cas
server.port=8443

server.ssl.key-store=file://opt/dev/tomcat8/conf/tomcat.keystore
server.ssl.key-store-password=mydomain
server.ssl.key-password=mydomain

# 贝到tomcat8的webapps目录中
cp -r cas-overlay-template/target/cas /opt/dev/tomcat8/webapps/
```

**第四步：启动tomcat验证**

```Bash
/opt/dev/tomcat8/bin/startup.sh #生成

```
在浏览器中输入 https://localhost:8443 如果出现登陆页面表示部署成功

(**其登陆用户名和密码在 WEB-INF/classes/application.properties中的*cas.authn.accept.users*配置项中配置的,格式为:用户名::密码**)
```XML
##
# CAS Authentication Credentials
#
cas.authn.accept.users=casuser::Mellon
```

**第五步：改为使用mysql数据库认证**

``` XML
# 修改cas-overlay-template文件夹下的pom.xml文件增加dependency
  <dependency>
   <groupId>org.apereo.cas</groupId>
   <artifactId>cas-server-support-jdbc-drivers</artifactId>
   <version>${cas.version}</version>
</dependency>
 <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>6.0.6</version>
    <scope>runtime</scope>
 </dependency>
```
