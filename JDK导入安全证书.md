目前在研究cas sso相关的搭建知识。开始便是关于HTTPS安全证书的一些问题，简单说一下本文的目的：安全证书的生成，jdk导入安全证书。

先说自己定义的域名

编辑文件 C:\\Windows\\System32\\drivers\\etc\\hosts 在文件末端添加下面的信息：

![](https://github.com/louisliaoxh1989/sso_learn/blob/master/images/1/1.png)

1：win+R 快捷键打开DOS命令窗口

2：输入以下命令，生成证书

    keytool -genkey -alias ssodemo -keyalg RSA -keysize 1024 -keypass ywj123 -validity 365 -keystore G:\ywj_cas.keystore -storepass ywj123

解释一下：

-alias后面的ssodemo 是我生成证书的别名 ，

-keypass ywj123是我要生成证书的密码，

-storepass ywj123要与上面的-keypass密码相同

-keystore G:\\ywj_cas.keystore 指定证书的位置

![](https://github.com/louisliaoxh1989/sso_learn/blob/master/images/1/2.png)

【注意】：第一个让你输入的“您的名字与姓氏是什么”，请必须输入在C:\\Windows\\System32\\drivers\\etc\\hosts文件中加入的服务端的域名。

我这里也就是server.ywj123.com，为何这么做？

首先cas只能通过域名来访问，不能通过ip访问，同时上方是生成证书，所以要求比较严格，所以如果不这么做的话，及时最终按照教程配置完成，cas也可以正常访问,访问一个客户端应用虽然能进入cas验证首页，但是，当输入信息正确后，cas在回调转入你想访问的客户端应用的时候，会出现No subject alternative names present错误异常信息，这个错误也就是在上面输入的第一个问题答案不是域名导致、或者与hosts文件配置的不一致导致

3：导出证书

在cmd窗口继续输入以下命令，导出证书:

    keytool -export -alias ssodemo -keystore G:\ywj_cas.keystore -file G:\ssodemo.crt -storepass ywj123

  

说明：-alias后面的名称要与生成证书的命令里面的alias的名称一致. –keystore后面指定证书存放的位置，这里我放在G盘根目录，同时证书名称要与【生成证书】对应的命令里的keystore名称一致.这里是ywj_cas.keystore，-file后面才crt路径，我也指定在G盘根目录. –storepass的证书密码要与上面输入的密码一致.

![](https://github.com/louisliaoxh1989/sso_learn/blob/master/images/1/3.png)

![]https://github.com/louisliaoxh1989/sso_learn/blob/master/images/1/4.png)

补充一下：这一步可能出现的权限问题如下图：

![](https://github.com/louisliaoxh1989/sso_learn/blob/master/images/1/5.png)

这个很简单，只要在所需要写入的磁盘中增加访问

![](https://github.com/louisliaoxh1989/sso_learn/blob/master/images/1/6.png)

  

4：JDK导入证书

由于是本地没有证书，证书是自己生成的，所以，务必将生成的证书导入到jre的证书链中，不然是无法支持CAS认证服务的。

 在cmd窗口输入命令:

  

    keytool -import -keystore "%JAVA_HOME%\jre\lib\security\cacerts" -file G:\ssodemo.crt -alias ssodemo

这里一定要注意一点首先使用%JAVA_HOME%这个路径你是在自己系统环境变量中配置好的jdk路径的，其次一定要主要双""别忘记加了。我就是在这里采坑的，另外还有就是输入这行命令的前提路径是进入你的cd G:\\ProgramFiles\\Java\\jdk1.7.0_17\\jre\\lib\\security这个jdk的安装路径

上图：  

![](https://github.com/louisliaoxh1989/sso_learn/blob/master/images/1/7.png)

第一个红色框是我进入的JDK目录第二个我没有加""报的错，第三个是我正确输入路径之后展示的效果就是正确的

接下来输入密码changeit这是默认额的密码建议不要修改，不是输入上面的keypass密码啊，不要弄错了。

![](https://github.com/louisliaoxh1989/sso_learn/blob/master/images/1/8.png)

紧接着输入y便可JDK导入安全证书成功。

本人借鉴http://www.cnblogs.com/zhoubang521/p/5200407.html此篇博客加以补充自己踩过的坑，介绍的会更加详细，后面我会把自己真的项目的部署过程公开给大家学习。

附常用命令：

//查看cacerts中的证书列表：

              keytool -list -keystore "%JAVA_HOME%/jre/lib/security/cacerts"  -storepass changeit

//删除cacerts中指定名称的证书：

              keytool -delete -alias ssodemo -keystore "%JAVA_HOME%/jre/lib/security/cacerts"  -storepass changeit

//导入指定证书到cacerts：  
              keytool -import -alias ssodemo -file ssodemo.cer -keystore"%JAVA_HOME%/jre/lib/security/cacerts"  -storepasschangeit-trustcacerts
