# linux 服务优化

------

## 修改单进程最大句柄数

```
vi /etc/security/limits.conf
#在文件末尾加上 * 代表所有用户 soft代表
* soft nofile 1000000
* hard nofile 1000000
```

## 修改全局文件句柄限制

```
#单次修改
cat /proc/sys/fs/file-max
echo 1000000 > /proc/sys/fs/file-max

#永久修改
vi /etc/sysctl.conf
fs.file-max=1000000
#修改生效
sysctl -p
```

## 修改网络连接限制（有k8s的时候会有iptable连接数限制）

```
# 查看最大值
# wc -l /proc/net/nf_conntrack
# cat /proc/sys/net/nf_conntrack_max
1024000
# vi /etc/sysctl.conf
net.nf_conntrack_max = 2000500
# or
sysctl -w net.nf_conntrack_max=2000500
```