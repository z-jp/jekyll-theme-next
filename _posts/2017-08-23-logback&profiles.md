---
layout: post
title: Logback & profiles
category: Java
tags: 
  - Java 
  - Spring
  - logback
---
在不同环境使用不同的日志记录方式是常见的需求，而在 Spring 下可以使用 profiles 区分环境。但网上大多数教程是基于 Spring-boot 的，SpringMvc 似乎不支持 logback-spring.xml 的配置方式。好在还有其他方式来支持 logback-extensions
1. 添加依赖
```xml
   <dependency>
       <groupId>org.logback-extensions</groupId>
       <artifactId>logback-ext-spring</artifactId>
       <version>${version}</version>
   </dependency>
```
2. 启动Logback    
```java
   public class Initializer extends AbstractAnnotationConfigDispatcherServletInitializer {

       @Override
       public void onStartup(ServletContext servletContext) throws ServletException {
           super.onStartup(servletContext);
           servletContext.addListener(LogbackConfigListener.class);
       }
   ...
   }
```
3. 配置Logback.xml
```xml
   <configuration>
       <appender name="appender" class="ch.qos.logback.ext.spring.DelegatingLogbackAppender"/>
       <root level="INFO">
           <appender-ref ref="appender"/>
       </root>
   </configuration>
```
配置非常简单，仅仅是作为占位符，真正的配置在 Java 代码中，这也方便使用 Spring 的 profiles 功能
4. Spring 配置   
```java
   @Configuration
   public class LogbackConfig {

       @Bean 
       public static ApplicationContextHolder applicationContextHolder() {
           return new ApplicationContextHolder ();
       }

       @Bean 
       public static LoggerContext loggerContext() {
           return (LoggerContext) LoggerFactory.getILoggerFactory();
       }

       @Bean (initMethod = "start", destroyMethod = "stop")
       public static PatternLayoutEncoder encoder (LoggerContext ctx) {
           PatternLayoutEncoder encoder = new PatternLayoutEncoder();
           encoder.setContext(ctx);
           encoder.setPattern("%date %-5level \[%thread\] %logger{36} %m%n");
           return encoder;
       }

       @Bean (initMethod = "start", destroyMethod = "stop")
       public static ConsoleAppender appender (LoggerContext ctx, PatternLayoutEncoder encoder) {
           ConsoleAppender appender = new ConsoleAppender();
           appender.setContext(ctx);
           appender.setEncoder(encoder);
           return appender;
       }
   }
   @Configuration
   public class LogbackConfig {

       @Bean 
       public static ApplicationContextHolder applicationContextHolder() {
           return new ApplicationContextHolder ();
       }

       @Bean 
       public static LoggerContext loggerContext() {
           return (LoggerContext) LoggerFactory.getILoggerFactory();
       }

       @Bean (initMethod = "start", destroyMethod = "stop")
       public static PatternLayoutEncoder encoder (LoggerContext ctx) {
           PatternLayoutEncoder encoder = new PatternLayoutEncoder();
           encoder.setContext(ctx);
           encoder.setPattern("%date %-5level \[%thread\] %logger{36} %m%n");
           return encoder;
       }

       @Bean (initMethod = "start", destroyMethod = "stop")
       public static ConsoleAppender appender (LoggerContext ctx, PatternLayoutEncoder encoder) {
           ConsoleAppender appender = new ConsoleAppender();
           appender.setContext(ctx);
           appender.setEncoder(encoder);
           return appender;
       }
   }
```
代码来自[官方示例](https://github.com/qos-ch/logback-extensions/wiki/Spring)，需要注意的是配置 Appender 的 方法名必须和 logback.xml 中的引用相同。要使用 profiles 只需要配置多个 Appender ，但又必须同名，我的作法是拆分到不同类中。

5. profiles 配置
```java
   @Configuration
   @Profile("prod")
   public class LogbackConfigProd { 
       @Bean(initMethod = "start", destroyMethod = "stop")
       public static FileAppender appender(LoggerContext ctx, PatternLayoutEncoder encoder) {
       ... 
       }
   }

   @Configuration
   @Profile("dev")
   public class LogbackConfigDev {

       @Bean(initMethod = "start", destroyMethod = "stop")
       public static ConsoleAppender appender(LoggerContext ctx, PatternLayoutEncoder encoder) {
       ...
       }
   }

   @Configuration
   @Import({LogbackConfigProd.class,LogbackConfigDev.class})
   public class LogbackConfig {
   ...
   }
```

参考链接：[https://github.com/qos-ch/logback-extensions/wiki/Spring](https://github.com/qos-ch/logback-extensions/wiki/Spring)