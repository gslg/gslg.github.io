title: csapp中的一个数组拷贝例子
author: gslg
tags:
  - csapp
  - java
categories:
  - csapp
date: 2019-01-24 14:05:00
---
在csapp中有一个数组拷贝的例子,有两个相同长度的二维数组a,b，将a的值拷贝到b中对应位置；一种是按行拷贝,另一种是按列拷贝，比较二者性能，我用java做了简单的例子:

###### 按行拷贝
```java
public static  void  copyij(int[][] a,int[][] b){
        for (int i = 0; i < 2048; i++) {
            for (int j = 0; j < 2048; j++) {
                b[i][j] = a[i][j];
            }
        }
    }
```

###### 按列拷贝
```java
public static  void  copyji(int[][] a,int[][] b){
        for (int j = 0; j < 2048; j++) {
            for (int i = 0; i < 2048; i++) {
                b[i][j] = a[i][j];
            }
        }
    }
```
###### 简单测试
```java
public static void main(String[] args) {
        int[][] a = new int[2048][2048];
        int[][] b = new int[2048][2048];
        for (int i = 0; i < 2048; i++) {
            for (int j = 0; j < 2048; j++) {
                a[i][j] = i;
            }
        }

        long start = System.nanoTime();
        copyij(a,b);
        //copyji(a,b);
        long end = System.nanoTime();

        System.out.println(end-start);

    }
```

###### 结果
发现结果果然如csapp上讲课老师说的，二者的运行时间差了一个数量级，按行拷贝比按列拷贝快了一个数量级。造成这个原因是内存的存储结构影响的，具体怎么影响还需要后续深入学习.
