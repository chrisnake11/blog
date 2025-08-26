---
title: shared_ptr和unique_ptr
published: 2025-08-26T16:30:25Z
description: '在写代码中，自己关于shared_ptr和unique_ptr的理解和使用场景'
image: ''
tags: []
category: ''
draft: false
---

# shared_ptr和unique_ptr

一般为了节省栈空间的内存大小，通常不会直接在类成员变量中直接存放结构较大的类。而是存储指向这些类的指针作为成员变量。

并且为了保证内存安全，通常使用智能指针（如shared_ptr和unique_ptr）来管理这些指针的生命周期。

## 场景1 在类中使用`shared_ptr`存储指针成员

在聊天服务器的用户管理或者消息管理模块中，通常需要存储大量的结构体`UserInfo, MessageInfo`等。为了节省内存空间需要使用智能指针来管理。一般使用`shared_ptr`而不是`unique_ptr`，因为`shared_ptr`可以在多个地方共享同一个对象的所有权，而`unique_ptr`则不支持这种共享。

```cpp
class UserManager{

public:
    UserManager();
    // 获取用户信息，共享给其他模块
    std::shared_ptr<UserInfo> getUserInfo(const int& uid);
private:
    // 使用shared_ptr存储用户信息，方便在多个地方共享
    // 错误：不要使用std::unique_ptr<UserInfo>
    std::unordered_map<int, std::shared_ptr<UserInfo>> users;
};

```

### 为什么不使用`unique_ptr`？

例如：当有其他模块需要访问这些用户信息或消息信息时，使用`shared_ptr`可以方便地将指针传递给其他模块，而不需要担心对象的生命周期问题。而`unique_ptr`则不适合这种场景，因为它只能有一个所有者，如果共享给其他成员，必须使用`std::move`将所有权转移，这会导致原始指针失效。

## 场景2 使用`unique_ptr`指向临时对象

在某些情况下，可能需要将一个对象的所有权临时转移给另一个函数或模块，这时可以使用`unique_ptr`。由于`unique_ptr`的所有权不可共享，因此可以确保在转移过程中不会出现悬空指针或重复释放的问题。

```cpp
void processMessage(std::unique_ptr<MessageInfo> msg) {
    // 处理消息
}

void handleMessage() {
    // 临时创建的消息，使用unique_ptr管理，可以在后续的过程中传递，确保唯一性。
    std::unique_ptr<MessageInfo> msg = std::make_unique<MessageInfo>();
    processMessage(std::move(msg));
    // msg在这里已经失效
}