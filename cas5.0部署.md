**第一步：下载cas官网提供的cas-overlay-template**

```Bash
https://github.com/apereo/cas-overlay-template.git
```

**第二步：生成storekey，为开启tomcat https访问做准备**

```Bash
cd cas-overlay-template
# 修改bulid.sh 中的 function gencert() 为下面的

function gencert() {
	if [[ ! -d /etc/cas ]] ; then 
		copy
	fi
	which keytool
	if [[ $? -ne 0 ]] ; then
	    echo Error: Java JDK \'keytool\' is not installed or is not in the path
	    exit 1
	fi
	# override DNAME and CERT_SUBJ_ALT_NAMES before calling or use dummy values
	DNAME="${DNAME:-CN=cas.example.org,OU=Example,OU=Org,C=US}"
	CERT_SUBJ_ALT_NAMES="${CERT_SUBJ_ALT_NAMES:-dns:example.org,dns:localhost,ip:127.0.0.1}"
	echo "Generating keystore for CAS with DN ${DNAME}"
	keytool -genkeypair -alias cas -keyalg RSA -keypass changeit -storepass changeit -keystore /etc/cas/thekeystore -dname ${DNAME} -ext SAN=${CERT_SUBJ_ALT_NAMES}
	keytool -exportcert -alias cas -storepass changeit -keystore /etc/cas/thekeystore -file /etc/cas/cas.cer
}
```
>> 其中CN一般为网站的域名，这里修改为cas.example.org,因为cas-overlay-template源代码中的etc下面的cas.properties文件配置的是cas.example.org,为了少修改文件所以就修改成了cas.example.org
>> 其他可设置字段解释
```
CN 一般为网站的域名 
OU 组织单位名称
O  组织名称 #如hanghai University, 
L  城市或区域名称  #如CD
ST 州或省份名称   #如SC
C  两字母国家代码 #如CN
```
**第三步Tomcat8开启https**

修改Tomcat8的conf下面的server.xml
```XML
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
               maxThreads="150" SSLEnabled="true">
        <SSLHostConfig>
            <Certificate certificateKeystoreFile="/etc/cas/thekeystore"
                         certificateKeystoreType="JKS" certificateKeystorePassword="changeit" />
        </SSLHostConfig>
    </Connector>

 <Host appBase="webapps" autoDeploy="true" name="cas.example.org" unpackWARs="true">
```
**第四步修改服务器的hosts**


```BASH
sudo vi /etc/hosts
```
加入

```XML
127.0.0.1   cas.example.org
127.0.0.1   example.org
```
**第五步：加入数据库认证依赖，启用mysql数据库验证方式**

> 修改  cas-overlay-template 下面的pom.xml文件增加下面的依赖,我这里的cas.version用的是5.1.5

``` XML
# 修改cas-overlay-template文件夹下的pom.xml文件增加dependency
#这个很重要，这是开启CAS的数据库认证
<dependency>  
<groupId>org.apereo.cas</groupId>  
<artifactId>cas-server-support-jdbc</artifactId>  
<version>${cas.version}</version>  
</dependency>
#这个配置CAS数据库驱动
 <dependency>
   <groupId>org.apereo.cas</groupId>
   <artifactId>cas-server-support-jdbc-drivers</artifactId>
   <version>${cas.version}</version>
</dependency>
#这个配置MySQL连接依赖
 <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>6.0.6</version>
    <scope>runtime</scope>
 </dependency>
```

**第六步：继续增加oauth,ticker等的依赖，以启用相应的功能**

```XML
 <!-- cas rest ticket start-->
        <dependency>
            <groupId>org.apereo.cas</groupId>
            <artifactId>cas-server-support-rest</artifactId>
            <version>${cas.version}</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.apereo.cas</groupId>
            <artifactId>cas-server-support-rest-services</artifactId>
            <version>${cas.version}</version>
            <scope>runtime</scope>
        </dependency>
        <!-- cas rest ticket end-->
        <!--
        <dependency>
            <groupId>org.apereo.cas</groupId>
            <artifactId>cas-server-support-json-service-registry</artifactId>
            <version>${cas.version}</version>
        </dependency>
        -->
        
        <dependency>
             <groupId>org.apereo.cas</groupId>
             <artifactId>cas-server-support-ldap</artifactId>
             <version>${cas.version}</version>
        </dependency>
        
        <dependency>
            <groupId>org.apereo.cas</groupId>
            <artifactId>cas-server-support-oauth-webflow</artifactId>
            <version>${cas.version}</version>
        </dependency>
        
        <dependency>
            <groupId>org.apereo.cas</groupId>
            <artifactId>cas-server-support-jpa-ticket-registry</artifactId>
            <version>${cas.version}</version>
        </dependency>
        
        <dependency>
            <groupId>org.apereo.cas</groupId>
            <artifactId>cas-server-support-jpa-service-registry</artifactId>
            <version>${cas.version}</version>
        </dependency>

```

**第七步：编译生成war并拷贝到tomcat8的webapps目录中**

```Bash
# 生成需要部署的web文件 war
./build.sh package
# 修改相应配置application.properties
cd cas-overlay-template/target/cas/WEB-INF/classes/
vi application.properties
# 修改其中的 
server.context-path=/cas
server.port=8443

server.ssl.key-store=/etc/cas/thekeystore
server.ssl.key-store-password=changeit
server.ssl.key-password=changeit

# 贝到tomcat8的webapps目录中
cp -r cas-overlay-template/target/cas /opt/dev/tomcat8/webapps/
```

**第八步：修改WEB-INF/classes/application.propertie下面的**

```
vi /opt/dev/tomcat8/webapps/WEB-INF/classes/application.properties
```

	1. 先 注释掉原有的cas.authn.accept.users=casuser::Mellon
	```XML
	# cas.authn.accept.users=casuser::Mellon
	```
	2. 增加数据认证方式

	```XML
	cas.authn.jdbc.query[0].sql=select password from cas_user where name=?
	cas.authn.jdbc.query[0].healthQuery=
	cas.authn.jdbc.query[0].isolateInternalQueries=false
	cas.authn.jdbc.query[0].url=jdbc:mysql://172.16.52.184:3306/castest?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&useSSL=false
	cas.authn.jdbc.query[0].failFast=true
	cas.authn.jdbc.query[0].isolationLevelName=ISOLATION_READ_COMMITTED
	cas.authn.jdbc.query[0].dialect=org.hibernate.dialect.MySQLDialect
	cas.authn.jdbc.query[0].leakThreshold=10
	cas.authn.jdbc.query[0].propagationBehaviorName=PROPAGATION_REQUIRED
	cas.authn.jdbc.query[0].batchSize=1
	cas.authn.jdbc.query[0].user=root
	cas.authn.jdbc.query[0].ddlAuto=create-drop
	cas.authn.jdbc.query[0].maxAgeDays=180
	cas.authn.jdbc.query[0].password=1
	cas.authn.jdbc.query[0].autocommit=false
	cas.authn.jdbc.query[0].driverClass=com.mysql.cj.jdbc.Driver
	cas.authn.jdbc.query[0].idleTimeout=5000
	#cas.authn.jdbc.query[0].credentialCriteria=
	#cas.authn.jdbc.query[0].name=
	#cas.authn.jdbc.query[0].order=0
	#cas.authn.jdbc.query[0].dataSourceName=
	#cas.authn.jdbc.query[0].dataSourceProxy=false

	#此处很关键，必须要配置查询字段的名字，否则认证失败，官方文档中未找到说明，跟踪源代码找到的。
	cas.authn.jdbc.query[0].fieldPassword=password

	#cas.authn.jdbc.query[0].fieldExpired=
	#cas.authn.jdbc.query[0].fieldDisabled=
	#cas.authn.jdbc.query[0].principalAttributeList=sn,cn:commonName,givenName

	#默认为NONE密码明文认证，可以自定义加密算法类(implements PasswordEncoder)
	#cas.authn.jdbc.query[0].passwordEncoder.type=NONE|DEFAULT|STANDARD|BCRYPT|SCRYPT|PBKDF2|com.example.CustomPasswordEncoder



**第步：启动tomcat验证**

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



**参考文档**

[基于CAS的单点登录SSO](http://blog.csdn.net/gobitan/article/details/77658360)

[Apereo CAS 5.0.X 简明实用教程](http://blog.csdn.net/xichenguan/article/details/60785451)
#cas.authn.jdbc.query[0].passwordEncoder.characterEncoding=
#当passwordEncoder.type为default时，算法可定义MD5等算法。
#cas.authn.jdbc.query[0].passwordEncoder.encodingAlgorithm=
#cas.authn.jdbc.query[0].passwordEncoder.secret=
#cas.authn.jdbc.query[0].passwordEncoder.strength=16
#cas.authn.jdbc.query[0].principalTransformation.suffix=
#cas.authn.jdbc.query[0].principalTransformation.caseConversion=NONE|UPPERCASE|LOWERCASE
#cas.authn.jdbc.query[0].principalTransformation.prefix=

#多属性返回(同样是各种试验，同事解决的哈)
cas.authn.attributeRepository.jdbc[0].singleRow=true
cas.authn.attributeRepository.jdbc[0].order=0
cas.authn.attributeRepository.jdbc[0].url=jdbc:mysql://172.16.52.184:3306/castest?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&useSSL=false
cas.authn.attributeRepository.jdbc[0].username=name
cas.authn.attributeRepository.jdbc[0].user=root
cas.authn.attributeRepository.jdbc[0].password=1
cas.authn.attributeRepository.jdbc[0].sql=select * from cas_user where name=?
#取消以下两行则在返回属性中仅包含表中的这两个字段属性，注释情况下返回该表行所有属性
#cas.authn.attributeRepository.jdbc[0].attributes.id=id
#cas.authn.attributeRepository.jdbc[0].attributes.name=name
cas.authn.attributeRepository.jdbc[0].dialect=org.hibernate.dialect.MySQLDialect
cas.authn.attributeRepository.jdbc[0].ddlAuto=create-drop
cas.authn.attributeRepository.jdbc[0].driverClass=com.mysql.cj.jdbc.Driver
cas.authn.attributeRepository.jdbc[0].leakThreshold=10
cas.authn.attributeRepository.jdbc[0].propagationBehaviorName=PROPAGATION_REQUIRED
cas.authn.attributeRepository.jdbc[0].batchSize=1
cas.authn.attributeRepository.jdbc[0].healthQuery=SELECT 1
cas.authn.attributeRepository.jdbc[0].failFast=true

```
