---
title: "cp复制文件目的文件权限"
tags: [linux, command, cp]
---

在脚本调试过程中发现一些由于文件权限导致的脚本执行错误问题。经过研究后发现出问题的原因是自己对linux 系统中文件系统的理解不够。现将过程及学习结果记录如下。

# 起因

在生成 docker 镜像的时候，准备了一个脚本，用于准备 `docker build` 所需要的文件。先将所有的文件放到一个临时目录下，并且设置了文件的相应权限。在 `docker build` 之前在将临时目录下的文件复制到需要的目录下。如下：

```sh
# 准备文件,并将文件设置为对应的权限
$ mkdir tmp
$ cd tmp/
$ touch data.txt
$ ll
total 0
-rw-rw-r-- 1 ubuntu ubuntu 0 Jun  2 17:16 data.txt
$ chmod 666 data.txt
$ ll
total 0
-rw-rw-rw- 1 ubuntu ubuntu 0 Jun  2 17:16 data.txt

# 将准备好的文件复制到目标文件
$ cd ../
$ mkdir dest
$ cd dest/
$ cp ../tmp/* .
$ ll
total 0
-rw-rw-r-- 1 ubuntu ubuntu 0 Jun  2 17:16 data.txt
```

通过上述命令发现在复制到目标文件夹后，之前设置好的文件权限发生了变化。 `data.txt` 之前设置的权限为 `666` 复制后新文件的权限为 `664`。使得之前设置好的权限失去了意义。具体是什么原因呢？

# 复制文件的过程

我们先来试验一下看看

先看一下当前环境中的 `umask` . 请记住这个值，后面会详细解释这个值对文件权限的影响。

```sh
$ umask
0022
```

当目的文件不存在时

```sh
$ strace cp data1.txt data1.cp.txt 2>&1 | egrep data1
execve("/bin/cp", ["cp", "data1.txt", "data1.cp.txt"], [/* 24 vars */]) = 0
stat("data1.cp.txt", 0x7ffd843bc780)    = -1 ENOENT (No such file or directory)
stat("data1.txt", {st_mode=S_IFREG|0666, st_size=10, ...}) = 0
stat("data1.cp.txt", 0x7ffd843bc4e0)    = -1 ENOENT (No such file or directory)
open("data1.txt", O_RDONLY)             = 3
open("data1.cp.txt", O_WRONLY|O_CREAT|O_EXCL, 0666) = 4
read(3, "data1.txt\n", 65536)           = 10
write(4, "data1.txt\n", 10)             = 10

$ ll
total 40
-rw-r--r-- 1 root   root   10 Jun  4 10:48 data1.cp.txt
-rw-rw-rw- 1 ubuntu ubuntu 10 Jun  2 19:39 data1.txt
```

当文件存在时

```sh
$ strace cp data1.txt data1.cp.txt 2>&1 | egrep data1
execve("/bin/cp", ["cp", "data1.txt", "data1.cp.txt"], [/* 24 vars */]) = 0
stat("data1.cp.txt", {st_mode=S_IFREG|0644, st_size=10, ...}) = 0
stat("data1.txt", {st_mode=S_IFREG|0666, st_size=10, ...}) = 0
stat("data1.cp.txt", {st_mode=S_IFREG|0644, st_size=10, ...}) = 0
open("data1.txt", O_RDONLY)             = 3
open("data1.cp.txt", O_WRONLY|O_TRUNC)  = 4
read(3, "data1.txt\n", 65536)           = 10
write(4, "data1.txt\n", 10)             = 10

$ ll
total 40
-rw-r--r-- 1 root   root   10 Jun  4 10:49 data1.cp.txt
-rw-rw-rw- 1 ubuntu ubuntu 10 Jun  2 19:39 data1.txt
```

小结：

文件复制过程包含了两个阶段：

1. 通过 `O_RDONLY` 权限打开文件, 这一步说明要复制文件，当前用户至少需要源文件的 `READ` 权限。
2. 当目标文件存在时，直接清空原文件，重新写入内容，目标文件权限不修改；当目标文件不存在时，创建新文件，注意此处创建时传入的参数是复制时 `source` 文件的权限。

在前面的例子中，我们源文件 `data1.txt` 的权限是 `666`。创建目标文件的时候也是使用源文件的权限作为参数，为什么生成的目标文件是 `644` 权限呢。这个跟本节刚开始提到的 `umask` 有关。具体请看下节。

# umask

`umask` 是用来指定"目前用户在新建文件或者目录时候的权限默认值"。

在linux下我们查看的方式有两种，一种可以直接输入 `umask` ，就可以看到数字形态的权限设置分数，一种则是加 `-S`（Symbolic）参数，就能以符号类型的方式来显示出权限了，如下：

```sh
$ umask
0002
$ umask -S
u=rwx,g=rwx,o=rx
```

## 默认权限

目录和文件的默认权限属性是不同的，因为对于一个目录来说它的x权限也就是执行权限是很重要的，进入目录等操作都是需要目录具有执行权限的，而对于文件来说，一般情况都是用于数据的记录操作，所以一般不需要执行权限。从而，在linux下默认的情况是这样的：

- 如果用户创建的是目录，则默认所有权限都开放，为 `777` ，默认为： `drwxrwxrwx`
- 如果创建的是文件，默认没有 `x` 权限，那么就只有 `r` 、 `w` 两项，最大值为 `666` ，默认为：`-rw-rw-rw-`

那么之前所说的拿走的权限就是这里这个默认值要减掉的权限，`r`、`w`、`x`分别是`4`、`2`、`1`，要拿掉读权限就输入 `4` ，拿掉写权限就输入 `2` ，以此类推。

如果 `umask` 为 `022` ，也就是说，对于当前用户没有拿掉权限， `group` 用户和 `other` 用户都被拿走了 `w` 权限，所以此时如果用户进行创建目录和文件的时候，默认权限是会进行如下的减法操作：

- 新建文件：`666-022=644`, 文件的默认权限是 `-rw-r--r--（644）`
- 新建目录：`777-022=755`, 目录的默认权限是 `rwxr-xr-x（755）`，

> **注意**
>
> 上面在计算创建的文件和目录的默认权限的时候，用：`666-022=644`；`777-022=755`，但这并不是做了对应数字的加减，刚刚看到数字相减的结果和最后验证的结果是一样的，但这只是巧合而已。准确的做法应该是，文件的权限 **`去除`** `umask` 中指定的权限
>
> 比如： 当前 umask 为 0003, 那么新建文件的权限:
>
> - 做减法 `666-003=663`, others 用户组拥有 w，x 权限(错误)
> - 扣除法 `003` 代表需要扣除 `w` 和 `x` 权限。文件默认为 `666`,由于没有 `x`权限，只扣除 `w`,所以文件最终的权限应该是 `664`

## 设置umask

权限掩码的设置： `umask` 的设置只需要在 `umask` 命令后加想要拿掉的权限数字就行，Linux下的 `etc/profile` 和 `etc/bashrc` 中都有默认的 `umask` 设置，vim打开文件后，可以看到根据不同的 `uid` 设置了不同的 `umask` ，那如果实在 `etc/profile` 中修改，只有在重新登录用户的时候才会发生改变，而在 `etc/bashrc` 中修改的话要是切换目录就会发生改变，因为 `profile` 是在登录用户的时候调用的，而 `bashrc` 是在打开一个shell时候调用。

另外，在权限掩码方面关于 `root` 用户和普通用户的区别，我切换到普通用户查看一下 `umask` ，如下图：

```sh
$ sudo su
$ umask
0022

$ su ubuntu
$ umask
0002
```

可以看到确实和 `root` 用户下不一样，此时的 `umask` 是 `0002` ，也就是说默认拿掉的权限少了。这是linux系统基于安全的考虑，对于一般用户身份，保留了用户组的写权限，而 `root` 用户下是拿掉了这项权限的。

# 小结

复制文件时，如果没有使用 `-p` 参数指明保存源文件的属性的话，那么就会根据 `cp` 命令的默认规则复制文件信息。

## 源文件权限

由于复制需要读取源文件的内容，因此执行复制操作的用户必须对源文件有读的权限。

## 目标文件权限

目标文件已经存在时，直接先清空文件内容，在写入源文件内容。目标文件权限不变化。

目标文件不存在时，使用源文件权限为参数创建新文件，然后在写入源文件内容。这里要注意的是，创建新文件时，源文件的权限作为参数，但是创建出来的目标文件权限并不一定就是源文件的权限。因为创建文件时还要扣除umask中指定要扣除的权限。因此源文件的权限只是指定了目标文件可能的最高权限，举例如下：

> 用户的 `umask` 为 `0002`
>
> 源文件权限为 `606`，那么新创建的目标文件权限就是 `606-002=604`
>
> 源文件权限为 `777`，那么新创建的目标文件权限就是 `777-002=775`

# References

- [LINUX的权限, bluemonkey](https://zhuanlan.zhihu.com/p/85360587)
