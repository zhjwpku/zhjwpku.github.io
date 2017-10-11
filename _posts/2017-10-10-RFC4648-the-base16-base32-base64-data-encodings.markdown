---
layout: post
title: "Base16, Base32, Base64 数据编码"
date: 2017-10-10 18:00:00 +0800
categories: rfc
tags:
- rfc
- base64
---

[RFC4648](https://tools.ietf.org/html/rfc4648) 介绍了 `base64`、`base32` 以及 `base16`编码方案。它还讨论了在编码数据中使用换行符、填充、非字母字符及不同的编码字母和标准编码的规范。本文摘取了其中广泛使用的 `base64` 进行讨论。

Base编码在许多情况下用于在环境中存储或传输数据，这些环境可能由于传统原因而被限制为US-ASCII数据。Base编码也可以用于没有遗留限制的新应用程序，因为它可以用文本编辑器来操作对象。

Base编码使用特定的简化的字母表对二进制数据进行编码。由于不同的应用有不同的要求，因此Base编码可能会有稍微不同的实现。[RFC4648](https://tools.ietf.org/html/rfc4648) 意在减少其他文档存在的歧义，进而实现更好的互操作性。

**Base 64 编码**

Base 64 编码被设计为允许使用大小写字母但不需要人类可读的形式表示任意八位字节序列。使用US-ASCII的65个字符的子集，用每个可打印字符表示6位。额外的第65个字符`=`用于编码填充。

<center>Table 1: The Base 64 Alphabet</center>
```
   Value Encoding  Value Encoding  Value Encoding  Value Encoding
       0 A            17 R            34 i            51 z
       1 B            18 S            35 j            52 0
       2 C            19 T            36 k            53 1
       3 D            20 U            37 l            54 2
       4 E            21 V            38 m            55 3
       5 F            22 W            39 n            56 4
       6 G            23 X            40 o            57 5
       7 H            24 Y            41 p            58 6
       8 I            25 Z            42 q            59 7
       9 J            26 a            43 r            60 8
      10 K            27 b            44 s            61 9
      11 L            28 c            45 t            62 +
      12 M            29 d            46 u            63 /
      13 N            30 e            47 v
      14 O            31 f            48 w         (pad) =
      15 P            32 g            49 x
      16 Q            33 h            50 y
```

对一个被编码的字符串，从左到右，每3个8位输入组形成一个24位输入组，然后将这24位视为4个级联的6位组，每组都被转换为base64字母表中的单个字符。6位组可以表示为0-63的数字，并作为字母表的索引对应到某个字母。

对于编码结尾少于24位的数据，对结尾进行填充处理。由于所有 base64 输入都是整数个八位字节，因此只会出现以下情况：
```
  1) 结尾为三个字节，此时不用填充
  2) 结尾为两个字节，此时需要填充8个0形成24位，编码结果为⌈16/6⌉=3个编码位，一个"="填充位
  3) 结尾为一个字节，此时需要填充16个0以形成24位，编码结果为⌈8/6⌉=2个编码位，两个"="填充位
```
当然，依据应用场景的不同，可以将填充位忽略（如数据长度已知的情况下）。

**Base64url 编码**

该编码应该与base64编码区别对待，除非特殊说明，`base64` 编码通常为 Table1 中的字母表。

base64url 与 base64 在技术上等同，只是字母表中最后两个编码有所区别。

<center>Table 2: The "URL and Filename safe" Base 64 Alphabet</center>
```
   Value Encoding  Value Encoding  Value Encoding  Value Encoding
       0 A            17 R            34 i            51 z
       1 B            18 S            35 j            52 0
       2 C            19 T            36 k            53 1
       3 D            20 U            37 l            54 2
       4 E            21 V            38 m            55 3
       5 F            22 W            39 n            56 4
       6 G            23 X            40 o            57 5
       7 H            24 Y            41 p            58 6
       8 I            25 Z            42 q            59 7
       9 J            26 a            43 r            60 8
      10 K            27 b            44 s            61 9
      11 L            28 c            45 t            62 - (minus)
      12 M            29 d            46 u            63 _
      13 N            30 e            47 v           (underline)
      14 O            31 f            48 w
      15 P            32 g            49 x
      16 Q            33 h            50 y         (pad) =
```

**C语言编码实现**

```
char base64set[]="ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
                 "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
                 "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
                 "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/=" ;

  // repeating the base64 characters in the index array avoids the  077 &  operations

char* Byte3toChar4(unsigned char ra[3]){
static char ar[4] = { base64set[       ra[0] >> 2                           ] ,
                      base64set[(byte)(ra[0] << 4)| ra[1] >> 4              ] ,
                      base64set[(byte)             (ra[1] << 2)| ra[2] >> 6 ] ,
                      base64set[                                 ra[2]      ] };  
return (char *)ar;}

  // replaces the traditional rudimentary approach, though it only uses base64set[0..63]

char* Byte3toChar4(unsigned char ra[3]){
static char ar[4] = { base64set[       ra[0] >> 2                           ] ,
                      base64set[ 077 & ra[0] << 4 | ra[1] >> 4              ] ,
                      base64set[ 077 &              ra[1] << 2 | ra[2] >> 6 ] ,
                      base64set[ 077 &                           ra[2]      ] };  
return (char *)ar;}
```

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Base64 From Wikipedia](https://en.wikipedia.org/wiki/Base64)
</span>
