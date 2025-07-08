---
title: SimpleConnPool
published: 2025-07-08T19:11:49Z
description: '简单的连接池实现，支持多线程的安全连接获取和释放。'
image: ''
tags: [Network Programming, C++]
category: 'C++'
draft: false
---

# SimpleConnPool

连接池是网络编程中常用的设计模式，用于**管理和复用连接资源**。

例如：当需要**频繁创建和销毁连接时，使用连接池可以显著提高性能和资源利用率**。连接池通过维护一个连接的队列或者集合来实现高效的连接管理。

连接池的基本功能包括：
- **连接获取`getConnection()`**：从池中获取一个可用的连接。
- **连接释放`returnConnection(conn)`**：将连接返回到池中以供后续使用。
- **多线程安全(多生产者多消费者)**：在多线程环境下，确保连接的获取和释放操作是线程安全的。
- 

以下是一个简单的连接池模板实现，通过C++模板类来支持不同类型的连接。在多线程环境下，使用**互斥锁和条件变量**，实现一个**多生产者多消费者模型**确保线程安全。
> 每个连接类需要实现一个`close()`方法来关闭连接。

```cpp
#pragma once

#include <mutex>
#include <queue>
#include <condition_variable>
#include <memory>

template <typename T>
class SimpleConnectionPool {

public:
	SimpleConnectionPool(int max_connections);

	~SimpleConnectionPool();

	std::shared_ptr<T> getConnection();
	void returnConnection(std::shared_ptr<T> conn);
	void close();

private:
	bool _b_stop; // 连接池状态
	std::queue<std::shared_ptr<T>> _queue;
	std::mutex _mute;
	std::condition_variable _cond;
	int _max_connections;
	std::shared_ptr<T> createConnection();

};

```

对应的实现部分如下：

```cpp

template <typename T>
SimpleConnectionPool<T>::SimpleConnectionPool(int max_connections) : _b_stop(false), _max_connections(max_connections) {
	for (int i = 0; i < max_connections; i++) {
		std::shared_ptr<T> conn = createConnection();
		if (conn) {
			_queue.push(createConnection());
		}
		// 处理创建失败
		else {
			std::cout << "create connection failed" << std::endl;
			break;
		}
	}
}

// 自定义创建连接
template <typename T>
std::shared_ptr<T> SimpleConnectionPool<T>::createConnection() {
	// 这里可以根据具体的连接类型来创建连接
    // 例如：return std::make_shared<T>(args...);
    return nullptr;
}

template <typename T>
void SimpleConnectionPool<T>::close() {
	std::unique_lock<std::mutex> lock(_mutex);
	_b_stop = true; // 设置连接池状态为停止
	// clear the queue
	while (!_queue.empty()) {
		std::shared_ptr<T> conn = _queue.front();
		_queue.pop();
		conn->close(); // 如果连接close()阻塞，整个线程都会被阻塞，可以优化。
	}
}

template <typename T>
SimpleConnectionPool<T>::~SimpleConnectionPool() {
	close();
}
 
template <typename T>
std::shared_ptr<T> SimpleConnectionPool<T>::getConnection() {
	std::unique_lock<std::mutex> lock(_mutex);
	_cond.wait(lock, [this] {
		return _b_stop || !_queue.empty();
	});
	if(_b_stop){
		return nullptr; // 如果连接池已关闭，返回空指针
	}
	std::shared_ptr<T> conn = _queue.front();
    _queue.pop();
    return conn;
}

template <typename T>
void SimpleConnectionPool<T>::returnConnection(std::shared_ptr<T> conn) {
	std::unique_lock<std::mutex> lock(_mutex);
	if(!_b_stop){
		_queue.push(conn);
		lock.unlock();
		_cond.notify_one();
	}
	else{
		// 如果连接池已关闭，直接关闭连接
		conn->close();
	}
	
}
```


