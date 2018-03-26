---
layout: post
title: 在 PostgreSQL 中使用 citext
category: PostgreSQL
tags: PostgreSQL
---
在数据存储中，有些数据我们希望是大小写不敏感的，比如将 'admin@example.com' 和 'Admin@example.com' 视为同一个邮箱地址，不允许重复注册账号。但 PostgreSQL 没有原生支持大小写不敏感的属性或类型。一种做法是在查询时统一转换为小写：
```SQL
SELECT email FROM account WHERE LOWER(email) = LOWER( ? );
```
这样做的缺点有
  - 需要在所有大小写不敏感的查询中手工地使用 `LOWER()` 函数
  - 如果有索引，需要重新创建表达式索引
  - 并没有带表达式的唯一约束，虽然唯一索引可以实现相同的效果，但语义不同了

另一个选择是使用 citext 数据类型--自动调用 `LOWER()` 函数的 text 类型。citext 基本解决了以上的缺陷，实现对应用提供透明的大小写不敏感数据查询。   
使用方法也很简单：
```SQL
CREATE EXTENSION IF NOT EXISTS citext;
ALTER TABLE account ALTER COLUMN email TYPE citext;
```
中文网络中对 citext 拓展的介绍并不多，但看到 [Product Hunt](https://mikecoutermarsh.com/storing-email-in-postgres-rails-use-citext/) 也在使用这个方案，便果断也用了。   
在 JDBC 中，还有一点额外的工作要做。因为 citext 在和其他字符串类型比较时，仅但另一方未指定类型或也为 citext 时才进行小写转换，而 PostgreSQL JDBC 默认将 `PreparedStatement` 的字符串参数视为 varchar 类型，解决方法有二：
  - 对于全局连接；使用 `stringtype=unspecified` 参数，让 PostgreSQL 服务器决定参数类型
  - 对于单个查询；指定参数类型 `SELECT email FROM account WHERE email = ( ? ::citext )`
   
参考链接：   
[Case insensitive UNIQUE constraints in Postgres](http://shuber.io/case-insensitive-unique-constraints-in-postgres/)   
[Improve case-insensitive queries in Postgres using smarter indexes](https://coderwall.com/p/6yhsuq/improve-case-insensitive-queries-in-postgres-using-smarter-indexes)   
[Connecting to the Database](https://jdbc.postgresql.org/documentation/91/connect.html#connection-parameters)   
[PgConnection.java](https://github.com/pgjdbc/pgjdbc/blob/master/pgjdbc/src/main/java/org/postgresql/jdbc/PgConnection.java)   