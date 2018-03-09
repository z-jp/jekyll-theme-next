---
layout: post
title: Startup Jetty Service under Systemd
category: Java
tags: 
  - Java 
  - Systemd
  - jetty
---
> ### 介绍如何配置 Jetty 及使用 Systemd 管理 Jetty

### 安装 Jetty & 分离 Jetty Base 和 Jetty Home  
Jetty Base 目录用于配置 Jetty 分发包，Jetty Home 目录存放各版本的分发包。分离这两个目录是官方推荐的做法，方便 Jetty 管理和升级。下面开始安装 Jetty 及配置。
1. 创建目录
    ```bash
    # mkdir -p /opt/jetty
    # mkdir -p /var/www/jetty-base
    # mkdir -p /var/www/jetty-base/temp
    # mkdir -P /var/www/jetty-base/jetty
    ```
    /opt/jetty : Jetty Home /var/www/jetty-base : Jetty Base /var/www/jetty-base/temp : 临时目录，存放 war 解压包 /var/www/jetty-base/jetty : .pid 文件目录，配置 Systemd 需要用到
2. 解压 Jetty
    ```bash
    # tar -zxf /home/user/downloads/jetty-distribution-9.4.8.tar.gz -C /opt/jetty/
    ```
    不要修改分发包解压目录下任何文件，永远保持像刚解压时那样
3. 添加需要的模块
    ```bash
    # cd /var/www/jetty-base
    # java -jar /opt/jetty/jetty-distribution-9.4.8/start.jar jetty.base=/var/www/jetty-base jetty.home=/opt/jetty/jetty-distribution-9.4.8 \
        --add-to-start=deploy,http,jsp,servlets,jstl,server
    ```
常用的模块还有 console-capture 、requestlog 等
  
现在就可以将 war 放在 $Jetty.Base/webapps 然后通过 `java -jar /opt/jetty/jetty-distribution-9.4.8/start.jar jetty.home=/opt/jetty/jetty-distribution-9.4.8 jetty.base=/var/www/jetty-base` 启动 Jetty 了。更新 Jetty 只需要将新的分发包解压到 `/opt/jetty/` 目录，并修改启动命令。但这还不方便管理 Jetty 的启动停止。  
### 配置 Systemd
1. 创建 Jetty 用户
    ```bash 
    # useradd -U jetty -s /usr/bin/nologin
    ```
    创建一个名为 Jetty 的用户，所属用户组和用户名相同，没有创建home目录也无法登录
2. 编写 Systemd 单元  
    创建 `/lib/systemd/system/jetty.service` 文件，写入以下配置
    ```ini
    [Unit]
    Description=Jetty Web Application Container
    After=network.target
    
    [Service]
    Type=forking
    PIDFile=/var/run/jetty.pid
    User=jetty
    Group=jetty
    ExecStart=/opt/jetty/jetty-distribution-9.4.8/bin/jetty.sh start
    ExecStop=/opt/jetty/jetty-distribution-9.4.8/bin/jetty.sh stop
    SuccessExitStatus=143
    
    [Install]
    WantedBy=multi-user.target
    ```
3. 编写启动配置文件  
    创建 `/etc/default/jetty` 文件，写入以下配置
    ```ini
    JETTY_HOME=/opt/jetty/jetty-distribution-9.4.8
    JETTY_BASE=/var/www/jetty-base
    TMPDIR=/var/www/jetty-base/temp
    ```
4. 刷新 Systemd 配置  
    执行 `systemctl daemon-reload`，此时应该可以启动 Jetty 了。但很不幸，单元启动失败，日志显示 `jetty.service: PID file /run/jetty/jetty.pid not readable (yet?) after start: No such file or directory` 看来是pid文件权限问题，将 pid 文件改在 Jetty Base 目录：
    1. 添加 `JETTY_PID=/var/www/jetty-base/jetty/jetty.pid` 到 `/etc/default/jetty`
    2. 同时修改 `/lib/systemd/system/jetty.service` 中 `PIDFile` 路径
    3. `# chown -R jetty:jetty /var/www/jetty-base/` 确保 Jetty Base 所有者和权限正确
5. 激活并启动 Jetty
    ```bash
    # systemctl enable jetty
    # systemctl start jetty
    ```
  
#### All Done!
参考链接：   
[https://www.eclipse.org/jetty/documentation/9.4.x/startup-unix-service.html](https://www.eclipse.org/jetty/documentation/9.4.x/startup-unix-service.html) [https://github.com/eclipse/jetty.project/issues/1485](https://github.com/eclipse/jetty.project/issues/1485)