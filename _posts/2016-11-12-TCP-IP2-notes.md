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

可移植的IP校验和实现，要非常注意里面奇数位，偶数位的交换的实现，很容易出错

```
// from https://github.com/chenshuo/4.4BSD-Lite2/blob/master/sys/netinet/in_cksum.c

#define ADDCARRY(x)  (x > 65535 ? x -= 65535 : x)
#define REDUCE {l_util.l = sum; sum = l_util.s[0] + l_util.s[1]; ADDCARRY(sum);} // 将sum的高16bit和低16bit相加，然后模2^16

int
in_cksum(m, len)
	register struct mbuf *m;
	register int len; // 表示需要处理的总字节数
{
	register u_short *w; // 代表用于计算的2个字节，16bit
	register int sum = 0; // 保存临时校验和	register int mlen = 0; // mbuf里的有效字节数
	int byte_swapped = 0; // 标志sum中的奇偶地址字节是否进行过交换

	union {
		char	c[2];
		u_short	s;
	} s_util;
	union {
		u_short s[2];
		long	l;
	} l_util;

	for (;m && len; m = m->m_next) {
		if (m->m_len == 0)
			continue;
		w = mtod(m, u_short *);
		if (mlen == -1) { // 表示上一个串还剩一个字节没处理
			/*
			 * The first byte of this mbuf is the continuation
			 * of a word spanning between this mbuf and the
			 * last mbuf.
			 *
			 * s_util.c[0] is already saved when scanning previous 
			 * mbuf.
			 */
			 
			s_util.c[1] = *(char *)w;
			sum += s_util.s;
			w = (u_short *)((char *)w + 1);
			mlen = m->m_len - 1;
			len--;
		} else
			mlen = m->m_len;
		if (len < mlen)
			mlen = len; // 接收到的IP分组的大小，可能比IP首部宣称的大小还要大，因此需要做检查，丢弃多余的字节
		len -= mlen;
		/*
		 * Force to even boundary.
		 */
		if ((1 & (int) w) && (mlen > 0)) {
          // 实现字节对齐，w为指针，要保证最低位为0；如果为1，则将当前的这个字节保存起来，和下个mbuf的第一个字节配对计算。
          
			REDUCE;
			sum <<= 8; // 交换sum的奇数地址字节和偶数地址字节，保证后面w中第一个字节为偶数地址字节，之后sum保存的是[偶数地址和，奇数地址和]
			s_util.c[0] = *(u_char *)w;
			w = (u_short *)((char *)w + 1);
			mlen--;
			byte_swapped = 1;
		}
			
		/*
		 * Unroll the loop to make overhead from
		 * branches &c small.
		 */
		while ((mlen -= 32) >= 0) {
			sum += w[0]; sum += w[1]; sum += w[2]; sum += w[3];
			sum += w[4]; sum += w[5]; sum += w[6]; sum += w[7];
			sum += w[8]; sum += w[9]; sum += w[10]; sum += w[11];
			sum += w[12]; sum += w[13]; sum += w[14]; sum += w[15];
			w += 16;
		}
		mlen += 32;
		while ((mlen -= 8) >= 0) {
			sum += w[0]; sum += w[1]; sum += w[2]; sum += w[3];
			w += 4;
		}
		mlen += 8;
		if (mlen == 0 && byte_swapped == 0)
			continue; // 刚好结束，且没有交换过奇偶地址字节，则直接进入下一个mbuf计算
		REDUCE;
		while ((mlen -= 2) >= 0) {
			sum += *w++;
		}
		if (byte_swapped) {   
			REDUCE;
			sum <<= 8; // 如果sum中的奇偶地址字节进行过交换，那么这里要交换回来，之后sum保存的是[奇数地址和，偶数地址和]
			byte_swapped = 0; 			
			if (mlen == -1) { // 表示最后还多出来一个字节，这个肯定是偶数地址字节；放在位置1，然后和之前放在位置0的奇数地址字节一起加起来
				s_util.c[1] = *(char *)w;
				sum += s_util.s;
				mlen = 0;
			} else
				mlen = -1; // 这里本来mlen=-2，表示刚好处理完了。因为之前保存了一个奇数地址字节，用-1告诉下个mbuf，这个mbuf还留了一位。这里sum保存的是[奇数地址和，偶数地址和]，刚好剩下的奇数地址字节在下个循环里会加到奇数地址和里
		} else if (mlen == -1)
	       // 注意这个分支里，说明mbuf一开始就是偶数地址，sum一开始保存的就是[偶数地址和，奇数地址和]，所以多出来的一个直接放在0位置。这个处理逻辑中，w第一个字节是偶数地址字节，在下个循环里会加到偶数地址和里，所以放在0位置
			s_util.c[0] = *(char *)w; // 最后多处一个字节
	}
	if (len)
		printf("cksum: out of data\n");
	if (mlen == -1) {
		/* The last mbuf has odd # of bytes. Follow the
		   standard (the odd byte may be shifted left by 8 bits
		   or not as determined by endian-ness of the machine) */
		s_util.c[1] = 0; // 容易漏的步骤
		sum += s_util.s;
	}
	REDUCE;
	return (~sum & 0xffff);
}

```

RFC 1071 mentions two optimizations that don’t appear in Net/3: a combined copy-with-checksum operation and incremental checksum updates. 

"a combined copy-with-checksum operation and incremental checksum updates." 被翻译成 "联合的有校验和的复制操作和递增的检验和更新" 特别别扭。这句话的意思是在复制的过程中进行校验和计算，与增量更新校验和。

==








