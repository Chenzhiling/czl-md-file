# 在不联网的服务器上，使用tar包安装mysql和redis

版本信息

- unbuntu18.04

- mysql5.7

- redis5.0.14

## 1、mysql tar包安装

### 1.1 解压缩，重命名

```shell
tar -zxvf mysql-5.7.43-linux-glibc2.12-x86_64.tar.gz -C /opt/

mv mysql-5.7.43-linux-glibc2.12-x86_64/ mysql5.7
```

### 1.2 创建data、log目录

```shell
cd mysql5.7
mkdir data log
```

### 1.3 创建mysql.log、mysql.pid、mysql.sock文件

```shell
cd log
touch mysql.log mysql.pid mysql.sock
```

### 1.4 创建mysql用户组并赋予mysql5.7目录权限

```shell
cd ../../
groupadd mysql
useradd -r -g mysql mysql
chown -R mysql:mysql mysql5.7/
```

### 1.5 etc目录下新建或修改my.cnf和my.cnf.d文件

`my.cnf`信息如下

```shell
[client]
#客户端连接端口
port=3306
#客户端连接sock
socket=/opt/mysql5.7/log/mysql.sock
#客户端编码
default-character-set=utf8
 
[mysqld]
#mysql服务端口
port=3306
#安装目录
basedir=/opt/mysql5.7
#数据存放目录
datadir=/opt/mysql5.7/data
#sock文件地址
socket=/opt/mysql5.7/log/mysql.sock
#错误日志存放地址
log-error=/opt/mysql5.7/log/mysql.log
#pid文件地址
pid-file=/opt/mysql5.7/log/mysql.pid
#服务端编码
character-set-server=utf8
 
!includedir /etc/my.cnf.d
```

### 1.6 初始化数据库

如果找不到`libaio1`，记得下载`apt-get install libaio1`

```shell
cd mysql5.7/bin
./mysqld --initialize --user=mysql --basedir=/opt/mysql5.7 --datadir=/opt/mysql5.7/data
```

### 1.7 启动mysql，并设置开机自启

```shell
cp support-files/mysql.server /etc/init.d/mysqld
systemctl enable mysqld
systemctl start mysqld
```

### 1.8 修改密码，赋予外部mysql权限

```shell
set password=password("root");
grant all privileges on *.* to root@'%' identified by 'root' with grant option;
flush privileges;
exit;
```



## 2、redis tar包安装

### 2.1 解压缩，重命名

```shell
tar -zxvf redis-5.0.14.tar.gz -C /opt/
mv redis-5.0.14 redis
```

### 2.2 编译

如果没有`make`和`gcc`,提前下载安装

```shell
make MALLOC=libc
```

如果出现`/deps/hiredis/libhiredis.a: 没有那个文件或目录`报错，执行以下命令

```shell
cd deps/
make lua hiredis linenoise
```



