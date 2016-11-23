---
layout: post
title: "Python的'BUG'"
categories: notes
tags: Python
---


今天抽空大概看了[Deep Dive Into Python's VM: Story of LOAD_CONST Bug](http://doar-e.github.io/blog/2014/04/17/deep-dive-into-pythons-vm-story-of-load_const-bug/)这篇文章。

简而言之，Python开发者认为，如果你可以操作Python字节码，那么你就有权执行任何二进制指令。虽然某种程度上，他们认可这是个BUG，但是并不打算花时间去修复。

在我还在上大学的时候，尝试过学习[LangFuzz](https://www.st.cs.uni-saarland.de/publications/files/holler-usenix-2012.pdf)去写一个针对Python的fuzzer。令我欣喜的是，fuzzer刚跑完没多久，就找到了一堆crash。

于是我兴冲冲的把这些crash中的一个报告给了security@python.org。

然后杳无音讯。

过了几天我发了一封邮件又去问了一遍，没想到Guido van Rossum亲自回复了我：

```
Dear MengYuan,

We don't consider this a security vulnerability. You can create function objects with faulty bytecode too. If you would like to prevent some of the crashes this can cause, submit a patch (or at least report a specific case as a bug) at bugs.python.org.

Take care,

--Guido

```

当时的crash的PoC是这样的：

```
import ast

m = ast.parse("[a]")
n = m.body[0].value.elts[0]
n.ctx = ast.Store()
exec compile(m, '<string>', 'exec')
```

很可能今天依然可以导致Python crash

当时发现的crash很多，我简单分析过其中一些，可以直接影响Python的引用计数，应该是可以构造UAF的。

不过后来，我发现了这个：

https://github.com/python/cpython/tree/c30098c8c6014f3340a369a31df9c74bdbacc269/Lib/test/crashers

```
This directory only contains tests for outstanding bugs that cause the
interpreter to segfault.  Ideally this directory should always be empty, but
sometimes it may not be easy to fix the underlying cause and the bug is deemed
too obscure to invest the effort.

Each test should fail when run from the command line:

	./python Lib/test/crashers/weakref_in_del.py

Put as much info into a docstring or comments to help determine the cause of the
failure, as well as a bugs.python.org issue number if it exists.  Particularly
note if the cause is system or environment dependent and what the variables are.

Once the crash is fixed, the test case should be moved into an appropriate test
(even if it was originally from the test suite).  This ensures the regression
doesn't happen again.  And if it does, it should be easier to track down.

Also see Lib/test_crashers.py which exercises the crashers in this directory.
In particular, make sure to add any new infinite loop crashers to the black
list so it doesn't try to run them.
```

原来开发者们，早就收集了一大堆可以导致Python crash的代码了。可能修复这些BUG需要整体进行较大的改动，于是他们决定暂时不修。大体还是基于上面的原则：如果你都可以在机器里执行这些代码了，为什么要限制你去控制机器呢。

可能其他开发者不这么认为。因为有SaaS的概念，软件的安全要求本身就被拓展了。

翻阅Python历史上的安全漏洞，大多都是函数调用即可导致攻击得惩的才算。不同的开发者或安全工作者，对于什么算安全漏洞，存在比较大的分歧(当然也和场景有关，比如Javascript，源码本身就是不受信的)。安全问题，还是要结合具体的场景去定义。

不过换个角度看，作为兴趣爱好，自然没必要纠结这些。只是Python的crash这么容易找到，很大的原因也就是因为它并不会导致严重的问题，否则早就被挖完修完了。同理，如果你在nginx里挖到了一个CVE，那就是很了不起的事。如果是nginx的非默认配置，可能需要打个折扣。如果是挖了一个没多少人知道的web server的CVE，虽然也不错，但是可能并代表不了什么，到现在还存在轻易就能被发现的漏洞，只是因为关注的人比较少罢了。


