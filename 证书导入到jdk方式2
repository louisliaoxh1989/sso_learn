通讯协议为HTTPS单向认证交易时报错,异常信息主要意思为服务器提供的证书不被我们客户端信任,此时需要将安全证书导入到java的cacerts证书库。步骤如下:

第一步、下载安全证书

  

在打开的窗口中,

![](https://github.com/louisliaoxh1989/sso_learn/raw/master/images/2/1.png)  

打开的窗口中,点击下一步即可,

![](https://github.com/louisliaoxh1989/sso_learn/raw/master/images/2/2.png)

在正式编码格式中,选择指定的格式,点击下一步;

![](https://github.com/louisliaoxh1989/sso_learn/raw/master/images/2/3.png)  

指定生成证书文件的名称(此处为vbooking.cer)

![](https://github.com/louisliaoxh1989/sso_learn/raw/master/images/2/4.png)  

  

  

第二步、将证书导入java的cacerts证书库

登录Tomcat所在的机器,切换到目录 ${JAVA_HOME}/jre/lib/security, 执行如下命令:

    keytool -import -alias vbooking -keystore cacerts -file ${JAVA_HOME}/jre/lib/security/vbooking.cer

  

其中：

-alias 指定别名(推荐和证书同名)

-keystore 指定存储文件(此处固定)

-file 指定证书文件全路径(证书文件所在的目录)

注意:当切换到 cacerts 文件所在的目录时,才可指定 -keystore cacerts, 否则应该指定全路径;

此时命令行会提示你输入cacerts证书库的密码,敲入changeit即可,这是java中cacerts证书库的默认密码,当然也可自行修改。

  

可使用如下命令查看证书信息：

    keytool -list -keystore cacerts -alias vbooking

  

结果如下：

![](https://github.com/louisliaoxh1989/sso_learn/raw/master/images/2/5.png)  

如需更新证书,应先删除原证书,再导入新证书:

    cd ${JAVA_HOME}/jre/lib/securitykeytool -delete -alias vbooking -keystore cacertskeytool -import -alias vbooking -keystore cacerts -file ${JAVA_HOME}/jre/lib/security/vbooking.cerkeytool -list -keystore cacerts -alias vbooking

  

重启服务器即可。

  

参考：

[http://blog.csdn.net/ybygjy/article/details/12147281](http://blog.csdn.net/ybygjy/article/details/12147281)

[http://blog.sina.com.cn/s/blog_56d8ea9001017uo4.html](http://blog.sina.com.cn/s/blog_56d8ea9001017uo4.html)
