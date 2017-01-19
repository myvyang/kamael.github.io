---
layout: post
title: "Java下常见的远程命令执行问题汇总"
categories: notes
tags: Java 命令执行
---

下面列出了目前java上较为常见的，可导致远程入侵的问题点。

像JMX，JMS，RMI，RPC这些，都是本只应该在内网环境使用的，因为这些功能本身在设计之初就预设了"只会在可信环境使用"的前提，因此本身就没怎么考虑过安全问题。

在实际使用中，为了实现高性能(鉴权会拉低性能)，高扩展性，分布式部署(鉴权会导致密钥管理困难)，业务方基本不会主动添加鉴权。进一步的导致了这些服务，本身只应在可信环境里使用。

因此，粗略的讲，任何讲这类服务，开放到非可信环境的行为，本身就等同于公开后门。

由于这些服务在长期的发展中，逐渐成为了企业级JAVA应用的基础，因此在各种WebServer，框架中被广泛的封装。应用使用时，都是通过直接，间接，或间接的间接的方式。在实际场景中，也因此存在碎片化，长尾的问题，进一步加大了管控的难度。

### JMX(Java Management Extensions)

JMX本身为一套规范，然后各个JAVA运行时环境(例如openjdk)都内置了对应的API。应用可以直接调用API来启动一个Server端，或者通过JAVA参数启动。

各种广泛的服务都内部集成了JMX。wikipedia列出了其中比较知名的：

```
JMX is supported by Java application servers such as OpenCloud Rhino Application Server [1], JBoss, JOnAS, WebSphere Application Server, WebLogic, SAP NetWeaver Application Server, Oracle Application Server 10g and Sun Java System Application Server.
JMX is supported by the UnboundID Directory Server, Directory Proxy Server, and Synchronization Server.[9]
Systems management tools that support the protocol include Empirix OneSight, GroundWork Monitor, Hyperic, HP OpenView, IBM Director, ITRS Geneos, Nimsoft NMS, OpenNMS,[10] Zabbix, Zenoss, and Zyrion, Solarwinds, Uptime Infrastructure Monitor, and LogicMonitor.[11]
JMX is also supported by servlet containers such as Apache Tomcat.[12] & Jetty (web server)
MX4J [2] is Open Source JMX for Enterprise Computing.
jManage [3] is an open source enterprise-grade JMX Console with Web and command-line interfaces.
MC4J [4] is an open source visual console for connecting to servers supporting JMX
snmpAdaptor4j [5] is an open source providing a simple access to MBeans via the SNMP protocol.
jvmtop is a lightweight open source JMX monitoring tool for the command-line
```
此外Spring等框架也都支持配置开放JMX功能，在企业内网分布异常广泛。

服务器对外开放的JMX端口和形式是多种多样的。然而，底层处理逻辑则都是JAVA原生的API。

需要全部监控起来，则需要将我们用到的所有的配置都分析一遍。

### RMI(Remote Method Invokation)

使用情况和安全难点和JMX基本一致，各框架都有包装的配置。部分RMI功能本身就支持原创命令执行，其他的也可以通过JAVA序列化漏洞可远程命令执行。

### RPC(Remote Procedura Call)

严格来讲，上面的RMI其实属于RPC的一种，只是JAVA原生有支持RMI的API，导致使用方式不同，上面单列。

### JMS(JAVA Message Service)

JMS规范本身规定了需要支持对象的传输，导致了所有实现JMS的消息队列中间件均存在JAVA反序列化漏洞。

例如开源的activeMQ，RakkitMQ那一堆，均存在该问题。

### 表达式

代表性的是Struts2的ognl，Spring的spel，如果被外部输入污染，也可导致远程命令执行。

Struts2的该类问题大爆发，而Spring主框架中目前没有出过漏洞，都是一些组件出的问题(spring-security-oauth, springboot)。经分析，spel在Spring中使用非常广泛，多用来实现强大的配置功能。如果不加限制，在可预见的将来，可能会变成跟Struts2一样的问题。

### 序列化问题

见 https://github.com/frohoff/ysoserial/ ，事实基本已经证明，序列化问题的锅并不在lib本身，而是这个功能本身就应视作等同于远程代码执行。因此修复方案需要以对功能本身进行限制为主要目标，而非新出一个POC，就强制全网升级相关的三方包。







