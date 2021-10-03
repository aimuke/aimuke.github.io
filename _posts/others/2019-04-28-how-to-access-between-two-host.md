---
title: "同一路由器下两台主机不能互相访问"
tags: ["win7", "网络", "ping", "防火墙"]

---

**2013年10月01日 00:00:29 阳光岛主**

Windows7出于安全考虑，默认情况下是不允许外部主机对其进行Ping测试的。但在一个安全的局域网环境中，Ping测试又是管理员进行网络测试所必须的，如何允许 Windows 7的ping测试回显呢? 

# 查看、开启或禁用系统防火墙 

打开命令提示符输入输入命令`netsh firewall show state`，然后回车可查看防火墙的状态
```
C:\Users\aaa>netsh firewall show state

防火墙状态:
-------------------------------------------------------------------
配置文件                          = 标准
操作模式                          = 启用
例外模式                          = 启用
多播/广播响应模式                 = 启用
通知模式                          = 启用
组策略版本                        = Windows 防火墙
远程管理模式                      = 禁用

所有网络接口上的端口当前均为打开状态:
端口   协议  版本  程序
-------------------------------------------------------------------
当前没有在所有网络接口上打开的端口。

重要信息: 已成功执行命令。
但不赞成使用 "netsh firewall"；
而应该使用 "netsh advfirewall firewall"。
有关使用 "netsh advfirewall firewall" 命令
而非 "netsh firewall" 的详细信息，请参阅
http://go.microsoft.com/fwlink/?linkid=121488
上的 KB 文章 947709。

```


从显示结果中，可看到防火墙各功能模块的禁用及启用情况。

命令`netsh firewall set opmode disable`用来禁用系统防火墙

命令`netsh firewall set opmode enable`可启用防火墙。



# 允许文件和打印共享 

文件和打印共享在局域网中常用的，如果要允许客户端访问本机的共享文件或者打印机，可分别输入并执行如下命令： 
```
netsh firewall add portopening UDP 137 Netbios-ns (允许客户端访问服务器UDP协议的137端口) 

netsh firewall add portopening UDP 138 Netbios-dgm (允许访问UDP协议的138端口) 

netsh firewall add portopening TCP 139 Netbios-ssn (允许访问TCP协议的139端口) 

netsh firewall add portopening TCP 445 Netbios-ds (允许访问TCP协议的445端口) 
```
命令执行完毕后，文件及打印共享所须的端口都被防火墙放行了。

# 允许ICMP回显 

默认情况下，Windows7出于安全考虑是不允许外部主机对其进行Ping测试的。但在一个安全的局域网环境中，Ping测试又是管理员进行网络测试所必须的，如何允许 Windows 7的ping测试回显呢? 

当然，通过系统防火墙控制台可在“入站规则”中将“文件和打印共享(回显请求– ICMPv4-In)”规则设置为允许即可，如果网络使用了 IPv6，则同时要允许 ICMPv6-In 的规则。

具体步骤： 
> 控制面板 ——》Windows 防火墙 ——》 高级设置 ——》 入站规则(Inbound Rules) ——》 文件和打印共享（File and Printer Sharing），如下图：



在上图，选中“File and Printer Sharing(Echo Request - ICMPv4-In”，右键属性（双击也行），勾选“Eanbled”，如下图：



主机开启防护墙的情况下，在虚拟机Ubuntu的终端中，ping主机（Host），ping通有回显了：


我们在命令行下通过netsh命令可快速实现。执行命令“netsh firewall set icmpsetting 8”可开启ICMP回显，反之执行“netsh firewall set icmpsetting 8 disable”可关闭回显。

# 参考文献

[win7 防火墙开启ping](https://blog.csdn.net/ithomer/article/details/12191283)
