---
title: "位图"
tags: [algorithm, bitmap, "位图"]
---

# 定义

位图法就是 `bitmap` 的缩写。所谓 `bitmap` ，就是用每一位来存放某种状态，适用于大规模数据，但数据状态又不是很多的情况。通常是用来判断某个数据存不存在的。在STL中有一个bitset容器，其实就是位图法，引用bitset介绍：
> A bitset is a special container class that is designed to store bits (elements with only two possible values: 0 or 1,true or false, ...).The class is very similar to a regular array, but optimizing for space allocation: each element occupies only one bit (which is eight times less than the smallest elemental type in C++: char).Each element (each bit) can be accessed individually: for example, for a given bitset named mybitset, the expression mybitset[3] accesses its fourth bit, just like a regular array accesses its elements.

位图中每一个位置包含了两个信息： 1. **位置**： 代表了对应的数据; 2. **值**：代表状态。位图常用于状态不是很多的情况，常见的使用 bit 来表示状态，就是两个状态（是否，存在/不存在等），当有多个状态时，可以考虑使用多个bit 为来表示的方式，将后面的 `2bitmap` 。

# 数据结构
unsigned int bit[N];
在这个数组里面，可以存储 `N * sizeof(int) * 8` 个数据，但是最大的数只能是 `N*sizeof(int)  *8-1`。假如，我们要存储的数据范围为 `0-15` ，则我们只需要使得 `N=1`，这样就可以把数据存进去。如下图数据为 `[5，1，7，15，0，4，6，10]`，则存入这个结构中的情况为

```
 15 | 14 | 13 | 12 | 11 | 10 |  9 |  8 |  7 |  6 |  5 |  4 |  3 |  2 |  1 |  0
  1 |  0 |  0 |  0 |  0 |  1 |  0 |  0 |  1 |  1 |  1 |  1 |  0 |  0 |  1 |  1    

```

# 相关操作
## 写入数据
定义一个数组： unsigned char bit[8 * 1024];这样做，能存 `8K*8=64K` 个 `unsigned short` 数据。`bit` 存放的字节位置和位位置（字节 `0~8191` ，位 `0~7` ）比如写 `1234` ，字节序： `1234/8 = 154`; 位序： `1234 &0b111 = 2` ，那么 `1234` 放在 bit 的下标 `154` 字节处，把该字节的 `2` 号位（ `0~7`）置为 `1`

字节位置： `int nBytePos =1234/8 = 154;`

位位置：   `int nBitPos = 1234 & 7 = 2;`

```c++
// 把数组的 154 字节的 2 位置为 1
unsigned short val = 1<<nBitPos;
bit[nBytePos] = bit[nBytePos] |val;  // 写入 1234 得到arrBit[154]=0b00000100
```
再比如写入 `1236` ，

字节位置： int nBytePos =1236/8 = 154;

位位置： int nBitPos = 1236 & 7 = 4
```c++
// / 把数组的 154 字节的 4 位置为 1
val = 1<<nBitPos; arrBit[nBytePos] = arrBit[nBytePos] |val;  
// 再写入 1236 得到arrBit[154]=0b00010100
```
函数实现：

```c++
#define SHIFT 5 
#define MAXLINE 32 
#define MASK 0x1F 
void setbit(int *bitmap, int i){     
    bitmap[i >> SHIFT] |= (1 << (i & MASK));
}
```

## 读指定位
```
bool getbit(int *bitmap1, int i){
    return bitmap1[i >> SHIFT] & (1 << (i & MASK));
}
```

# 位图法的缺点
- 可读性差
- 位图存储的元素个数虽然比一般做法多，但是存储的元素大小受限于存储空间的大小。位图存储性质：存储的元素个数等于元素的最大值。比如， 1K 字节内存，能存储 8K 个值大小上限为 8K 的元素。（元素值上限为 8K ，这个局限性很大！）比如，要存储值为 65535 的数，就必须要 65535/8=8K 字节的内存。要就导致了位图法根本不适合存 unsigned int 类型的数（大约需要 2^32/8=5 亿字节的内存）。
- 位图对有符号类型数据的存储，需要 2 位来表示一个有符号元素。这会让位图能存储的元素个数，元素值大小上限减半。 比如 8K 字节内存空间存储 short 类型数据只能存 8K*4=32K 个，元素值大小范围为 -32K~32K 。

# 位图法的应用
1. 给40亿个不重复的 unsigned int 的整数，没排过序的，然后再给一个数，如何快速判断这个数是否在那 40 亿个数当中？
   思路：首先，将这 40 亿个数字存储到 bitmap 中，然后对于给出的数，判断是否在 bitmap 中即可。

2. 使用位图法判断整形数组是否存在重复？
   答：遍历数组，一个一个放入 bitmap，并且检查其是否在 bitmap 中出现过，如果没出现放入，否则即为重复的元素。

3. 如何使用位图法进行整形数组排序？
   答：首先遍历数组，得到数组的最大最小值，然后根据这个最大最小值来缩小 bitmap 的范围。这里需要注意对于 int 的负数，都要转化为 unsigned int 来处理，而且取位的时候，数字要减去最小值。

4. 在 2.5 亿个整数中找出不重复的整数？（注：内存不足以容纳这 2.5 亿个整数）
   参考的一个方法是：采用 2-Bitmap（每个数分配 2bit，00 表示不存在，01 表示出现一次，10 表示多次，11 无意义）。其实，这里可以使用两个普通的 Bitmap，即第一个 Bitmap 存储的是整数是否出现，如果再次出现，则在第二个 Bitmap 中设置即可。这样的话，就可以使用简单的 1-Bitmap 了。

# 2-BitMap

看个小场景：在3亿个整数中找出不重复的整数，限制内存不足以容纳3亿个整数。

对于这种场景我可以采用`2-BitMap`来解决，即为每个整数分配`2bit`，用不同的`0`、`1`组合来标识特殊意思，如 `00` 表示此整数没有出现过，`01` 表示出现一次，`11` 表示出现过多次，就可以找出重复的整数了，其需要的内存空间是正常BitMap的2倍，为：3亿*2/8/1024/1024=71.5MB。

具体的过程如下：

扫描着3亿个整数组成BitMap，先查看BitMap中的对应位置，如果 `00` 则变成 `01` ，是 `01` 则变成 `11` ，是 `11` 则保持不变，当将3亿个整数扫描完之后也就是说整个BitMap已经组装完毕。最后查看BitMap将对应位为 `11` 的整数输出即可。

这个方法同时还能判断哪些数据不存在，即位图中对应位置为 `00`的数字.

# 参考文献

- [原文 究竟什么是位图？](http://jartto.wang/2018/12/09/bitmap/)

- [数据结构：位图法](https://www.iteblog.com/archives/148.html)
