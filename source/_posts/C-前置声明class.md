---
title: C++/前置声明class
date: 2022-02-14 11:42:07
tags: c++
---

https://www.ibm.com/docs/en/zos/2.3.0?topic=only-incomplete-class-declarations-c

编译器在编译头文件时，要计算类大小、函数签名。如果class A有个成员变量class B，那么编译A时要首先编译B。如果刚好class B也引用class
A作为成员变量，那么无解了：**A、B无限循环引用，A、B的类大小无法确定**。此时推荐用指针引用。因为指针大小是确定的：4个字节或8个。

```c++
// a.h
class B;
class A{
    B* b;
    B* foo();
};

// b.h
class A;
class B{
    A* a;
    A* foo();
};
```

所以c++这种文件就很操蛋，分离了声明和定义（.h和.cxx）,导致我们不得不去理解编译机制。对比golang，就不存在这种考虑了。