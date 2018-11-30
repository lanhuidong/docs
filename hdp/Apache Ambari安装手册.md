## 一、服务器环境准备

### 1.1 设置最大文件打开数

- 编辑/etc/security/limits.conf， 加入以下配置

  ```
  * soft nofile 100000
  * hard nofile 100000
  ```

- 通过如下命令查看结果

  ```shell
  #ulimit -Sn
  #ulimit -Hn
  ```

- 编辑/etc/sysctl.conf，加入以下配置

  ```
  fs.file-max=100000
  ```

- 执行如下命令使配置生效

  ```shell
  #/sbin/sysctl -p
  ```

- 通过如下命令查看结果

  ```shell
  #cat /proc/sys/fs/file-max
  ```

### 1.2 修改hostname

- 编辑/etc/cloud/cloud.cfg(ubuntu 18)

  ```
  preserve_hostname: true
  ```

- 编辑/etc/sysconfig/network(CentOS)

  ```
  NETWORKING=yes
  HOSTNAME=ambari.nexusy.com
  ```

- 编辑/etc/hostname

  ```
  ambari.nexusy.com
  ```

- 编辑/etc/hosts

  ```
  172.16.16.224 ambari.nexusy.com
  ```

### 1.3 设置无密码SSH

- 在Ambari服务器主机上生产SSH公钥和私钥

  ```shell
  #ssh-keygen
  ```

- 复制SSH公钥(id_rsa.pub)到目标主机的root账户的~/.ssh目录下

- 添加SSH公钥到authorized_keys文件中

  ```shell
  #cat id_rsa.pub >> authorized_keys
  ```

- 修改~/.ssh和authorized_keys文件的权限

  ```shell
  #chmod 700 ~/.ssh
  #chmod 600 ~/.ssh/authorized_keys
  ```

### 1.4 启用NTP

CentOS

```shell
#yum install -y ntp
#systemctl enable ntpd
```

Ubuntu

```shell
#apt-get install ntp
#update-rc.d ntp defaults
```

### 1.5 禁用iptables

CentOS

```shell
#systemctl disable firewalld
#service firewalld stop
```

Ubuntu

```shell
#sudo ufw disable
#sudo iptables -X
#sudo iptables -t nat -F
#sudo iptables -t nat -X
#sudo iptables -t mangle -F
#sudo iptables -t mangle -X
#sudo iptables -P INPUT ACCEPT
#sudo iptables -P FORWARD ACCEPT
#sudo iptables -P OUTPUT ACCEPT
```

### 1.6 禁用SELinux

- 临时禁用

  ```shell
  #setenforce 0
  ```


- 永久禁用

  编辑/etc/selinux/config文件

  ```
  SELINUX=disabled
  ```

### 1.7 禁用PackageKit（ubuntu默认未启用）

编辑/etc/yum/pluginconf.d/refresh-packagekit.con

```
enable=0
```

### 1.8 检查并设置umask

- 检查，如果umask为022(0022)或者027(0027)，则无需修改

  ```shell
  #umask
  ```

- 设置当前会话

  ```shell
  #umask 0022
  ```

- 永久设置

  ```shell
  #echo umask 0022 >> /etc/profile
  ```

### 1.9 安装JDK和JCE

- 将local_policy.jar和US_export_policy.jar放到JAVA_HOME/jre/lib/security目录

### 1.10 配置数据库(本文使用MySQL)

- 创建用户

  ```
  CREATE USER 'rangerdba'@'localhost' IDENTIFIED BY 'rangerdba';
  GRANT ALL PRIVILEGES ON *.* TO 'rangerdba'@'localhost';
  CREATE USER 'rangerdba'@'%' IDENTIFIED BY 'rangerdba';
  GRANT ALL PRIVILEGES ON *.* TO 'rangerdba'@'%';
  GRANT ALL PRIVILEGES ON *.* TO 'rangerdba'@'localhost' WITH GRANT OPTION;
  GRANT ALL PRIVILEGES ON *.* TO 'rangerdba'@'%' WITH GRANT OPTION;
  FLUSH PRIVILEGES;
  ```


- 下载JDBC驱动mysql-connector-java.jar

- 创建数据库

  ```
  CREATE DATABASE registry;
  CREATE DATABASE streamline;
  CREATE USER 'registry'@'%' IDENTIFIED BY 'R12$%34qw';
  CREATE USER 'streamline'@'%' IDENTIFIED BY 'R12$%34qw';
  GRANT ALL PRIVILEGES ON registry.* TO 'registry'@'%' WITH GRANT OPTION ;
  GRANT ALL PRIVILEGES ON streamline.* TO 'streamline'@'%' WITH GRANT OPTION ;
  CREATE DATABASE druid DEFAULT CHARACTER SET utf8;
  CREATE DATABASE superset DEFAULT CHARACTER SET utf8;
  CREATE USER 'druid'@'%' IDENTIFIED BY '9oNio)ex1ndL';
  CREATE USER 'superset'@'%' IDENTIFIED BY '9oNio)ex1ndL';
  GRANT ALL PRIVILEGES ON *.* TO 'druid'@'%' WITH GRANT OPTION;
  GRANT ALL PRIVILEGES ON *.* TO 'superset'@'%' WITH GRANT OPTION;
  ```

## 二、配置本地仓库

### 2.1 安装HTTP服务器

可以选择apache httpd或者nginx

### 2.2 下载安装包

- [ambari-2.7.1.0-ubuntu16.tar.gz](http://public-repo-1.hortonworks.com/ambari/ubuntu16/2.x/updates/2.7.1.0/ambari-2.7.1.0-ubuntu16.tar.gz)
- [HDP-3.0.1.0-ubuntu16-deb.tar.gz](http://public-repo-1.hortonworks.com/HDP/ubuntu16/3.x/updates/3.0.1.0/HDP-3.0.1.0-ubuntu16-deb.tar.gz)
- [HDP-GPL-3.0.1.0-ubuntu16-gpl.tar.gz](http://public-repo-1.hortonworks.com/HDP-GPL/ubuntu16/3.x/updates/3.0.1.0/HDP-GPL-3.0.1.0-ubuntu16-gpl.tar.gz)
- [HDP-UTILS-1.1.0.22-ubuntu16.tar.gz](http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.22/repos/ubuntu16/HDP-UTILS-1.1.0.22-ubuntu16.tar.gz)

### 2.3 解压安装包到HTTP服务器根目录

```
/nginx/html/ambari/ubuntu16/2.7.1.0-169
/nginx/html/HDP/ubuntu16/3.0.1.0-187
/nginx/html/HDP-GPL/ubuntu16/3.0.1.0-187
/nginx/html/HDP-UTILS/ubuntu16/1.1.0.22
```

## 三、安装Ambari

###  3.1 下载Ambari仓库

```shell
#wget -O /etc/apt/sources.list.d/ambari.list http://public-repo-1.hortonworks.com/ambari/ubuntu16/2.x/updates/2.7.1.0/ambari.list
#apt-key adv --recv-keys --keyserver keyserver.ubuntu.com B9733A7A07513CAD
#apt-get update
```

确认是否成功

```shell
#apt-cache showpkg ambari-server
#apt-cache showpkg ambari-agent
#apt-cache showpkg ambari-metrics-assembly
```

### 3.2 下载HDP仓库

```shell
#wget -O /etc/apt/sources.list.d/hdp.list http://public-repo-1.hortonworks.com/HDP/ubuntu16/3.x/updates/3.0.1.0/hdp.list
```

### 3.3 编辑仓库文件

- /etc/apt/source.list.d/ambari.list

  ```
  #VERSION_NUMBER=2.7.1.0-169
  #json.url = http://public-repo-1.hortonworks.com/HDP/hdp_urlinfo.json
  deb http://ambari.nexusy.com/ambari/ubuntu16/2.7.1.0-169 Ambari main
  ```


- /etc/apt/source.list.d/hdp.list

  ```
  #VERSION_NUMBER=3.0.1.0-187
  deb http://ambari.nexusy.com/HDP/ubuntu16/3.0.1.0-187 HDP main
  deb http://ambari.nexusy.com/HDP-GPL/ubuntu16/3.0.1.0-187 HDP-GPL main
  deb http://ambari.nexusy.com/HDP-UTILS/ubuntu16/1.1.0.22 HDP-UTILS main
  ```

### 3.4  安装

```shell
#apt-get install ambari-server
#ambari-server setup
```

使用/var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql创建数据库

### 3.5 启动/停止/检查状态

```shell
#ambari-server start
#ambari-server status
#ambari-server stop
```

### 3.6 登录Ambari

默认用户名/密码（admin/admin）

```
http://<your.ambari.server>:8080
```