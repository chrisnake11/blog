---
title: 重构MySQL数据库的Manager和Dao层
published: 2025-08-25T10:23:07Z
description: '重构MySQL数据库的Manager和Dao层，提升代码的可维护性和可扩展性。'
image: ''
tags: [MySQL, 重构, ChatRoom, C++]
category: 'ChatRoom'
draft: false
---

# 重构MySQL数据库的Manager和Dao层

在实际开发中，随着业务需求的不断变化，数据库的结构和访问方式也需要不断调整。为了提高代码的可维护性和可扩展性，我们可以对MySQL数据库的Manager和Dao层进行重构。

## 理解Manager和Dao层的职责

在重构之前，我们需要明确Manager和Dao层的职责：

- **Manager层**：负责业务逻辑的处理，调用Dao层的方法，协调各个Dao之间的关系。
  - Manager层只负责业务逻辑的处理，当需要访问数据库时，它会调用相应的Dao层方法。
- **Dao层**：负责与数据库进行直接交互，执行SQL语句，映射数据库记录到对象。
  - Dao层只负责数据的持久化，即：对数据库的增删改查操作，然后将结果封装为对象，提供给上层调用。

之前的代码将Dao层完全负责业务逻辑，并且在MySQL中直接编写SQL函数，导致了代码的混乱和难以维护。

## SQL函数的作用

SQL函数适合用于封装那些“数据相关”的、“计算密集型的”、“与业务规则无关”的核心操作。

SQL函数一般只负责一些简单的操作，例如：简单的算术计算，数据格式化，数据格式检验（长度检验），数据排序等。

但实际上SQL函数的作用与DAO层的有部分冲突。DAO层有时也会涉及到一些数据处理的逻辑，例如数据的过滤、排序等。

在本次重构中，原有的SQL函数包含了业务逻辑，我们将这些业务逻辑迁移到Manager层中，删除了SQL函数，并在DAO层访问数据。

+ **例如根据用户生日查询用户年龄就可以使用SQL函数实现，减少字段。**
```sql
CREATE DEFINER = CURRENT_USER FUNCTION `getUserAge`(p_user_id INT) 
RETURNS INT
DETERMINISTIC
BEGIN
    DECLARE birth_date DATE;
    DECLARE user_age INT;
    
    -- 查询用户生日，若不存在则birth_date保持NULL
    SELECT birthday INTO birth_date 
    FROM user_info 
    WHERE user_id = p_user_id;
    
    -- 判断用户是否存在
    IF birth_date IS NULL THEN
        -- 用户不存在时返回NULL或特定标识（如-1）
        RETURN NULL;
        -- 也可以使用：RETURN -1; 视业务需求而定
    ELSE
        -- 计算年龄
        SET user_age = TIMESTAMPDIFF(YEAR, birth_date, CURDATE());
        RETURN user_age;
    END IF;
END;

```

## 原来的代码

直接在SQL函数中实现注册功能相关的业务，DAO层直接调用对应的call语句来执行，而Manager层直接调用dao层，所有的业务逻辑集中在SQL函数中。

```sql
CREATE DEFINER=`root`@`localhost` PROCEDURE `reg_user`(IN `new_name` VARCHAR(255), 
    IN `new_email` VARCHAR(255), 
    IN `new_pwd` VARCHAR(255), 
    OUT `result` INT)
BEGIN
    
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        
        ROLLBACK;
        
        SET result = -1;
    END;
    
    START TRANSACTION;
    
    IF EXISTS (SELECT 1 FROM `user_info` WHERE `name` = new_name) THEN
        SET result = 0; 
        COMMIT;
    ELSE
        
        IF EXISTS (SELECT 1 FROM `user_info` WHERE `email` = new_email) THEN
            SET result = 0; 
            COMMIT;
        ELSE
            
            UPDATE `user_id` SET `id` = `id` + 1;
            
            SELECT `id` INTO @new_id FROM `user_id`;
            
            INSERT INTO `user_info` (`uid`, `name`, `email`, `passwd`) VALUES (@new_id, new_name, new_email, new_pwd);
            
            SET result = @new_id; 
            COMMIT;
        END IF;
    END IF;
END
```


## 解耦后的代码

删除了SQL函数，将业务逻辑迁移到Manager层中，DAO层只负责数据的持久化。

+ MysqlManager层

```cpp
int MysqlManager::registerUser(const std::string& name, const std::string& email, const std::string& passwd)
{
	// 1. 检查用户名和邮箱是否存在
	if (!_dao.existUserByName(name)) {
		return ERROR_USER_EXIST;
	} 
	if(!_dao.existUserByEmail(email)) {
		return ERROR_EMAIL_NOT_MATCH;
	}
	// 2. 更新用户id
	int uid = _dao.updateUserId();
	if (uid < 0) {
		return ERROR_USER_EXIST;
	}
	// 3. 创建用户基本信息
	BaseUserInfo base_info;
	base_info.uid = uid;
	base_info.username = name;
	base_info.email = email;
	base_info.passwd = passwd;
	int ret = _dao.createBaseUserInfo(base_info);
	if (ret < 0) {
		return ERROR_USER_EXIST;
	}
	return uid;
}

int MysqlManager::resetUserPasswd(const std::string& passwd, const std::string& email)
{
	if (_dao.existUserByEmail(email)) {
		return ERROR_EMAIL_NOT_MATCH;
	}

	int res = _dao.updateUserPasswdByEmail(passwd, email);
	if (res < 0) {
		return ERROR_EMAIL_NOT_MATCH;
	}
	return res;
}

bool MysqlManager::checkNameAndPasswd(const std::string& name, const std::string& passwd, UserInfo& user_info)
{
	return _dao.checkNameAndPasswd(name, passwd, user_info);
}
```

+ DAO层

```cpp
#include "MysqlDao.h"
#include <chrono>
#include "MysqlManager.h"

/*
    ...
    MysqlConnectionPool相关代码

*/

MysqlDao::MysqlDao()
{
    // 获取数据库连接信息，初始化连接池
    auto& config = ConfigManager::GetInstance();
    const auto& host = config["MySQL"]["Host"];
    const auto& user = config["MySQL"]["User"];
    const auto& passwd = config["MySQL"]["Password"];
    const auto& schema = config["MySQL"]["Schema"];
    const auto& port = config["MySQL"]["Port"];

    _pool.reset(new MysqlPool(host + ":" + port, user, passwd, schema, 5));
}

MysqlDao::~MysqlDao()
{
    _pool->close();
}

bool MysqlDao::existUserByName(const std::string& username)
{
    auto conn = _pool->getConnection();
    const Defer defer([this, &conn]() {
        _pool->returnConnection(std::move(conn));
        });

    try {
        if(conn == nullptr) {
            _pool->returnConnection(std::move(conn));
            return false;
		}
        std::unique_ptr<sql::PreparedStatement> pstmt(
            conn->_con->prepareStatement("select 1 from user_info where username = ?")
        );
        pstmt->setString(1, username);
        std::unique_ptr<sql::ResultSet> res(pstmt->executeQuery());
        
        // if result set is empty, return false
        if (res->next()) {
            return true;
        }
    }
    catch (std::exception& e) {
		std::cerr << "exception: " << e.what() << std::endl;
    }

    return false;
}

bool MysqlDao::existUserByEmail(const std::string& email)
{
    auto conn = _pool->getConnection();
    const Defer defer([this, &conn]() {
        _pool->returnConnection(std::move(conn));
        });

    try {
        if (conn == nullptr) {
            return false;
        }
        std::unique_ptr<sql::PreparedStatement> pstmt(
            conn->_con->prepareStatement("select 1 from user_info where email = ?")
        );
        pstmt->setString(1, email);
        std::unique_ptr<sql::ResultSet> res(pstmt->executeQuery());

        if (res->next()) {
            return true;
        }
    }
    catch (std::exception& e) {
        std::cerr << "exception: " << e.what() << std::endl;
    }
    return false;
}

int MysqlDao::updateUserId()
{
    std::unique_ptr<SqlConnection> conn = _pool->getConnection();
    const Defer defer([this, &conn]() {
        _pool->returnConnection(std::move(conn));
        });
    try {
        if (conn == nullptr) {
			std::cerr << "get mysql connection is nullptr" << std::endl;
            return -1;
        }
        std::unique_ptr<sql::PreparedStatement> pstmt(
            conn->_con->prepareStatement("UPDATE user_id SET id=(id+1)")
		);
        pstmt->executeUpdate();

		// 更新成功，返回更新后的用户ID
        return getMaxUserId();
    }
    catch (std::exception& e) {
		std::cout << "update user id exception: " << e.what() << std::endl;
    }
    return -1;
}

int MysqlDao::getMaxUserId()
{
    std::unique_ptr<SqlConnection> conn = _pool->getConnection();
    const Defer defer([this, &conn]() {
        _pool->returnConnection(std::move(conn));
        });
    try {
        if (conn == nullptr) {
            std::cerr << "get mysql connection is nullptr" << std::endl;
            return -1;
        }
        std::unique_ptr<sql::Statement> stmt(conn->_con->createStatement());
        std::unique_ptr<sql::ResultSet> res(stmt->executeQuery("SELECT id FROM user_id"));
        if (res->next()) {
            int id = res->getInt("id");
            return id;
        }
    }
    catch (std::exception& e) {
        std::cout << "update user id exception: " << e.what() << std::endl;
    }
    return -1;
}

int MysqlDao::createBaseUserInfo(const BaseUserInfo& user_info)
{
	auto conn = _pool->getConnection();
    const Defer defer([this, &conn]() {
        _pool->returnConnection(std::move(conn));
		});
    try {
        if (conn == nullptr) {
            return -1;
        }
        std::unique_ptr<sql::PreparedStatement> pstmt(
            conn->_con->prepareStatement("INSERT INTO user_info(uid, username, email, passwd) VALUES(?,?,?,?)")
		);
        pstmt->setInt(1, user_info.uid);
        pstmt->setString(2, user_info.username);
        pstmt->setString(3, user_info.email);
        pstmt->setString(4, user_info.passwd);
        int ret = pstmt->executeUpdate();
        if (ret == 1) {
            return static_cast<int>(pstmt->getUpdateCount());
        }
    }
    catch(std::exception& e) {
        std::cout << "create base user info exception: " << e.what() << std::endl;
	}
    return -1;
}

int MysqlDao::createUserInfo(const UserInfo& user_info)
{
    auto conn = _pool->getConnection();
    const Defer defer([this, &conn]() {
        _pool->returnConnection(std::move(conn));
        });
    try {
        if (conn == nullptr) {
            return -1;
        }
        std::unique_ptr<sql::PreparedStatement> pstmt(
            conn->_con->prepareStatement("INSERT INTO user_info(username, email, nickname, passwd, avatar, gender, address, phone, birthday, psersonal_signature, online_status, last_login, register_time) VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?)")
        );
		pstmt->setString(1, user_info.username);
		pstmt->setString(2, user_info.email);
		pstmt->setString(3, user_info.nickname);
		pstmt->setString(4, user_info.passwd);
		pstmt->setString(5, user_info.avatar);
        pstmt->setString(6, user_info.gender);
        pstmt->setString(6, user_info.address);
		pstmt->setString(7, user_info.phone);
		pstmt->setString(8, user_info.birthday);
		pstmt->setString(9, user_info.sign);
        pstmt->setInt(10, user_info.online_status);
		pstmt->setDateTime(11, user_info.last_login);
		pstmt->setDateTime(12, user_info.register_time);
        int ret = pstmt->executeUpdate();
        if (ret == 1) {
            return static_cast<int>(pstmt->getUpdateCount());
        }
    }
    catch (std::exception& e) {
        std::cout << "create base user info exception: " << e.what() << std::endl;
    }
    return -1;
}

std::vector<UserInfo> MysqlDao::getUsersByNameAndPasswd(const std::string& name, const std::string passwd)
{
	std::vector<UserInfo> user_list;
	auto conn = _pool->getConnection();
    const Defer defer([this, &conn]() {
        _pool->returnConnection(std::move(conn));
		});
    try {
        if (conn == nullptr) {
			return user_list;
        }
        std::unique_ptr<sql::PreparedStatement> pstmt(
            conn->_con->prepareStatement("SELECT * FROM user_info WHERE username = ? AND passwd = ?"));
		pstmt->setString(1, name);
		pstmt->setString(2, passwd);
		std::unique_ptr<sql::ResultSet> res(pstmt->executeQuery());
        if (res->next()) {
            UserInfo user_info;
            user_info.uid = res->getInt("uid");
            user_info.username = res->getString("username");
            user_info.email = res->getString("email");
            user_info.passwd = res->getString("passwd");
            user_info.nickname = res->getString("nickname");
            user_info.phone = res->getString("phone");
            user_info.address = res->getString("address");
            user_info.avatar = res->getString("avatar");
            user_info.gender = res->getInt("gender");
			user_info.birthday = res->getString("birthday");
			user_info.sign = res->getString("personal_signature");
			user_info.online_status = res->getInt("online_status");
			user_info.last_login = res->getString("last_login");
			user_info.register_time = res->getString("register_time");
			user_list.push_back(user_info);
        }

    }
    catch (std::exception& e) {
		std::cout << "get user by name and passwd exception: " << e.what() << std::endl;
    }
    return user_list;
}

std::vector<UserInfo> MysqlDao::getUserByName(const std::string& name)
{
    std::vector<UserInfo> user_list;
    auto conn = _pool->getConnection();
    const Defer defer([this, &conn]() {
        _pool->returnConnection(std::move(conn));
        });
    try {
        if (conn == nullptr) {
            return user_list;
        }
        std::unique_ptr<sql::PreparedStatement> pstmt(
            conn->_con->prepareStatement("SELECT * FROM user_info WHERE username = ?"));
        pstmt->setString(1, name);
        std::unique_ptr<sql::ResultSet> res(pstmt->executeQuery());
        if (res->next()) {
            UserInfo user_info;
            user_info.uid = res->getInt("uid");
            user_info.username = res->getString("username");
            user_info.email = res->getString("email");
            user_info.passwd = res->getString("passwd");
            user_info.nickname = res->getString("nickname");
            user_info.phone = res->getString("phone");
            user_info.address = res->getString("address");
            user_info.avatar = res->getString("avatar");
            user_info.gender = res->getInt("gender");
            user_info.birthday = res->getString("birthday");
            user_info.sign = res->getString("personal_signature");
            user_info.online_status = res->getInt("online_status");
            user_info.last_login = res->getString("last_login");
            user_info.register_time = res->getString("register_time");
            user_list.push_back(user_info);
        }

    }
    catch (std::exception& e) {
        std::cout << "get user by name and passwd exception: " << e.what() << std::endl;
    }
    return user_list;
}

std::vector<UserInfo> MysqlDao::getUserById(int uid)
{
    std::vector<UserInfo> user_list;
    auto conn = _pool->getConnection();
    const Defer defer([this, &conn]() {
        _pool->returnConnection(std::move(conn));
        });
    try {
        if (conn == nullptr) {
            return user_list;
        }
        std::unique_ptr<sql::PreparedStatement> pstmt(
            conn->_con->prepareStatement("SELECT * FROM user_info WHERE uid=?"));
        pstmt->setInt(1, uid);
        std::unique_ptr<sql::ResultSet> res(pstmt->executeQuery());
        while (res->next()) {
            UserInfo user_info;
            user_info.uid = res->getInt("uid");
            user_info.username = res->getString("username");
            user_info.email = res->getString("email");
            user_info.passwd = res->getString("passwd");
            user_info.nickname = res->getString("nickname");
            user_info.phone = res->getString("phone");
            user_info.address = res->getString("address");
            user_info.avatar = res->getString("avatar");
            user_info.gender = res->getInt("gender");
            user_info.birthday = res->getString("birthday");
            user_info.sign = res->getString("personal_signature");
            user_info.online_status = res->getInt("online_status");
            user_info.last_login = res->getString("last_login");
            user_info.register_time = res->getString("register_time");
            user_list.push_back(user_info);
        }
    }
    catch (std::exception& e) {
        std::cout << "get user by name and passwd exception: " << e.what() << std::endl;
    }
    return user_list;   
}

int MysqlDao::updateUserPasswdByEmail(const std::string& email, const std::string& passwd)
{
	auto conn = _pool->getConnection();
    const Defer defer([this, &conn]() {
        _pool->returnConnection(std::move(conn));
        });
    try {
        std::unique_ptr<sql::PreparedStatement> pstmt(
            conn->_con->prepareStatement("UPDATE user_info SET passwd=? WHERE email=?"));
        pstmt->setString(1, passwd);
		pstmt->setString(2, email);
        int ret = pstmt->executeUpdate();
        if (ret == 1) {
            return static_cast<int>(pstmt->getUpdateCount());
		}
    }
    catch(std::exception& e) {
        std::cout << "update user passwd by email exception: " << e.what() << std::endl;
    }
    return -1;
}

bool MysqlDao::checkNameAndPasswd(const std::string& name, const std::string& passwd, UserInfo& user_info) {
    auto connect = _pool->getConnection();
    Defer defer([this, &connect] {
        _pool->returnConnection(std::move(connect));
        });

    try {
        if (connect == nullptr) {
            return false;
        }

        std::unique_ptr<sql::PreparedStatement> pstmt(
            connect->_con->prepareStatement("select * from user where username = ? and passwd = ?")
        );
        pstmt->setString(1, name);
        pstmt->setString(2, passwd);

        std::unique_ptr<sql::ResultSet> res(pstmt->executeQuery());
        
        // if result set is empty, return false
        if (!res->next()) {
            std::cout << "password is not matched" << std::endl;
            return false;
        }
        user_info.uid = res->getInt("uid");
        user_info.username = res->getString("username");
        user_info.email = res->getString("email");
        user_info.passwd = res->getString("passwd");
        user_info.nickname = res->getString("nickname");
        user_info.phone = res->getString("phone");
        user_info.address = res->getString("address");
        user_info.avatar = res->getString("avatar");
        user_info.gender = res->getInt("gender");
        user_info.birthday = res->getString("birthday");
        user_info.sign = res->getString("personal_signature");
        user_info.online_status = res->getInt("online_status");
        user_info.last_login = res->getString("last_login");
        user_info.register_time = res->getString("register_time");
        return true;
    }

    catch (sql::SQLException &e) {
        std::cerr << "SQLException: " << e.what();
        std::cerr << " (MySQL error code: " << e.getErrorCode();
        std::cerr << ", SQLState: " << e.getSQLState() << " )" << std::endl;
        return false;
    }
}   

```