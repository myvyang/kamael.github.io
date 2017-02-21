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
    
    主要介绍BeyondCorp整体的架构。文笔详实。大体上和阿里的体系非常接近，大多数设施可以找到一一对应的点。不过阿里缺失Access Proxy这个模块，而谷歌的"开放内网"策略在很大程度上依赖于此。
    
    反过来说，BeyondCorp项目的目的即在于通过"开放内网"来倒逼内部应用和外部应用一样安全，如果不包含BeyondCorp就不可能达到这个目的了。

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
    
    "Mutual authentication between the proxy and the back end"。前端到proxy需要认证这个没问题。proxy到后端是内网的部分。这部分的认证大多数公司都是忽略的，而谷歌对此作了要求。谷歌内网的不同服务之前保持相互不可信的状态。
    
    "Implementing a domain-specific language for ACLs was key in
tackling challenges of centralized authorization"。谷歌的研发能力确实强大，一言不合就上DSL。由于ACL都是嵌入程序中使用的，而应用的场景非常庞杂，因此简单的配置难以保证要求，最后的结果会是开发渐渐懒于配置，使用类似通配符之类的功能，ACL形同虚设。而使用DSL保证了配置的灵活性，同时开发肯定具备使用DSL的能力。

    "Challenges with Multi-Platform Authentication"。文章中表述的谷歌的解决方案并不完美。首先实现了两套认证体系，一套桌面设备的证书体系，一套手机设备的设备标识体系。优点在于实现了"Wrapping XXX traffic in HTTP over TLS"，这样后端就只用考虑对HTTP的支持就行，简化设计。当然文中也提到，"we plan to build
a desktop device manager, which will look quite similar to the
mobile device manager. It will provide a common identifier in the
form of a Device-User-Session-ID (DUSI) that’s shared across
all browsers and tools using a common OAuth token-granting
daemon."，希望能够以一套认证机制取代目前的两套的状况。

    统一系统的好处不言而喻。很多策略，拓展，如果系统没有统一，导致的爆炸的复杂度的影响是非常可怕的，可能会直接导致很多东西因为成本过高而不可行。
    
    "Emergencies Happen"。这个就不说了。
    
    "Engineers Need Support"。完善的文档很重要。    
    
    
    完。














    


    

