---
layout: post
title: "escapeshellcmd绕过探究"
categories: notes
tags: php bypass
---

网上看到针对escapeshellcmd存在两种绕过方式：

1. 在windows里，如果将接收到的命令写入.bat文件，然后执行该.bat文件，则可以通过0x1a绕过
2. 宽字符绕过

首先看到第一种方法的时候，没太懂是什么意思。经过一番搜索stackoverflow后，发现ASCII 26(0x1a)在windows上其实表示的是文件终止符(EOF)。.bat文件在解析的时候，碰到终止符则表示当前命令执行结束了。然后接下来执行下面的一行命令。

第二种没什么好说的了，C语言(php用C开发的)在处理输入的时候，如果按单字节处理，那么就存在较为严重的多字节问题，特别是对于GBK这类编码。在Java，Python等语言中就不存在这样的问题，因为这些语言在设计的时候就考虑到了内部处理的一致性，采用定长UTF-16作为内码。