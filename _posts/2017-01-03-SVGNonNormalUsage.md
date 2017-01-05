---
layout: post
title: "SVG中非显而易见的用法"
categories: notes
tags: SVG
---

### \<use\>

```
<svg width="140" height="170">
<g id="whiskers">
   <line x1="75" y1="95" x2="135" y2="85" style="stroke: black;"/>
   <line x1="75" y1="95" x2="135" y2="105" style="stroke: black;"/>
</g>
<use xlink:href="#whiskers" transform="scale(-1 1) translate(-140 0)"/>
</svg>
```

表示重用。出于安全考虑，有些浏览器并不支持外部 SVG 文件的引用，尤其是 IE。有部分浏览器也仅支持引用同域或者配置了允许跨域访问的文件。

### \<set\>

```
<circle cx="60" cy="60" r="30" style="fill: #ff9; stroke: gray;">
    <animate id="c1" attributeName="r" attributeType="XML"
        begin="0s" dur="4s" from="30" to="0" fill="freeze"/>
</circle>

<text text-anchor="middle" x="60" y="60" style="visibility: hidden;">
    <set attributeName="visibility" attributeType="CSS"
        to="visible" begin="4.5s" dur="1s" fill="freeze"/>  
    All gone!
</text>
```

和动画大体一样。不同之处在于`to`可以设置为非数字值，可以实现比较特殊的变换。

### 动画标签的targetElement和href

可以指定动画应用到哪个节点上。默认用在父节点。

### \<script\>的href属性

```
xlink:href = "<iri>"
```

An IRI reference to an external resource containing the script code.

### XSS Auditor Bypass

```
<svg><set href=#script attributeName=href to=data:,alert() /><script id=script src=foo></script>
```

如果没有把SVG相关的加黑名单，可以通过这种变换来实现JS执行。




