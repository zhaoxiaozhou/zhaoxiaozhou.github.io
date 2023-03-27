---
title: 【c++多线程编程】多线程编程专栏二
excerpt: "C++11多线程编程竞争关系、互斥锁"
date: 2022-03-07 21:49:53
tags: ['C++', '多线程']
category: learning
---


## 1. 竞争条件

并发代码中最常见的错误之一就是**竞争条件(race condition)**。而其中最常见的就是**数据竞争(data race)**，从整体上来看，所有线程之间共享数据的问题，都是修改数据导致的，如果所有的共享数据都是只读的，就不会发生问题。但是这是不可能的，大部分共享数据都是要被修改的。

而`c++`中常见的`cout`就是一个共享资源，如果在多个线程同时执行`cout`，你会发发现很奇怪的问题：

```c++
#include <iostream>
#include <thread>
#include <string>
using namespace std;

// 普通函数 无参
void function_1() {
    for(int i=0; i>-100; i--)
        cout << "From t1: " << i << endl;
}

int main()
{
    std::thread t1(function_1);

    for(int i=0; i<100; i++)
        cout << "From main: " << i << endl;

    t1.join();
    return 0;
}
```

你有很大的几率发现打印会出现类似于`From t1: From main: 64`这样奇怪的打印结果。`cout`是基于流的，会先将你要打印的内容放入缓冲区，可能刚刚一个线程刚刚放入`From t1: `，另一个线程就执行了，导致输出变乱。而`c`语言中的`printf`不会发生这个问题

## 2. 使用互斥元来保护共享数据

在`c++`中，可以使用互斥锁`std::mutex`进行资源保护，头文件是`#include <mutex>`，共有两种操作：**锁定(lock)**与**解锁(unlock)**。将`cout`重新封装成一个线程安全的函数：

```c++
#include <iostream>
#include <thread>
#include <string>
#include <mutex>
using namespace std;

std::mutex mu;
// 使用锁保护
void shared_print(string msg, int id) {
    mu.lock(); // 上锁
    cout << msg << id << endl;
    mu.unlock(); // 解锁
}

void function_1() {
    for(int i=0; i>-100; i--)
        shared_print(string("From t1: "), i);
}

int main()
{
    std::thread t1(function_1);

    for(int i=0; i<100; i++)
        shared_print(string("From main: "), i);

    t1.join();
    return 0;
}
```

修改完之后，运行可以发现打印没有问题了。但是还有一个隐藏着的问题，如果`mu.lock()`和`mu.unlock()`之间的语句发生了异常，会发生什么？`unlock()`语句没有机会执行！导致导致`mu`一直处于锁着的状态，其他使用`shared_print()`函数的线程就会阻塞。

解决这个问题也很简单，使用`c++`中常见的`RAII`技术，即**获取资源即初始化(Resource Acquisition Is Initialization)**技术，这是`c++`中管理资源的常用方式。简单的说就是在类的构造函数中创建资源，在析构函数中释放资源，因为就算发生了异常，`c++`也能保证类的析构函数能够执行。我们不需要自己写个类包装`mutex`，`c++`库已经提供了`std::lock_guard`类模板，使用方法如下：

```c++
void shared_print(string msg, int id) {
    //构造的时候帮忙上锁，析构的时候释放锁
    std::lock_guard<std::mutex> guard(mu);
    //mu.lock(); // 上锁
    cout << msg << id << endl;
    //mu.unlock(); // 解锁
}
```

可以实现自己的`std::lock_guard`，类似这样：

```c++
class MutexLockGuard
{
 public:
  explicit MutexLockGuard(std::mutex& mutex)
    : mutex_(mutex)
  {
    mutex_.lock();
  }

  ~MutexLockGuard()
  {
    mutex_.unlock();
  }

 private:
  std::mutex& mutex_;
};
```

## 3. 为保护共享数据精心组织代码

上面的`std::mutex`互斥元是个全局变量，他是为`shared_print()`准备的，这个时候，我们最好将他们绑定在一起，比如说，可以封装成一个类。由于`cout`是个全局共享的变量，没法完全封装，就算你封装了，外面还是能够使用`cout`，并且不用通过锁。下面使用文件流举例：

```c++
#include <iostream>
#include <thread>
#include <string>
#include <mutex>
#include <fstream>
using namespace std;

std::mutex mu;
class LogFile {
    std::mutex m_mutex;
    ofstream f;
public:
    LogFile() {
        f.open("log.txt");
    }
    ~LogFile() {
        f.close();
    }
    void shared_print(string msg, int id) {
        std::lock_guard<std::mutex> guard(mu);
        f << msg << id << endl;
    }
};

void function_1(LogFile& log) {
    for(int i=0; i>-100; i--)
        log.shared_print(string("From t1: "), i);
}

int main()
{
    LogFile log;
    std::thread t1(function_1, std::ref(log));

    for(int i=0; i<100; i++)
        log.shared_print(string("From main: "), i);

    t1.join();
    return 0;
}
```

上面的`LogFile`类封装了一个`mutex`和一个`ofstream`对象，然后`shared_print`函数在`mutex`的保护下，是线程安全的。使用的时候，先定义一个`LogFile`的实例`log`，主线程中直接使用，子线程中通过引用传递过去（也可以使用单例来实现）,这样就能保证资源被互斥锁保护着，外面没办法使用但是使用资源。

但是这个时候还是得小心了！用互斥元保护数据并不只是像上面那样保护每个函数，就能够完全的保证线程安全，如果将资源的指针或者引用不小心传递出来了，所有的保护都白费了！要记住一下两点：

1. 不要提供函数让用户获取资源

   ```c++
   std::mutex mu;
   class LogFile {
       std::mutex m_mutex;
       ofstream f;
   public:
       LogFile() {
           f.open("log.txt");
       }
       ~LogFile() {
           f.close();
       }
       void shared_print(string msg, int id) {
           std::lock_guard<std::mutex> guard(mu);
           f << msg << id << endl;
       }
       // Never return f to the outside world
       ofstream& getStream() {
           return f;  //never do this !!!
       }
   };
   ```

2. 不要资源传递给用户的函数

   ```c++
   class LogFile {
       std::mutex m_mutex;
       ofstream f;
   public:
       LogFile() {
           f.open("log.txt");
       }
       ~LogFile() {
           f.close();
       }
       void shared_print(string msg, int id) {
           std::lock_guard<std::mutex> guard(mu);
           f << msg << id << endl;
       }
       // Never return f to the outside world
       ofstream& getStream() {
           return f;  //never do this !!!
       }
       // Never pass f as an argument to user provided function
       void process(void fun(ostream&)) {
           fun(f);
       }
   };
   ```

   以上两种做法都会将资源暴露给用户，造成不必要的安全隐患

   ## 4. 接口设计中也存在竞争条件

   `STL`中的`stack`类是线程不安全的，于是你模仿着想写一个属于自己的线程安全的类`Stack`。于是，你在`push`和`pop`等操作得时候，加了互斥锁保护数据。但是在多线程环境下使用使用你的`Stack`类的时候，却仍然有可能是线程不安全的，why？

   假设你的`Stack`类的接口如下：

   ```c++
   class Stack
   {
   public:
       Stack() {}
       void pop(); //弹出栈顶元素
       int& top(); //获取栈顶元素
       void push(int x);//将元素放入栈
   private:
       vector<int> data; 
       std::mutex _mu; //保护内部数据
   };
   ```

   类中的每一个函数都是线程安全的，但是**组合起来却不是**。加入栈中有`9,3,8,6`共4个元素，你想使用两个线程分别取出栈中的元素进行处理，如下所示：

   ```c++
      Thread A                Thread B
   int v = st.top(); // 6
                         int v = st.top(); // 6
   st.pop(); //弹出6
                         st.pop(); //弹出8
                         process(v);//处理6
   process(v); //处理6
   ```

   可以发现在这种执行顺序下， 栈顶元素被处理了两遍，而且多弹出了一个元素`8`，导致`8没有被处理！这就是由于接口设计不当引起的竞争。解决办法就是将这两个接口合并为一个接口！就可以得到线程安全的栈。

   ```c++
   class Stack
   {
   public:
       Stack() {}
       int& pop(); //弹出栈顶元素并返回
       void push(int x);//将元素放入栈
   private:
       vector<int> data; 
       std::mutex _mu; //保护内部数据
   };
   
   //下面这样使用就不会发生问题
   int v = st.pop(); // 6
   process(v);
   ```

   但是注意：这样修改之后**是线程安全**的，但是**并不是异常安全**的，这也是为什么`STL`中栈的出栈操作分解成了两个步骤的原因。（为什么不是异常安全的还没想明白。。）

   所以，为了保护共享数据，还得好好设计接口才行。

   ## 参考文献

   [【C++并发编程实战】](https://link.segmentfault.com/?enc=s5BJZJv05EaPcblnrKXNaQ%3D%3D.ASihcN%2BCsh8vW2GbIeZKpDlZDNpSv8ipr%2BhvOyd1NPVAFC3Ts0N%2B5lZkOsqvHcSV)
   
   [【C++ Threading #3: Data Race and Mutex】](https://link.segmentfault.com/?enc=j%2FiVWm9hZ5%2Fo6P3%2F%2FKxbeA%3D%3D.6VcvndvV2QVpE7Jnc3AXcfQvKCjbJe9Lur6Furr0bM08IkjBibSH8BnEaQJoaQgL1nzKnvMLOC2DPuXKyctZ8Dmv4LK24fNvJnQC%2Btu9%2FyFad9xMnzrPJlzl32ffeJJ%2FnuC811SlwBt6FqKiw1fnAw%3D%3D)
   
   [【多线程编程(三)——竞争条件与互斥锁】](https://segmentfault.com/a/1190000016201729)
