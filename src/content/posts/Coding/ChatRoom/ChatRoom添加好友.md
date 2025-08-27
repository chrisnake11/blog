---
title: ChatRoom添加好友
published: 2025-08-22T14:03:12Z
description: 'ChatRoom添加好友功能，包括数据表设计、接口实现等'
image: ''
tags: [ChatRoom, QT, C++, MySQL, Redis]
category: 'ChatRoom'
draft: false
---

# ChatRoom添加好友

- [ ] 设计表、数据结构、业务逻辑
- [ ] 实现数据表
- [ ] 实现客户端界面
- [ ] 实现客户端TCP接口
- [ ] 实现ChatServer数据库接口
- [ ] 实现ChatServer业务逻辑接口


## 基本原则

在客户端查询用户信息时，为了保证责任最小化，所有的用户信息交给ChatServer在Redis或者MySQL中查询，客户端只需要传递用户的唯一标识符，例如用户ID即可。

为了区分用户的数据访问权限，我们将数据暂时分为3种。
+ 用户私密数据：包括用户的所有完整信息。
+ 用户公开数据：用于查找用户时展示的部分信息。
+ 添加用户数据：用于展示用户请求时所需的信息。

为了保存这些数据在MySQL数据库和Redis中，我们需要设计3种表。
1. 用户表`user_info`保存完整的用户信息。
```sql
CREATE TABLE user_info (
    uid BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) UNIQUE NOT NULL,
    nickname VARCHAR(50),
    passwd VARCHAR(255) NOT NULL,
    avatar_url VARCHAR(255),
    gender TINYINT,         -- 0: unknown, 1: male, 2: female
    address VARCHAR(255),
    email VARCHAR(100),
    phone VARCHAR(20),
    birthday DATE,
    personal_signature VARCHAR(255),
    online_status TINYINT     -- online/offline/busy
    last_login DATETIME,
    register_time DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

公开信息：
```
uid, username, nickname, avatar_url, 
gender, email, phone, birthday, 
personal_signature, online_status, register_time
```

申请好友所需的信息：
```
uid, username, nickname, avatar_url,
gender, address, request_message,create_time
```

2. 好友关系表，记录两个用户最终的好友关系（双向的）。
```sql
CREATE TABLE friend_relationship (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    friend_id BIGINT NOT NULL,
    friend_status TINYINT NOT NULL,       -- 0: accepted, 1: blocked
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    last_message_id BIGINT UNSIGNED,
    
    -- 生成列保证双向唯一
    uid_min BIGINT GENERATED ALWAYS AS (LEAST(user_id, friend_id)) STORED,
    uid_max BIGINT GENERATED ALWAYS AS (GREATEST(user_id, friend_id)) STORED,
    UNIQUE KEY (uid_min, uid_max), -- 唯一约束
    FOREIGN KEY (user_id) REFERENCES user_info(uid),
    FOREIGN KEY (friend_id) REFERENCES user_info(uid)
);
```


3. 添加好友请求表，记录用户向好友发送的请求信息（单向的）。
```sql
CREATE TABLE add_friend_request (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    from_id BIGINT NOT NULL,
    to_id BIGINT NOT NULL,
    request_message VARCHAR(255),
    status TINYINT NOT NULL DEFAULT 0, -- 0: pending, 1: accepted, 2: rejected
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY (from_id, to_id), -- 唯一约束
    FOREIGN KEY (from_id) REFERENCES user_info(uid),
    FOREIGN KEY (to_id) REFERENCES user_info(uid)
);
```

对应的3种C++数据结构如下:
```cpp
// 完整用户数据
struct PrivateUserInfo{
    int uid;
    std::string username;
    std::string nickname;
    std::string avatar_url;
    int gender;
    std::string address;
    std::string email;
    std::string phone;
    std::string birthday;
    std::string personal_signature;
    std::string online_status;
    std::string last_login;
    std::string register_time;
};

// 用户公开数据
struct PublicUserInfo{
    int uid;
    std::string username;
    std::string nickname;
    std::string avatar_url;
    int gender;
    std::string email;
    std::string phone;
    std::string birthday;
    std::string personal_signature;
    std::string online_status;
    std::string register_time;
};

// 请求数据
struct AddFriendInfo{
    int from_uid;
    int to_uid;
    std::string request_message;
};
```


## 流程

### 1. 用户搜索

1. 用户点击添加按钮，跳出添加好友界面。
2. 在添加好友界面输入用户`username`, 根据用户名搜索用户。
3. 请求发送给ChatServer，ChatServer查询用户公开信息，以及是否为好友关系，全部返回给客户端。
   1. 如果为空，返回错误。
4. 客户端展示用户信息。
   1. 根据错误，提示用户。


### 2. 用户添加

1. 选择要添加的**非好友用户**，点击添加。
2. 点击添加后，弹出添加好友请求的确认框，输入自我介绍，然后发送请求。
3. 将数据封装为`(from_uid, to_uid, request_message)`并序列化，经过TcpManager发送给ChatServer。
4. ChatServer接收到消息后在Redis或者MySQL中查询用户。
5. 用户存在，再查询好友关系表，判断双方是否已经是好友关系。
   1. 用户不存在，返回错误。
   2. 如果是好友关系，返回错误。
6. 找不到说明不是好友关系，查询add_friend_request表，判断是否已经发送过请求。
   1. 如果已经发送过请求，报错返回已添加。
   2. 如果对方发送了请求，直接添加成功(更新add_friend_request, friend_relationship表)，并推送消息给双方。
   3. 如果不存在请求，新建添加好友请求。
7. ChatServer将添加好友请求，通知给双方用户（前端）
   1. 如果通知对方，通过GRPC通知对方的ChatServer
   2. ChatServer打包用户信息和请求状态传递给客户端。


### 3. 用户审核

1. 初始化，查询add_friend_request表，遍历所有待审核的请求。
2. 对于每个未处理的请求，由用户判断是否被接受或拒绝。
   1. 如果被接受，发送接收请求给Chat Server，ChatServer添加好友关系到friend_relationship表，并更新add_friend_request表中的记录。
   2. 如果被拒绝，发送拒绝请求给Chat Server，ChatServer更新add_friend_request表中的记录。
3. 随后ChatServer将处理结果推送给双方用户（前端）。
   1. 如果通知对方，通过GRPC通知对方的ChatServer
   2. ChatServer打包用户信息和请求状态传递给客户端。