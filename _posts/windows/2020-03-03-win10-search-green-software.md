---
title: "Windows10 中如何搜索绿色软件"
tags: [win10, windows, "搜索", cortana, "绿色软件"]
---

在系统中通过压缩包解压缩的方式存在很多的绿色软件，由于这些软件没有安装注册，所以直接通过 windows的 `cortana` 进行搜索是搜索不到的。怎样可以直接通过搜索的方式搜索到呢？查找了一下方法为：

- 在 `C:\ProgramData\Microsoft\Windows\Start Menu\Programs` 目录中创建对应子目录
- 将绿色软件可执行程序的快捷方式复制一个到新建的目录中

比如我新建了 `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\vscode1` 目录，然后将vscode的快捷方式 `vscode` 放到目录中，这样就可以直接通过搜索 `vscode` 搜索到了。

> 这里用 `vscode1` 是搜索不到的，我之前就是将快捷方式的名字写错了，所以一直都搜索不成功。

另外添加这个目录后在 `开始` 菜单中也会出现vscode这个选项。可以通过在选项上右键 `固定到开始屏幕` 将其添加到开始菜单的贴片中。

# References

- [如何让cortana搜索代替win+R/wox等快速启动功能？](https://meta.appinn.net/t/cortana-win-r-wox/1484/9)
