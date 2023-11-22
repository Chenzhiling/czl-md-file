# Ubuntu使用tar包安装nginx

简单记录nginx安装过程

## 版本

- `ubuntu` 18.04
- `nginx` 1.24.0

## 解压缩

```shell
sudo tar -zxvf nginx-1.24.0.tar -C /opt/nginx-1.24.0
cd /opt/nginx-1.24.0
```

## 安装

```shell
全部采用默认安装
sudo ./configure
sudo make && make install
```

## 如果缺包，安装相关依赖

```shell
sudo apt-get install libpcre3-dev
sudo apt-get install zlib1g-dev
```

## 启动

```shell
cd /usr/local/nginx/sbin
./nginx
```

