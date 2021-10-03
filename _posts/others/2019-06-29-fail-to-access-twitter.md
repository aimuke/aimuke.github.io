---
title: "无法访问twitter的解决方法"
tags: [error, twitter]
---


# 场景

在google等网站能够正常访问的情况下，在win7浏览器中访问 twitter，发现网站不能访问。

# 解决方法

1. 清空本地DNS缓存（开始->运行->输入“CMD”->回车->运行“ipconfig /flushdns”命令）
2. 将本地网络连接的DNS服务器修改为Google的DNS: 8.8.8.8和8.8.4.4设置google dns

事实上我只执行了第2步，就可以访问了

# 参考 

- [原文地址](http://www.yuyuhunter.com/post/vpn-facebook-twitter.html)
