---
title: CSession
published: 2025-07-23T17:02:46Z
description: 'ChatRoom会话处理类，负责处理客户端连接和消息交互。'
image: ''
tags: [C++, ChatRoom, 网络编程]
category: 'ChatRoom'
draft: false
---

# CSession

在ChatRoom中，`CSession`类是一个重要的组件，负责处理客户端连接和消息交互。它通常包含以下功能：
- **消息处理**：接收和发送消息。
- **连接管理**：处理客户端的连接和断开连接事件。
- **状态管理**：维护会话的状态信息，例如用户ID、会话ID等。

CSession会在服务器端创建一个会话对象`(io_context, socket)`来代表每个连接的客户端。存储在服务器`CServer`类中，通常会有一个会话列表`std::map<std::string _uuid, std::shared_ptr<CSession>>`来管理所有活跃的会话。

## CSession类定义

```cpp
#pragma once
#include <boost/asio.hpp>
#include <boost/uuid/uuid.hpp>
#include <boost/uuid/uuid_generators.hpp>
#include <boost/uuid/uuid_io.hpp>
#include <queue>
#include "Const.h"
#include "MsgNode.h"
#include "CServer.h"

/*
CSession 类用于表示一个网络会话，封装了与客户端的连接、数据接收和发送等功能。

功能解析：
1. 构造函数和析构函数：
   - CSession()：初始化会话，设置默认值。
   - ~CSession()：清理资源，关闭连接。
2. 获取套接字和UUID：
   - getSocket()：返回与客户端的套接字引用。
   - getUuid()：返回会话的唯一标识符。
3. 会话管理：
   - start()：开始会话，通常用于启动异步操作。
   - close()：关闭会话，释放资源。
4. 异步操作：
   - asyncReadHead()：异步读取数据包头部，指定总长度。
   - asyncReadBody()：异步读取数据包体，指定总长度。
   - send()：发送数据到客户端。
5. 数据处理：
   - asyncReadFull()：异步读取完整数据包。
   - asyncReadLen()：异步读取指定长度的数据。

*/

class CSession : public std::enable_shared_from_this<CSession>{
public:
	CSession(boost::asio::io_context& io_context, CServer* server);
	~CSession();
	boost::asio::ip::tcp::socket& getSocket();
	std::string& getUuid();
	void start();
	void close();
	// 异步解析头部
	void asyncReadHead(int total_length);
	// 解析并处理数据包体
	void asyncReadBody(int total_length);
	// send data
	void send(const char* msg, short max_length, short msg_id);
	void send(const std::string& msg, short msg_id);
	// 支持右值引用，临时字符串对象传递。
	void send(std::string&& msg, short msg_id);

private:
	// 解析完整的数据包(head + body)
	// max_length表示头部长度
	// handler为回调函数
	void asyncReadFull(std::size_t max_length, std::function<void(const boost::system::error_code& error, std::size_t)> handler);
	// 封装async_read_some异步读取函数
	// 读取指定长度的数据，read_length为已处理的数据，total_length为包的总长度。
	void asyncReadLen(std::size_t read_length, std::size_t total_length, std::function<void(const boost::system::error_code& error, std::size_t)> handler);

	void handleWrite(const boost::system::error_code& error, std::shared_ptr<CSession> self_shared);

	// server
	boost::asio::ip::tcp::socket _socket;
	CServer* _server;
	bool _b_close;

	// session uid
	std::string _uuid;

	// data, 单个消息的缓冲区
	char _data[MAX_LENGTH];

	// send data
	std::queue<std::shared_ptr<SendNode>> _send_queue;
	std::mutex _send_lock;
	

	bool _b_head_parse;
	// head data node
	std::shared_ptr<MsgNode> _recv_head_node;
	// msg data node
	std::shared_ptr<RecvNode> _recv_msg_node;
	
};

class LogicNode {
	friend class LogicSystem;
public:
	LogicNode(std::shared_ptr<CSession>, std::shared_ptr<RecvNode>);
private:
	std::shared_ptr<CSession> _session;
	std::shared_ptr<RecvNode> _recv_node;
};
```

## 异步消息处理

### 消息接收

`CSession`类使用Boost.Asio库来处理异步IO操作。它通过`asyncReadHead()`和`asyncReadBody()`方法来异步读取数据包的头部和体。

`asyncReadHead()`和`asyncReadBody()`方法会调用`asyncReadFull()`来读取指定长度的数据。这样可以确保在处理大数据包时不会阻塞IO线程。

`asyncReadFull()`方法会进一步调用`asyncReadLen()`来异步读取数据，`asyncReadLen()`会调用`asio`中的`async_read_some()`函数，异步的读取指定长度的数据到缓冲区`_data`，并在读取完成后调用指定的回调函数。

在回调函数中，会解析缓冲区`_data`中的数据包头部和体的数据，并将其存储在`_recv_head_node`和`_recv_msg_node`中。

随后`_recv_msg_node`会被传递给`LogicSystem`，由逻辑系统处理数据。


#### 代码实现

```cpp
#include "CSession.h"
#include <iostream>
#include "LogicSystem.h"

void CSession::asyncReadFull(std::size_t max_length, std::function<void(const boost::system::error_code& error, std::size_t)> handler)
{
	::memset(_data, 0, max_length);
	asyncReadLen(0, max_length, handler);
}

void CSession::asyncReadLen(std::size_t read_length, std::size_t total_length, std::function<void(const boost::system::error_code& error, std::size_t)> handler) {
	auto self = shared_from_this();
	_socket.async_read_some(boost::asio::buffer(_data + read_length, total_length - read_length), 
		[self, read_length, total_length, handler](const boost::system::error_code& error, std::size_t bytes_transfered) {
			// 出错，错误交给回调函数处理。
			if (error) {
				handler(error, read_length + bytes_transfered);
				return;
			}
			// 读取到的长度足够了，回调函数
			if (read_length + bytes_transfered >= total_length) {
				handler(error, read_length + bytes_transfered);
				return;
			}

			self->asyncReadLen(read_length + bytes_transfered, total_length, handler);
				
			
		});
}

void CSession::asyncReadHead(int total_len)
{
	auto self = shared_from_this();
	asyncReadFull(HEAD_TOTAL_LENGTH, [self, this](const boost::system::error_code& error, std::size_t bytes_transfered) {
		try {
			if (error) {
				std::cout << "handle read failed, error is: " << error.what() << std::endl;
				close();
				_server->clearSession(_uuid);
				return;
			}

			if (bytes_transfered < HEAD_TOTAL_LENGTH) {
				std::cout << "read head length not match, read [" << bytes_transfered << "] , total[" << HEAD_TOTAL_LENGTH << "] " << std::endl;
				close();
				_server->clearSession(_uuid);
				return;
			}

			_recv_head_node->clear();
			memcpy(_recv_head_node->_data, _data, bytes_transfered);

			// get MSGID
			short msg_id = 0;
			memcpy(&msg_id, _recv_head_node->_data, HEAD_ID_LENGTH);
			msg_id = boost::asio::detail::socket_ops::network_to_host_short(msg_id);
			std::cout << "msg id is " << msg_id;
			
			// msg_id 非法
			if (msg_id > MAX_LENGTH) {
				std::cout << "msg_id too long" << std::endl;
				_server->clearSession(_uuid);
				return;
			}

			short msg_len = 0;
			memcpy(&msg_len, _recv_head_node->_data + HEAD_ID_LENGTH, HEAD_DATA_LENGTH);
			msg_len = boost::asio::detail::socket_ops::network_to_host_short(msg_len);
			std::cout << "msg_len is" << msg_len << std::endl;
			if (msg_len > MAX_LENGTH) {
				std::cout << "msg_len too long" << std::endl;
				_server->clearSession(_uuid);
				return;
			}

			// 构造RecvNode，开始读取消息体
			_recv_msg_node = std::make_shared<RecvNode>(msg_len, msg_id);
			asyncReadBody(msg_len);

		}
		catch (const std::exception& e) {
			// Handle exception
			std::cerr << "Exception in asyncReadHead: " << e.what() << std::endl;
		}
		});
}

void CSession::asyncReadBody(int total_length) {
	auto self = shared_from_this();
	asyncReadFull(total_length, [self, this, total_length](const boost::system::error_code& error, std::size_t bytes_transfered) {
		try {
			if (error) {
				std::cout << "handle read failed, error is " << error.what() << std::endl;
				close();
				_server->clearSession(_uuid);
				return;
			}
			if (bytes_transfered < total_length) {
				std::cout << "read length not match read[" << bytes_transfered << "], total[" << total_length << "]" << std::endl;
				close();
				_server->clearSession(_uuid);
				return;
			}

			_recv_msg_node->clear();
			memcpy(_recv_msg_node->_data, _data + HEAD_TOTAL_LENGTH, bytes_transfered);
			_recv_msg_node->_cur_length += bytes_transfered;
			_recv_msg_node->_data[_recv_msg_node->_total_length] = '\0';
			std::cout << "recv_msg_data is" << _recv_msg_node->_data << std::endl;

			// 构造LogicNode，放入消息队列
			LogicSystem::getInstance()->PostMsgToQueue(std::make_shared<LogicNode>(shared_from_this(), _recv_msg_node));

            // 当前消息处理完成，继续读取下一个消息头
			asyncReadHead(HEAD_TOTAL_LENGTH);

		}
		catch (std::exception& e) {
			std::cerr << "Exception code is" << e.what() << std::endl;
		}
		
	});
}

LogicNode::LogicNode(std::shared_ptr<CSession> session, std::shared_ptr<RecvNode> recv_node): _session(session), _recv_node(recv_node)
{

}
```


### 消息发送


当LogicSystem处理完数据后，会调用`CSession::send()`方法来发送数据。该方法会将待发送的消息放入一个队列中，并使用异步IO操作来发送数据。

这发送队列`_send_queue`是一个线程安全的队列，使用`std::mutex`来保护对队列的访问。在CSession和LogicSystem之间构建了一个生产者-消费者模型，CSession负责发送数据，LogicSystem负责产生数据。

每次发送数据时，CSession会检查发送队列是否为空，如果队列只有1个消息，则从队列中取出一个消息并发送。如果大于1个消息，则将所有消息放入队列中，等待发送。这样可以确保始终有且只有一个消息在执行发送操作。

数据发送通过`boost::asio::async_write()`方法实现。该方法会将数据异步写入到套接字中，并在写入完成后调用指定的回调函数。

在回调函数中重复检查发送队列是否为空，如果不为空，则再次调用`boost::asio::async_write()`来发送下一个消息。

#### 代码实现

```cpp
void CSession::handleWrite(const boost::system::error_code& error, std::shared_ptr<CSession> self_shared)
{
	try {
		if(error){
			std::cout << "handle write failed, error is " << error.what() << std::endl;
			close();
			_server->clearSession(_uuid);
		}
		std::lock_guard<std::mutex> lock(_send_lock);
		_send_queue.pop();
		if (!_send_queue.empty()) {
			auto& send_node = _send_queue.front();
			boost::asio::async_write(_socket, boost::asio::buffer(send_node->_data, send_node->_total_length), std::bind(&CSession::handleWrite, this, std::placeholders::_1, self_shared));
		}
	}
	catch (std::exception& e) {
		std::cout << "Exception code: " << e.what() << std::endl;
	}
}

void CSession::send(const char* msg, short max_length, short msg_id) {
	std::lock_guard<std::mutex> lock(_send_lock);
	int send_queue_size = _send_queue.size();
	if (_send_queue.size() >= MAX_SEND_QUEUE) {
		std::cout << "session:" << _uuid << ", send queue is full" << std::endl;
		return;
	}
	_send_queue.push(std::make_shared<SendNode>(msg, max_length, msg_id));
	if (_send_queue.size() > 1) {
		return;
	}
	auto& send_node = _send_queue.front();
	boost::asio::async_write(_socket, boost::asio::buffer(send_node->_data, send_node->_total_length), 
		std::bind(&CSession::handleWrite, this, std::placeholders::_1, shared_from_this()));

}

void CSession::send(const std::string& msg, short msg_id) {
	// 使用 c_str() 转换为 const char*
	send(msg.c_str(), static_cast<short>(msg.length()), msg_id);
}

// 支持移动语义的重载（C++11及以上）
void CSession::send(std::string&& msg, short msg_id) {
	send(msg.c_str(), static_cast<short>(msg.length()), msg_id);
}

```

# MessageNode的实现

```cpp
#pragma once
#include "Const.h"
class MsgNode{
public:
	MsgNode(int max_len): _total_length(max_len), _cur_length(0) {
		_data = new char[_total_length + 1]();
		_data[_total_length] = '\0';
	}
	void clear();
	char* _data;
	int _total_length;
	int _cur_length;
};

class RecvNode : public MsgNode{
public:
	RecvNode(int total_length, int msg_id) : MsgNode(total_length), _msg_id(msg_id){
	
	}
private:
	int _msg_id;
};

class SendNode : public MsgNode {
public:
	SendNode(const char* msg, int max_length, int msg_id):
		MsgNode(max_length + HEAD_TOTAL_LENGTH), _msg_id(msg_id){
		// 发送前将头部数据转化为大端序。
		short host_msg_id = boost::asio::detail::socket_ops::host_to_network_short(_msg_id);
		memcpy(_data, &host_msg_id, HEAD_ID_LENGTH);
		short host_max_length = boost::asio::detail::socket_ops::host_to_network_short(_total_length);
		memcpy(_data + HEAD_ID_LENGTH, &host_max_length, HEAD_DATA_LENGTH);
		memcpy(_data + HEAD_TOTAL_LENGTH, msg, max_length);
	}
private:
	int _msg_id;
};
```