---
layout: wiki
title: CentOS7 中 MongoDB 配置流程
categories: linux
description: CentOS7 中MongoDB配置流程
keywords: CentOS MongoDB linux
---

记录一下centos7 中 mongodb 的配置流程

#### 使用配置文件启动
##### 创建配置文件
```
cd /usr/local/etc
```
```
 vi mongod.conf
```
##### 复制下面的内容
```
systemLog:
  destination: file
  path: /usr/local/var/log/mongodb/mongo.log
  logAppend: true
storage:
  dbPath: /usr/local/var/mongodb
net:
  bindIp: 127.0.0.1
  port: 27017
processManagement:
  fork: true
security:
  authorization: enabled
```
##### 启动服务
```
 mongod --config /usr/local/etc/mongod.conf
```

##### 进入mongodb交互shell
```
 mongo
```

#### 创建管理员角色
**需要切换到对应数据库创建用户，进入admin数据库创建管理员**

```
use admin
```
```
db.createUser(  
  { user: "root",  
    pwd: "xxxxx",  
    roles: [ { role: "root", db: "admin" } ]  
  }  
)
```
#### 创建用户角色

```
use blog
```

```
db.createUser(  
  { user: "user1",  
    pwd: "xxxx123",  
    roles: [ { role: "readWrite", db: "blog" } ]  
  }  
) 
```

#### 用户认证
**要到对应的数据库上去认证用户, 否则会认证授权失败**

```
use blog
```

```
db.auth('user1','xxxx123')
```
#### 数据备份
```
mongodump -h 127.0.0.1 --port 27017 -d blog -u user1 -p xxxx123 -o /Users/zhangning/Desktop/mongoData
```
#### 数据恢复
```
mongorestore -h 127.0.0.1 --port 27017 -u user1 -p xxxx123 -d iBlog2 --drop /Users/zhangning/Desktop/mongoData/iBlog2
```
#### 数据导入
```
mongoimport -h 127.0.0.1 --port 27017 -u user1 -p xxxx123 -d iBlog2 -c post --upsertFields _id --drop /Users/zhangning/Desktop/test.csv
```

#### 数据导出
```
mongoexport -h 127.0.0.1 --port 27017 -u user1 -p xxxx123 -d iBlog2 -c post -f _id -o /Users/zhangning/Desktop/test.csv
```
#### 定时备份脚本
```
mkdir -p ~/crontab
vi ~/crontab/mongod_bak.sh
```
**脚本内容**

```
#!/bin/sh
DUMP=mongodump
OUT_DIR=/data/backup/mongod/tmp   // 备份文件临时目录
TAR_DIR=/data/backup/mongod       // 备份文件正式目录
DATE=`date +%Y_%m_%d_%H_%M_%S`    // 备份文件将以备份时间保存
DB_USER=<USER>                    // 数据库操作员
DB_PASS=<PASSWORD>                // 数据库操作员密码
DAYS=14                           // 保留最新14天的备份
TAR_BAK="mongod_bak_$DATE.tar.gz" // 备份文件命名格式
cd $OUT_DIR                       // 创建文件夹
rm -rf $OUT_DIR/*                 // 清空临时目录
mkdir -p $OUT_DIR/$DATE           // 创建本次备份文件夹
$DUMP -u $DB_USER -p $DB_PASS -o $OUT_DIR/$DATE  // 执行备份命令
tar -zcvf $TAR_DIR/$TAR_BAK $OUT_DIR/$DATE       // 将备份文件打包放入正式目录
find $TAR_DIR/ -mtime +$DAYS -delete             // 删除14天前的旧备份
```

**脚本权限改为可执行**

```
chmod +x ~/crontab/mongod_bak.sh
```

**使用linux crontab命令自动运行**

```
vi /etc/crontab
```

**添加以下内容**

```
每天凌晨02:00以 root 身份运行备份数据库的脚本。
0 2 * * * root ~/crontab/mongod_bak.sh
```

```
//然后重启 crond 使其生效：
/bin/systemctl restart  crond.service
```
```
// 设为开机启动
chkconfig crond on   
```


#### 文件传输
```
scp -P 29698 Desktop/mongo.tar root@66.98.123.33:/root
```


#### 文件压缩
```
tar -cvf tutorial.tar tutorial/
```

```
tar -zcvf tutorial.tar.gz tutorial/
```

#### 文件解压
```
tar -xvf tutorial.tar
```
```
tar -zxvf tutorial.tar.gz
```



