# Hadoop和spark常用端口限制

## 安装iptables-services

```shell
sudo yum install -y iptables-services
sudo systemctl start iptables
sudo systemctl enable iptables
```

## 拒绝策略

```shell
# 允许来自特定IP的访问
sudo iptables -A INPUT -p tcp -s 10.0.97.106 --dport 9870 -j ACCEPT
# 拒绝所有其他IP访问Hadoop Ui9870端口
sudo iptables -A INPUT -p tcp --dport 9870 -j DROP

# 允许来自特定IP的访问
sudo iptables -A INPUT -p tcp -s 10.0.97.106 --dport 8088 -j ACCEPT
# 拒绝所有其他IP访问Yarn Ui 8088端口
sudo iptables -A INPUT -p tcp --dport 8088 -j DROP

# 允许来自特定IP的访问
sudo iptables -A INPUT -p tcp -s 10.0.97.106 --dport 8080 -j ACCEPT
# 拒绝所有其他IP访问Spark Master Ui8080端口
sudo iptables -A INPUT -p tcp --dport 8080 -j DROP

# 允许来自特定IP的访问
sudo iptables -A INPUT -p tcp -s 10.0.97.106 --dport 8081 -j ACCEPT
# 拒绝所有其他IP访问Spark worker Ui 8081端口
sudo iptables -A INPUT -p tcp --dport 8081 -j DROP

# 允许来自特定IP的访问
sudo iptables -A INPUT -p tcp -s 10.0.97.106 --dport 4000 -j ACCEPT
# 拒绝所有其他IP访问Spark Log Ui 4000端口
sudo iptables -A INPUT -p tcp --dport 4000 -j DROP

# 允许来自特定IP的访问
sudo iptables -A INPUT -p tcp -s 10.0.97.106 --dport 80 -j ACCEPT
# 拒绝所有其他IP访问nginx 80端口
sudo iptables -A INPUT -p tcp --dport 80 -j DROP

sudo service iptables save
sudo iptables -L INPUT -n -v --line-numbers
sudo iptables -D INPUT 5
```

