# Linux

### CentOS 删除软件包

```bash
# 以删除 mongodb-org 为例
sudo yum erase $(rpm -qa | grep mongodb-org)
```

### CentOS 删除 systemctl 服务

> 来自：[https://superuser.com/a/936976](https://superuser.com/a/936976)

```bash
systemctl stop [servicename]
systemctl disable [servicename]
rm /etc/systemd/system/[servicename]
rm /etc/systemd/system/[servicename] # and symlinks that might be related
rm /usr/lib/systemd/system/[servicename] 
rm /usr/lib/systemd/system/[servicename] # and symlinks that might be related
systemctl daemon-reload
systemctl reset-failed
```

