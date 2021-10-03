---
title: 为docker设置代理
tags: [docker, proxy]
---

# 创建目录

```sh
mkdir -p /etc/systemd/system/docker.service.d
```

# 创建文件

/etc/systemd/system/docker.service.d/http-proxy.conf
内容如下：
```sh
[Service]
Environment="HTTP_PROXY=http://pill:pill@node2:3128/"
```
# 重启docker
```sh
systemctl daemon-reload
systemctl restart docker
```
# 验证docker代理是否设置成功
```sh
systemctl show --property=Environment docker　　
```
显示如下结果说明设置成功
```sh
Environment=GOTRACEBACK=crash HTTP_PROXY=http://pill:pill@node2:3128/
```
# 参考文献
- [为docker配置HTTP代理服务器](https://www.cnblogs.com/lixiaolun/p/7449017.html)
