---
layout: post
title: "ReDoS静态分析论文阅读笔记"
categories: notes
tags: DOS REGEX
---

## 提要

论文为`Static Analysis for Regular Expression
Denial-of-Service Attacks`和`Static Analysis for Regular Expression
Exponential Runtime via Substructural Logics`，分别对应RXXR和RXXR2两个软件。

二者功能一致，均为通过静态分析的方式进行整治表达式的ReDoS问题的分析。后者为前者的改进版。

我先读的前者论文，读的云里雾里。然后找到了后者。后者的篇幅除了对"如何进行静态分析"外，还花了大笔墨证明了这种分析的正确性，搞学术的人还真是辛苦。。。

为了更好的理解这个问题，我找来了代码进行详细分析。

代码由Ocaml编写，里面变量命名很抽象，读懂废了好大劲。。。

不过读过一遍后再回头来看，却发现逻辑还是很简单清晰的，和论文描述基本一致。

## 代码结构

我阅读和注释的代码存放在[https://github.com/kamael/rxxr2](https://github.com/kamael/rxxr2)，见其中的README。

代码的主要逻辑在`AnalyserMain-> search_optimized`中，满足如下调用关心。

```
search_optimized ->
    search_x ->
        XAnalyser.next
        search_y1 ->
            Y1Analyser.next
            search_y2 ->
                Y2Analyser.next
                search_z ->
                    ZAnalyser.next
```
其中`X, Y, Z`的含义见论文第7段。

可见主要的搜索过程在几个`next`函数中。

## 摘要分析

在分析过程中采用如下输入：

```
# Vulnerable
/zzz(a|ab|b)*kk/
```

`RegexScanner.ml` 第35行，`let p = ParsingMain.parse_pattern lexbuf` 的输出值为

```
p: ParsingData.pattern =
  (r, 8)

r: ParsingData.regex = 
  (Conc
    ((Atom (Char 'z'),
      {spos = 0; epos = 0; scount = 0; nullable = false; cflags = 0}),
    (Conc
      ((Atom (Char 'z'),
        {spos = 1; epos = 1; scount = 0; nullable = false; cflags = 0}),
      (Conc
        ((Atom (Char 'z'),
          {spos = 2; epos = 2; scount = 0; nullable = false; cflags = 0}),
        (Conc
          ((Kleene (Gq,
             (Group (CAP 1, 0, 0,
               (Alt
                 ((Atom (Char 'a'),
                   {spos = 4; epos = 4; scount = 0; nullable = false;
                    cflags = 0}),
                 (Alt
                   ((Conc
                      ((Atom (Char 'a'),
                         {spos = 6; epos = 6; scount = 0; nullable = false;
                          cflags = 0}),
                        (Atom (Char 'a'),
                          {spos = 7; epos = 7; scount = 0; nullable = false;
                           cflags = 0})),
                        {spos = 6; epos = 7; scount = 0; nullable = false;
                         cflags = 0}),
                      (Atom (Char 'b'),
                       {spos = 9; epos = 9; scount = 0; nullable = false;
                        cflags = 0})),
                     {spos = 6; epos = 9; scount = 0; nullable = false;
                      cflags = 0})),
                  {spos = 4; epos = 9; scount = 0; nullable = false;
                   cflags = 0})),
                {spos = 3; epos = 10; scount = 0; nullable = false;
                 cflags = 0})),
              {spos = 3; epos = 11; scount = 0; nullable = false; cflags = 0}),
             (Conc
               ((Atom (Char 'k'),
                 {spos = 12; epos = 12; scount = 0; nullable = false;
                  cflags = 0}),
               (Atom (Char 'k'),
                {spos = 13; epos = 13; scount = 0; nullable = false;
                 cflags = 0})),
              {spos = 12; epos = 13; scount = 0; nullable = false;
               cflags = 0})),
            {spos = 3; epos = 13; scount = 0; nullable = false; cflags = 0})),
         {spos = 2; epos = 13; scount = 0; nullable = false; cflags = 0})),
       {spos = 1; epos = 13; scount = 0; nullable = false; cflags = 0})),
     {spos = 0; epos = 15; scount = 0; nullable = false; cflags = 0})
     
结构图：

  // zzz(a|ab|b)*kk

  Conc
    Atom (Char 'z')
    Conc
      Atom (Char 'z')
      Conc
        Atom (Char 'z')
        Conc
          Kleene Gq
             Group
               CAP
               1
               0
               0
               Alt
                 Atom (Char 'a')
                 Alt
                   Conc
                     Atom (Char 'a')
                     Atom (Char 'b')
                   Atom (Char 'b')
        Conc
          Atom (Char 'k'),
          Atom (Char 'k'),

```

一个巨大的tuple，通过这个tuple的结构，可以得知`ParsingData.pattern`，即Regex语法解析的各个字段的含义。


转换后的NFA：

```
nfa: t =
  {states =
    [|Match ([('z', 'z')], 1); Match ([('z', 'z')], 2);
      Match ([('z', 'z')], 11); BeginCap (1, 4); BranchAlt (5, 6);
      Match ([('a', 'a')], 10); BranchAlt (7, 9); Match ([('a', 'a')], 8);
      Match ([('b', 'b')], 10); Match ([('b', 'b')], 10); EndCap (1, 11);
      BranchKln (true, 3, 12); Match ([('k', 'k')], 13);
      Match ([('k', 'k')], 14); End|];
   transitions =
    [|
    Some [('z', 'z', 1)];
    Some [('z', 'z', 2)];
    Some [('z', 'z', 11)];
    None; = @10
    None; = @10
    Some [('a', 'a', 10)];
    Some [('a', 'a', 8); ('b', 'b', 10)];
    Some [('a', 'a', 8)];
    Some [('b', 'b', 10)];
    Some [('b', 'b', 10)];
    Some [('k', 'k', 13); ('a', 'a', 10); ('a', 'a', 8); ('b', 'b', 10)];
    None; = @10
    Some [('k', 'k', 13)];
    Some [('k', 'k', 14)];
    Some [(zmin, zmax, 14)]; = @End
    |];
   positions =
    [|(0, 0); (1, 1); (2, 2); (3, 3); (4, 9); (4, 4); (6, 9); (6, 6);
      (7, 7); (9, 9); (10, 10); (3, 11); (12, 12); (13, 13); (15, 15)|];
   root = 0}

```

NFA的立体结构

```
0('z') -> 1('z') -> 2('z') -> 11('*') -> 12('k') -> 13('k') -> 14(End)
                             /    ^
                            /      \
                           V        \
                       3('(')        \
                          V           \
                      4('|')           \
                         / \            \
                        V   V            \ 
                     5('a') 6('|')        \
                       |      / \         |
                       |     V   V        |
                       | 7('a')  9('b')   |
                       |    V      |     /
                       | 8('b')    |    /
                        \    |    /    /
                         V   V   V    /
                           10(')') __/
```

`nfa.t.transitions` 表示的是消去 $\epsilon$ 的NFA转移路径。

碰到的一个分析了好久的问题是`Beta->otree`这个数据结构到底是干嘛的。

最后多次尝试后发现，是用来合并多余的跳转分枝的。语言难以描述，用如下输入输出比较好理解

输入

```
/z([b-e]kkk|flll|[c-e]mmm|[a-c]nnn|[a-f]ooo)zz(a|ab|b)*kk/
```

对应的`otree`

```
  OTNode ('c', 'c', [23; 19; 14; 4],
   OTNode ('b', 'b', [23; 19; 4],
    OTNode ('a', 'a', [23; 19], OTNull, OTNull), OTNull),
   OTNode ('f', 'f', [23; 9], OTNode ('d', 'e', [23; 14; 4], OTNull, OTNull),
    OTNull))
```

`otree`保证了在后路径中，单个字母对应的多个跳转路径的合并，整理的比较合理。这个可能是正则表达式实现里的一个常用手法(未知)。

## 最后

先到这里，要理解还是得自己看代码，弄懂里面的数据结构和逻辑。实现这样一个分析器某种意义上等于实现了一个正则表达式引擎了，还是很麻烦的。RXXR2的实现还有些兼容问题。比较，各个语言支持的正则语法还有区别呢。

此外确实不喜Ocaml这个语言。。。花了几个小时学了学语法，就为了看懂这代码，看的还挺累。除此之外实现没兴趣学这个语言。








