---
title: "CSAPPlab1 datalab通关记录"
date: 2026-05-10
categories: 公开课
tags: CSAPP
top_img: false
---

本文记录了完成datalab13个问题的思考及修改~~（踩坑）~~全过程。

还会介绍补码的本质，和ieee754浮点数的本质（以32位为例）。

## bitXor

要求：使用~(非)和&(与)实现异或

异或的本质是相同为0，不同为1，针对结果为1的情况拆开来看：a为0、b为1的时候结果为1；a为1、b为0的时候结果为1，两种情况|(或)即为结果。题干中没有|，但是|可以通过&来实现，即德摩根律：a | b = ~(~a & ~b)。代码如下：

```c
/* 
 * bitXor - x^y using only ~ and & 
 *   Example: bitXor(4, 5) = 1
 *   Legal ops: ~ &
 *   Max ops: 14
 *   Rating: 1
 */
int bitXor(int x, int y) {
	int a0b1 = ~x & y;
	int a1b0 = x & ~y;
	return ~(~a0b1 & ~a1b0); // a | b = ~(~a & ~b)
}
```

## 补码本质

首先我们知道，正数的补码表示等于无符号数表示。以四位二进制数为例（下文未提及则默认为4位二进制数），0b0000代表0，0b0111代表7。这时新的问题出现了，能否实现一种编码，使得减法也可以通过二进制加法直接运算呢？补码就是为了解决这个问题。

首先考虑无符号数，它至多表示2 ^ 4 = 16个状态，能表示的最小值为0b0000 = 0，最大值为0b1111 = 15。当最大值执行二进制加法，加1时就会变成0b10000，而只有四位表示空间，所以天然进行了**取模**操作，变成了0b0000。这和钟表很像，11点加1点为0点，也是12点。四位二进制画成钟表，见下图：

![补码钟表](/img/csapp/1.jpg)

其中黑色为二进制表示，蓝色为对应的无符号数十进制值，红色为对应的补码十进制值。

然后考虑补码，补码的根本目的是为了让减法执行加法的操作。假设当前表针指向3，想计算3 - 5，在表盘中减法意味着逆时针往回走，走5步到刻度14的位置。如何使用加法来表示减法呢？我们发现在这个只有16个刻度的表盘上，逆时针退5步等于顺时针往前走11步，因为16 - 5 = 11。从刻度3往前走11步，3 + 11 = 14，奇迹般地同样站在了刻度14上。这就是补码最核心的秘密了！**在模2^n的系统里，减去x等于加上x在这个系统里的互补数(2^n - x)**。当我们要算减法A - B时，我们把它映射成加法：A + 2^n - B。

至此我们知道了负数是为什么是0b1xxxx这样的。下面推导一下为什么取负等于按位取反再加1，见下图：

![按位取反末位加1](/img/csapp/2.jpg)

还有一个小小的特殊点需要介绍，即0b1000为什么等于-8。当前共有2 ^ 4 = 16个数可以表示，0b0000 ~ 0b0111占据了8个数，即0~7。我们规定负数也占据8个数，即16的一半（这就是为什么补码能表示的负数永远比正数多一个）。而-1 ~ -7又占据了7个数，还剩一个数可以表示内容，二进制为0b1000，它是多少呢？回到我们的 -x = 2 ^ n - x，-8 = 16 - 8，所以-8占据了这个位置。

最后再推导一下CSAPP课本中介绍的，为什么补码负数最高位的权重为负，见下图：

![最高位权为负](/img/csapp/3.jpg)

至此我们可以总结补码特性了：

- 0b1111 = -1
- 0b1000 = -8（最小）
- tmax + 1 = tmin（0b0111 + 1 = 0b1000）
- 表示范围 [- 2 ^ (n - 1), 2 ^ (n - 1) - 1]

## tmin

要求：返回二进制补码的最小数。

经过上文“补码本质”的分析，这题答案很显然了。

```c
/* 
 * tmin - return minimum two's complement integer 
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 4
 *   Rating: 1
 */
int tmin(void) {
  	return 1 << 31; //tmin = 0x80000000 = 1 followed by 31 zeros
}
```

## isTmax

要求：判断是否为tmax

思路：tmax的特点是tmax + 1 = tmin，~tmax = tmin

就有了我的第一个想法：

```c
return !(~((1 << 31) ^ x));
/* 返回结果是1
* 1 = !0
* ~((1 << 31) ^ x) = 0
* (1 << 31) ^ x = 0xffffffff
* x = 0x7fffffff
* 逻辑上很完美
* 但是题干限制，不让用移位，所以不可以呦
*/
```

猪脑过载的我想出了第二版，忘了符号限制还在用移位，根本逻辑其实和第一版相同，运行结果却不对让我很晕，要知道第一版运行结果可是对的！发现是触发了神奇的c语言UB(未定义行为)，代码如下：

```c
int y = x + 1;
return !(~((1 << 31) ^ ~y));
/*
 * tmax + 1 = tmin是c语言中的ub
 * 编译器假设程序员不会出现正 + 正 = 负
 * 所以在if !这种逻辑推断时，编译器推断return 1时y必须是0x80000000
 * 而y = x + 1，正数 + 正数不该有负数
 * 所以编译器认为不可能返回1，自动把语句优化为return 0;
 */
```

第三版的思路优化了一下，由于没有移位符号构造tmin，就只有通过x(tmax)来构造了。tmax ^ tmin = 0xffffffff，所以有了!(~((x + 1) ^ x))，不过还要排除x = -1。解题代码见下：

```c
/*
 * isTmax - returns 1 if x is the maximum, two's complement number,
 *     and 0 otherwise 
 *   Legal ops: ! ~ & ^ | +
 *   Max ops: 10
 *   Rating: 1
 */
 // 不要用越界/溢出和非法移位产生的魔数进行判断。
 // UB: 1. 越界/溢出 2. 非法移位
 // y = x + 1;
 // return !(~((1 << 31) ^ ~y));
 // 这里的y是越界产生的结果，编译器假设程序员从不溢出，所以直接给它优化了一个常数
 // 下面的解法照样使用了越界/溢出产生的魔数进行判断
 // 但是由于-1的存在让编译器无法优化
 // 所以编译器无法将其优化成一个常数
int isTmax(int x) {
	return !(~((x + 1) ^ x)) & !!(x + 1);
}
```

## allOddBits

要求：奇数位（31 30 …… 1 0，即最低位从0开始）的值都是1，

思路：构造一个掩码把偶数位全都置1，结果若为真则此时值为0xffffffff，取反后为0，!0 = 1

**置1用或1，置0用与0**

```c
/* 
 * allOddBits - return 1 if all odd-numbered bits in word set to 1
 *   where bits are numbered from 0 (least significant) to 31 (most significant)
 *   Examples allOddBits(0xFFFFFFFD) = 0, allOddBits(0xAAAAAAAA) = 1
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 12
 *   Rating: 2
 */
int allOddBits(int x) {
  	int mask = 0x55;
	mask = (mask << 8) + mask; // 0x5555
	mask = (mask << 16) + mask; // 0x55555555
	return !(~(x | mask));
}
```

## negate

要求：取反

思路：按位取反再加一，上文已证明原因

```c
/* 
 * negate - return -x 
 *   Example: negate(1) = -1.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 5
 *   Rating: 2
 */
int negate(int x) {
  	return ~x + 1;
}
```

## isAsciiDigit

要求：0x30 <= x <= 0x39

思路：

- 先减去0x30，判断结果是否在0~9之间。
- 高28位全为0
- 低4位（3 2 1 0，如此描述）第3位可以为0可以为1
- 第3位为0必然在0~7之间，符合要求
- 第3位为1的话，第2位和第1位必须都为0

```c
//3
/* 
 * isAsciiDigit - return 1 if 0x30 <= x <= 0x39 (ASCII codes for characters '0' to '9')
 *   Example: isAsciiDigit(0x35) = 1.
 *            isAsciiDigit(0x3a) = 0.
 *            isAsciiDigit(0x05) = 0.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 15
 *   Rating: 3
 */
int isAsciiDigit(int x) {
  	int y = x + (~0x30 + 1); // y = x - 0x30
	// return !(y >> 4) & (!(y >> 3) | (!(y >> 2) & !(y >> 1)));
    return !(y >> 4) & !(y >> 3 & (y >> 2 | y >> 1));
    //下面这个是上面那个把！提出来的结果
}
```

## conditional

要求：x ? y : z，x为真则返回y，为假则返回z

思路：没有if用，就是说条件判断必须在return里通过逻辑真假来判断，也就是说return语句里一定同时有y和z。考虑构造一个掩码，x真时为全1，x假时为全0。

```c
/* 
 * conditional - same as x ? y : z 
 *   Example: conditional(2,4,5) = 4
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 16
 *   Rating: 3
 */
int conditional(int x, int y, int z) {
  	int mask = ~!!x + 1; 
    /* x为0，!!x = 0, ~0 + 1 = 0(全0)
     * x不为0，!!x = 1, ~1 + 1 = -1(全1)
     */
    return (~mask & z) | (mask & y);
}
```

## isLessOrEqual

要求：判断x <= y

思路：容易想到y - x，判断符号位即可，但是存在溢出问题。比如y正x负，y > x，y - x符号位应该为0，但是溢出时符号位为1；再比如y负x正，y < x，y - x符号位应该为1，但是溢出时符号位为0。观察发现溢出的情况为两数异号，而异号时正数必然大于负数。

```c
/* 
 * isLessOrEqual - if x <= y  then return 1, else return 0 
 *   Example: isLessOrEqual(4,5) = 1.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 24
 *   Rating: 3
 */
int isLessOrEqual(int x, int y) {
  	int xSignal = x >> 31;
	int ySignal = y >> 31;
	int signDiff = xSignal ^ ySignal;
	int ySubxSignal = (y + ~x + 1) >> 31;
	return (signDiff & !(xSignal + 1)) | (!signDiff & !ySubxSignal);
}
```

## logicalNeg

要求：实现!运算符

思路：根源上是在找0，观察0有什么特殊性质。0的相反数还是0，不过tmin的相反数也是tmin，其他数取相反数必然变化，尤其是符号位。

**观察目标特点很重要**

```c
//4
/* 
 * logicalNeg - implement the ! operator, using all of 
 *              the legal operators except !
 *   Examples: logicalNeg(3) = 0, logicalNeg(0) = 1
 *   Legal ops: ~ & ^ | + << >>
 *   Max ops: 12
 *   Rating: 4 
 */
int logicalNeg(int x) {
	int xSignal = x >> 31;
	int minusxSignal = (~x + 1) >> 31;
	int signal = xSignal | minusxSignal;
	return signal + 1;
}
```

## howManyBits

~~最帅的一集来了,二分查找且不用循环究竟有多帅~~

要求：计算补码表示一个数最少需要多少位

思路：

* 正数划掉多余的0，例如：0b0001 = 0b01 = 1
* 负数划掉多余的1，例如：0b1101 = 0b101 = -3
* 正数找第一个1，从它开始的个数+1就是结果
* 负数找第一个0，从它开始的个数+1就是结果
* 把负数取反，正负数大一统，全都找第一个1，最后结果+1就是结果
* 如何设计一个，查找从左到右第一个1的算法呢？没有循环哦
* 二分查找，每次如果高一半不为0，则第一个1在高一半里，否则在低一半

```c
/* howManyBits - return the minimum number of bits required to represent x in
 *             two's complement
 *  Examples: howManyBits(12) = 5
 *            howManyBits(298) = 10
 *            howManyBits(-5) = 4
 *            howManyBits(0)  = 1
 *            howManyBits(-1) = 1
 *            howManyBits(0x80000000) = 32
 *  Legal ops: ! ~ & ^ | + << >>
 *  Max ops: 90
 *  Rating: 4
 */
int howManyBits(int x) {
  	int nums = 0;
	int xSignal = x >> 31; // + 0000 - 1111
	int target = xSignal ^ x; // 正负数统一

	int b16;
	int b8;
	int b4;
	int b2;
	int b1;
	int b0;

	// target >> 16之后，若为0则b16为0，非0则b16 = 16
	// 原本类似于使用condition函数，优化之后优美得多
	b16 = !!(target >> 16) << 4;
	nums += b16;
	target = target >> b16;

	b8 = !!(target >> 8) << 3;
	nums += b8;
	target = target >> b8;

	b4 = !!(target >> 4) << 2;
	nums += b4;
	target = target >> b4;

	b2 = !!(target >> 2) << 1;
	nums += b2;
	target = target >> b2;

	b1 = !!(target >> 1);
	nums += b1;
	target = target >> b1;

	b0 = target;
	nums += b0;
	
	return nums + 1;
}
```

## ieee754浮点数

说实话不知道该单独写点啥了，非规格化浮点数之间距离相等，和规格化浮点数之间过渡平滑？这些CSAPP课本上讲得很清晰了。阶码全1时Nax和∞的设计也讲了。

离0越远浮点数之间的距离越大，本质是用范围换精度这个如果有朋友好奇的话我可以写一写。~~是的开个小坑~~

## floatScale2

要求：计算f * 2的位级结果（f是浮点数）

思路：按部就班

- 非规格化数小数部分直接左移一位
- 阶码为255直接返回uf（∞）
- 阶码为254则阶码加1尾数部分全为0（∞）
- 其他情况阶码加一，拼上尾数

```c
//float
/* 
 * floatScale2 - Return bit-level equivalent of expression 2*f for
 *   floating point argument f.
 *   Both the argument and result are passed as unsigned int's, but
 *   they are to be interpreted as the bit-level representation of
 *   single-precision floating point values.
 *   When argument is NaN, return argument
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
unsigned floatScale2(unsigned uf) {
	unsigned sign = uf >> 31;
	unsigned exponent = (uf & 0x7f800000) >> 23;
	unsigned frac = uf & 0x007fffff;
	unsigned answer = 0;

	if (exponent == 0) {
		answer |= frac << 1;
	} else if (exponent == 255) {
		return uf;
	} else {
		if (exponent == 254) {
			exponent += 1;
			answer |= exponent << 23;
		} else{
			exponent += 1;
			answer |= (exponent << 23) | frac;
		}
	}
	answer |= sign << 31;

	return answer; 
}
```

## floatFloat2Int

要求：float转int

思路：float这边思路都没啥特别特殊的，观察特点写就行，直接看代码吧，我结构还是蛮清晰的

```c
/* 
 * floatFloat2Int - Return bit-level equivalent of expression (int) f
 *   for floating point argument f.
 *   Argument is passed as unsigned int, but
 *   it is to be interpreted as the bit-level representation of a
 *   single-precision floating point value.
 *   Anything out of range (including NaN and infinity) should return
 *   0x80000000u.
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
int floatFloat2Int(unsigned uf) {
	unsigned sign = uf >> 31;
	unsigned exponent = (uf & 0x7f800000) >> 23;
	int exp = exponent - 127;
	unsigned frac = uf & 0x007fffff;
	unsigned answer;

	if (exp < 0) {
		return 0;
	} else if (exp > 30) {
		return 0x80000000u;
	} else {
		answer = frac | 0x00800000;

		if (exp >= 23) {
			answer <<= exp - 23;
		} else {
			answer >>= 23 - exp;
		}

		if (sign) {
			answer = ~answer + 1;
		}

		return answer;
	}
}
```

## floatPower2

要求：2.0^x

思路：同上

```c
/* 
 * floatPower2 - Return bit-level equivalent of the expression 2.0^x
 *   (2.0 raised to the power x) for any 32-bit integer x.
 *
 *   The unsigned value that is returned should have the identical bit
 *   representation as the single-precision floating-point number 2.0^x.
 *   If the result is too small to be represented as a denorm, return
 *   0. If too large, return +INF.
 * 
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. Also if, while 
 *   Max ops: 30 
 *   Rating: 4
 */
unsigned floatPower2(int x) {
	unsigned answer;
	if (x < -149) {
		return 0;
	} else if (x > 127) {
		return 0x7f800000u;
	} else {
		if (x < -126) { // 非规格化数 x ∈ [-149, -127]
			answer = 0x00800000;
			answer >>= -(x + 126);
		} else { // 规格化数
			answer = (x + 127) << 23;
		}
	}

	return answer;
}
```



--------------------------------------------------------



## 后记

- 做任何题目的破题点就两方面：1.观察题设，找出研究对象的特质，根据特质做研究。2.思考结果的大概形式，从结果倒推过程。不仅做lab是这样，我自己写toy或者做数学题的时候也是这种感觉
- 下次我一定一边做一边写笔记 /(ㄒoㄒ)/~~
