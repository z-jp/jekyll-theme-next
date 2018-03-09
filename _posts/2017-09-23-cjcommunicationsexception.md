---
layout: post
title: CJCommunicationsException
categories: 
  - Java
  - MySQL
tags: 
  - MySQL 
  - Java
---
部署的 web 项目总是在第二天就挂了，一旦操作数据库便抛出 `CJCommunicationsException` 异常。原因是MySQL会释放超时的不活跃连接，默认是8小时，由 `wait_timeout` 属性指定。而在应用中我们通常使用数据库连接池，不会在每次操作数据库时创建新的连接，便有可能使用到了已经被服务器关闭的连接去服务数据库。 两种解决方法在异常中已经给出：
> You should consider either expiring and/or testing connection validity before use in your application, increasing the server configured values for client timeouts, or using the Connector/J connection property 'autoReconnect=true' to avoid this problem.  
    
1. 在连接中使用`autoReconnect=true`属性 在连接关闭的情况下自动重连，这听起来是个好方法，但实际上数据库操作依然会失败。数据库仅仅重新建立连接而不会再次执行操作。而且官方也并不推荐这种作法
  > The use of this feature is not recommended, because it has side effects related to session state and data consistency when applications don't handle SQLExceptions properly, and is only designed to be used when you are unable to configure your application to handle SQLExceptions resulting from dead and stale connections properly.
    
2. 设置服务器，延长超时时间
  > Alternatively, as a last option, investigate setting the MySQL server variable "wait_timeout" to a high value, rather than the default of 8 hours.  
  但是，不管怎么延长时间也总会有超时的时候，不释放空闲连接也会对服务器造成压力。每隔一段时间主动“Ping”服务器以保持连接的方式也是同理。

最后解决方法：
- 去掉 `autoReconnect` 属性
- 使用 [Tomcat JDBC Connection Pool](https://tomcat.apache.org/tomcat-8.0-doc/jdbc-pool.html#Introduction)
- 配置连接池如下
  ```ini
  testOnBorrow=true
  validationQuery=SELECT 1
  removeAbandoned=true
  ```
  或者代码配置
  ```java
  dataSource.setTestOnBorrow(true);
  dataSource.setRemoveAbandoned(true);
  dataSource.setValidationQuery("SELECT 1");
  ```
这样，连接池在取出连接时会执行`SELECT 1`来验证连接是否可用，并移除已关闭的连接。

参考链接:  
[https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-reference-configuration-properties.html](https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-reference-configuration-properties.html)  
[https://stackoverflow.com/questions/11301707/attempt-to-reconnect-jdbc-pool-datasource-after-database-restarts](https://stackoverflow.com/questions/11301707/attempt-to-reconnect-jdbc-pool-datasource-after-database-restarts)  
[https://tomcat.apache.org/tomcat-8.0-doc/jdbc-pool.html](https://tomcat.apache.org/tomcat-8.0-doc/jdbc-pool.html)
