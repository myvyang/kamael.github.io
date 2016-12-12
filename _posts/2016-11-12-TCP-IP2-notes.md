---
layout: post
title: "TCP/IP详解卷二读书笔记"
categories: notes
tags: 读书笔记
---

==

2.5.2 里有个编程技巧

```
#define MGET(m. how, type) { \
  MALLOC((m), ...) \
  if (m) { \
    ... \
  } else { \
    (m) = m_retry((how), (type)); \
  }
  
struct mbuf *
m_retry(i, t)
int i, t
{
  struct mbuf *m;
  m_reclaim();
#define m_retry(i, t) (struct mbuf *)0
  MGET(m, i , t)
#undef m_retry
  return (m);
}

```

很有趣， 通过这种技术，可以实现重试

==

C语言可以用一种很简洁的方式实现抽象

```
struct A {
  struct B b;
  type1 other;
  char end[12];
}

struct B {
  int (func*)(int);
}
```

此外，这里的 end 数组可以实现变长，只需要 alloc 的时候取大一点的地址就行，反正C语言没有边界检查

==

3.11 中 "宏LLADDR将一个指向sockaddr_dl结构的指针转换成一个指向这个文本名称的第一个字节的指针" 这句翻译错误，应该是 "宏LLADDR将一个指向sockaddr_dl结构的指针转换成一个指向越过文本名称的第一个字节的指针"，即LLADDR指向的是后面的硬件地址的开头。

翻了下英文版，原来是把 beyond 翻译错了。

看后面的内容，last 也是一个比较容易翻不好的词，很多时候 last 指的是 "最近的" 而非 "最后的"

==

图3-35里有大量技巧

```
#define _offsetof(t, m) ((int)((caddr_t)&(t *)0)->m)
  masklen = _offsetof(struct sockaddr_dl, sdl_data[0]) +
            unitlen + namelen;
  socksize = masklen + ifp->if_addrlen;
#define ROUNDUP(a) (1 + (((a) - 1) | (sizeof(long) - 1)))
```

`((t *)0)->m`是C语言的惯用技巧了，在V8里也经常看到，取t结构里的m的偏移，其实就是从开头到这里的长度了，然后类型转换成char指针后，直接把地址转换成int，因为是从0地址算起的，所以指针的值就是偏移的长度了。

下面加上后面几个后，就是整个sockaddr_dl结构的实际大小。

ROUNDUP这个就比较迷了，`(sizeof(long) - 1)`等于取了`sizeof(long)`长度的掩码，或了之后，等于`(a)-1`的低这么多位都是1，最后加上1那么这些位就都是0了，而且高出的那一位肯定会进1，这样ROUNDUP的结果必然比原来大，但是不会大过`(a) + sizeof(long)`，而且是`sizeof(long)`的倍数，其实就是向上取模，这种技巧在nginx，V8里也多有用到。

把低位抹平的好处一般在于，字节对齐，缓存友好。额外好处是指向这些结构的指针的低位可以放一些标志位，节省空间。而且对于一些特定的结构，指针是某个数的倍数，可以实现快速访问，nginx就经常这么用。

==

结构的继承关系

```
struct sockaddr {
    u_char sa_len;
    u_char sa_family;
    char sa_data[14];
}

struct sockaddr_dl {
    u_char sa_len;
    u_char sa_family;
    u_short sdl_index;
    
    u_char sdl_type;
    u_char sdl_nlen;
    
    u_char sdl_alen;
    u_char sdl_slen;
    u_char sdl_data[12];
}
```

如上定义的`sockaddr`，`sockaddr_dl`都是可以被继承的，后者正是继承前者。

在这里，通过`sa_len`来存储结构体的长度，然后在结构体的末尾定义一个char数组，来表明，这个结构体的长度可以被拓展。

在C中，指针的类型的转换非常容易，我们定义一个结构体，包含指向`sockaddr`的成员，然后指向sockaddr_dl结构，没有任何问题。

```
struct ifaddr {
    ...
    struct sockaddr *ifa_addr;
    struct sockaddr *ifa_netmask;
    ...
} ifa;

struct sockaddr_dl *sdl = ...;
ifa->ifa_addr = (struct sockaddr *)sdl;
``` 
==

读代码的脉络：

1. 搞清楚数据结构
2. 调查函数的参数，返回值的含义，进而弄清楚函数的功能，再去细抠
3. 弄清楚代码要解决的问题的背景
4. 学习代码中的一些优化点的技巧

== 

8.4.2，IP允许链路层填充分组，因此接收到的IP分组的大小，可能比IP首部宣称的大小还要大，因此需要做检查，丢弃多余的字节。

IP包处理非常细致的做了很多检查，以保证协议的各个部分符合预期。

==









