---
title: OI中的IO优化
date: 2018-02-20 21:09	
categories: 
- OI
---
本文主要讲述常用的2种读入优化方法。  
输出优化很少使用，在此简单提一下：也就是把输出的东西先放进字符串，再一次性puts\printf出去。提升不大，不常用。  
首先当然需要先知道，scanf/printf比cin/cout快不少。  
读入优化： 
1. getchar  
使用getchar一个一个读入字符，转化成数字。比scanf快一些。
```cpp
    inline int read()
    {
        int f=1,x=0;//f是正负的标识
        char ch;
        do {
            ch=getchar();
            if(ch=='-')
                f=-1;
        } while(ch<'0'||ch>'9');
        do {
            x=x*10+ch-'0';
            ch=getchar();
        } while(ch>='0'&&ch<='9');
        return f*x;
    }
```
2.fread  （非常快！）
fread将stdin里的内容读到字符串里，然后利用指针处理。  
首先定义指针和读入的数组：  
```cpp
#define MAXB 10000000
//定义读入最长的长度
    char buf[MAXB],*cp=buf;
```
接下来是读入：  
```cpp
    fread(buf,1,MAXB,stdin);//函数具体参数含义请善用搜索引擎
```
最后是从中处理出数据（现在这个函数是为了处理int整型而设计）  
```cpp
    inline int read()
    {
        int f=1,x=0;
        while(*cp<'0'||*cp>'9') {
            if(*cp=='-')
                f=-1;
            cp++;
        }
        while(*cp>='0'&&*cp<='9') {
            x=x*10+*cp-'0'; 
            cp++;
        }
        return f*x;
    }
```