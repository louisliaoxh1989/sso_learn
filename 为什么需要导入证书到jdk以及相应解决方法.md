## 为什么

```
jdk的security manager默认有一堆的根证书信任。如果你的https站点证书是花钱申请的，被这些根证书所信任，那使用java来访问此https站点会非常方便。

但假如，你的https证书是自己签名的，就需要将证书导入至JDK的信任证书中，否则访问时会报SSL错误,这是由于https站点的安全证书不被JSSE所信任

假如，你的webserice是基于https来进行访问，而此站点证书是自签名类型的，那么部署时一定要使用keytool进行证书导入，否则无法正常访问。

```

## 解决方法

```
1. 一种解决这个问题的方法是按照信任管理器的处理规则，把站点的证书放到证书库文件jssecacerts中，或者把证书存放到任一TrustStore文件中，

然后设置系统属性javax.net.sll.trustStore指向该文件

2.另一种解决方法则是自己实现信任管理器类，让它信任我们指定的证书


```

### 1. 证书导入jdk

Java提供了命令行工具keytool用于创建证书或者把证书从其它文件中导入到Java自己的TrustStore文件中。把证书从其它文件导入到TrustStore文件中的命令行格式为： 
``` java
keytool -import -file src_cer_file –keystore dest_cer_store 
```
```　　
其中，src_cer_file为存有证书信息的源文件名，dest_cer_store为目标TrustStore文件。 

　　在使用keytool之前，首先要取得源证书文件，这个源文件可使用IE浏览器获得，IE浏览器会把访问过的HTTPS站点的证书保存到本地。从IE浏览器导出证书的方法是打开“Internet 选项”，选择“内容”选项卡，点击“证书…”按钮，在打开的证书对话框中，选中一个证书，然后点击“导出…”按钮，按提示一步步将该证书保存到一文件中。最后就可利用keytool把该证书导入到Java的TrustStore文件中。为了能使Java程序找到该文件，应该把这个文件复制到jre安装路径下的lib/security/目录中。 

　　这样，只需在程序中设置系统属性javax.net.sll.trustStore指向文件dest_cer_store，就能使JSSE信任该证书，从而使程序可以访问使用未经验证的证书的HTTPS站点。 

　　使用这种方法，编程非常简单，但需要手工导出服务器的证书。当服务器证书经常变化时，就需要经常进行手工导出证书的操作。下面介绍的实现X509证书信任管理器类的方法将避免手工导出证书的问题
```

### 2. X509证书信任管理器类的实现及应用

在JSSE中，证书信任管理器类就是实现了接口X509TrustManager的类。我们可以自己实现该接口，让它信任我们指定的证书。接口X509TrustManager有下述三个公有的方法需要我们实现： 
```java
　　void checkClientTrusted(X509Certificate[] chain, String authType) throws CertificateException 
```
> 该方法检查客户端的证书，若不信任该证书则抛出异常。由于我们不需要对客户端进行认证，因此我们只需要执行默认的信任管理器的这个方法。JSSE中，默认的信任管理器类为TrustManager。 

```java
　　void checkServerTrusted(X509Certificate[] chain, String authType) throws CertificateException 
```
> 该方法检查服务器的证书，若不信任该证书同样抛出异常。通过自己实现该方法，可以使之信任我们指定的任何证书。在实现该方法时，也可以简单的不做任何处理，即一个空的函数体，由于不会抛出异常，它就会信任任何证书。 

```java
X509Certificate[] getAcceptedIssuers() 
```
> 该方返回受信任的X509证书数组。 

```
自己实现了信任管理器类，如何使用呢？类HttpsURLConnection似乎并没有提供方法设置信任管理器。其实，HttpsURLConnection通过SSLSocket来建立与HTTPS的安全连接，SSLSocket对象是由 SSLSocketFactory生成的。HttpsURLConnection提供了方法 setSSLSocketFactory(SSLSocketFactory)设置它使用的SSLSocketFactory对象。 SSLSocketFactory通过SSLContext对象来获得，在初始化SSLContext对象时，可指定信任管理器对象。下面用一个图简单表示这几个JSSE类的关系： 

假设自己实现的X509TrustManager类的类名为：MyX509TrustManager，下面的代码片断说明了如何使用MyX509TrustManager： 

```

``` java  
//创建SSLContext对象，并使用我们指定的信任管理器初始化  
TrustManager[] tm = {new MyX509TrustManager ()};  
SSLContext sslContext = SSLContext.getInstance("SSL","SunJSSE");  
sslContext.init(null, tm, new java.security.SecureRandom());  
  
//从上述SSLContext对象中得到SSLSocketFactory对象  
SSLSocketFactory ssf = sslContext.getSocketFactory();  
  
//创建HttpsURLConnection对象，并设置其SSLSocketFactory对象  
HttpsURLConnection httpsConn = (HttpsURLConnection)myURL.openConnection();  
httpsConn.setSSLSocketFactory(ssf);  
```
```
这样，HttpsURLConnection对象就可以正常连接HTTPS了，
无论其证书是否经权威机构的验证，只要实现了接口X509TrustManager的类MyX509TrustManager信任该证书。 
```
