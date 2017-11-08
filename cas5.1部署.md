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

**第八步：修改WEB-INF/classes/application.properties 文件，增加数据库验证方式等配置**

```
vi /opt/dev/tomcat8/webapps/WEB-INF/classes/application.properties
```

```XML
##
# CAS Authentication Credentials
#
#cas.authn.accept.users=casuser::Mellon
# 增加数据库验证方式
cas.authn.jdbc.query[0].sql=select * from cas_user where name=?
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
cas.authn.jdbc.query[0].principalAttributeList=id,name,roles
#sn,cn:commonName,givenName

#默认为NONE密码明文认证，可以自定义加密算法类(implements PasswordEncoder)
#cas.authn.jdbc.query[0].passwordEncoder.type=NONE|DEFAULT|STANDARD|BCRYPT|SCRYPT|PBKDF2|com.example.CustomPasswordEncoder
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
cas.authn.attributeRepository.jdbc[0].attributes.id=id
cas.authn.attributeRepository.jdbc[0].attributes.name=name
cas.authn.attributeRepository.jdbc[0].attributes.roles=roles
cas.authn.attributeRepository.jdbc[0].dialect=org.hibernate.dialect.MySQLDialect
cas.authn.attributeRepository.jdbc[0].ddlAuto=create-drop
cas.authn.attributeRepository.jdbc[0].driverClass=com.mysql.cj.jdbc.Driver
cas.authn.attributeRepository.jdbc[0].leakThreshold=10
cas.authn.attributeRepository.jdbc[0].propagationBehaviorName=PROPAGATION_REQUIRED
cas.authn.attributeRepository.jdbc[0].batchSize=1
cas.authn.attributeRepository.jdbc[0].healthQuery=SELECT 1
cas.authn.attributeRepository.jdbc[0].failFast=true

#############ticket registry##################
cas.ticket.registry.jpa.jpaLockingTimeout=3600
cas.ticket.registry.jpa.healthQuery=SELECT 1
cas.ticket.registry.jpa.isolateInternalQueries=false
cas.ticket.registry.jpa.url=jdbc:mysql://172.16.52.184:3306/castest?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&useSSL=false
cas.ticket.registry.jpa.failFast=true
cas.ticket.registry.jpa.dialect=org.hibernate.dialect.MySQL5Dialect
# cas.ticket.registry.jpa.leakThreshold=10
cas.ticket.registry.jpa.jpaLockingTgtEnabled=false
cas.ticket.registry.jpa.batchSize=1
# cas.ticket.registry.jpa.defaultCatalog=
cas.ticket.registry.jpa.defaultSchema=castest
cas.ticket.registry.jpa.user=root

cas.ticket.registry.jpa.ddlAuto=create-drop
cas.ticket.registry.jpa.password=1
cas.ticket.registry.jpa.autocommit=true
cas.ticket.registry.jpa.driverClass=com.mysql.cj.jdbc.Driver

######service registry##################
cas.serviceRegistry.jpa.healthQuery=SELECT 1
cas.serviceRegistry.jpa.isolateInternalQueries=false
cas.serviceRegistry.jpa.url=jdbc:mysql://172.16.52.184:3306/castest?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&useSSL=false
cas.serviceRegistry.jpa.failFast=true
cas.serviceRegistry.jpa.dialect=org.hibernate.dialect.MySQL5Dialect
# cas.serviceRegistry.jpa.leakThreshold=10
# cas.serviceRegistry.jpa.batchSize=1
# cas.serviceRegistry.jpa.defaultCatalog=
# cas.serviceRegistry.jpa.defaultSchema=
cas.serviceRegistry.jpa.user=root
cas.serviceRegistry.jpa.ddlAuto=create-drop
cas.serviceRegistry.jpa.password=1
cas.serviceRegistry.jpa.autocommit=true
cas.serviceRegistry.jpa.driverClass=com.mysql.cj.jdbc.Driver
```
**第九步：修改WEB-INF/classes/services/HTTPSandIMAPS-10000001.json 文件，修改为http和https开头的所有客户端都可以用CAS Server进行认证。并且修改attributeReleasePolicy为org.apereo.cas.services.ReturnAllAttributeReleasePolicy已增加认证通过后返回的用户属性**

```JS
{
  "@class" : "org.apereo.cas.services.RegexRegisteredService",
  "serviceId" : "^(https|imaps|http)://.*",
  "name" : "HTTPS and IMAPS",
  "id" : 10000001,
  "description" : "This service definition authorized all application urls that support HTTPS and IMAPS protocols.",
  "proxyPolicy" : {
    "@class" : "org.apereo.cas.services.RefuseRegisteredServiceProxyPolicy"
  },
  "evaluationOrder" : 10000,
  "usernameAttributeProvider" : {
    "@class" : "org.apereo.cas.services.DefaultRegisteredServiceUsernameProvider"
  },
  "logoutType" : "BACK_CHANNEL",
  "attributeReleasePolicy" : {
    "@class" : "org.apereo.cas.services.ReturnAllAttributeReleasePolicy"
  },
  "accessStrategy" : {
    "@class" : "org.apereo.cas.services.DefaultRegisteredServiceAccessStrategy",
    "enabled" : true,
    "ssoEnabled" : true
  },
  "logo" : "images/logo_cas.png"
}
```
>> ReturnAllAttributeReleasePolicy会返回application.properties中定义的cas.authn.jdbc.query[0].principalAttributeList的属性列表,还涉及cas.authn.attributeRepository.jdbc[0].attributes.xxx属性设置

>> 备注客户端获取属性

```JAVA
   AttributePrincipal principal = (AttributePrincipal) request.getUserPrincipal();

   final Map attributes = principal.getAttributes();
   
```

**第十步：新建数据库及表**

```SQL
CREATE DATABASE `castest` /*!40100 DEFAULT CHARACTER SET utf8 */
CREATE TABLE `cas_user` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `name` varchar(50) DEFAULT NULL,
  `password` varchar(50) DEFAULT NULL,
  `roles` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8

DROP TABLE IF EXISTS `OAUTH_TOKENS`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `OAUTH_TOKENS` (
  `TYPE` varchar(31) NOT NULL,
  `ID` varchar(255) NOT NULL,
  `NUMBER_OF_TIMES_USED` int(11) DEFAULT NULL,
  `CREATION_TIME` datetime DEFAULT NULL,
  `EXPIRATION_POLICY` longblob NOT NULL,
  `LAST_TIME_USED` datetime DEFAULT NULL,
  `PREVIOUS_LAST_TIME_USED` datetime DEFAULT NULL,
  `AUTHENTICATION` longblob NOT NULL,
  `SERVICE` longblob NOT NULL,
  `scopes` longblob,
  `ticketGrantingTicket_ID` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`ID`),
  KEY `FKq0iyiw6b2rufxe2ocmpsqdo50` (`ticketGrantingTicket_ID`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `OAUTH_TOKENS`
--

LOCK TABLES `OAUTH_TOKENS` WRITE;
/*!40000 ALTER TABLE `OAUTH_TOKENS` DISABLE KEYS */;
/*!40000 ALTER TABLE `OAUTH_TOKENS` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `RegisteredService_Contacts`
--

DROP TABLE IF EXISTS `RegisteredService_Contacts`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `RegisteredService_Contacts` (
  `AbstractRegisteredService_id` bigint(20) NOT NULL,
  `contacts_id` bigint(20) NOT NULL,
  `contacts_ORDER` int(11) NOT NULL,
  PRIMARY KEY (`AbstractRegisteredService_id`,`contacts_ORDER`),
  UNIQUE KEY `UK_s7mf4a23wejqx62tt4vh3tgwi` (`contacts_id`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `RegisteredService_Contacts`
--

LOCK TABLES `RegisteredService_Contacts` WRITE;
/*!40000 ALTER TABLE `RegisteredService_Contacts` DISABLE KEYS */;
/*!40000 ALTER TABLE `RegisteredService_Contacts` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `RegexRegisteredService`
--

DROP TABLE IF EXISTS `RegexRegisteredService`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `RegexRegisteredService` (
  `expression_type` varchar(50) NOT NULL DEFAULT 'regex',
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `access_strategy` longblob,
  `attribute_release` longblob,
  `description` varchar(255) DEFAULT NULL,
  `evaluation_order` int(11) NOT NULL,
  `expiration_policy` longblob,
  `informationUrl` varchar(255) DEFAULT NULL,
  `logo` varchar(255) DEFAULT NULL,
  `logout_type` int(11) DEFAULT NULL,
  `logout_url` varchar(255) DEFAULT NULL,
  `mfa_policy` longblob,
  `name` varchar(255) NOT NULL,
  `privacyUrl` varchar(255) DEFAULT NULL,
  `proxy_policy` longblob,
  `public_key` longblob,
  `required_handlers` longblob,
  `serviceId` varchar(255) NOT NULL,
  `theme` varchar(255) DEFAULT NULL,
  `username_attr` longblob,
  `bypassApprovalPrompt` bit(1) DEFAULT NULL,
  `clientId` varchar(255) DEFAULT NULL,
  `clientSecret` varchar(255) DEFAULT NULL,
  `generateRefreshToken` bit(1) DEFAULT NULL,
  `jsonFormat` bit(1) DEFAULT NULL,
  `supported_grants` longblob,
  `supported_responses` longblob,
  PRIMARY KEY (`id`)
) ENGINE=MyISAM AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;
/*!40101 SET character_set_client = @saved_cs_client */;
DROP TABLE IF EXISTS `RegisteredServiceImpl_Props`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `RegisteredServiceImpl_Props` (
  `AbstractRegisteredService_id` bigint(20) NOT NULL,
  `properties_id` bigint(20) NOT NULL,
  `properties_KEY` varchar(255) NOT NULL,
  PRIMARY KEY (`AbstractRegisteredService_id`,`properties_KEY`),
  UNIQUE KEY `UK_i2mjaqjwxpvurc6aefjkx5x97` (`properties_id`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `RegisteredServiceImpl_Props`
--

LOCK TABLES `RegisteredServiceImpl_Props` WRITE;
/*!40000 ALTER TABLE `RegisteredServiceImpl_Props` DISABLE KEYS */;
/*!40000 ALTER TABLE `RegisteredServiceImpl_Props` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `cas_user`
--

DROP TABLE IF EXISTS `cas_user`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `cas_user` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `name` varchar(50) DEFAULT NULL,
  `password` varchar(50) DEFAULT NULL,
  `roles` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `cas_user`
--

LOCK TABLES `cas_user` WRITE;
/*!40000 ALTER TABLE `cas_user` DISABLE KEYS */;
INSERT INTO `cas_user` VALUES (2,'lxh','lxh','ROLE_ADMIN');
/*!40000 ALTER TABLE `cas_user` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `locks`
--

DROP TABLE IF EXISTS `locks`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `locks` (
  `application_id` varchar(255) NOT NULL,
  `expiration_date` datetime DEFAULT NULL,
  `unique_id` varchar(255) DEFAULT NULL,
  `lockVer` int(11) NOT NULL DEFAULT '0',
  PRIMARY KEY (`application_id`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `locks`
--

LOCK TABLES `locks` WRITE;
/*!40000 ALTER TABLE `locks` DISABLE KEYS */;
INSERT INTO `locks` VALUES ('cas-ticket-registry-cleaner',NULL,NULL,1);
/*!40000 ALTER TABLE `locks` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `RegexRegisteredServiceProperty`
--

DROP TABLE IF EXISTS `RegexRegisteredServiceProperty`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `RegexRegisteredServiceProperty` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `property_values` longblob,
  PRIMARY KEY (`id`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `RegexRegisteredServiceProperty`
--

LOCK TABLES `RegexRegisteredServiceProperty` WRITE;
/*!40000 ALTER TABLE `RegexRegisteredServiceProperty` DISABLE KEYS */;
/*!40000 ALTER TABLE `RegexRegisteredServiceProperty` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `SERVICETICKET`
--

DROP TABLE IF EXISTS `SERVICETICKET`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `SERVICETICKET` (
  `TYPE` varchar(31) NOT NULL,
  `ID` varchar(255) NOT NULL,
  `NUMBER_OF_TIMES_USED` int(11) DEFAULT NULL,
  `CREATION_TIME` datetime DEFAULT NULL,
  `EXPIRATION_POLICY` longblob NOT NULL,
  `LAST_TIME_USED` datetime DEFAULT NULL,
  `PREVIOUS_LAST_TIME_USED` datetime DEFAULT NULL,
  `FROM_NEW_LOGIN` bit(1) NOT NULL,
  `TICKET_ALREADY_GRANTED` bit(1) NOT NULL,
  `SERVICE` longblob NOT NULL,
  `ticketGrantingTicket_ID` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`ID`),
  KEY `FK60oigifivx01ts3n8vboyqs38` (`ticketGrantingTicket_ID`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `SERVICETICKET`
--

LOCK TABLES `SERVICETICKET` WRITE;
/*!40000 ALTER TABLE `SERVICETICKET` DISABLE KEYS */;
/*!40000 ALTER TABLE `SERVICETICKET` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `TICKETGRANTINGTICKET`
--

DROP TABLE IF EXISTS `TICKETGRANTINGTICKET`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `TICKETGRANTINGTICKET` (
  `TYPE` varchar(31) NOT NULL,
  `ID` varchar(255) NOT NULL,
  `NUMBER_OF_TIMES_USED` int(11) DEFAULT NULL,
  `CREATION_TIME` datetime DEFAULT NULL,
  `EXPIRATION_POLICY` longblob NOT NULL,
  `LAST_TIME_USED` datetime DEFAULT NULL,
  `PREVIOUS_LAST_TIME_USED` datetime DEFAULT NULL,
  `AUTHENTICATION` longblob NOT NULL,
  `DESCENDANT_TICKETS` longblob NOT NULL,
  `EXPIRED` bit(1) NOT NULL,
  `PROXIED_BY` longblob,
  `PROXY_GRANTING_TICKETS` longblob NOT NULL,
  `SERVICES_GRANTED_ACCESS_TO` longblob NOT NULL,
  `ticketGrantingTicket_ID` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`ID`),
  KEY `FKiqyu3qw2fxf5qaqin02mox8r4` (`ticketGrantingTicket_ID`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
/*!40101 SET character_set_client = @saved_cs_client */;
DROP TABLE IF EXISTS `RegisteredServiceImplContact`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `RegisteredServiceImplContact` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `department` varchar(255) DEFAULT NULL,
  `email` varchar(255) DEFAULT NULL,
  `name` varchar(255) DEFAULT NULL,
  `phone` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
/*!40101 SET character_set_client = @saved_cs_client */;
```

**第十一步：启动tomcat8，进行验证**

在浏览器中输入 https://example.org:8443/cas 如果出现登陆页面表示部署成功,如果使用cas_user表中的用户名和密码进行登陆表示认证功能正常。



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
