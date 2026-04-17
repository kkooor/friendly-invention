---
title: "how to download mysql in centos 7"
date: 2026-04-17
draft: false          
categories: ["技术笔记"]
tags: ["Linux", "MySQL", "CentOS", "数据库安装"]
author: "kkorcicv"
toc: true
---

```
# CentOS 7 安装 MySQL 5.7.37 完整教程

## 一、下载 MySQL 安装包
```bash
yum install axel -y
```

下载 MySQL 5.7.37：

```
axel -n 15 https://mirrors.tuna.tsinghua.edu.cn/mysql/downloads/MySQL-5.7/mysql-5.7.37-el7-x86_64.tar.gz
```

小技巧：卡顿直接 Ctrl+C，重新执行命令支持断点续传。

## 三、解压并移动安装目录

解压压缩包，移动至 Linux 标准软件目录 `/usr/local/mysql`。

解压文件：

```
tar -zxvf mysql-5.7.37-el7-x86_64.tar.gz
```

移动并重命名：

```
mv mysql-5.7.37-el7-x86_64 /usr/local/mysql
```

验证目录：

```
ll /usr/local/mysql
```

## 四、卸载系统冲突组件

CentOS 7 自带 mariadb，与 MySQL 冲突，必须卸载。

卸载命令：

```
rpm -e --nodeps mariadb-libs
```

## 五、创建专用用户与数据目录

MySQL 禁止 root 直接运行，需创建专用用户并授权数据目录。

创建用户组与用户：

```
groupadd mysql
useradd -r -g mysql mysql
```

创建数据目录并授权：

```
mkdir -p /data/mysql
chown -R mysql:mysql /data/mysql
chmod -R 755 /data/mysql
```

## 六、配置 MySQL 核心文件

生成 `/etc/my.cnf` 配置文件，直接复制执行即可。

配置命令：

```
cat > /etc/my.cnf << EOF
[mysqld]
basedir=/usr/local/mysql
datadir=/data/mysql
socket=/tmp/mysql.sock
user=mysql
port=3306
character-set-server=utf8
default_storage_engine=InnoDB
max_connections=200
symbolic-links=0

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
EOF
```

## 七、初始化 MySQL（生成临时密码）

初始化数据库，生成 root 临时登录密码，务必保存。

进入目录：

```
cd /usr/local/mysql/bin
```

执行初始化：

```
./mysqld --initialize --user=mysql --datadir=/data/mysql --basedir=/usr/local/mysql
```

查看临时密码：

```
cat /var/log/mysqld.log | grep password
```

示例输出：

```
A temporary password is generated for root@localhost: eaDg3wsfjw:j
```

## 八、配置全局环境变量

解决 `bash: mysql: 未找到命令` 问题。

配置命令：

```
echo "export PATH=$PATH:/usr/local/mysql/bin" >> /etc/profile
source /etc/profile
```

## 九、启动服务并设置开机自启

配置系统服务，启动 MySQL 并设置开机自启。

配置启动脚本：

```
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
chmod +x /etc/init.d/mysqld
```

启动 MySQL：

```
service mysqld start
```

设置开机自启：

```
chkconfig --add mysqld
chkconfig mysqld on
```

## 十、登录并修改 root 密码

使用临时密码登录，修改为自定义密码。

登录 MySQL：

```
mysql -uroot -p
```

修改密码（示例：123456）：

```
ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';
FLUSH PRIVILEGES;
exit;
```

验证登录：

```
mysql -uroot -p123456
```

## 十一、配置远程连接

允许 Navicat 等工具远程连接，开放防火墙端口。

授权远程访问：

sql

```
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;
FLUSH PRIVILEGES;
exit;
```

开放 3306 端口：

```
firewall-cmd --add-port=3306/tcp --permanent
firewall-cmd --reload
```
