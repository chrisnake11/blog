---
title: LogicSystem
published: 2025-07-23T16:13:39Z
description: ''
image: ''
tags: [C++, ChatRoom, 网络编程]
category: 'ChatRoom'
draft: false
---

# LogicSystem

LogicSystem 是一个用于处理服务器消息的单线程的逻辑系统。它负责CSession传递的消息，并将其分发到相应的处理函数。

## 主要功能

+ 维护一个消息处理函数的映射表
+ 将消息分发到对应的处理函数
+ 提供消息队列的消息的入口
+ 单线程地处理消息列表中的消息
+ 支持多种消息类型的处理
+ 调用CSession中的Send函数来发送消息

## 代码实现
```cpp
#pragma once
#include "Const.h"
#include "Singleton.h"
#include <map>
#include <functional>
#include <mutex>
#include <queue>
#include <unordered_map>
class CSession;
class LogicNode;

typedef std::function<void(std::shared_ptr<CSession>, const short& msg_id, const std::string& msg_data)> node_callback;

class LogicSystem : public Singleton<LogicSystem>{
	friend class Singleton<LogicSystem>;
public:
	~LogicSystem();
	void postMsgToQueue(std::shared_ptr<LogicNode> msg_node);
private:
	LogicSystem();
	void registerHanlder();
	void dealMsg();

	// 登录逻辑处理
	void loginHandler(std::shared_ptr<CSession> session, const short& msg_id, const std::string& msg_data);

	// 
	bool _b_stop;

	// 单线程处理消息节点。
	std::mutex _node_mutex;
	std::queue<std::shared_ptr<LogicNode>> _node_queue;
	std::thread _work_thread;
	std::condition_variable _consume;

	// 不同消息的逻辑函数列表
	std::map <short, node_callback> _handlers;

	// 待处理的用户列表
	//std::unordered_map<int, std::shared_ptr<UserInfo>> _users;
};
```

## 代码实现
```cpp
#include "LogicSystem.h"
#include "CSession.h"

LogicSystem::LogicSystem() : _b_stop(false) {
	registerHanlder();
	_work_thread = std::thread(&LogicSystem::dealMsg, this);

}

LogicSystem::~LogicSystem() {
	std::cout << "LogicSystem destroy~" << std::endl;
	_b_stop = true;
	_consume.notify_one();
	_work_thread.join();
}

void LogicSystem::dealMsg() {
	std::shared_ptr<LogicNode> node;
	while (true) {
		std::unique_lock<std::mutex> lock(_node_mutex);
		while (_node_queue.empty() || !_b_stop) {
			_consume.wait(lock);
		}

		if (_b_stop) {
			while (!_node_queue.empty()) {
				auto node = _node_queue.front();
				auto msg_id = node->_recv_node->_msg_id;
				std::cout << "recv_msg id is " << node->_recv_node->_msg_id << std::endl;
				auto callback_iter = _handlers.find(msg_id);
				if (callback_iter == _handlers.end()) {
					_node_queue.pop();
					std::cout << "msg_id " << msg_id << " handler not found." << std::endl;
					continue;
				}
				callback_iter->second(node->_session, node->_recv_node->_msg_id, 
					std::string(node->_recv_node->_data, node->_recv_node->_cur_length));
			}
			break;
		}

		auto node = _node_queue.front();
		auto msg_id = node->_recv_node->_msg_id;
		std::cout << "logicSystem msg_id is: " << msg_id << std::endl;
		auto callback_iter = _handlers.find(msg_id);
		if (callback_iter == _handlers.end()) {
			_node_queue.pop();
			std::cout << "msg_id " << msg_id << " handler not found." << std::endl;
			continue;
		}
		callback_iter->second(node->_session, node->_recv_node->_msg_id,
			std::string(node->_recv_node->_data, node->_recv_node->_cur_length));

		_node_queue.pop();
	}
}

void LogicSystem::postMsgToQueue(std::shared_ptr<LogicNode> node) {
	std::unique_lock<std::mutex> lock(_node_mutex);
	_node_queue.push(node);
	// 如果由空变为不空
	if (_node_queue.size() == 1) {
		lock.unlock();
		_consume.notify_one();
	}
}

void LogicSystem::registerHanlder()
{
	_handlers[MSG_CHAT_LOGIN] = std::bind(&LogicSystem::loginHandler, this, std::placeholders::_1, std::placeholders::_2, std::placeholders::_3);
}

void LogicSystem::loginHandler(std::shared_ptr<CSession> session, const short& msg_id, const std::string& msg_data) {
	Json::Reader reader;
	Json::Value root;
	reader.parse(msg_data, root);
	std::cout << "user login uid is: " << root["uid"].asInt() << 
		", user token is: " << root["token"].asString() << std::endl;

	// process data
	// ...

	// response data
	std::string return_str = root.toStyledString();
	session->send(return_str, msg_id);
}
```