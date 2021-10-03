---
title: "每天一个linux命令（21）：find命令之xargs"
tags: [linux, command, find, xargs]
list_number: n
---

# 1. exec执行的问题
## 1.1 参数太长
在使用 `find`命令的`-exec`选项处理匹配到的文件时， `find`命令将所有匹配到的文件一起传递给`exec`执行。但有些系统对能够传递给`exec`的命令长度有限制，这样在`find`命令运行几分钟之后，就会出现溢出错误。错误信息通常是“参数列太长”或“参数列溢出”。这就是`xargs`命令的用处所在，特别是与`find`命令一起使用。  

`find`命令把匹配到的文件传递给`xargs`命令，而`xargs`命令每次只获取一部分文件而不是全部，不像`-exec`选项那样。这样它可以先处理最先获取的一部分文件，然后是下一批，并如此继续下去。  

## 1.2进程过多

在有些系统中，使用`-exec`选项会为处理每一个匹配到的文件而发起一个相应的进程，并非将匹配到的文件全部作为参数一次执行；这样在有些情况下就会出现进程过多，系统性能下降的问题，因而效率不高； 而使用`xargs`命令则只有一个进程。另外，在使用`xargs`命令时，究竟是一次获取所有的参数，还是分批取得参数，以及每一次获取参数的数目都会根据该命令的选项及系统内核中相应的可调参数来确定。

# 2. 实例

## 2.1 Demo

```sh
[root@localhost test]# find . -type f -print | xargs file
./log2014.log: empty
./log2013.log: empty
./log2012.log: ASCII text
[root@localhost test]#
```

## 2.2 查找文件中是否包含字符

```sh
find . -type f -print | xargs grep "hostname"
```

## 2.3 参数太长处理

argument line too long解决方法

```sh
[root@pd test4]#  find . -type f -atime +0 -print0 | xargs -0 -l1 -t rm -f
rm -f 
[root@pdtest4]#
```

`-l1`是一次处理一个；`-t`是处理之前打印出命令

## 2.4 修改结果集表示方式

`-i` 默认的前面输出用{}代替

`-I` 指定其他代替字符，如例子中的[] 

如将当前目录中的文件移动到上层, 下面两条命令都可以
```sh
find . -type f | xargs -i mv {} ..
```
使用了 `-i` 则前面`find`的结果默认用`{}` 表示

```sh
find . -type f | xargs -I [] mv [] ..
```
使用了 `-I []` 则前面`find`的结果使用指定的`[]` 表示


## 2.5 提示前确认
`-p` 参数会提示让你确认是否执行后面的命令,y执行，n不执行。

```sh
[root@localhost test3]#  find . -name "*.log" | xargs -p -i mv {} ..
mv ./log2015.log .. ?...y
[root@localhost test3]# ll
```

# 参考文献

- [每天一个linux命令（21）：find命令之xargs](http://www.cnblogs.com/peida/archive/2012/11/15/2770888.html)
