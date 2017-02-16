---
layout: post
title: "Google BeyondCorp 论文阅读"
categories: notes
tags: 内网安全
---

## 论文

总共三篇：

1. BeyondCorp: A New Approach to Enterprise Security

    https://research.google.com/pubs/archive/43231.pdf
    
    主要介绍BeyondCorp整体的架构。读起来无比轻松无比熟悉，因为跟阿里的内网认证几乎完全一致，只是阿里相对少了Access Proxy。而也就是缺少这个，导致了阿里内网不能开放出去，同时决定了阿里的内网安全简直不能看。
    
    反过来看，BeyondCorp的目的就是为了开放内网，让内网变得跟外网一样安全。所以。。。

2. BeyondCorp: Design to Deployment at Google

    https://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/44860.pdf

    介绍了BeyondCorp在谷歌的实现。没有什么特别可说的，case by case。

3. Beyond Corp: The Access Proxy

    https://research.google.com/pubs/archive/45728.pdf

    重点讲了下Access Proxy的实现细节。
    
    "We hope that by sharing
our solutions to challenges like multi-platform authentication
and special cases and exceptions, and the lessons we learned
during this project, our experience can help other organizations
to undertake similar solutions with minimal pain."。

    其中有几点值得关注。
    
    "Mutual authentication between the proxy and the back end"。前端到proxy需要认证这个没问题。proxy到后端就是内网的部分了。这部分的认证估计大多数公司都是忽略的，进而导致了验证的风险。这就涉及到一个安全边界和纵深防御的问题了。
    
    "Implementing a domain-specific language for ACLs was key in
tackling challenges of centralized authorization"。不得不说谷歌的研发能力确实强大，一言不合就上DSL。大多团队一想到自己要设计DSL就放弃了，而使用既有的语言来实现domain specific的功能又容易被用户滥用，像JAVA世界里的groovy，已经被玩坏了。

    "Challenges with Multi-Platform Authentication"。文章中表述的谷歌的解决方案并不完美。首先实现了两套认证体系，一套桌面设备的证书体系，一套手机设备的设备标识体系。优点在于实现了"Wrapping XXX traffic in HTTP over TLS"，这样后端就只用考虑对HTTP的支持就行，简化设计。当然文中也提到，"we plan to build
a desktop device manager, which will look quite similar to the
mobile device manager. It will provide a common identifier in the
form of a Device-User-Session-ID (DUSI) that’s shared across
all browsers and tools using a common OAuth token-granting
daemon."，希望能够以一套认证机制取代目前的两套的状况。

    统一系统的好处不言而喻。很多策略，拓展，如果系统没有统一，导致的爆炸的复杂度的影响是非常可怕的，可能会直接导致很多东西因为成本过高而不可行。
    
    "Emergencies Happen"。这个就不说了。
    
    "Engineers Need Support"。文档！文档！文档！又是一声叹息。。。
    
    完。














    


    

