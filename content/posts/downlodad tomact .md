---
title: "linux 部署tomact"
date: 2026-04-17
draft: false          # 必须为 false 才会发布
categories: ["部署步骤"]
tags: ["Linux", "tomact"]
author: "kkorcicv"
toc: true
---


本文基于 CentOS 7 系统，全程实操验证，手把手带你完成 Tomcat 9 完整部署。教程覆盖**yum源修复、JDK环境配置、安装包下载校验、服务启停、开机自启、端口开放、项目部署**全流程，解决了CentOS 7官方源失效、安装包404、javac命令缺失、远程访问失败等所有新手常见坑，所有命令可直接复制使用，零基础也能一次成功。


## 一、环境说明
- **操作系统**：CentOS 7 x86_64
- **Tomcat版本**：9.0.100（稳定生产版，兼容JDK 8+）
- **JDK版本**：OpenJDK 1.8.0（完整开发版）
- **安装方式**：离线tar包手动安装
- **适配场景**：VMware虚拟机、云服务器通用

## 二、前置准备：修复CentOS 7 yum源（必做）
CentOS 7 官方源已于2024年6月30日永久下架，不修复源会导致所有yum安装命令失败，先执行以下命令一键切换阿里云Vault归档源。

### 1. 修复DNS解析（解决无法解析主机问题）
```bash
# 备份原DNS配置
cp /etc/resolv.conf /etc/resolv.conf.bak

# 配置阿里云公共DNS，稳定无解析失败
cat > /etc/resolv.conf << EOF
nameserver 223.5.5.5
nameserver 223.6.6.6
EOF
```

### 2. 替换为阿里云 CentOS 7 Vault 源

```
# 备份原有yum源
mkdir -p /etc/yum.repos.d/bak
mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/bak/

# 写入阿里云Vault源配置
cat > /etc/yum.repos.d/CentOS-Base.repo << EOF
[base]
name=CentOS-7 - Base - Aliyun Vault
baseurl=https://mirrors.aliyun.com/centos-vault/7.9.2009/os/x86_64/
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/centos-vault/RPM-GPG-KEY-CentOS-7
enabled=1

[updates]
name=CentOS-7 - Updates - Aliyun Vault
baseurl=https://mirrors.aliyun.com/centos-vault/7.9.2009/updates/x86_64/
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/centos-vault/RPM-GPG-KEY-CentOS-7
enabled=1

[extras]
name=CentOS-7 - Extras - Aliyun Vault
baseurl=https://mirrors.aliyun.com/centos-vault/7.9.2009/extras/x86_64/
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/centos-vault/RPM-GPG-KEY-CentOS-7
enabled=1
EOF

# 清理并重新生成yum缓存
yum clean all
yum makecache
```

## 三、安装 JDK 8 环境（Tomcat 运行必备）

Tomcat 是 Java 开发的 Web 服务器，必须依赖 JDK 环境才能运行，这里安装**完整开发版 JDK**，包含运行环境和编译工具，解决`javac: 未找到命令`问题。

### 1. 一键安装 JDK 8 完整开发包

```
yum install java-1.8.0-openjdk-devel -y
```

### 2. 验证安装是否成功

必须同时验证`java`和`javac`两个命令，均输出版本信息即为安装成功：

```
java -version
javac -version
```

**成功输出示例**：

```
openjdk version "1.8.0_412"
OpenJDK Runtime Environment (build 1.8.0_412-b08)
OpenJDK 64-Bit Server VM (build 25.412-b08, mixed mode)
javac 1.8.0_412
```

## 四、下载 Tomcat 安装包（解决 404 问题）

使用国内华为云镜像站，解决 Apache 官方镜像下载慢、旧版本下架 404 问题，提供两种下载方式任选其一。

### 方式一：系统自带 wget 下载（简单直接，无需额外安装）

```
wget https://mirrors.huaweicloud.com/apache/tomcat/tomcat-9/v9.0.100/bin/apache-tomcat-9.0.100.tar.gz
```

### 方式二：axel 多线程下载（速度更快，推荐）

```
# 先安装EPEL扩展源
yum install epel-release -y --nogpgcheck
# 安装axel多线程下载工具
yum install axel -y --nogpgcheck
# 多线程下载Tomcat
axel -n 15 https://mirrors.huaweicloud.com/apache/tomcat/tomcat-9/v9.0.100/bin/apache-tomcat-9.0.100.tar.gz
```

### 必做：校验安装包完整性

避免安装包下载不完整、被篡改导致部署失败，下载完成后执行 MD5 校验：

```
md5sum apache-tomcat-9.0.100.tar.gz
```

**正确输出结果**：

```
5c0331b5a52508aca5944061f4224fb4  apache-tomcat-9.0.100.tar.gz
```

输出的 32 位 MD5 值与上述一致，代表安装包完整无损，可放心使用；不一致需重新下载。

## 五、解压并移动安装目录

将安装包解压后，移动到 Linux 标准软件安装目录`/usr/local/tomcat`，方便后续管理。

### 1. 解压压缩包

```
tar -zxvf apache-tomcat-9.0.100.tar.gz
```

### 2. 移动并重命名目录

```
mv apache-tomcat-9.0.100 /usr/local/tomcat
```

### 3. 验证目录

```
ll /usr/local/tomcat
```

输出`bin、conf、webapps、logs`等核心文件夹，即为解压移动成功。

## 六、创建专用用户与权限配置（安全规范）

不推荐直接使用 root 用户运行 Tomcat，创建专用运行用户并授权，提升服务安全性，符合生产环境规范。

### 1. 创建 tomcat 用户组和专用用户

```
groupadd tomcat
useradd -r -g tomcat tomcat
```

### 2. 给 Tomcat 目录完整授权

```
chown -R tomcat:tomcat /usr/local/tomcat
chmod -R 755 /usr/local/tomcat
```

## 七、核心配置说明（新手必看）

Tomcat 核心配置均在`/usr/local/tomcat/conf`目录下，以下是新手最常用的配置项：

### 1. 核心配置文件说明

|     配置文件     |                      作用                       |
| :--------------: | :---------------------------------------------: |
|    server.xml    | Tomcat 核心服务配置，修改端口、字符集、主机配置 |
| tomcat-users.xml |             管理后台用户与权限配置              |
|     web.xml      |                全局 Web 应用配置                |

### 2. 常用修改：更改默认端口 + 解决中文乱码

Tomcat 默认端口为 8080，如需修改或解决中文乱码，编辑`server.xml`文件：

```
vi /usr/local/tomcat/conf/server.xml
```

找到如下配置，修改`port`值即可更换端口，新增`URIEncoding="UTF-8"`解决中文乱码：

```
<Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" URIEncoding="UTF-8" />
```

修改完成后，需重启 Tomcat 服务才能生效。

## 八、服务启停与 systemd 配置（开机自启）

提供两种服务管理方式，新手可先用脚本启停，生产环境推荐使用 systemd 管理，支持进程守护、开机自启。

### 方式一：脚本手动启停

Tomcat 启停脚本均在`/usr/local/tomcat/bin`目录下：

```
# 启动Tomcat
/usr/local/tomcat/bin/startup.sh

# 停止Tomcat
/usr/local/tomcat/bin/shutdown.sh

# 查看运行状态
ps -ef | grep tomcat
```

启动成功会输出`Tomcat started.`，可通过进程命令验证是否正常运行。

### 方式二：systemd 系统服务配置（推荐，支持开机自启）

CentOS 7 推荐使用 systemd 统一管理服务，可实现开机自启、异常自动重启、统一启停命令。

#### 1. 创建 systemd 服务文件

```
cat > /usr/lib/systemd/system/tomcat.service << EOF
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target syslog.target

[Service]
Type=forking
User=tomcat
Group=tomcat
Environment="JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk"
PIDFile=/usr/local/tomcat/tomcat.pid
ExecStart=/usr/local/tomcat/bin/startup.sh
ExecStop=/usr/local/tomcat/bin/shutdown.sh
Restart=always
RestartSec=10
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
```

#### 2. 配置 PID 文件（配合服务管理）

```
cat > /usr/local/tomcat/bin/setenv.sh << EOF
CATALINA_PID="/usr/local/tomcat/tomcat.pid"
EOF

# 赋予执行权限并重新授权
chmod +x /usr/local/tomcat/bin/setenv.sh
chown -R tomcat:tomcat /usr/local/tomcat
```

#### 3. 重载 systemd 配置

```
systemctl daemon-reload
```

#### 4. 服务常用管理命

```
# 启动Tomcat
systemctl start tomcat
# 停止Tomcat
systemctl stop tomcat
# 重启Tomcat
systemctl restart tomcat
# 查看运行状态
systemctl status tomcat
# 设置开机自启
systemctl enable tomcat
# 关闭开机自启
systemctl disable tomcat
```

## 九、防火墙配置与访问验证

默认防火墙会拦截 Tomcat 端口，需开放端口才能通过浏览器访问。

### 1. 防火墙开放 8080 端口



```
# 永久开放8080端口
firewall-cmd --zone=public --add-port=8080/tcp --permanent
# 重载防火墙规则
firewall-cmd --reload
# 验证端口是否开放成功
firewall-cmd --list-ports
```

> 若修改了 Tomcat 默认端口，需将命令中的 8080 替换为你的自定义端口。

### 2. 访问验证

- 虚拟机本机访问：`http://127.0.0.1:8080`
- 宿主机 / 远程访问：`http://服务器IP:8080`
- 示例访问地址：`http://192.168.174.129:8080`

浏览器访问出现 Tomcat 官方默认首页，即为部署成功！

## 十、管理后台配置（可选）

Tomcat 自带 Web 管理后台，可在线管理 Web 应用，需配置用户权限与远程访问限制。

### 1. 配置管理员用户

编辑用户配置文件：

```
vi /usr/local/tomcat/conf/tomcat-users.xml
```

在`</tomcat-users>`标签前，添加如下配置（可自定义用户名和密码）：

```
<role rolename="manager-gui"/>
<role rolename="admin-gui"/>
<user username="admin" password="123456" roles="manager-gui,admin-gui"/>
```

### 2. 放开远程访问限制

默认 Tomcat 仅允许本机访问管理后台，需修改配置放开限制：

```
# 放开Manager后台远程访问限制
vi /usr/local/tomcat/webapps/manager/META-INF/context.xml
# 放开Host Manager后台远程访问限制
vi /usr/local/tomcat/webapps/host-manager/META-INF/context.xml
```

将文件中如下配置注释掉（前后添加`<!--`和`-->`）：

```
<!--
<Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
-->
```

### 3. 重启 Tomcat 生效

```
systemctl restart tomcat
```

重启后，访问`http://服务器IP:8080/manager/html`，输入配置的用户名密码，即可进入管理后台。

## 十一、Web 项目部署

Tomcat 最常用的项目部署方式为 war 包自动部署，操作简单，新手零门槛。

### 1. 自动部署（推荐）

将打包好的`xxx.war`项目文件，上传到 Tomcat 的`webapps`目录：

```
# 示例：将war包移动到webapps目录
mv xxx.war /usr/local/tomcat/webapps/
```

Tomcat 会自动解压 war 包并完成部署，无需重启服务，直接访问`http://服务器IP:8080/xxx`即可进入项目。

### 2. 根路径部署（无需项目名访问）

如需直接通过`http://服务器IP:8080`访问项目，编辑`server.xml`文件：

```
vi /usr/local/tomcat/conf/server.xml
```

在`Host`标签内添加如下配置：

```
<Context path="" docBase="/usr/local/tomcat/webapps/xxx" debug="0" reloadable="true" />
```

> `docBase`为你的项目解压后的完整路径，修改后重启 Tomcat 即可生效。

## 十二、新手常见问题排查

**yum 安装命令失败，提示无法找到软件包**

优先检查是否完成了 CentOS 7 yum 源修复，官方源已永久失效，必须切换为 Vault 归档源。

**javac: 未找到命令**

安装的是仅含运行环境的 JRE，需执行`yum install java-1.8.0-openjdk-devel -y`安装完整开发版 JDK。

**Tomcat 下载链接 404**

Apache 镜像站会定期下架旧版本，更换为教程中的华为云镜像最新稳定版链接即可。

**浏览器无法访问 Tomcat 首页**

排查顺序：① Tomcat 服务是否正常运行 ② 防火墙是否开放对应端口 ③ 服务器 IP 是否可正常 ping 通 ④ 云服务器安全组是否放行端口。

**启动失败，提示端口被占用**

查看端口占用：`netstat -tulpn | grep 8080`，杀死占用进程，或修改 Tomcat 默认端口。

**项目中文乱码**

在`server.xml`的端口配置中添加`URIEncoding="UTF-8"`，重启 Tomcat 生效。

**管理后台访问 403 拒绝访问**

检查是否放开了远程访问限制，以及`tomcat-users.xml`中的用户权限是否配置正确。
