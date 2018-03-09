---
layout: post
title: Embedded jetty in project without web.xml
category: Java
tags: 
  - Java 
  - web
  - jetty
---
`AbstractAnnotationConfigDispatcherServletInitializer` 可以替代 web.xml 配置 DispatcherServlet ，部署时需要支持 Servlet3.0 的 web 服务器如 Tomcat 7，Jetty 8。
- Maven  
在 Maven 项目集成 Jetty 插件时注意 org.mortbay.jetty:jetty-maven-plugin 已经废弃，jetty:Run 会在检查 web.xml 时直接抛出 NPE ，可以使用 org.eclipse.jetty:jetty-maven-plugin
```xml
   <dependency>
       <groupId>org.eclipse.jetty</groupId>
       <artifactId>jetty-maven-plugin</artifactId>
       <version>${jetty.version}</version>
   </dependency>
```
参考链接：[https://stackoverflow.com/questions/19737236/can-i-use-jetty-to-deploy-my-project-which-doesnt-have-a-web-xml](https://stackoverflow.com/questions/19737236/can-i-use-jetty-to-deploy-my-project-which-doesnt-have-a-web-xml)
- Gradle  
Jetty 的 Gradle 插件也只支持 Jetty 6 ==、 可以用 org.akhikhl.gretty 替代，配置 servletContainer 属性指定 Jetty 版本  
代码参考：[https://www.mkyong.com/spring-mvc/gradle-spring-4-mvc-hello-world-example-annotation/](https://www.mkyong.com/spring-mvc/gradle-spring-4-mvc-hello-world-example-annotation/)