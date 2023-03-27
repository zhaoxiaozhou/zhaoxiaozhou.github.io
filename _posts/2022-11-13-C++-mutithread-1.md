---
layout: post
title: 【c++多线程编程】多线程编程专栏一
excerpt: "C++多线程编程基础、thread类的基础知识"
date: 2022-03-06 20:57:40
tags: ['C++', '多线程']
category: learning
---
# 第一章

## 1. 多编程编程基础

### 1.1 并发

多核CPU的并发才是真正的并发，单核CPU的并发，需要切换上下文，需要很多的时间

### 1.2 并发的方式

并发分为多进程和多多线程的方式

#### 多进程

进程可以通过常规的[进程间通信机制](https://link.segmentfault.com/?enc=2oF1MCCVfmtLM39hky8QYw%3D%3D.pUMpcnHH%2FjWGmHkplGWRMGmk%2F6nSip6GLhxHBYgt9c7bmcZpo3EUcY5jRDaQZy0pmwWlBGCZWEbLvAS6J9Vw8g%3D%3D)进行通信，如管道、信号、消息队列、共享内存、存储映射I/O、信号量、套接字等。

- 优点
  - 启动进程的开销比线程大，使用的系统资源也更多
  - 进程间通信较为复杂，速度相对线程间的通信更慢。
- 缺点
  - 进程间通信的机制相对于线程更加安全
  - 能够很容易的将一台机器上的多进程程序部署在不同的机器上（如果通信机制选取的是套接字的话）

#### 多线程

线程很像轻量级的进程，但是一个进程中的所有线程都共享相同的地址空间，线程间的大部分数据都可以共享。线程间的通信一般都通过共享内存来实现

- 优点
  - 由于可以共享数据，多线程间的通信开销比进程小的多
  - 线程启动的比进程快，占用的资源更少
- 缺点
  - 共享数据太过于灵活，为了维护正确的共享，代码写起来比较复杂
  - 无法部署在分布式系统上



### 1.3 简单的多线程编程

`C++11`中终于提供了多线程的标准库，提供了管理线程、保护共享数据、线程间同步操作、原子操作等类。

多线程库对应的头文件是`#include <thread>`，类名为`std::thread`

一个简单的多线程的程序

```c++
#include <iostream>
#include <thread>

void function_1() {
    std::cout << "I'm function_1()" << std::endl;
}

int main() {
    std::thread t1(function_1);
    // do other things
    t1.join();
    return 0;
}
```

分析：

1. 首先，构建一个`std::thread`对象`t1`，构造的时候传递了一个参数，这个参数是一个函数，这个函数就是这个线程的**入口函数**，函数执行完了，整个线程也就执行完了。
2. 线程创建成功后，就会**立即启动**，并没有一个类似`start`的函数来显式的启动线程。
3. 一旦线程开始运行， 就需要显式的决定是要等待它完成(join)，或者分离它让它自行运行(detach)。注意：只需要在`std::thread`对象**被销毁之前**做出这个决定。这个例子中，对象`t1`是栈上变量，在`main`函数执行结束后就会被销毁，所以需要在`main`函数结束之前做决定。
4. 这个例子中选择了使用`t1.join()`，主线程会一直阻塞着，直到子线程完成，`join()`函数的另一个任务是回收该线程中使用的资源。

**线程对象和对象内部管理的线程的生命周期并不一样**，如果线程执行的快，可能内部的线程已经结束了，但是线程对象还活着，也有可能线程对象已经被析构了，内部的线程还在运行。

假设`t1`线程是一个执行的很慢的线程，主线程并不想等待子线程结束就想结束整个任务，直接删掉`t1.join()`是不行的，程序会被终止（析构`t1`的时候会调用`std::terminate`，程序会打印`terminate called without an active exception`）。

与之对应，我们可以调用`t1.detach()`，从而将`t1`线程放在后台运行，所有权和控制权被转交给`C++`运行时库，以确保与线程相关联的资源在线程退出后能被正确的回收。参考`UNIX`的**守护进程(daemon process)**的概念，这种被分离的线程被称为**守护线程(daemon threads)**。线程被分离之后，即使该线程对象被析构了，线程还是能够在后台运行，只是由于对象被析构了，主线程不能够通过对象名与这个线程进行通信。例如：

```c++
#include <iostream>
#include <thread>

void function_1() {
    //延时500ms 为了保证test()运行结束之后才打印
    std::this_thread::sleep_for(std::chrono::milliseconds(500)); 
    std::cout << "I'm function_1()" << std::endl;
}

void test() {
    std::thread t1(function_1);
    t1.detach();
    // t1.join();
    std::cout << "test() finished" << std::endl;
}

int main() {
    test();
    //让主线程晚于子线程结束
    std::this_thread::sleep_for(std::chrono::milliseconds(1000)); //延时1s
    return 0;
}

// 使用 t1.detach()时
// test() finished
// I'm function_1()

// 使用 t1.join()时
// I'm function_1()
// test() finished
```

分析：

1. 由于线程入口函数内部有个`500ms`的延时，所以在还没有打印的时候，`test()`已经执行完成了，`t1`已经被析构了，但是它负责的那个线程还是能够运行，这就是`detach()`的作用。
2. 如果去掉`main`函数中的`1s`延时，会发现**什么都没有打印**，因为主线程执行的太快，整个程序已经结束了，那个后台线程被`C++`运行时库回收了。
3. 如果将`t1.detach()`换成`t1.join()`，`test`函数会在`t1`线程执行结束之后，才会执行结束。

一旦一个线程被分离了，就不能够再被`join`了。如果非要调用，程序就会崩溃，可以使用`joinable()`函数判断一个线程对象能否调用`join()`。

```c++
void test() {
    std::thread t1(function_1);
    t1.detach();

    if(t1.joinable())
        t1.join();

    assert(!t1.joinable());
}
```
# 第二章
## 1. 线程类的构造函数

`std::thread`类的构造函数是使用**可变参数模板**实现的，也就是说，可以传递任意个参数，第一个参数是线程的入口**函数**，而后面的若干个参数是该函数的**参数**。

第一参数的类型并不是`c`语言中的函数指针（`c`语言传递函数都是使用函数指针），在`c++11`中，增加了**可调用对象(Callable Objects)**的概念，总的来说，可调用对象可以是以下几种情况：

- 函数指针
- 重载了`operator()`运算符的类对象，即仿函数
- `lambda`表达式（匿名函数）
- `std::function`

#### 函数指针示例

```c++
// 普通函数 无参
void function_1() {
}

// 普通函数 1个参数
void function_2(int i) {
}

// 普通函数 2个参数
void function_3(int i, std::string m) {
}

std::thread t1(function_1);
std::thread t2(function_2, 1);
std::thread t3(function_3, 1, "hello");

t1.join();
t2.join();
t3.join();
```

实验的时候还发现一个问题，如果将重载的函数作为线程的入口函数，会发生编译错误！编译器搞不清楚是哪个函数，如下面的代码：

```c++
// 普通函数 无参
void function_1() {
}

// 普通函数 1个参数
void function_1(int i) {
}
std::thread t1(function_1);
t1.join();
// 编译错误
/*
C:\Users\Administrator\Documents\untitled\main.cpp:39: 
error: no matching function for call to 'std::thread::thread(<unresolved overloaded function type>)'
     std::thread t1(function_1);
                              ^
*/
```

#### 仿函数

```c++
// 仿函数
class Fctor {
public:
    // 具有一个参数
    void operator() () {

    }
};
Fctor f;
std::thread t1(f);  
// std::thread t2(Fctor()); // 编译错误 
std::thread t3((Fctor())); // ok
std::thread t4{Fctor()}; // ok
```

一个仿函数类生成的对象，使用起来就像一个函数一样，比如上面的对象`f`，当使用`f()`时就调用`operator()`运算符。所以也可以让它成为线程类的第一个参数，如果这个仿函数有参数，同样的可以写在线程类的后几个参数上。

而`t2`之所以编译错误，是因为编译器并没有将`Fctor()`解释为一个临时对象，而是将其解释为一个函数声明，编译器认为你声明了一个函数，这个函数不接受参数，同时返回一个`Factor`对象。解决办法就是在`Factor()`外包一层小括号`()`，或者在调用`std::thread`的构造函数时使用`{}`，这是`c++11`中的新的同意初始化语法。

但是，如果重载的`operator()`运算符有参数，就不会发生上面的错误。

#### 匿名函数

```c++
std::thread t1([](){
    std::cout << "hello" << std::endl;
});

std::thread t2([](std::string m){
    std::cout << "hello " << m << std::endl;
}, "world");
```

#### std::function

```c++
class A{
public:
    void func1(){
    }

    void func2(int i){
    }
    void func3(int i, int j){
    }
};

A a;
std::function<void(void)> f1 = std::bind(&A::func1, &a);
std::function<void(void)> f2 = std::bind(&A::func2, &a, 1);
std::function<void(int)> f3 = std::bind(&A::func2, &a, std::placeholders::_1);
std::function<void(int)> f4 = std::bind(&A::func3, &a, 1, std::placeholders::_1);
std::function<void(int, int)> f5 = std::bind(&A::func3, &a, std::placeholders::_1, std::placeholders::_2);

std::thread t1(f1);
std::thread t2(f2);
std::thread t3(f3, 1);
std::thread t4(f4, 1);
std::thread t5(f5, 1, 2);
```

## 2. 传值还是引用

先提出一个问题：如果线程入口函数的的参数是引用类型，在线程内部修改该变量，主线程的变量会改变吗？

代码如下：

```c++
#include <iostream>
#include <thread>
#include <string>

// 仿函数
class Fctor {
public:
    // 具有一个参数 是引用
    void operator() (std::string& msg) {
        msg = "wolrd";
    }
};



int main() {
    Fctor f;
    std::string m = "hello";
    std::thread t1(f, m);

    t1.join();
    std::cout << m << std::endl;
    return 0;
}

// vs下： 最终是："hello"
// g++编译器： 编译报错
```

事实上，该代码使用`g++`编译会报错，而使用`vs2015`并不会报错，但是子线程并没有成功改变外面的变量`m`。

我是这么认为的：`std::thread`类，内部也有若干个变量，当使用构造函数创建对象的时候，是将参数先赋值给这些变量，所以这些变量只是个副本，然后在线程启动并调用线程入口函数时，传递的参数只是这些副本，所以内部怎么操作都是改变副本，而不影响外面的变量。`g++`可能是比较严格，这种写法可能会导致程序发生严重的错误，索性禁止了。

而如果可以想真正传引用，可以在调用线程类构造函数的时候，用`std::ref()`包装一下。如下面修改后的代码：

```c++
std::thread t1(f, std::ref(m));
```

然后`vs`和`g++`都可以成功编译，而且子线程可以修改外部变量的值。

当然这样并不好，多个线程同时修改同一个变量，会发生数据竞争。

同理，构造函数的第一个参数是可调用对象，默认情况下其实传递的还是一个副本。

```c++
#include <iostream>
#include <thread>
#include <string>

class A {
public:
    void f(int x, char c) {}
    int g(double x) {return 0;}
    int operator()(int N) {return 0;}
};

void foo(int x) {}

int main() {
    A a;
    std::thread t1(a, 6); // 1. 调用的是 copy_of_a()
    std::thread t2(std::ref(a), 6); // 2. a()
    std::thread t3(A(), 6); // 3. 调用的是 临时对象 temp_a()
    std::thread t4(&A::f, a, 8, 'w'); // 4. 调用的是 copy_of_a.f()
    std::thread t5(&A::f, &a, 8, 'w'); //5.  调用的是 a.f()
    std::thread t6(std::move(a), 6); // 6. 调用的是 a.f(), a不能够再被使用了
    t1.join();
    t2.join();
    t3.join();
    t4.join();
    t5.join();
    t6.join();
    return 0;
}
```

对于线程`t1`来说，内部调用的线程函数其实是一个副本，所以如果在函数内部修改了类成员，并不会影响到外面的对象。只有传递引用的时候才会修改。所以在这个时候就必须想清楚，到底是传值还是传引用！

## 3. 线程对象只能移动不可复制

线程对象之间是不能复制的，只能移动，移动的意思是，将线程的所有权在`std::thread`实例间进行转移。

```c++
void some_function();
void some_other_function();
std::thread t1(some_function);
// std::thread t2 = t1; // 编译错误
std::thread t2 = std::move(t1); //只能移动 t1内部已经没有线程了
t1 = std::thread(some_other_function); // 临时对象赋值 默认就是移动操作
std::thread t3;
t3 = std::move(t2); // t2内部已经没有线程了
t1 = std::move(t3); // 程序将会终止，因为t1内部已经有一个线程在管理了
```


## 参考文献

[【segmentFault】C++多线程编程](https://segmentfault.com/a/1190000016171072)

[【[Concurrent Programming with C++ 11](https://www.youtube.com/playlist?list=PL5jc9xFGsL8E12so1wlMS0r0hTQoJL74M)】](https://www.youtube.com/watch?v=LL8wkskDlbs&list=PL5jc9xFGsL8E12so1wlMS0r0hTQoJL74M&index=2&ab_channel=BoQian)

