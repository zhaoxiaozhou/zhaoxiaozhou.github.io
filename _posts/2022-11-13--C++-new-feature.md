---
layout: post
title: 【c++基础知识】C++11 新特性专栏
excerpt: "C++11新特性，包括auto、function、lambda、final、thread、bind、右值引用、移动语义、构造函数等等"
date: 2022-03-11 22:50:58
tags: ['C++', 'c++11新特性']
category: learning
---

# C++11 新特性

## auto关键字

```c++
auto i = 42;        // i is an int
auto l = 42LL;      // l is an long long
auto p = new foo(); // p is a foo*
```

## std::function

类模版`std::function`是一种通用、多态的函数封装。`std::function`的实例可以对任何可以调用的目标实体进行存储、复制、和调用操作，这些目标实体包括普通函数、`Lambda`表达式、函数指针、以及其它函数对象等。`std::function`对象是对C++中现有的可调用实体的一种类型安全的包裹（我们知道像函数指针这类可调用实体，是类型不安全的）

通常`std::function`是一个函数对象类，它包装其它任意的函数对象，被包装的函数对象具有类型为T1, …,TN的N个参数，并且返回一个可转换到R类型的值。`std::function`使用 模板转换构造函数接收被包装的函数对象；特别是，闭包类型可以隐式地转换为`std::function`

最简单的理解就是：

通过`std::function`对C++中各种可调用实体（普通函数、`Lambda`表达式、函数指针、以及其它函数对象等）的封装，形成一个新的可调用的`std::function`对象；让我们不再纠结那么多的可调用实体

```c++
//代码出自链接：http://www.jellythink.com/archives/771
#include <functional>
#include <iostream>
using namespace std;

std::function< int(int)> Functional;

// 普通函数
int TestFunc(int a)
{
    return a;
}

// Lambda表达式
auto lambda = [](int a)->int{ return a; };

// 仿函数(functor)
class Functor
{
public:
    int operator()(int a)
    {
        return a;
    }
};

// 1.类成员函数
// 2.类静态函数
class TestClass
{
public:
    int ClassMember(int a) { return a; }
    static int StaticMember(int a) { return a; }
};

int main()
{
    // 普通函数
    Functional = TestFunc;
    int result = Functional(10);
    cout << "普通函数："<< result << endl;

    // Lambda表达式
    Functional = lambda;
    result = Functional(20);
    cout << "Lambda表达式："<< result << endl;

    // 仿函数
    Functor testFunctor;
    Functional = testFunctor;
    result = Functional(30);
    cout << "仿函数："<< result << endl;

    // 类成员函数
    TestClass testObj;
    Functional = std::bind(&TestClass::ClassMember, testObj, std::placeholders::_1);
    result = Functional(40);
    cout << "类成员函数："<< result << endl;

    // 类静态函数
    Functional = TestClass::StaticMember;
    result = Functional(50);
    cout << "类静态函数："<< result << endl;

    return 0;
}

int main()
{
    vector<std::function<int(int)>> funcVec;

    funcVec.push_back(testFunc);
    funcVec.push_back(lambda);
    funcVec.push_back(Functor());
    funcVec.push_back(std::bind(&testClass::classMember, testClass(), std::placeholders::_1));
    funcVec.push_back(testClass::staticMember);

    for (auto const& func : funcVec) {
        std::cout << func(20) << std::endl;
    }

    return 0;
}
```

于可调用实体转换为`std::function`对象需要遵守以下两条原则： 
转换后的`std::function`对象的参数能转换为可调用实体的参数； 
可调用实体的返回值能转换为`std::function`对象的返回值。 
`std::function`对象最大的用处就是在**实现函数回调**（实际工作中就是用到了这一点），使用者需要注意，它不能被用来检查相等或者不相等，但是可以与`NULL`或者`nullptr`进行比较

 ## 正则表达式

```c++
#include <iostream>
#include <iterator>
#include <string>
#include <regex>

int main()
{
    std::string s = "Some people, when confronted with a problem, think "
        "\"I know, I'll use regular expressions.\" "
        "Now they have two problems.";

    std::regex self_regex("REGULAR EXPRESSIONS",
            std::regex_constants::ECMAScript | std::regex_constants::icase);
    if (std::regex_search(s, self_regex)) {
        std::cout << "Text contains the phrase 'regular expressions'\n";
    }

    std::regex word_regex("(\\S+)");
    auto words_begin = 
        std::sregex_iterator(s.begin(), s.end(), word_regex);
    auto words_end = std::sregex_iterator();

    std::cout << "Found "
              << std::distance(words_begin, words_end)
              << " words\n";

    const int N = 6;
    std::cout << "Words longer than " << N << " characters:\n";
    for (std::sregex_iterator i = words_begin; i != words_end; ++i) {
        std::smatch match = *i;
        std::string match_str = match.str();
        if (match_str.size() > N) {
            std::cout << "  " << match_str << '\n';
        }
    }

    std::regex long_word_regex("(\\w{7,})");
    std::string new_s = std::regex_replace(s, long_word_regex, "[$&]");
    std::cout << new_s << '\n';
}
Output:
Text contains the phrase 'regular expressions'
Found 19 words
Words longer than 6 characters:
  people,
  confronted
  problem,
  regular
  expressions."
  problems.
Some people, when [confronted] with a [problem], think 
"I know, I'll use [regular] [expressions]." Now they have two [problems].
```

[【正则表达式30分钟入门教程】](https://deerchao.cn/tutorials/regex/regex.htm)

## Lambda表达式

“Lambda 表达式”(lambda expression)是一个匿名函数，`Lambda`表达式基于数学中的λ演算得名，直接对应于其中的lambda抽象(lambda abstraction)，是一个匿名函数，即没有函数名的函数。

**[capture list] (parameter list) ->return type { function body }**

[] // 沒有定義任何變數。使用未定義變數會導致錯誤。 
[x, &y] // x 以傳值方式傳入(預設)，y 以傳參考方式傳入。 
[&] // 任何被使用到的外部變數皆隱式地以參考方式加以引用。 
[=] // 任何被使用到的外部變數皆隱式地以傳值方式加以引用。 
[&, x] // x 顯示地以傳值方式加以引用。其餘變數以參考方式加以引用。 
[=, &z] // z 顯示地以參考方式加以引用。其餘變數以傳值方式加以引用。

```c++
class CTest 
{ 
 public:  
   CTest() : m_nData(20) { NULL; }  
   void TestLambda()  
   {   
     vector<int> vctTemp;   
     vctTemp.push_back(1);   
     vctTemp.push_back(2);    

   // 无函数对象参数，输出：1 2   
   {    
     for_each(vctTemp.begin(), vctTemp.end(), [](int v){ cout << v << endl; });   
   }   

   // 以值方式传递作用域内所有可见的局部变量（包括this），输出：11 12   
   {    
     int a = 10;    
     for_each(vctTemp.begin(), vctTemp.end(), [=](int v){ cout << v+a << endl; });   
   }    

   // 以引用方式传递作用域内所有可见的局部变量（包括this），输出：11 13 12   
   {    
     int a = 10;   
     for_each(vctTemp.begin(), vctTemp.end(), [&](int v)mutable{ cout << v+a << endl; a++; });    
     cout << a << endl;   
   }    

   // 以值方式传递局部变量a，输出：11 13 10   
   {    
     int a = 10;    
     for_each(vctTemp.begin(), vctTemp.end(), [a](int v)mutable{ cout << v+a << endl; a++; });    
     cout << a << endl;   
   }    

   // 以引用方式传递局部变量a，输出：11 13 12   
   {    
     int a = 10;    
     for_each(vctTemp.begin(), vctTemp.end(), [&a](int v){ cout << v+a << endl; a++; });    
     cout << a << endl;  
   }    

   // 传递this，输出：21 22 
   {  
     for_each(vctTemp.begin(), vctTemp.end(), [this](int v){ cout << v+m_nData << endl; });   
   }    

   // 除b按引用传递外，其他均按值传递，输出：11 12 17   
   {    
     int a = 10;    
     int b = 15;    
     for_each(vctTemp.begin(), vctTemp.end(), [=, &b](int v){ cout << v+a << endl; b++; });    
     cout << b << endl;   
   }     
   // 操作符重载函数参数按引用传递，输出：2 3   
   {    
     for_each(vctTemp.begin(), vctTemp.end(), [](int &v){ v++; });    
     for_each(vctTemp.begin(), vctTemp.end(), [](int v){ cout << v << endl; });   
   }    
   // 空的Lambda表达式   
   {    
     [](){}();    []{}();   
   }  
 }  
 private:  int m_nData; 
};
```

其中除了“[ ]”（其中捕获列表可以为空）和“复合语句”（相当于具名函数定义的函数体），其它都是可选的。

C++中，一个`lambda`表达式表示一个可调用的代码单元。我们可以将其理解为一个未命名的内联函数。它与普通函数不同的是，`lambda`必须使用尾置返回来指定返回类型。

你可能会问，除了那些表达式爱好者，谁会用`lambda`表达式呢？ 
1、距离 
程序员认为，让定义位于使用的地方附近很有用，因此这样无需翻阅多页的源代码。从这个角度看，`Lambda`表达式是理想的选择，因为定义和使用是在同一个地方。 
2、简洁 
函数符代码比函数和`Lambda`表达式更加繁琐，函数和`Lambda`表达式的简洁程度相当。但是不同于函数的是，`Lambda`可以定义于函数内部。 
3、`lambda`可访问作用域内的任何动态变量

##  override和final

C++ 11添加了两个继承控制关键字：`override`和`final`。override确保在派生类中声明的重载函数跟基类的虚函数有相同的签名。final阻止类的进一步派生和虚函数的进一步重载

**虚函数重载**

一个派生类可以重载在基类中声明的成员函数，这是面向对象设计的基础。然而像重载一个函数这么简单的操作也会出错。关于重载虚函数的两个常见错误如下： 
**无意中重载**
**签名不匹配**

首先，我们来分析一下无意中重载的综合症。你可能只是通过声明了一个与基类的某个虚成员函数具有相同的名字和签名的成员函数而无意中重载了这个虚函数。编译器和读代码的人很难发现这个bug因为他们通常以为这个新函数是为了实现对基类函数的重载：

```c++
class A
{
 public:
    virtual void func();
};            
class B: A{};
class F{};
class D: A, F
{
 public:
  void func();//meant to declare a new function but

 //accidentally overrides A::func};
```

阅读以上代码，你不能确定成员函数D::func()是否故意重载了`A::func()`.它也可能是个偶然发生的重载，因为两个函数的参数列表和名字都碰巧一样。

签名不匹配是一个更为常见的情景。这导致意外创建一个新的虚函数（而不是重载一个已存在的虚函数），正如以下例子所示：

```c++
class  G
{
public:
 virtual void func(int);
};

class H: G
{
public:
 virtual void func(double); 
};
```

这种情况下，程序员本打算在类H中重载`G::func()`的。然而，由于`H::func()`拥有不同的签名，结果创建了一个新的虚函数，而非对基类函数的重载：

```c+=
H *p=new H;
p->func(5); //calls G::f
p->func(5.0); // calls H::f
```

基于上面的两个错误

在C++11中，通过使用新关键字override可以消除这两个bugs。**override明确地表示一个函数是对基类中一个虚函数的重载**。更重要的是，它会检查基类虚函数和派生类中重载函数的签名不匹配问题。如果签名不匹配，编译器会发出错误信息。

我们来看看override如何消除签名不匹配bug的：

```c++
class G
{
public:
 virtual void func(int);
};
class H: G
{
public:
 virtual void func(double) override; //compilation error
};
```

**final函数和类**

C++11的关键字final有两个用途。第一，它阻止了从类继承；第二，阻止一个虚函数的重载。我们先来看看final类吧。

程序员常常在没有意识到风险的情况下坚持从`std::vector`派生。在C++11中，无子类类型将被声明为如下所示：

```c++
class TaskManager {/*..*/} final; 
class PrioritizedTaskManager: public TaskManager {
};  //compilation error: base class TaskManager is final
```

同样，你可以通过声明它为final来禁止一个虚函数被进一步重载。如果一个派生类试图重载一个final函数，编译器就会报错：

```c++
class A
{
pulic:
  virtual void func() const;
};
class  B: A
{
pulic:
  void func() const override final; //OK
};
class C: B
{
pulic:
 void func()const; //error, B::func is final
};
```

`C::func()`是否声明为override没关系，一旦一个虚函数被声明为final，派生类不能再重载它。

## std::thread

默认构造函数： 
thread() noexcept; 
构造一个任何线程不执行的线程对象。

初始化函数：

```c++
template <class Fn, class... Args>
explicit thread (Fn&& fn, Args&&... args);
```

构造一个线程对象，可以开启线程执行 
新执行的线程调用函数fn，并传递args作为参数 
fn 
可以指向函数，指向成员，或是移动构造函数 
args… 
传递给fn的参数，这些参数可以移动赋值构造。如果fn是一个成员指针，那么第一个args参数就必须是一个对象，或是引用，或是指向该对象的指针。

拷贝构造函数： 
thread (const thread&) = delete; 
删除构造的线程

**std::thread::join**
该函数返回时，线程执行完成。 
当 a thread 调用Join方法的时候，MainThread 就被停止执行，直到 a thread 线程执行完毕。

```c++
#include <iostream>       // std::cout
#include <thread>         // std::thread, std::this_thread::sleep_for
#include <chrono>         // std::chrono::seconds

void pause_thread(int n) 
{
  std::this_thread::sleep_for (std::chrono::seconds(n));
  std::cout << "pause of " << n << " seconds ended\n";
}

int main() 
{
  std::cout << "Spawning 3 threads...\n";
  std::thread t1 (pause_thread,1);
  std::thread t2 (pause_thread,2);
  std::thread t3 (pause_thread,3);
  std::cout << "Done spawning threads. Now waiting for them to join:\n";
  t1.join();
  t2.join();
  t3.join();
  std::cout << "All threads joined!\n";

  return 0;
}

//输出
Output (after 3 seconds):
Spawning 3 threads...
Done spawning threads. Now waiting for them to join:
pause of 1 seconds ended
pause of 2 seconds ended
pause of 3 seconds ended
All threads joined!
```

**std::thread::detach**
分离线程的对象，使它们从彼此独立地执行所表示的线程。这两个线程继续没有阻止，也没有以任何方式同步。注意，当任一结束执行，其资源被释放。

```c++
#include <iostream>       // std::cout
#include <thread>         // std::thread, std::this_thread::sleep_for
#include <chrono>         // std::chrono::seconds

void pause_thread(int n) 
{
  std::this_thread::sleep_for (std::chrono::seconds(n));
  std::cout << "pause of " << n << " seconds ended\n";
}

int main() 
{
  std::cout << "Spawning and detaching 3 threads...\n";
  std::thread (pause_thread,1).detach();
  std::thread (pause_thread,2).detach();
  std::thread (pause_thread,3).detach();
  std::cout << "Done spawning threads.\n";

  std::cout << "(the main thread will now pause for 5 seconds)\n";
  // give the detached threads time to finish (but not guaranteed!):
  pause_thread(5);
  return 0;
}
//输出
Output (after 5 seconds):
Spawning and detaching 3 threads...
Done spawning threads.
(the main thread will now pause for 5 seconds)
pause of 1 seconds ended
pause of 2 seconds ended
pause of 3 seconds ended
pause of 5 seconds ended
```

**给线程传递参数**

```c++
#include <iostream>
#include <thread>
#include <string>

void thread_function(std::string s)
{
    std::cout << "thread function ";
    std::cout << "message is = " << s << std::endl;
}

int main()
{
    std::string s = "Kathy Perry";
    std::thread t(&thread_function, s);
    std::cout << "main thread message = " << s << std::endl;
    t.join();
    return 0;
}
```

传递引用的参数，需要使用std::ref()或者std::move

**std::thread::get_id()**;

```c++
int main()
{
    std::string s = "Kathy Perry";
    std::thread t(&thread_function, std::move(s));
    std::cout << "main thread message = " << s << std::endl;

    std::cout << "main thread id = " << std::this_thread::get_id() << std::endl;
    std::cout << "child thread id = " << t.get_id() << std::endl;

    t.join();
    return 0;
}
Output:

thread function message is = Kathy Perry
main thread message =
main thread id = 1208
child thread id = 5224
```

## 参数绑定std::bind

std::bind函数定义在头文件functional中，是一个函数模板，它就像一个函数适配器，接受一个可调用对象（callable object），生成一个新的可调用对象来“适应”原对象的参数列表。一般而言，我们用它可以把一个原本接收N个参数的函数fn，通过绑定一些参数，返回一个接收M个（M可以大于N，但这么做没什么意义）参数的新函数。同时，使用std::bind函数还可以实现参数顺序调整等操作
std::bind函数有两种函数原型，定义如下：

```c++
template< class F, class... Args >
/*unspecified*/ bind( F&& f, Args&&... args );
 
template< class R, class F, class... Args >
/*unspecified*/ bind( F&& f, Args&&... args );
```

```c++

#include <iostream>
#include <functional>
 
void fn(int n1, int n2, int n3) {
	std::cout << n1 << " " << n2 << " " << n3 << std::endl;
}
 
int fn2() {
	std::cout << "fn2 has called.\n";
	return -1;
}
 
int main()
{
	using namespace std::placeholders;
	auto bind_test1 = std::bind(fn, 1, 2, 3);
	auto bind_test2 = std::bind(fn, _1, _2, _3);
	auto bind_test3 = std::bind(fn, 0, _1, _2);
	auto bind_test4 = std::bind(fn, _2, 0, _1);
 
	bind_test1();//输出1 2 3
	bind_test2(3, 8, 24);//输出3 8 24
	bind_test2(1, 2, 3, 4, 5);//输出1 2 3，4和5会被丢弃
	bind_test3(10, 24);//输出0 10 24
	bind_test3(10, fn2());//输出0 10 -1
	bind_test3(10, 24, fn2());//输出0 10 24，fn2会被调用，但其返回值会被丢弃
	bind_test4(10, 24);//输出24 0 10
	return 0;
}
```

https://en.cppreference.com/w/cpp/utility/functional/bind这个里面的例子也很好的

## initializer_list

类的构造函数使用初始化列表来初始化成员变量

```c++
std::vector v = { 1, 2, 3, 4 };
```

这就是所谓的initializer list。

更进一步，有一个关键字叫initializer list

C++11允许构造函数和其他函数把初始化列表当做参数。

```c++
#include <iostream>
#include <vector>

class MyNumber
{
public:
    MyNumber(const std::initializer_list<int> &v) {
        for (auto itm : v) {
            mVec.push_back(itm);
        }
    }

    void print() {
    for (auto itm : mVec) {
        std::cout << itm << " ";
    }
    }
private:
    std::vector<int> mVec;
};

int main()
{
    MyNumber m = { 1, 2, 3, 4 };
    m.print();  // 1 2 3 4

    return 0;
}
```

最后写一个类，可以对比一下，加深理解

```c++
class CompareClass 
{
  CompareClass (int,int);
  CompareClass (initializer_list<int>);
};

int main（）
{
    myclass foo {10,20};  // calls initializer_list ctor
    myclass bar (10,20);  // calls first constructor 
}
```

## CALLBACKS

提到了std::function作为回调函数。

今天主要讨论不同情况下std::function作为回调使用。

```c++
#include <functional>
#include <iostream>
namespace {
using cb1_t = std::function<void()>;
using cb2_t = std::function<void(int)>;

void foo1()
{
    std::cout << "foo1 is called\n";
}

void foo2(int i)
{
    std::cout << "foo2 is called with: " << i << "\n";
}

struct S {
    void foo3()
    {
        std::cout << "foo3 is called.\n";
    }
};

}

int main()
{
    // Bind a free function.
    cb1_t f1 = std::bind(&foo1);
    // Invoke the function foo1.
    f1();

    // Bind a free function with an int argument.
    // Note that the argument can be specified with bind directly.
    cb1_t f2 = std::bind(&foo2, 5);
    // Invoke the function foo2.
    f2();

    // Bind a function with a placeholder.
    cb2_t f3 = std::bind(&foo2, std::placeholders::_1);
    // Invoke the function with an argument.
    f3(42);

    // Bind a member function.
    S s;
    cb1_t   f4 = std::bind(&S::foo3, &s);
    // Invoke the method foo3.
    f4();

    // Bind a lambda.
    cb1_t f5 = std::bind([] { std::cout << "lambda is called\n"; });
    f5();

    return 0;
}
```

**存储回调**
使用回调是非常好的，但是更多的情况下，我们往往需要存储一些回调函数，并稍后使用。例如，注册一个客户端到某个事件上。（也许注册和事件是C Sharp的词汇）

我们经常使用std::vector来完成任务。 
缺点就是，我们不能再vector中存储不同的类型 。 
但是，大多数情况下，一种类型就可以满足我们了。

```c++
#include <vector>
#include <functional>
#include <iostream>

namespace {

using cb1_t = std::function<void()>;
using callbacks_t = std::vector<cb1_t>;

callbacks_t callbacks;

void foo1()
{
    std::cout << "foo1 is called\n";
}

void foo2(int i)
{
    std::cout << "foo2 is called with: " << i << "\n";
}

} // end anonymous namespace

int main()
{
    // Bind a free function.
    cb1_t f1 = std::bind(&foo1);
    callbacks.push_back(f1);

    // Bind a free function with an int argument.
    // Here the argument is statically known.
    cb1_t f2 = std::bind(&foo2, 5);
    callbacks.push_back(f2);

    // Bind a free function with an int argument.
    // Here the argument is bound and can be changed at runtime.
    int n = 15;
    cb1_t f3 = std::bind(&foo2, std::cref(n));
    callbacks.push_back(f3);

    // Invoke the functions
    for(auto& fun : callbacks) {
        fun();
    }
    return 0;
}
```

**包装函数**
上面提到都不是重点，个人觉得特别重要的就是std::function作为函数的参数使用。 
下面一个例子就将展示一个函数被copy不同的参数。

```c++
#include <functional>
#include <iostream>

namespace {

using cb1_t = std::function<void()>;
using cb2_t = std::function<void(int)>;

// Wrapper function with std::function without arguments.
template<typename R>
void call(std::function<R(void)> f)
{
    f();
}

// Wrapper function with std::function with arguments.
template<typename R, typename ...A>
void call(std::function<R(A...)> f, A... args)
{
    f(args...);
}

// Wrapper function for generic callable object without arguments.
// Delegates to the std::function call.
template<typename R>
void call(R f(void))
{
    call(std::function<R(void)>(f));
}

// Wrapper function for generic callable object with arguments.
// Delegates to the std::function call.
template<typename R, typename ...A>
void call(R f(A...), A... args)
{
    call(std::function<R(A...)>(f), args...);
}

// Wrapper for a function pointer (e.g. a lambda without capture) without
// arguments.
using fp = void (*)(void);
void call(fp f)
{
    call(std::function<void()>(f));
}

void foo1()
{
    std::cout << "foo1 is called\n";
}

void foo2(int i)
{
    std::cout << "foo2 is called with: " << i << "\n";
}

} // end anonymous namespace

int main()
{
    // Call function 1.
    call(&foo1);

    // Alternative to call function 1.
    cb1_t f1 = std::bind(&foo1);
    call(f1);

    // Call function 2.
    call(&foo2, 5);

    // Alternative to call function 2.
    cb2_t f2 = std::bind(&foo2, std::placeholders::_1);
    call(f2, 5);

    // Here is an example with a lambda. It calls the function that takes a
    // function pointer.
    call([] { std::cout << "lambda called\n"; });

    return 0;
}
```

上面的内容比较难一点，需要好好看一下子

## std::array container

但是某些场合，使用vector是多余的，尤其能明确元素个数的情况下，这样我们就付出了效率稍低的代价，与vector不同的是，array对象的长度是固定的，使用了静态存储区，即存储在栈上，效率跟数组相同，但是更加的安全。

首先需要包含头文件array，而且语法与vector也有所不同

```c++
#include<array>
...
std::array<int,5> ai;//创建一个有5个int成员的对象
std::array<double,4> ad = {1.2, 2.1, 3.1, 4.1};
```

顺便提一嘴，C++11允许使用初始化列表对vetcor和array进行赋值，详情请见博客《 [c++11特性之initializer_list](http://blog.csdn.net/wangshubo1989/article/details/49622871)》

个人觉得array相比于数组最大的优势就是可以将一个array对象赋值给另一个array对象：

```c++
std::array<double, 4> ad = {1.2, 2.1, 3.1, 4.1};
std::array<double, 4> ad1;
ad1 = ad;
```

数组索引越界也是我们往往躲不过的坑儿，array和vector使用中括号索引时也不会检查索引值的正确性。

但是，他们有一个成员函数可以替代中括号进行索引，这样越界就会进行检查:

```c++
std::array<double, 4> ad = {1.2, 2.1, 3.1, 4.1};
ad[-2] = 0.5;//合法
ad.at(1) = 1.1;
```

使用at()，将在运行期间捕获非法索引的，默认将程序中断。

最后上一段代码：

```c++
#include <string>
#include <iterator>
#include <iostream>
#include <algorithm>
#include <array>

int main()
{
    // construction uses aggregate initialization
    std::array<int, 5> i_array1{ {3, 4, 5, 1, 2} };  // double-braces required
    std::array<int, 5> i_array2 = {1, 2, 3, 4, 5};   // except after =
    std::array<std::string, 2> string_array = { {std::string("a"), "b"} };

    std::cout << "Initial i_array1 : ";
    for(auto i: i_array1)
        std::cout << i << ' ';
    // container operations are supported
    std::sort(i_array1.begin(), i_array1.end());

    std::cout << "\nsored i_array1 : ";
    for(auto i: i_array1)
        std::cout << i << ' ';

    std::cout << "\nInitial i_array2 : ";
    for(auto i: i_array2)
        std::cout << i << ' ';

    std::cout << "\nreversed i_array2 : ";
    std::reverse_copy(i_array2.begin(), i_array2.end(),
                      std::ostream_iterator<int>(std::cout, " "));

    // ranged for loop is supported
    std::cout << "\nstring_array : ";
    for(auto& s: string_array)
        std::cout << s << ' ';

    return 0;
}
```

##  rvalue Reference(右值引用)

右值引用可以使我们区分表达式的左值和右值。

C++11引入了右值引用的概念，使得我们把引用与右值进行绑定。使用两个“取地址符号”：

```c++
int&& rvalue_ref = 99;
```

需要注意的是，只有左值可以付给引用，如：

```c++
int& ref = 9;   
```

我们会得到这样的错误： “invalid initialization of non-const reference of type int& from an rvalue of type int”

我们只能这样做：

```c++
int nine = 9;
int& ref = nine;
```

看下面的例子，你会明白的：

```c++
#include <iostream>

void f(int& i) { std::cout << "lvalue ref: " << i << "\n"; }
void f(int&& i) { std::cout << "rvalue ref: " << i << "\n"; }

int main()
{
    int i = 77;
    f(i);    // lvalue ref called
    f(99);   // rvalue ref called

    f(std::move(i));  // 稍后介绍

    return 0;
}
```

**右值有更隐晦的，记住如果一个表达式的结果是一个暂时的对象，那么这个表达式就是右值**，同样看看代码：

```c++
#include <iostream>

int getValue ()
{
    int ii = 10;
    return ii;
}

int main()
{
    std::cout << getValue();
    return 0;
}
```

这里需要注意的是 getValue() 是一个右值。

我们只能这样做，必须要有const

```c++
const int& val = getValue(); // OK
int& val = getValue(); // NOT OK
```

但是C++11中的右值引用允许这样做：

```c++
const int&& val = getValue(); // OK
int&& val = getValue(); //  OK
```

现在做一个比较吧：

```c++
void printReference (const int& value)
{
        cout << value;
}

void printReference (int&& value)
{
        cout << value;
}
```

第一个printReference()可以接受参数为左值，也可以接受右值。 
而第二个printReference()只能接受右值引用作为参数。

In other words, by using the rvalue references, we can use function overloading to determine whether function parameters are lvalues or rvalues by having one overloaded function taking an lvalue reference and another taking an rvalue reference. In other words, C++11 introduces a new non-const reference type called an rvalue reference, identified by T&&. **This refers to temporaries that are permitted to be modified after they are initialized, which is the cornerstone of move semantics.**

```c++
#include <iostream>

using namespace std;

void printReference (int& value)
{
        cout << "lvalue: value = " << value << endl;
}

void printReference (int&& value)
{
        cout << "rvalue: value = " << value << endl;
}

int getValue ()
{
    int temp_ii = 99;
    return temp_ii;
}

int main()
{ 
    int ii = 11;
    printReference(ii);
    printReference(getValue());  //  printReference(99);
    return 0;
}
/*----------------------
输出
lvalue: value = 11
rvalue: value = 99
----------------------*/
```

## Move semantics(移动语义)

按值传递的意义是什么？ 
当一个函数的参数按值传递时，这就会进行拷贝。当然，编译器懂得如何去拷贝。 
**而对于我们自定义的类型，我们也许需要提供拷贝构造函数**。

但是不得不说，拷贝的代价是昂贵的。

所以我们需要寻找一个避免不必要拷贝的方法，即C++11提供的移动语义。 上一篇博客中有一个句话用到了：

```c++
#include <iostream>

void f(int& i) { std::cout << "lvalue ref: " << i << "\n"; }
void f(int&& i) { std::cout << "rvalue ref: " << i << "\n"; }

int main()
{
    int i = 77;
    f(i);    // lvalue ref called
    f(99);   // rvalue ref called

    f(std::move(i));  // 稍后介绍

    return 0;
}
```

**实际上，右值引用主要用于创建移动构造函数和移动赋值运算。**

移动构造函数类似于拷贝构造函数，把类的实例对象作为参数，并创建一个新的实例对象。 
但是 移动构造函数可以避免**内存的重新分配**，因为我们知道右值引用提供了一个暂时的对象，而不是进行copy，所以我们可以进行移动。

换言之，在设计到关于临时对象时，右值引用和移动语义允许我们避免不必要的拷贝。我们不想拷贝将要消失的临时对象，所以这个临时对象的资源可以被我们用作于其他的对象。

右值就是典型的临时变量，并且他们可以被修改。如果我们知道一个函数的参数是一个右值，我们可以把它当做一个临时存储。这就意味着我们要移动而不是拷贝右值参数的内容。这就会节省很多的空间。

说多无语，看代码：

**下面的代码四种构造函数都有，非常值得学习一下**

```c++
#include <iostream>
#include <algorithm>

class A
{
public:

    // Simple constructor that initializes the resource.
    explicit A(size_t length)
        : mLength(length), mData(new int[length])
    {
        std::cout << "A(size_t). length = "
        << mLength << "." << std::endl;
    }

    // Destructor.
    ~A()
    {
    std::cout << "~A(). length = " << mLength << ".";

    if (mData != NULL) {
            std::cout << " Deleting resource.";
        delete[] mData;  // Delete the resource.
    }

    std::cout << std::endl;
    }

    // Copy constructor.
    A(const A& other)
        : mLength(other.mLength), mData(new int[other.mLength])
    {
    std::cout << "A(const A&). length = "
        << other.mLength << ". Copying resource." << std::endl;

    std::copy(other.mData, other.mData + mLength, mData);
    }

    // Copy assignment operator.
    A& operator=(const A& other)
    {
    std::cout << "operator=(const A&). length = "
             << other.mLength << ". Copying resource." << std::endl;

    if (this != &other) {
        delete[] mData;  // Free the existing resource.
        mLength = other.mLength;
            mData = new int[mLength];
            std::copy(other.mData, other.mData + mLength, mData);
    }
    return *this;
    }

    // Move constructor.
    A(A&& other) : mData(NULL), mLength(0)
    {
        std::cout << "A(A&&). length = " 
             << other.mLength << ". Moving resource.\n";

        // Copy the data pointer and its length from the 
        // source object.
        mData = other.mData;
        mLength = other.mLength;

        // Release the data pointer from the source object so that
        // the destructor does not free the memory multiple times.
        other.mData = NULL;
        other.mLength = 0;
    }

    // Move assignment operator.
    A& operator=(A&& other)
    {
        std::cout << "operator=(A&&). length = " 
             << other.mLength << "." << std::endl;

        if (this != &other) {
          // Free the existing resource.
          delete[] mData;

          // Copy the data pointer and its length from the 
          // source object.
          mData = other.mData;
          mLength = other.mLength;

          // Release the data pointer from the source object so that
          // the destructor does not free the memory multiple times.
          other.mData = NULL;
          other.mLength = 0;
       }
       return *this;
    }

    // Retrieves the length of the data resource.
    size_t Length() const
    {
        return mLength;
    }

private:
    size_t mLength; // The length of the resource.
    int* mData;     // The resource.
};
```

**移动构造函数**
语法：

```c++
A(A&& other) noexcept    // C++11 - specifying non-exception throwing functions
{
  mData =  other.mData;  // shallow copy or referential copy
  other.mData = nullptr;
}
```

最主要的是没有用到新的资源，是移动而不是拷贝。 
假设一个地址指向了一个有一百万个int元素的数组，使用move构造函数，我们没有创造什么，所以代价很低。

```c++
// Move constructor.
A(A&& other) : mData(NULL), mLength(0)
{
    // Copy the data pointer and its length from the 
    // source object.
    mData = other.mData;
    mLength = other.mLength;

    // Release the data pointer from the source object so that
    // the destructor does not free the memory multiple times.
    other.mData = NULL;
    other.mLength = 0;
}
```

**移动比拷贝更快！！！**

**移动赋值运算符**
语法：

**移动赋值运算符**
语法：

```c++
A& operator=(A&& other) noexcept
{
  mData =  other.mData;
  other.mData = nullptr;
  return *this;
}
```

工作流程这样的：Google上这么说的：

Release any resources that *this currently owns. 
Pilfer other’s resource. 
Set other to a default state. 
Return *this.

```c++
// Move assignment operator.
A& operator=(A&& other)
{
    std::cout << "operator=(A&&). length = " 
             << other.mLength << "." << std::endl;

    if (this != &other) {
      // Free the existing resource.
      delete[] mData;

      // Copy the data pointer and its length from the 
      // source object.
      mData = other.mData;
      mLength = other.mLength;

      // Release the data pointer from the source object so that
      // the destructor does not free the memory multiple times.
      other.mData = NULL;
      other.mLength = 0;
   }
   return *this;
}
```

让我们看几个move带来的好处吧！ 
vector众所周知，C++11后对vector也进行了一些优化。例如vector::push_back()被定义为了两种版本的重载，一个是cosnt T&左值作为参数，一个是T&&右值作为参数。例如下面的代码：

```c++
std::vector<A> v;
v.push_back(A(25));
v.push_back(A(75));
```

上面两个push_back()都会调用push_back(T&&)版本，因为他们的参数为右值。这样提高了效率。

而 当参数为左值的时候，会调用push_back(const T&) 。

```c++
#include <vector>

int main()
{
    std::vector<A> v;
    A aObj(25);       // lvalue
    v.push_back(aObj);  // push_back(const T&)
}
```

但事实我们可以使用 static_cast进行强制：

```c++
// calls push_back(T&&)
v.push_back(static_cast<A&&>(aObj));
```

我们可以使用std::move完成上面的任务：

```c++
v.push_back(std::move(aObj));  //calls push_back(T&&)
```

似乎push_back(T&&)永远是最佳选择，但是一定要记住： 
push_back(T&&) 使得参数为空。如果我们想要保留参数的值，我们这个时候需要使用拷贝，而不是移动。

最后写一个例子，看看如何使用move来交换两个对象：

```c++
#include <iostream>
using namespace std;

class A
{
  public:
    // constructor
    explicit A(size_t length)
        : mLength(length), mData(new int[length]) {}

    // move constructor
    A(A&& other)
    {
      mData = other.mData;
      mLength = other.mLength;
      other.mData = nullptr;
      other.mLength = 0;
    }

    // move assignment
    A& operator=(A&& other) noexcept
    {
      mData =  other.mData;
      mLength = other.mLength;
      other.mData = nullptr;
      other.mLength = 0;
      return *this;
    }

    size_t getLength() { return mLength; }

    void swap(A& other)
    {
      A temp = move(other);
      other = move(*this);
      *this = move(temp);
    }

    int* get_mData() { return mData; }

  private:
    int *mData;
    size_t mLength;
};

int main()
{
  A a(11), b(22);
  cout << a.getLength() << ' ' << b.getLength() << endl;
  cout << a.get_mData() << ' ' << b.get_mData() << endl;
  swap(a,b);
  cout << a.getLength() << ' ' << b.getLength() << endl;
  cout << a.get_mData() << ' ' << b.get_mData() << endl;
  return 0;
}
```

## default and delete specifiers

**default**
首先我们清楚，如果自己提供了任何形式的构造函数，那么编译器将不会产生一个默认构造函数，这是一个放之四海而皆准的原则。

但凡是都是双面性，看看一下的代码：

```c++
class A
{
public:
    A(int a){};
};
```

然后我们这样使用：

```c++
A a;
```

悲剧发生了，编译器不会为我们提供默认的构造函数！

所以呢，我们要暴力一些，对编译器做一些强制的规定。这时候default关键字出场了：

```c++
class A
{
public:
    A(int a){}
    A() = default;
};
```

于是再也不用担心下面的代码报错了：

```c++
A a;
```

下面谈一谈delete 

**delete**
有这样一个类：

```c++
class A
{
public:
    A(int a){};
};
```

下面的使用都会正确：

```c++
A a(10);     // OK
A b(3.14);   // OK  3.14 will be converted to 3
a = b;       // OK  We have a compiler generated assignment operator
```

然而，如果我们不想让上面的某种调用成功呢，比如不允许double类型作为参数呢？

你当然想到了，关键字delete：

```c++
class A
{
public:
    A(int a){};
    A(double) = delete;         // conversion disabled
    A& operator=(const A&) = delete;  // assignment operator disabled
};
```

这时候你使用代码：

```c++
A a(10);     // OK
A b(3.14);   // Error: conversion from double to int disabled
a = b;       // Error: assignment operator disabled
```

通过关键字default和delete 我们随心所欲！！！！

## Static assertions 和constructor delegation

**Static assertion**
static_assert 是在编译时期的断言，作用不言而喻的。 
语法是这样：

```c++
static_assert ( bool_constexpr , string )   
```

其中： 
bool_constexpr: 常量表达式 
string: 如果bool_constexpr表达式为false, 这个string就是编译时候报的错误。

看看代码：

```c++
// run-time assert
assert(ptr != NULL)

// C++ 11
// compile-time assert
    
static_assert(sizeof(void *) == 4, "64-bit is not supported.");
```

## 类的各种构造函数

```c++
#include <iostream>
using namespace std;

class A {
public:
    int x;
    A(int x) : x(x)
    {
        cout << "Constructor" << endl;
    }
    A(A& a) : x(a.x)
    {
        cout << "Copy Constructor" << endl;
    }
    A& operator=(A& a)
    {
        x = a.x;
        cout << "Copy Assignment operator" << endl;
        return *this;
    }
    A(A&& a) : x(a.x)
    {
        cout << "Move Constructor" << endl;
    }
    A& operator=(A&& a)
    {
        x = a.x;
        cout << "Move Assignment operator" << endl;
        return *this;
    }
};

A GetA()
{
    return A(1);
}

A&& MoveA(A& a)
{
    return std::move(a);
}

int main()
{
    cout << "-------------------------1-------------------------" << endl;
    A a(1);
    cout << "-------------------------2-------------------------" << endl;
    A b = a;
    cout << "-------------------------3-------------------------" << endl;
    A c(a);
    cout << "-------------------------4-------------------------" << endl;
    b = a;
    cout << "-------------------------5-------------------------" << endl;
    A d = A(1);
    cout << "-------------------------6-------------------------" << endl;
    A e = std::move(a);
    cout << "-------------------------7-------------------------" << endl;
    A f = GetA();
    cout << "-------------------------8-------------------------" << endl;
    A&& g = MoveA(f);
    cout << "-------------------------9-------------------------" << endl;
    d = A(1);
}
```

```c++
-------------------------1-------------------------
Constructor
-------------------------2-------------------------
Copy Constructor
-------------------------3-------------------------
Copy Constructor
-------------------------4-------------------------
Copy Assignment operator
-------------------------5-------------------------
Constructor
Move Constructor
-------------------------6-------------------------
Move Constructor
-------------------------7-------------------------
Constructor
Move Constructor
Move Constructor
-------------------------8-------------------------
-------------------------9-------------------------
Constructor
Move Assignment operator
```

第1行毋庸置疑，调用构造函数。
 第2行创建新对象b，使用a初始化b，因此调用拷贝构造函数。
 第3行创建新对象c，使用a初始化c，因此调用拷贝构造函数。
 第4行使用a的值更新对象b，因为不需要创建新对象，所以调用拷贝赋值运算符。
 第5行创建新对象d，使用临时对象A(1)初始化d，由于临时对象是一个右值，所以调用移动构造函数。
 第6行创建新对象e，使用a的值初始化e，但调用std::move(a)将左值a转化为右值，所以调用移动构造函数。
 第7行创建新对象f，使用GetA()函数返回的临时对象初始化f，由于临时对象是右值，所以调用移动构造函数。值得注意的是，这里调用了两次移动构造函数。第一次是GetA()返回前，A(1)移动构造了一个临时对象。第二次是临时对象移动构造f。
 第8行没有创建新对象，也不更新任何对象，只是将MoveA()的返回值绑定到右值引用g。因此不调用构造函数，也不调用赋值运算符。
 第9行使用临时对象A(1)更新d，因为不需要创建新对象，所以调用移动赋值运算



## 参考文献

[【极客时间《现代C++实战30讲》】](https://www.bookstack.cn/read/CPlusPlusThings/1f6e953ef3c2346b.md)

[【吐血整理C++11新特性】](https://www.kancloud.cn/wangshubo1989/new-characteristics/99704)
