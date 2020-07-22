---
title: 金额计算为什么不能用float或者double
tags: Java 
---



最近在做一个电商相关的项目，自然是要和钱打交道的，很多小伙伴都知道涉及到金额不应该用float或者double存储，因为浮点数结果不精确。话说回来，为什么float或者double不精确呢？

### 为什么不能用float存储金额

为了故事的顺利发展，上个栗子

```java
public void test() {
  	float a = 2.6f;
  	float b = 2.3f;
  	System.out.println(a+b);
}
// 4.8999996
```

输出结果为：4.8999996，竟然不是4.9！！！

因为计算机只认识0和1，所有的数字计算都会转化为二进制来计算，以下我们来看下2.6f+2.3f的过程

### 从二进制角度看2.6f+2.3f的过程

##### float底层存储

现代计算机的浮点数一般采用IEEE 754标准，由以下三部分组成： sign bit(符号位)、 exponent field(指数部分)、 significand or mantissa(有效数或尾数部分)，其中sign占1位，exponent占8位，fraction占23位

<img src="https://moonto.org/assets/images/java/float_example.png" alt="float_example" style="zoom:50%;" />



##### 转换为二进制

小数转二进制是分两部分转换的，整数和小数部分是分开的，2.6的整数部分2转换为二进制位0010，小数部分0.6转化为二进制为1001 1001 1001 ...（1001循环），所以2.6转换为二进制为10.100110011001...，同理2.3对应的二进制为10.010011001100...(1100循环)，关于怎样的十进制数转为二进制数会无限循环，[维基百科](https://en.wikipedia.org/wiki/Floating-point_arithmetic)有一段描述

> Whether or not a rational number has a terminating expansion depends on the base. For example, in base-10 the number 1/2 has a terminating expansion (0.5) while the number 1/3 does not (0.333...). In base-2 only rationals with denominators that are powers of 2 (such as 1/2 or 3/16) are terminating. Any rational with a denominator that has a prime factor other than 2 will have an infinite binary expansion. This means that numbers which appear to be short and exact when written in decimal format may need to be approximated when converted to binary floating-point. For example, the decimal number 0.1 is not representable in binary floating-point of any finite precision; the exact binary representation would have a "1100" sequence continuing endlessly:

##### 规格化

规格化类似科学计数法，保证整数位为1，2.6规格化后为1.010011001*2^1

##### 偏置指数

指数偏移值=固定值+规格化的指数值，其中固定值=2^(e-1)-1，e为存储指数部分的位数，float为8位，所以固定值为127，2.6f规格化后为1.010011001*2^1，指数位为1，所以指数偏移值为127+1=128，二进制表示为10000000

##### 2.6最终存储形式

2.6为整数，所以符号位为0，指数部分为10000000，有效部分为规格化后的小数部分(因为规格化后最高位一定为1，所以这里只记录尾数的小数部分)，取小数部分的前23位：01001100110011001100110，三部分拼接到一起为01000000001001100110011001100110，到这里就可以看出为什么float会不精确了，因为小数部分转换为二进制后是无限循环的，但是这里只能存储23位有效数字，存储时存在精度丢失

<img src="https://moonto.org/assets/images/java/float-2.6.png" alt="float_2.6f" style="zoom:50%;" />

##### 求和

在*计算机组成原理*这本书中有详细的说明浮点数之间的加减运算，浮点数的加减运算操作过程大概可以分为四步：

1）0操作数的检查

如果判断两个操作数有一个操作数0，即可得知运算结果有没有必要再进行后续的操作

2）比较指数位大小并完成对阶

两浮点数加减要先看两数的指数位是否相同，即小数点位置是否对齐，若指数位相同，则直接尾数即有效位数进行相加减就可以了，反之则需要先进行**对阶**

如何对阶：

假设有两个浮点数X和Y对应的指数分别为Ex和Ey，通过尾数的移动使Ex和Ey相等的操作为对阶，因为尾数左移会引起最高有效位的丢失，所以一般都采用右移的方式，这样造成的误差比较小，右移一阶相应的指减一，以小阶看向大阶的原则调整小阶数使其指数位与大阶数一致

3）有效位数进行加减运算

对阶完成后就可以对尾数即有效位数进行求和

4）结果规格化并进行四舍五入

以上述两数相加为例：

2.6f的存储格式为：0  10000000  01001100110011001100110

2.3f的存储格式为：0  10000000  00100110011001100110011

因为两数的指数位相同，所以直接对尾数进行计算即可，尾数相加结果为：0 10000000 01110011001100110011001，尾数加上最高位和为10.01110011001100110011001，指数位为1，最后规格化结果为1.00111001100110011001100*2^2，转换为十进制4.899999618530273，出现了！！！

### 使用BigDecimal进行浮点型运算

再来看一段代码

```java
public static void test() {
    BigDecimal price1 = new BigDecimal("2.6");
    BigDecimal price2 = new BigDecimal("2.3");
    System.out.println(price1.add(price2));
}
// 4.9
```

输出结果为4.9，这里要用String的构造器，不然精度也是不保证的，使用BigDecimal也有缺点，它占用存储空间比较多



#### 

参考资料：

[Floating Point Tutorial](https://www.rfwireless-world.com/Tutorials/floating-point-tutorial.html)

