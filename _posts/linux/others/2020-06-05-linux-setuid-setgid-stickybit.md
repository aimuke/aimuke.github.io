---
title: "特殊权限——SetUID、SetGID、Sticky BIT"
tags: [linux, SetUID, SetGID, BIT]
---

**[micro小宝](https://me.csdn.net/wxbmelisky)**

对于 `SetUID` 、`SetGID` 、 `Sticky BIT` 这三个文件特殊权限，分别介绍如下：

# 1. SetUID 权限

只有可以执行的二进制程序才能设定 `SetUID` 权限，并且命令执行者要对该程序拥有 `x` （执行）权限。对于设定了  `SetUID` 权限的命令来说，其功能是命令执行者在执行该程序文件时获得该程序文件所有者的身份。 `SetUID` 权限只在该程序执行过程中有效，也就是说身份改变只在程序执行过程中有效。

例如：

用户真正的加密密码存在于 `/etc/shadow` 文件中，通过 `passwd` 命令修改密码即是修改 `/etc/shadow` 文件，通过命令可以看出该文件的权限为 `000` ：

```sh
$ ll /etc/shadow
---------- 1 root root 2505 May  7 10:26 /etc/shadow
```

所以按道理说，一个普通用户是不能修改自己的密码的，但事实上是可以做到的。我们看一下 `passwd` 命令：

```sh
$ ll /usr/bin/passwd
-rwsr-xr-x. 1 root root 27832 Aug 15  2016 /usr/bin/passwd
```

可以看到其所有者权限上有一个 `s`，这个 `s` 就表示 `passwd` 命令拥有 `SetUID` 权限。因此，一个普通用户在执行 `passwd` 命令时会获得它的所有者 `root` 的身份，而 `root` 是可以修改 `/etc/shadow` 文件的，从而一个普通用户可以可以修改自己的密码。

再查看 `cat` 命令，发现它没有 `SetUID` 权限：

```sh
$ ll /usr/bin/cat
-rwxr-xr-x. 1 root root 54080 Jun 13  2018 /usr/bin/cat
```

所以普通用户不能查看 `/etc/shadow` 文件内容。区别在于:

普通用户执行 `/usr/bin/passwd` 时， 由于其权限为 `-rwsr-xr-x`。所以在执行期间，用户获得了 `/usr/bin/passwd` 所有者 `root` 的身份。由于 `root` 有权限操作任何文件，所以可以修改 `/etc/shadow`的内容，即权限被修改。

普通用户执行 `cat` 命令查看 `/etc/shadow` 时，其身份还是执行者身份，由于没有对应的 `READ` 权限，所以会报没有权限的错误。

```sh
$ cat /etc/shadow
cat: /etc/shadow: Permission denied
```

## 设定与取消

在所有者权限之前加上 `4` 代表SetUID，设定方法为：`chmod 4755 文件名`

相应的取消 `SetUID` 方法为：`chmod 755 文件名`

还有设定方法为： `chmod u+s 文件名`, `u` 代表当前用户

相应的取消 `SetUID` 方法为：`chmod u-s 文件名`。

例如：

```sh
$ touch a.file
$ ll
total 0
-rw-rw-r-- 1 ubuntu ubuntu 0 Jun  4 16:40 a.file

$ chmod 4755 a.file
$ ll
total 0
-rwsr-xr-x 1 ubuntu ubuntu 0 Jun  4 16:40 a.file

$ chmod 755 a.file
$ ll
total 0
-rwxr-xr-x 1 ubuntu ubuntu 0 Jun  4 16:40 a.file

$ chmod u+s a.file
$ ll
total 0
-rwsr-xr-x 1 ubuntu ubuntu 0 Jun  4 16:40 a.file

$ chmod u-s a.file
$ ll
total 0
-rwxr-xr-x 1 ubuntu ubuntu 0 Jun  4 16:40 a.file
```

`SetUID` 权限是存在一定的危险的，使用 `SetUID` 要注意以下三点：

- 关键目录应严格控制写权限，比如 /、/usr、/etc 等 ；
- 用户的密码设置要严格遵守密码三原则；
- 对系统中默认应该具有 SetUID 权限的文件作一列表，定时检查有没有这些之外的文件被设置了 SetUID 权限。

# 2. SetGID 权限

可以对可执行的二进制程序文件设置 `SetGID` 权限，也可以对目录设置 `SetGID` 权限。

## SetGID 针对文件的作用

对于文件只有可执行的二进制程序才能设置 `SetGID` 权限，并且命令执行者要对该程序拥有 `x`（执行）权限。对于设定了 `SetGID` 权限的二进制程序来说，命令执行者在执行程序的时候，组身份升级为该程序文件的所属组。 `SetGID` 权限同样只在该程序执行过程中有效，也就是说组身份改变只在程序执行过程中有效。

**例如：**

对于 `locate` 命令，一个普通用户可以通过它在文件资料库 `/var/lib/mlocate/mlocate.db` 中查找文件。下面先看一下文件资料库的权限：

```sh
$ ll /var/lib/mlocate/mlocate.db
-rw-r-----. 1 root slocate 3693648 6月 11 17:16 /var/lib/mlocate/mlocate.db
```

可以看到，从其权限来看普通用户对其是没有任何权限的，那为什么却可以访问该文件呢？

下面再看一下 `locate` 命令的权限：

```sh
$ ll /usr/bin/locate
-rwx--s--x. 1 root slocate 40496 6月 10 2014 /usr/bin/locate
```

`/usr/bin/locate` 是可执行二进制程序，可以赋予 `SetGID` ，看到它的所属组权限上有一个 `s`，这个 `s` 就表示 `locate` 命令拥有 `SetGID` 权限。普通用户对 `/usr/bin/locate` 命令拥有执行权限因此，一个普通用户在执行 `locate` 命令时组身份会升级为 `locate` 命令的所属组 `slocate` ，而 `slocate` 对文件资料库 `/var/lib/mlocate/mlocate.db` 拥有 `r`（读）权限，所以 `locate` 命令可以访问文件资料库，从而普通用户可以通过 `locate` 命令查找文件。当然，命令结束后普通用户的组身份返回为它原来的所属组。

## SetGID 针对目录的作用

普通用户必须对一个目录拥有 `r` 和 `x` 权限，才能进入此目录。对于设定了 `SetGID` 权限的目录来说，普通用户在此目录中的有效组会变成此目录的所属组，若普通用户对此目录拥有 `w` 权限时，在目录中新建的文件的默认所属组是这个目录的所属组。

例如：

```sh

$ mkdir dirtest
$ ll
total 0
drwxr-xr-x 2 root root 6 Jun  4 17:25 dirtest
$ chmod g+s dirtest
$ ll -d dirtest/
drwxr-sr-x 2 root root 6 Jun  4 17:25 dirtest/


$ chmod 777 dirtest/
$ ll -d dirtest/
drwxrwsrwx 2 root root 6 Jun  4 17:25 dirtest/


$ su ubuntu
$ cd dirtest/
$ touch aa
$ ll
total 0
-rw-rw-r-- 1 ubuntu root 0 Jun  4 17:27 aa
```

> 注意: 新建的文件 `aa` 的用户组是 `root`

设定与取消 `SetGID` 的方法如下：

在所有者权限之前加上 `2` 代表 `SetGID` ，设定方法为：`chmod 2755 文件名`，相应的取消 SetGID 方法为：`chmod 755 文件名`。 

还有设定方法为： `chmod g+s 文件名`，相应的取消 SetGID 方法为：`chmod g-s 文件名`。

# Sticky BIT 权限

`Sticky BIT` 表示的是粘着位，主要是用来避免其他用户对文件的误操作。

粘着位目前只对目录有效，普通用户要对该目录拥有 `w` 和 `x` 权限，即普通用户可以在此目录拥有写入权限。如果没有粘着位，因为普通用户拥有 `w` 权限，所以可以删除此目录下所有文件，包括其他用户建立的文件。一但赋予了粘着位，除了 `root` 可以删除所有文件，普通用户就算拥有 `w` 权限也只能删除自己建立的文件，但是不能删除其他用户建立的文件。

例如：

最常见的系统中拥有粘着位的目录是 `/tmp`，通过查看权限可以看到 `/tmp` 的其他人权限有一个 `t`，表示拥有粘着位，即拥有 `Sticky BIT` 权限：

```sh
$ ll -d /tmp
drwxrwxrwt. 14 root root 4096 Jun  4 17:32 /tmp

$ touch aaa
$ su - tmp
$ cd /tmp
$ touch bbb
$ su - tmp
$ cd /tmp
$ touch ccc

$ rm -f ccc
$ rm -f bbb
rm：无法删除"bbb": 不允许的操作
$ rm -f aaa
rm：无法删除"aaa": 不允许的操作
```

设置与取消粘着位 `Sticky BIT` 权限如下：
设置粘着位 ：`chmod 1777 目录名`  或  `chmod o+t 目录名`
取消粘着位 ：`chmod 777 目录名`  或  `chmod o-t 目录名`

关于黏着位找了很多资料也没有找到明确的描述，网上众说纷纭也没有清晰的描述出它的作用，最后还是在《UNIX环境高级编程》中找到一些更明确的解释：

> The S_ISVTX bit has an interesting history. Onversions of the UNIX System that predated demand paging, this bit was known as the sticky bit.If it was set for an executable program ﬁle, then the ﬁrst time the program was executed, a copy of the program’s text was saved in the swap area when the process terminated. (The text portion of a program is the machine instructions.) The program would then load into memory morequickly the next time it was executed, because the swap area was handled as a contiguous ﬁle, as compared to the possibly random location of data blocks in a normal UNIX ﬁle system. The sticky bit was often set for common application programs, such as the text editor and the passes of the C compiler. Naturally,therewas a limit to the number of sticky ﬁles that could be contained in the swap area beforerunning out of swap space, but it was a useful technique. The name sticky came about because the text portion of the ﬁle stuck around in the swap area until the system was rebooted. Later versions of the UNIX System referred to this as the saved-text bit; hence the constant S_ISVTX.With today’s newer UNIX systems, most of which have a virtual memory system and a faster ﬁle system, the need for this technique has disappeared.

在早期的unix系统中，如果一个程序被设置了黏着位，那么当它第一次执行结束之后，程序的正文段会被写入到交换空间中，以此加快后续使用的加载速度。因为交换空间是顺序存放，而磁盘上是随机的。它通常被设置成经常被使用的公用程序例如文本编辑器、编译器等，会一直持续到系统重启。

在后续的unix系统都设计了新的更快速的文件系统，所以这种用法逐渐消失。而对于这一黏着位“现在的用途”的描述是：

> On contemporary systems, the use of the sticky bit has been extended. The Single UNIX Speciﬁcation allows the sticky bit to be set for a directory. If the bit is set for a directory, a file in the directory can be removed or renamed only if the user has write permission for the directory and meets one of the following criteria:
>
> Owns the file
>
> Owns the directory
>
> Is the superuser
>
> The directories /tmp and /var/tmp are typical candidates for the sticky bit—they are directories in which any user can typically create files. The permissions for these two directories are often read, write, and execute for everyone (user, group, and other). But users should not be able to delete or rename files owned by others.

现在的系统里面，黏着位已经被扩展了，它被用于目录权限上，如果一个目录设置了这一位，这个目录下的文件就只能被满足以下条件的用户重命名或者删除：

- 所有者
- 当前目录所有者
- 超级用户

目录 `/tmp` 和 `/var/tmp` 是设置粘住位的候选者—这两个目录是任何用户都可在其中创建文件的目录。这两个目录对任一用户 (用户、组和其他)的许可权通常都是读、写和执行。但是用户不应能删除或更名属于其他人的文件，为此在这两个目录的文件方式中都设置了粘住位

手动给目录添加 `sticky` 位：

```sh
> mkdir 123
> chmod +t 123
> ll -d 123
drwxrwxr-t. 2 ma ma 6 Nov  3 02:18 123
```

# 权限位含义

`Chmod` 命令中的特殊权限位含义：

- `S_ISUID` `04000` 文件的 (set user-id on execution)位
- `S_ISGID` `02000` 文件的 (set group-id on execution)位
- `S_ISVTX` `01000` 文件的 sticky 位

如何设置UID、GID、STICK_BIT：

SUID ：置于 `u` 的 `x` 位，原位置有执行权限，就置为 `s`，没有了为 `S` .

- chmod u+s  xxx # 设置setuid权限
- chmod u-s  xxx # 取消setuid权限
- chmod 4551 file // 权限： r-sr-x—x
- chmod 551 file // 权限： r-xr-x—x

SGID：置于 `g` 的 `x` 位，原位置有执行权限，就置为 `s`，没有了为 `S` .

- chmod g+s  xxx # 设置setgid权限
- chmod g-s  xxx # 取消setgid权限
- chmod 2551 file // 权限： r-xr-s--x
- chmod 551 file // 权限： r-xr-x--x

STICKY：粘滞位，置于 o 的 x 位，原位置有执行权限，就置为 t ，否则为T .

- chmod o+t  xxx # 设置stick bit权限，针对目录
- chmod 1551 file // 权限： r-xr-x--t
- chmod 551 file // 权限： r-xr-x--x

# References

- [原文 Linux文件特殊权限——SetUID、SetGID、Sticky BIT](https://blog.csdn.net/wxbmelisky/article/details/51649343), [micro小宝](https://me.csdn.net/wxbmelisky)

- [理解setuid()、setgid()和sticky位](https://www.cnblogs.com/puyangsky/p/5307030.html), [puyangsky](https://www.cnblogs.com/puyangsky/)

- [linux中的setuid、setgid以及sticky bit](https://www.dyxmq.cn/linux/linux-setuid-setgid-sticky-bit.html), [马谦马谦马谦](https://www.dyxmq.cn/author/%E8%B0%A6/)
