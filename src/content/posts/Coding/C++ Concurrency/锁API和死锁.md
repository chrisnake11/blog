---
title: 锁API和死锁
published: 2025-08-01T19:44:18Z
description: 'C++中锁的API使用，以及死锁的产生和避免的原则方法。'
image: ''
tags: [C++, Concurrency, Lock, 读书笔记]
category: 'C++'
draft: false
---

# 锁API和死锁

# 锁相关的API

C++中锁相关的API主要包括以下几种（括号内为引入的最低C++标准）：
- `std::mutex`（C++11）：互斥量，提供互斥访问机制来保护共享资源。
- `std::lock_guard()`（C++11）：RAII风格的锁，自动管理锁的获取和释放。
- `std::lock()`（C++11）：用于同时锁定多个互斥量，避免死锁。
- `std::unique_lock()`（C++11）：`std::mutex`的封装，更灵活的锁管理，可以手动控制锁的释放。
- `std::scoped_lock()`（C++17）：用于多个互斥量的锁定，确保在同一作用域内锁的获取和释放。

## 示例

### 1. `std::mutex` 基本使用
```cpp
#include <iostream>
#include <thread>
#include <mutex>

std::mutex mtx;
int shared_data = 0;

void increment() {
    mtx.lock();           // 手动加锁
    ++shared_data;
    std::cout << "Data: " << shared_data << std::endl;
    mtx.unlock();         // 手动解锁
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();
    return 0;
}
```

### 2. `std::lock_guard` RAII风格锁管理
```cpp
#include <iostream>
#include <thread>
#include <mutex>

std::mutex mtx;
int shared_data = 0;

void safe_increment() {
    std::lock_guard<std::mutex> lock(mtx);  // 构造时自动加锁
    ++shared_data;
    std::cout << "Data: " << shared_data << std::endl;
    // 作用域结束时自动解锁
}

int main() {
    std::thread t1(safe_increment);
    std::thread t2(safe_increment);
    t1.join();
    t2.join();
    return 0;
}
```

### 3. `std::unique_lock` 灵活锁管理

unique_lock提供了更灵活的锁管理，可以手动解锁和重新加锁，或者在需要时延迟锁的获取。

当多个unique_lock绑定同一个互斥量时，可以使用`std::adopt_lock`来表示该互斥量已经被锁定。如果有其他的`unique_lock`对这个互斥量进行锁定，则会抛出异常。


> unique_lock相比于lock_guard提供了更多的灵活性，但是由于它需要更多的资源来管理锁状态，因此性能略低于lock_guard。

```cpp
#include <iostream>
#include <thread>
#include <mutex>

std::mutex mtx;
int shared_data = 0;

void flexible_lock() {
    std::unique_lock<std::mutex> lock(mtx);
    ++shared_data;
    
    // 可以手动解锁
    lock.unlock();
    
    // 做一些不需要锁保护的工作
    std::this_thread::sleep_for(std::chrono::milliseconds(10));
    
    // 重新加锁
    lock.lock();
    std::cout << "Data: " << shared_data << std::endl;
    // 析构时自动解锁
}
```

### 4. `std::lock` 同时锁定多个互斥量
```cpp
#include <iostream>
#include <thread>
#include <mutex>

std::mutex mtx1, mtx2;
int data1 = 0, data2 = 0;

void safe_swap() {
    // 同时锁定两个互斥量，避免死锁
    std::lock(mtx1, mtx2);
    
    // 使用lock_guard接管已锁定的互斥量
    std::lock_guard<std::mutex> lock1(mtx1, std::adopt_lock);
    std::lock_guard<std::mutex> lock2(mtx2, std::adopt_lock);
    
    std::swap(data1, data2);
    std::cout << "Swapped: data1=" << data1 << ", data2=" << data2 << std::endl;
}
```


### 补充：`std::adopt_lock` 详细示例

`std::adopt_lock` 是一个标签类型，表示当前锁管理类（如 `std::lock_guard` 或 `std::unique_lock`）接管了互斥量，这样就可以转移互斥量对应的锁的所有权。但是只仅限于同一个线程中。

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <chrono>

std::mutex mtx1, mtx2;
int shared_resource1 = 0, shared_resource2 = 0;

void adopt_lock_example() {
    // 手动加锁
    mtx1.lock();
    
    try {
        // 使用adopt_lock告诉lock_guard互斥量已经被锁定
        std::lock_guard<std::mutex> guard(mtx1, std::adopt_lock);
        
        shared_resource1++;
        std::cout << "Resource1: " << shared_resource1 << std::endl;
        
        // 可能抛出异常的操作
        if (shared_resource1 > 5) {
            throw std::runtime_error("Error occurred!");
        }
        
    } catch (const std::exception& e) {
        std::cout << "Exception caught: " << e.what() << std::endl;
        // lock_guard析构时会自动解锁，即使发生异常
    }
    // 注意：如果没有使用adopt_lock，mtx1会被重复加锁导致未定义行为
}

// 配合std::lock使用adopt_lock的完整示例
void transfer_money(int& from_account, int& to_account, 
                   std::mutex& from_mtx, std::mutex& to_mtx, 
                   int amount) {
    // 同时锁定两个互斥量，避免死锁
    std::lock(from_mtx, to_mtx);
    
    // 使用adopt_lock接管已锁定的互斥量
    std::lock_guard<std::mutex> lock1(from_mtx, std::adopt_lock);
    std::lock_guard<std::mutex> lock2(to_mtx, std::adopt_lock);
    
    if (from_account >= amount) {
        from_account -= amount;
        to_account += amount;
        std::cout << "Transfer successful: " << amount << " units" << std::endl;
    } else {
        std::cout << "Insufficient funds" << std::endl;
    }
    // lock_guard析构时自动解锁所有互斥量
}

// unique_lock与adopt_lock的使用
void unique_lock_adopt_example() {
    mtx2.lock();  // 手动加锁
    
    // 使用unique_lock接管已锁定的互斥量
    std::unique_lock<std::mutex> ulock(mtx2, std::adopt_lock);
    
    shared_resource2++;
    std::cout << "Resource2: " << shared_resource2 << std::endl;
    
    // unique_lock可以手动解锁和重新加锁
    ulock.unlock();
    
    // 做一些不需要锁保护的工作
    std::this_thread::sleep_for(std::chrono::milliseconds(10));
    
    ulock.lock();  // 重新加锁
    shared_resource2++;
    std::cout << "Resource2 after relock: " << shared_resource2 << std::endl;
}
```

**`std::adopt_lock` 的关键点：**
- 用于告诉锁管理类互斥量已经被锁定
- 避免重复加锁导致的未定义行为
- 常与 `std::lock()` 函数配合使用
- 锁管理类析构时仍会正常解锁

### 5. `std::scoped_lock` (C++17) 多互斥量锁定

`std::scoped_lock`是C++17引入的一个新特性，用于同时锁定多个互斥量，确保在同一作用域内锁的获取和释放。可以完美地替换多互斥量情况下的`std::lock`，简化代码。

```cpp
#include <iostream>
#include <thread>
#include <mutex>

std::mutex mtx1, mtx2;
int data1 = 0, data2 = 0;

void scoped_safe_swap() {
    // C++17风格，更简洁的多互斥量锁定
    std::scoped_lock lock(mtx1, mtx2);
    
    std::swap(data1, data2);
    std::cout << "Swapped: data1=" << data1 << ", data2=" << data2 << std::endl;
    // 作用域结束时自动解锁所有互斥量
}
```

# 死锁

死锁是指两个或多个线程在执行过程中，因为争夺资源而造成的一种互相等待的现象。每个线程都在等待其他线程释放资源，导致所有线程都无法继续执行。


## 死锁发生的必要条件

1. 互斥：资源不能被共享，必须独占。
2. 占有且等待：线程已持有资源，并请求其他资源。
3. 不抢占：已分配的资源不能被强制剥夺，只能在使用完成后自行释放。
4. 循环等待：存在一个线程等待循环链，链中的每个线程都在等待下一个线程持有的资源。


## 为避免死锁，可以遵循以下原则

1. **按顺序加锁**：确保线程的代码中按照相同的顺序获取锁，避免循环等待。
2. **在持有锁时减少工作**：尽量减少在持有锁时的工作量，快速释放锁。
3. **在持有锁时尽量少调用其他函数**：避免在持有锁的情况下调用可能导致其他锁的获取。
---

## 额外的方法来避免死锁

4. **使用超时机制**：在尝试获取锁时设置超时时间，如果超过时间未获取到锁，则放弃尝试。
5. **使用死锁检测机制**：定期检查系统中是否存在死锁，并采取措施恢复。