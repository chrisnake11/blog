---
title: 使用gRPC提供服务
published: 2025-07-13T10:34:18Z
description: ''
image: ''
tags: [
    gRPC, C++, 网络编程, ChatRoom
]
category: 'C++'
draft: false
---

# 使用gPRC提供服务

gPRC是一个高性能、开源和通用的RPC框架，支持多种编程语言。它基于HTTP/2协议，提供了流式传输、双向流、流控等特性。

在C++中使用gRPC提供服务的步骤如下：
1. 安装gRPC和protobuf
2. 编写.proto文件
3. 使用protoc编译.proto文件
4. 编写服务端代码
5. 编译和运行服务端

## protobuf编写

编写.proto文件，定义服务和消息格式。

.proto文件中主要包含以下内容：
- 消息类型的定义(meesage)：
  定义服务端和客户端之间传输的数据结构。
- 服务的定义(service)：定义服务的接口和方法。
- 选项的定义(option)：生成代码时的选项。
- 导入其他.proto文件(import)：可以导入其他.proto文件中的消息类型和服务定义。
- 包名(package)：用于组织消息类型和服务的命名空间。

以下是一个简单的.proto文件示例：

```protobuf
syntax = "proto3";

package message;

// 定义消息类型
message GetChatServerReq {
  int32 uid = 1;
}


message GetChatServerRsp {
  int32 error = 1;
  string host = 2;
  string port = 3;
  string token = 4;
}

// 定义服务的接口
service StatusService {
	rpc GetChatServer (GetChatServerReq) returns (GetChatServerRsp) {}
}
```

## protoc编译

使用protoc编译.proto文件，生成处理消息序列化的C++代码。

```bash
protoc.exe --cpp_out=. "message.proto"
``` 

+ `--cpp_out`选项指定生成C++代码的输出目录。
+ `message.proto`是要编译的.proto文件。

生成gRPC服务的C++代码，用于rpc服务。


```bash
protoc.exe  -I="." --gRPC_out="." --plugin=protoc-gen-gRPC="gRPC_cpp_plugin.exe" "message.proto"
```

+ `-I`选项指定.proto文件的搜索路径。
+ `--gRPC_out`选项指定生成gRPC代码的输出目录。
+ `--plugin`选项指定gRPC插件的路径。


### 生成gRPC服务的脚本
```bash
@echo off

set PROTOC_PATH=D:\dev\grpc\visualstudio\third_party\protobuf\Debug\protoc.exe
set GRPC_PLUGIN_PATH=D:\dev\grpc\visualstudio\Debug\grpc_cpp_plugin.exe
set PROTO_FILE=message.proto

echo Generating gRPC code...
%PROTOC_PATH% -I="." --grpc_out="." --plugin=protoc-gen-grpc="%GRPC_PLUGIN_PATH%" "%PROTO_FILE%"

echo Generating C++ code...
%PROTOC_PATH% --cpp_out="." "%PROTO_FILE%"

echo Done
```

## 编写服务端代码

在C++中编写服务端代码，使用gRPC提供的API来实现服务。

主要步骤如下：
1. 引入生成的头文件
2. 实现服务端类，继承生成的服务基类
3. 实现服务方法
4. 创建gRPC服务器，注册服务
5. 启动服务器，监听端口


## 服务端代码示例：

使用gRPC提供ChatServer的轮询负载均衡服务，返回有效的服务器地址，并使用boost::asio来监听终止信号。

gRPC服务端实现步骤：
1. 创建gRPC服务实例
2. 监听端口
3. 注册服务
4. 启动服务器
```cpp
void RunServer() {
	auto& cfg = ConfigManager::GetInstance();

	// gRPC监听的地址
	std::string server_address(cfg["StatusServer"]["Host"] + ":" + cfg["StatusServer"]["Port"]);
	
	// gRPC服务实例
	StatusServiceImpl service;

	// 创建gRPC服务器
	gRPC::ServerBuilder builder;
	// 监听端口和添加服务
	builder.AddListeningPort(server_address, gRPC::InsecureServerCredentials());
	builder.RegisterService(&service);

	// 构建并启动gRPC服务器
	std::unique_ptr<gRPC::Server> server(builder.BuildAndStart());
	std::cout << "Server listening on " << server_address << std::endl;

	// 创建Boost.Asio的io_context监听终止信号
	boost::asio::io_context io_context;
	boost::asio::signal_set signals(io_context, SIGINT, SIGTERM);
	signals.async_wait([&server](const boost::system::error_code& error, int signal_number) {
		if (!error) {
			std::cout << "Shutting down server..." << std::endl;
			server->Shutdown(); // 优雅地关闭服务器
		}
		});

	// 守护线程运行io_context，捕获SIGINT信号
	std::thread([&io_context]() { io_context.run(); }).detach();

	// 等待gRPC服务器关闭
	server->Wait();
	// 停止boost的io_context服务
	io_context.stop();
}
```

## gRPC服务类的实现
```cpp
#pragma once
#include <gRPCpp/gRPCpp.h>
#include "message.gRPC.pb.h"

using gRPC::Server;
using gRPC::ServerBuilder;
using gRPC::ServerContext;
using gRPC::Status;
using message::GetChatServerReq;
using message::GetChatServerRsp;
using message::StatusService;

// ChatServer类，用地址和端口表示
struct ChatServer {
	std::string host;
	std::string port;
};

class StatusServiceImpl final : public StatusService::Service
{
public:
	StatusServiceImpl();
	Status GetChatServer(ServerContext* context, const GetChatServerReq* request,
		GetChatServerRsp* response) override;

	// 聊天服务器列表
	std::vector<ChatServer> _servers;
	// 指定下一个新聊天的服务器，简单的负载均衡
	int _server_index;
};
```

```cpp
#include "StatusServiceImpl.h"
#include "ConfigManager.h"
#include "const.h"
std::string generate_unique_string() {
	// 创建UUID对象
	boost::uuids::uuid uuid = boost::uuids::random_generator()();

	// 将UUID转换为字符串
	std::string unique_string = to_string(uuid);

	return unique_string;
}

Status StatusServiceImpl::GetChatServer(ServerContext* context, const GetChatServerReq* request, GetChatServerRsp* response)
{
	std::string prefix("status server has received :  ");
	// 这里数组越界
	//_server_index = (_server_index++) % (_servers.size());
	// 轮询方式选择聊天服务器。
	_server_index = (_server_index + 1) % (_servers.size());
	auto& server = _servers[_server_index];
	
	// 设置返回的聊天服务器的地址和端口
	// 设置状态码，和访问令牌
	response->set_host(server.host);
	response->set_port(server.port);
	response->set_error(ErrorCodes::SUCCESS);
	response->set_token(generate_unique_string());
	std::cout << "Sending grcp response: " << response->DebugString() << std::endl;
	return Status::OK;
}

StatusServiceImpl::StatusServiceImpl() :_server_index(0)
{
	auto& cfg = ConfigManager::GetInstance();
	ChatServer server;
	// 初始化可用的聊天服务器列表
	server.port = cfg["ChatServer1"]["Port"];
	server.host = cfg["ChatServer1"]["Host"];
	_servers.push_back(server);

	server.port = cfg["ChatServer2"]["Port"];
	server.host = cfg["ChatServer2"]["Host"];
	_servers.push_back(server);
}
```