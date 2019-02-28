---
layout: post
title:  "举例说明sql注入"
categories: Mysql
tags:  sql java
author: Utachi
---

* content
{:toc}

## 引子

* 平时的工作中很多WAF（web应用防火墙）的用户会发现拦截sql注入***条，过来问我什么是sql注入，能不呢简单举个例子说明一下，今天元旦没什么事做就写一篇记录一下，分享给大家

## 什么是sql注入

* 维基百科：
SQL注入攻击（英语：SQL injection），简称SQL攻击或注入攻击，是发生于应用程序与数据库层的安全漏洞。简而言之，是在输入的字符串之中注入SQL指令，在设计不良的程序当中忽略了字符检查，那么这些注入进去的恶意指令就会被数据库服务器误认为是正常的SQL指令而运行，因此遭到破坏或是入侵。

## 一个例子
以常见的JDBC来说：

JDBC Statement 形式的数据库操作，是将一个组装好的带有数据的SQL直接提交给MySQL服务，这有SQL注入的危险，是不安全的。

下面是一个不安全的例子：





### 数据库和表设计

```bash
-- 创建数据库
CREATE DATABASE `utachi`;

-- 切换到 utachi 库
USE `utachi`;

-- 创建表
CREATE TABLE `user_balance` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT ,
  `name` VARCHAR(45) NOT NULL ,
  `balance` BIGINT NOT NULL ,
  PRIMARY KEY (`id`)
)
ENGINE = InnoDB
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_general_ci;
```

* 在 user_balance 表中准备两条数据：

```bash
mysql> select * from user_balance;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | zhangs |    1000 |
|  2 | zhaosi |    1001 |
+----+--------+---------+
2 rows in set (0.00 sec)
```
### 一个不安全的查询案例

```bash
package demo;

import com.sun.tools.internal.ws.wsdl.document.soap.SOAPUse;

import java.sql.*;

/**
 * 使用 Statement 是不安全的
 * sql 注入
 */
public class UnsafeStatement {

    private static final String USER = "root";
    private static final String PASSWORD = "123456";

    private static final String JDBC_DRIVER = "com.mysql.jdbc.Driver";
    private static final String DB_URL = "jdbc:mysql://127.0.0.1:3306/utachi";

    public static void select(String name) throws ClassNotFoundException, SQLException {
        Class.forName(JDBC_DRIVER);
        Connection conn =  DriverManager.getConnection(DB_URL, USER, PASSWORD);
        Statement stmt = null;
        try {
            stmt = conn.createStatement();
            String sql = String.format("SELECT * FROM user_balance WHERE name='%s'", name);
            System.out.println(sql);
            ResultSet resultSet = stmt.executeQuery(sql);
            while (resultSet.next()) {
                System.out.printf("id: %s, name: %s, balance: %s\n",
                        resultSet.getLong("id"),
                        resultSet.getString("name"),
                        resultSet.getLong("balance"));
            }
            resultSet.close();
        } finally {
            if (stmt != null) {
                stmt.close();
            }
            conn.close();
        }
    }


    public static void main(String[] args) throws SQLException, ClassNotFoundException {
        select("zhangs' OR '1'='1");
    }
}
```

执行结果：

```bash
SELECT * FROM user_balance WHERE name='zhangs' OR '1'='1'
id: 1, name: zhangs, balance: 1000
id: 2, name: zhaosi, balance: 1001
```

select 方法的本意是根据 name 查询对应的记录。但是"有人"精心构造了name的值('zhangs' OR '1'='1')，最终导致组装的SQL变成了：
```bash
SELECT * FROM user_balance WHERE name='zhangs' OR '1'='1'
```

这个SQL会查询所有的数据，不是select 方法预期的功能。

### 如何改进

使用JDBC PreparedStatement 查询数据


```bash

package demo;

import java.sql.*;

public class PreparedStatementSelect {


    private static final String USER = "root";
    private static final String PASSWORD = "123456";

    private static final String JDBC_DRIVER = "com.mysql.jdbc.Driver";
    private static final String DB_URL = "jdbc:mysql://127.0.0.1:3306/utachi";

    public static void select(String name) throws ClassNotFoundException, SQLException {
        Class.forName(JDBC_DRIVER);
        Connection conn =  DriverManager.getConnection(DB_URL, USER, PASSWORD);
        PreparedStatement pstmt = null;
        try {
            pstmt = conn.prepareStatement("SELECT * FROM user_balance WHERE name=?");
            pstmt.setString(1, name);
            ResultSet resultSet = pstmt.executeQuery();
            while (resultSet.next()) {
                System.out.printf("id: %s, name: %s, balance: %s\n",
                        resultSet.getLong("id"),
                        resultSet.getString("name"),
                        resultSet.getLong("balance"));
            }
            resultSet.close();
        } finally {
            if (pstmt != null) {
                pstmt.close();
            }
            conn.close();
        }
    }


    public static void main(String[] args) throws SQLException, ClassNotFoundException {
        select("zhangs");
        select("zhaosi");
    }

}
```
执行结果：

```bash
id: 1, name: zhangs, balance: 1000
id: 2, name: zhaosi, balance: 1001
```
