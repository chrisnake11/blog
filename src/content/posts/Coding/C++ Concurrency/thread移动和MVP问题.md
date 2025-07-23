---
title: thread移动和MVP问题
published: 2025-07-23T14:34:54Z
description: 'std::thread所属权的移动，以及C++中最烦人的解析(Most Vexing Parse)问题。构造函数中右值引用参数的使用。'
image: ''
tags: [C++, Concurrency, Thread]
category: 'C++'
draft: false
---

# thread移动和MVP问题

在阅读[《C++ Concurrency in Action》](https://www.amazon.com/C-Concurrency-Action-Anthony-Williams/dp/1617294691) 2.3 移交线程归属权时，发现了一个有趣的问题。

源代码如下:
```cpp
// test.cpp : 此文件包含 "main" 函数。程序执行将在此处开始并结束。  
//  

#include <iostream>  
#include <thread>  

class scoped_thread  
{  
   std::thread t;  
public:
   // 值传递
   explicit scoped_thread(std::thread t_) :
      t(std::move(t_)) // 使用std::move将线程对象转移到成员变量  
   {
	  std::cout << "scoped_thread constructor called, moving thread." << std::endl;
      if (!t.joinable())
          throw std::logic_error("No thread");
   }
//    // 问题1：为什么不用右值引用参数
//    scoped_thread(std::thread&& t_) :  
//        t(std::move(t_)) // 使用std::move将线程对象转移到成员变量  
//    {  
// 	   std::cout << "scoped_thread right value constructor called, moving thread." << std::endl;
//        if (!t.joinable())  
//            throw std::logic_error("No thread");  
//    }
   ~scoped_thread()  
   {  
       std::cout << "scoped_thread destructor called, joining thread." << std::endl;  
       t.join();  
   }  
   scoped_thread(scoped_thread const&) = delete;  
   scoped_thread& operator=(scoped_thread const&) = delete;  
};  

void do_something() {  
   // 模拟一些工作  
   std::this_thread::sleep_for(std::chrono::seconds(1));  
   std::cout << "Doing something in thread " << std::endl;  
}  

int main()  
{  
    std::thread t2(do_something);
    // 问题2：Most Vexing Parse
    // scoped_thread st1( std::thread(do_something) ); // Most Vexing Parse问题，无法传递临时对象 
    scoped_thread st1{ std::thread(do_something) }; // 使用scoped_thread管理线程  
    scoped_thread st2(std::move(t2)); // 使用scoped_thread管理线程  
}

```

代码中，我们定义了一个实现RAII机制的`scoped_thread`类，用于管理线程的生命周期。它接收一个`std::thread`对象，并在析构时调用`join()`方法。

## 问题1：为什么不用右值引用参数

在构造函数中，我们使用了一个值传递的参数`std::thread t_`，并在构造函数体内使用`std::move(t_)`将其转移到成员变量`t`中。

在参数的值传递过程中，`std::thread t`会被调用默认的移动构造函数，这样就可以将线程对象的所有权从临时对象转移到`scoped_thread`实例中，随后调用`std::move(t_)`将其转移到成员变量`t`中。总计2次移动。

如果我们使用右值引用参数`std::thread&& t_`，在右值引用参数传递时不会调用移动构造函数，直接在构造函数中直接使用`std::move(t_)`将其转移到成员变量`t`中，这样只需要1次移动。

但是实际代码实践中，使用值传递的方式更为常见和安全，因为它可以避免一些潜在的错误，性能损失可以忽略。并且常见的C++库和教材中也多采用这种方式。

## 问题2：Most Vexing Parse

`scoped_thread st1( std::thread(do_something) );`这行代码，会被编译器解析为一个函数的声明(Declaration)，而不是一个对象的定义(Definition)。这是因为C++的解析规则会优先考虑函数声明。

> 即：当编译器看到一个括号内的表达式时，会尽量将其解析为声明，而不是定义。
> 

[WikiPedia]中的来源解释： 
> The term "most vexing parse" was first used by Scott Meyers in his 2001 book Effective STL.

### 解决方法

为了解决这个问题，我们可以使用统一初始化(Uniform Initialization)的方式，使用大括号 `{}` 来创建`scoped_thread`对象，例如：
```cpp
scoped_thread st1{ std::thread(do_something) };
```
这样就可以避免Most Vexing Parse的问题。

> 或者，在内部对象使用统一初始化`{}`，外部仍然使用小括号`()`。
> ```
> scoped_thread st1( std::thread> {do_something} );
> ```

### 参考资料
- [C++ Concurrency in Action 2nd Edition](https://www.amazon.com/C-Concurrency-Action-2nd-Anthony-Williams/dp/1617294691)
- [Most Vexing Parse](https://en.wikipedia.org/wiki/Most_vexing_parse)
- [What is the purpose of the Most Vexing Parse?](https://stackoverflow.com/questions/14077608/what-is-the-purpose-of-the-most-vexing-parse)