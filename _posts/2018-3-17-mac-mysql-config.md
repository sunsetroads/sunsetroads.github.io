---
layout: post
title: Mac 中 MySQL 的配置流程
categories: Linux
description: some word here
keywords: keyword1, keyword2
---

记录下 MySQL 的配置和使用流程。

**安装 MySQL**
```
brew install MySQL
```
如果没有brew 环境，先安装 brew
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

**初始化**

为了快速使用，MySQL 安装时跳过了一些配置选项，使用此命令来配置 root 密码等。
```
mysql_secure_installation
```

**启动 MySQL**
```
mysql.server start
```

**停止 MySQL 运行**
```
mysql.server stop
```

**查看运行状态** 
```
mysql.server status
```

**终端中登录 MySQL**

当 MySQL 服务已经运行时, 我们可以通过 MySQL 自带的客户端工具登录到 MySQL 数据库中, 首先打开命令提示符, 输入以下格式的命名:
```
mysql -h 主机名 -u 用户名 -p
```
参数说明：
- -h : 指定客户端所要登录的 MySQL 主机名, 登录本机该参数可以省略。
- -u : 登录的用户名。
- -p : 使用一个密码来登录, 如果所要登录的用户名密码为空, 可以忽略此选项。


**备注**

先记录下，后面用到时再添加。

其它 MySql 相关命令查看 [MySql 教程](https://www.runoob.com/mysql/mysql-tutorial.html) 。