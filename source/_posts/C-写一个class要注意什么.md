---
title: C++/写一个class要注意什么
date: 2022-03-19 09:58:14
tags:
---

* const/mutable关键字

C++是一门有缺陷的语言，越用越觉得别扭。

## 前置声明

整体来看，前置声明的根本问题，是c++编译机制有缺陷。没有包机制，包内的符号应该是互相认识的，不用特殊声明。

当然，两个class始终无法直接引用彼此作为成员，这会导致无限递归。

```c++
// a.h和b.h定义了相互引用的两个类A和B
// 但是没法互相include，编译器报错： error: #include nested too deeply
// 那么我们需要引入前置声明，高速编译器，a.h里引用了一个class B，但是不知道符号在哪里定义，需要link时决定。这是ok的。
// 由于a.h对class B一无所知，不知道他的成员变量、成员函数，所以无法引用任何变量/函数。
// a.h只能声明指针，需要在.cc文件里去引用他的成员/函数
class A;
class B{
    // 虽然我们不知道A到底有多少成员变量，占多少内存，但是可以引入它的指针。因为指针大小是固定的
    std::shared_ptr<A> a1; 
    A* a2;
    // ERROR! A的大小不确定，编译器没法编排B的内存大小
    A a3;
};

```

## 虚析构函数

C++用继承实现多态，用构造/析构函数实现资源获取和释放。所以你想让一个基类的指针，能够调用到子类的析构函数，就得用继承来解决。

声明虚析构函数，是为了资源安全释放。声明纯虚析构函数，是强制子类来实现它。

```c++
class A{
public:
    // 语法规定，显示声明virtual，让这个析构函数进入虚函数表。
    // = 0 代表它没有实现，任何子类都必须实现它，才能编译通过。这个写法特别适合做接口class
    virtual ~A() = 0;
};

class B: public A{
public:
    ~B() override = default; // 自然的成为一个虚函数。显式override声明，是告诉人类，这玩意儿是个虚函数；也告诉编译器，让编译器工作的更快
};

A* a=new B();
delete a; // b的析构函数会被调用
```

## static和inline

### 头文件里可以声明static function嘛

在头文件里声明一个static function，会有奇妙的作用。每个编译单元都可以引用它，但是，各个编译单元都会拷贝一份这个函数。

如果恰好这个static function里，使用了static variable，就神奇了。

```c++
// hello.h
static void Hello(){
    static int count=0;
    std::cout<<"hello, with count "<<++count<<std::endl;
}

// a.h
#include "hello.h"
void HelloA(){
    Hello();
    Hello();
    Hello();
}

// b.h
#include "hello.h"
void HelloB(){
    Hello();
    Hello();
    Hello();
}

// main.cc
HelloA(); // hello, with count 1
HelloA(); // hello, with count 2
HelloA(); // hello, with count 3
// b.h有自己的一份Hello(),里面的count也是独有的。所以并不是输出4！
HelloB(); // hello, with count 1
HelloB(); // hello, with count 2
HelloB(); // hello, with count 3
```

这个特性实在是易燃易爆炸。如果你在static function里面搞了一点计数器，或者资源释放的东西，很容易引起误解。所以说，头文件里的static function，不要乱用。不对，是最好就不要用。

### 有必要声明inline吗

没有。我们认为，现代编译器足够智能，可以自动展开一些小函数。当你认为有inline的必要时，编译器也会识别出来。编译器比我们更聪明。

## explicit

explicit用来禁止编译通过隐式转换，调用构造函数。直白点，就是禁止编译器把某个类型的对象，直接调用构造函数转成另一个对象。

因为有时候，我们要调用对象的某个“初始化方法”，才能激活它的全部功能，进而调用某些特殊的函数。如果没有explicit，编译器很容易帮忙我们偷偷调用构造函数转换对象，跳过了“初始化方法”。举个例子：

```c++
// 有人写了个蹩脚的buffer，必须调用Allocate来激活内存
class Buffer {
 public:
  Buffer(int n) { memory_size = n; }
  void Alloate() { memory = new char[memory_size]; }
  void Access() const { std::cout << int(memory[0]) << std::endl; }

 private:
  int memory_size = 0;
  char *memory = nullptr;
};

// 有个函数使用了buferr里的某个方法
void PrintBuffer(const Buffer &buffer) { buffer.Access(); }

// 编译器允许我们这样写，路径是3->调用Buffer(int n)构造函数->得到Buffer对象
// **但显然这是有大问题的，我们还没有调用Allocate方法去初始化内存！**
PrintBuffer(3);
```

Buffer这个类算是比较合理，只有在真正要使用的时候，才去分配内存。也就是我们要显式的去分配内存。但是你某个粗心的队友写错了，让编译器偷偷帮忙搞了隐式转换，从int直接调用构造函数转成Buffer，就很不好了。为了禁止编译器这种自作主张的行为，就得加上explicit关键字。

```c++
// 有人写了个蹩脚的buffer，必须调用Allocate来激活内存
class Buffer {
 public:
  explicit Buffer(int n) { memory_size = n; }
  void Alloate() { memory = new char[memory_size]; }
  void Access() const { std::cout << int(memory[0]) << std::endl; }

 private:
  int memory_size = 0;
  char *memory = nullptr;
};

// 有个函数使用了buferr里的某个方法
void PrintBuffer(const Buffer &buffer) { buffer.Access(); }

// 有了explicit，编译器就不允许这样写。不能隐式的去调用构造函数Buffer(int n)
PrintBuffer(3); // compile error!

Buffer buffer(3);
buffer.Allocate();
PrintBuffer(buffer); // ok !
```

可以认为，explicit就是为了防止我们粗心的(也可能是故意留坑)调用隐式转换。很多class在设计的时候，不是在构造函数里获取资源，而是有特殊的初始化方法。**构造并不等同于对象可用**，所以不能随意地隐式转换。其实这也很蹩脚。即使有了explicit，我们还是很容易搞出一个对象，在忘记调用初始化方法的时候，使用其他成员方法。**只要人类想要犯错，编译器是挡不住的**。
