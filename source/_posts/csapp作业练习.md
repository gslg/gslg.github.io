title: csapp作业练习
author: gslg
date: 2019-02-22 10:05:37
tags:
---
#### 2.5 大小端问题:  

```c
//show_bytes.c
#include <stdio.h>

typedef unsigned char *byte_pointer;

void show_bytes(byte_pointer start,size_t len){
     size_t i;
     for(i=0;i<len;i++){
        printf(" %.2x",start[i]);
     }
     printf("\n");
}

void show_int(int x){
  show_bytes((byte_pointer)&x,sizeof(int));
}

void show_float(float x){
   show_bytes((byte_pointer)&x,sizeof(float));
}

void show_pointer(void *x){
   show_bytes((byte_pointer)&x,sizeof(void *));
}

int main(){
   int val = 12345;
   int ival = val;
   float fval = (float)ival;
   int *pval = &ival;
   show_int(ival);
   show_float(fval);
   show_pointer(pval);
}
```

```c
int val = 0x87654321;
byte_pointer valp = (byte_pointer)&val;
show_bytes(valp,1); /*A.*/
show_bytes(valp,2);/*B.*/
show_bytes(valp,3);/*C.*/
```
在小端和大端机器上分别调用的结果:  

| 值    |  小端 | 大端 |
| :-------- | :-------| :-- |
| 0x87654321  | 21 43 65 87 |  87 65 43 21 |
| A     | 21 |  87 |
| B     | 21 43 |  87 65 |
| C     | 21 43 65 |  87 65 43|

#### 2.6 正数与浮点数二进制位比较
|  值   |  十六进制 | 二进制 |
| :-------- | :-------| :-- |
| 3510593  | 0x00359141 |  00000000001<font color=red face=“黑体”>101011001000101000001</font> |
| 3510593.0 | 0x4a564504 |&nbsp;&nbsp;&nbsp; 010010100 <font color=red face=“黑体”>101011001000101000001</font>00 |
整数中除了第一位的1其它都包含在浮点数的二进制中

#### 2.7
```c 
const char *s="abcdef";
show_bytes((byte_pointer)s,strlen(s)); //输出结果
```
`注意字母'a'～‘z’的ASCII码为0x61～0x7A.在unix中运行man ascii可以查看ascii表.`  
输出为61 62 63 64 65 66 00其中00是字符串终止字节
#### 2.8 位向量布尔运算及相关定理
|  运算   |  结果 |
| :-------- | :-------|
| a  | [01101001] |
| b  | [01010101] |
| ~a  | [10010110] |
| ~b  | [10101010] |
|a&b |[01000001]|
|a&#124;b |[01111101]|
|a^b|[00111100]|
注: 类似于乘法分配律,  
- &对|分配律: `a&(b|c)=(a&b)|(a&c);  `  
- 反过来，|对&也具有分配律: `a|(b&c)=(a|b)&(a|c)`
- 对于任何a值,有:`a^a=0`;即使交换顺序也成立:`a^b^a=(a^a)^b=b`

#### 2.9 RGB三原色  
基于红(R),绿(G)，蓝(B)三种光源的打开(1)和关闭(0)可以创建以下8种不同颜色:  

|  R   |  G | B | 颜色||  R   |  G | B |颜色|
| :--: | :--:|:--:|:--:|| :--: | :--:|:--:|:--:|
| 0   |  0  | 0 | 黑色||  1   | 0  | 0  |红色|  
| 0   |  0  | 1 | 蓝色||  1   | 0  | 1  |红紫色|  
| 0   |  1  | 0 | 绿色||  1   | 1  | 0  |黄色|  
| 0   |  1  | 1 | 蓝绿色||  1  | 1  | 1  |白色|  

A.每种颜色的互补关系:
![image](三原色.png)
B.对颜色进行布尔运算: 
- 蓝色|绿色 = 001|010 = 011= 蓝绿色
- 黄色&蓝绿色 = 110&011 = 010 = 绿色
- 红色^红紫色 = 100^101 = 001 = 蓝色

#### 2.10 不使用临时变量的交换值
```c
void inplace_swap(int *x,int *y){
  *y = *x ^ *y; /* Step 1*/
  *x = *x ^ *y; /* Step 2*/
  *y = *x ^ *y; /* Step 3*/
}
```
根据定理a^a=0,得到每一步骤的结果:  

|  步骤   |  *x | *y|
| :--: | :--:|:--:|
| 初始  |a |b|
| 第1步  | a|a^b|
| 第2步  | a^(a^b)=b|a^b|
| 第3步  | b|b^(a^b)=a|

这样就利用^实现了不引入第三个临时变量实现了两个变量的交换，这种交换方式在性能上并没有什么优势，仅仅是一种智力游戏.

#### 2.11 利用inplace_swap函数实现数组两端依次对调
```c
void reverse_array(int a[],int cnt){
  int first,last;
  for(first = 0,last = cnt-1;
      first <= last;
      first++,last--){
        inplace_swap(&a[first],&a[last]);
  }
}
```
上面这个函数，对包含1,2,3,4元素的数组来说可以按预期那样变为4,3,2,1;但是，当一个包含1,2,3,4,5元素的数组使用该函数时却会看到5,4,0,2,1的结果.实际上，上面的函数对偶数项数组都能正确工作，但是对于奇数项的总会把中间项设置位0. 
- A.对于一个长度为奇数的数组，长度cnt=2k+1,函数reverse_array最后一次循环中，变量first和last的值分别是什么? `first=last=k`
- B.为什么这时候调用inplace_swap函数会将数组元素置为0? `因为&a[k]^&a[k]=0`
- C.对reverse_array的代码做哪些改动即可解决该问题? `因为中间元素不需要交换，故把first<=last条件改为first<last即可`.

#### 2.12 掩码运算
对于下面的值，写出C语言表达式，并且代码对于任何w≥8都能工作.我们给出0x87654321以及w=32时的表达式求值结果，以供参考:
- A.x的最低有效字节,其它位置均为0. [0x00000021] &nbsp;&nbsp;&nbsp;&nbsp;`x&0xFF`
- B.除了x的最低有效字节外,其它位置都取补.[0x789ABC21] &nbsp;&nbsp;&nbsp;&nbsp; `x^~0xFF`
- C.x的最低有效字节全部设置为1，其它字节保持不变.[0x876543FF] &nbsp;&nbsp;&nbsp;&nbsp; `x|0xFF`

#### 2.13 bis(位设置)和bic(位清除)函数
这两种指令输入都是一个数据字x和一个掩码m,生成一个结果z,z是根据掩码m的位来设置x的位得到的.  
- bis：在m为1的每个位置上，将x对应的位设置为1
- bic：在m为1的每个位置上，将x对应的位设置为0

现在，不使用任何其它C语言运算，实现|和^运算
```c
/*bis和bic函数声明*/
int bis(int x,int m);
int bic(int x,int m);

/*使用bis和bic函数实现x|y运算*/
int bool_or(int x,int y){
  int result = bis(x,y);
  return result;
}

/*使用bis和bic函数实现x^y运算*/
int bool_xor(int x,int y){
  int result = bis(bic(x,y),bic(y,x));
  return result;
}
```
`注:bis运算等价于布尔or；bic(x,m)等价于x&~m,即实现x对应位为1且m对应位为0时，该位等于1.  
由此，我们可以调用一次bis来实现|运算。对于实现^运算则需要使用以下性质:  
  x^y=(x&~y)|(~x&y) 可以推导:x^y=bic(x,y)|bic(y,x)=bis(bic(x,y),bic(y,x))`
#### 2.14 逻辑运算与位级运算比较
假设x和y的值分别是0x66和0x39，填写下表:  

|  值   |  二进制 |~|
| :-- | :--|:--|
| 0x66  | 0110 0110 |1001 1001|
| 0x39  | 0011 1001 |1100 0110|

|  表达式   |  值 ||  表达式   |  值 |
| :-- | :--|:--| :--|
| x & y  | 0010 0000=0x20 | | x && y|0x01|
| x &#124; y  | 0111 1111=0x7F||x &#124;&#124;y|0x01|
| ~x &#124; ~y  | 1101 1111=0xDF||~x &#124;&#124;~y|0x00|
| ~x & !y  | 0000 0000=0x00||x && ~y|0x01|

`注意!是逻辑运算符，因此!y=0x00`

#### 2.15 
只用位级和逻辑运算，编写一个C表达式，它等价于x==y.换句话说，当x和y相等时它将返回1,否则返回0. &nbsp;&nbsp;&nbsp;&nbsp;`!(x^y)`

#### 2.16 位移运算  
<table><tr><td colspan=2 style="border:1px solid;text-align:center">x</td><td colspan=2 style="border:1px solid;text-align:center">x<<3</td><td colspan=2 style="border:1px solid;text-align:center">x>>2(逻辑的)</td><td colspan=2 style="border:1px solid;text-align:center">x>>2(算术的)</td></tr><tr><td style="border:1px solid;text-align:center">十六进制</td><td style="border:1px solid;text-align:center">二进制</td><td style="border:1px solid;text-align:center">二进制</td><td style="border:1px solid;text-align:center">十六进制</td><td style="border:1px solid;text-align:center">二进制</td><td style="border:1px solid;text-align:center">十六进制</td><td style="border:1px solid;text-align:center">二进制</td><td style="border:1px solid;text-align:center">十六进制</td></tr><tr><td style="border-left:1px solid;border-right:1px solid;">0xC3</td><td style="border-right:1px solid;">1100 0011</td><td style="border-right:1px solid;">0001 1000</td><td style="border-right:1px solid;">0x18</td><td style="border-right:1px solid;">0011 0000</td><td style="border-right:1px solid;">0x30</td><td style="border-right:1px solid;">1111 0000</td><td style="border-right:1px solid;">0xF0</td></tr><tr><td style="border-left:1px solid;border-right:1px solid;">0x75</td><td style="border-right:1px solid;">0111 0101</td><td style="border-right:1px solid;">1010 1000</td><td style="border-right:1px solid;">0xA8</td><td style="border-right:1px solid;">0001 1101</td><td style="border-right:1px solid;">0x1D</td><td style="border-right:1px solid;">0001 1101</td><td style="border-right:1px solid;">0x1D</td></tr><tr><td style="border-left:1px solid;border-right:1px solid;">0x87</td><td style="border-right:1px solid;">1000 0111</td><td style="border-right:1px solid;">0011 1000</td><td style="border-right:1px solid;">0x38</td><td style="border-right:1px solid;">0010 0001</td><td style="border-right:1px solid;">0x21</td><td style="border-right:1px solid;">1110 0001</td><td style="border-right:1px solid;">0xE1</td></tr><tr><td style="border-bottom:1px solid;border-left:1px solid;border-right:1px solid;">0x66</td><td style="border-bottom:1px solid;border-right:1px solid;">0110 0110</td><td style="border-bottom:1px solid;border-right:1px solid;">0011 0000</td><td style="border-bottom:1px solid;border-right:1px solid;">0x30</td><td style="border-bottom:1px solid;border-right:1px solid;">0001 1001</td><td style="border-bottom:1px solid;border-right:1px solid;">0x19</td><td style="border-bottom:1px solid;border-right:1px solid;">00001 1001</td><td style="border-bottom:1px solid;border-right:1px solid;">0x19</td></tr></table>
#### 2.17 无符号数、有符号数转换
{% raw %}

<script type="text/x-mathjax-config">
  MathJax.Hub.Config({tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}});
</script>
<script type="text/javascript" async
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/latest.js?config=TeX-MML-AM_CHTML">
</script>
 <style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:black;}
.tg th{font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:black;text-align:center}
.tg .tg-nb2d{font-size:100%;border-color:inherit;text-align:center;vertical-align:middle}
.tg .tg-c3ow{border-color:inherit;text-align:center;vertical-align:middle}
.tg .tg-0pky{border-color:inherit;text-align:center;vertical-align:middle}
.tg .tg-0lax{text-align:center;vertical-align:middle}
</style>
<table class="tg">
  <tr>
    <th class="tg-0pky" colspan="2">$\vec{x}$</th>
    <th class="tg-nb2d" rowspan="2">$B2U_{4}(\vec{x})$</th>
    <th class="tg-c3ow" rowspan="2">$B2T_{4}(\vec{x})$</th>
  </tr>
  <tr>
    <td class="tg-0pky">十六进制</td>
    <td class="tg-0pky">二进制</td>
  </tr>
  <tr>
    <td class="tg-0pky">0xE</td>
    <td class="tg-0pky">[1110]</td>
    <td class="tg-0pky">$2^3+2^2+2^1=14$</td>
    <td class="tg-0pky">$-2^3+2^2+2^1=-2$</td>
  </tr>
  <tr>
    <td class="tg-0pky">0x0</td>
    <td class="tg-0pky">[0000]</td>
    <td class="tg-0pky">0</td>
    <td class="tg-0pky">0</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x5</td>
    <td class="tg-0lax">[0101]</td>
    <td class="tg-0lax">$2^2+2^1=5$</td>
    <td class="tg-0lax">5</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x8</td>
    <td class="tg-0lax">[1000]</td>
    <td class="tg-0lax">$2^3=8$</td>
    <td class="tg-0lax">$-2^3=-8$</td>
  </tr>
  <tr>
    <td class="tg-0lax">0xD</td>
    <td class="tg-0lax">[1101]</td>
    <td class="tg-0lax">$2^3+2^2+2^0=13$</td>
    <td class="tg-0lax">$-2^3+2^2+2^0=-3$</td>
  </tr>
  <tr>
    <td class="tg-0lax">0xF</td>
    <td class="tg-0lax">[1111]</td>
    <td class="tg-0lax">$2^3+2^2+2^1+2^0=15$</td>
    <td class="tg-0lax">$-2^3+2^2+2^1+2^0=-1$</td>
  </tr>
</table>
{% endraw %}
#### 2.19 有符号数转无符号数
{% raw %}
<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:black;}
.tg th{font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:black;}
.tg .tg-s6z2{text-align:center}
</style>
<table class="tg">
  <tr>
    <th class="tg-s6z2">x</th>
    <th class="tg-s6z2">$T2U_{4}(x)$</th>
  </tr>
  <tr>
    <td class="tg-s6z2">-8</td>
    <td class="tg-s6z2">8</td>
  </tr>
  <tr>
    <td class="tg-s6z2">-3</td>
    <td class="tg-s6z2">13</td>
  </tr>
  <tr>
    <td class="tg-s6z2">-2</td>
    <td class="tg-s6z2">14</td>
  </tr>
  <tr>
    <td class="tg-s6z2">-1</td>
    <td class="tg-s6z2">15</td>
  </tr>
  <tr>
    <td class="tg-s6z2">0</td>
    <td class="tg-s6z2">0</td>
  </tr>
  <tr>
    <td class="tg-s6z2">5</td>
    <td class="tg-s6z2">5</td>
  </tr>
</table>
原理: 补码转无符号数:  
对满足$TMin_{w} \leqslant x \leqslant TMax_{w}$的x有:
$$T2U_{w}(x)= \left\{\begin{matrix}
 & x + 2^{w},x< 0 & \\ 
 & x,x\geqslant 0 & 
\end{matrix}\right. $$
{% endraw%}
