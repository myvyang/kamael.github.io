### Important: Remote Code Execution CVE-2016-8735

这个漏洞某种程度上可看作一个漏洞系列。

漏洞的原因见 http://engineering.pivotal.io/post/java-deserialization-jmx/ ，大意就是，当开发用JMX对外提供服务的时候，JMX是可以设置账号密码的。然而，账号密码的提交，本身也是通过序列化方式解析的，这就导致了攻击者可以利用反序列化漏洞，在未认证的情况下(即使服务器要求认证)任意命令执行。

最开始漏洞提交者发现Tomcat存在这个问题，但是想不到好的修复方案，因此把漏洞提交给了Oracle。然后Oracle为JMX添加了一个[限制功能](https://www.java.com/zh_CN/download/faq/release_changes.xml)，即在提交认证的时候，采用白名单的类限制。Oracle编号CVE-2016-3427。

于是tomcat就[在通用功能的基础上，加强了自己的保护](http://svn.apache.org/viewvc/tomcat/tc8.5.x/trunk/java/org/apache/catalina/mbeans/JmxRemoteLifecycleListener.java?r1=1767646&r2=1767645&pathrev=1767646&diff_format=s)。显然，如果Java不升级，只升级Tomcat的意义不大。

从另一方面讲，由于JMX服务本就不应该开放给不受信的网络(特别是公网)，因此这个漏洞本身可以视为无影响的。

### CVE-2016-6816

关于这类漏洞的详情，可以搜索[HTTP-Request-Smuggling]()。老问题了。

简而言之，只要webserver在处理HTTP头的时候没用严格按照规范来，就有可能存在这类漏洞。只要你找到一个proxy和一个webserver，对于同一个HTTP头的处理不一致的实例，就可以提交一个漏洞了。