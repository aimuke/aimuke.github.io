---
title: "如何查看文件的时间信息"
tags: [linux, stat, touch]
---

利用 `stat` 指令查看文件信息

```sh
$ stat data.txt
  File: ‘data.txt’
  Size: 18              Blocks: 8          IO Block: 4096   regular file
Device: fd00h/64768d    Inode: 51736618    Links: 1
Access: (0600/-rw-------)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2020-06-01 14:51:58.026000000 +0800
Modify: 2020-06-01 14:50:44.429000000 +0800
Change: 2020-06-01 15:02:56.320000000 +0800
 Birth: -
```

# 三种时间的介绍

通过如上的方式显示了文件的相关信息，其中包含了三个时间，其含义分别为:

- `Access` 文件最后一次访问时间
- `Modify` 文件内容最后一次被修改时间
- `Change` 文件属性被修改时间

其中 `Modify` 主要是针对文件的内容变化。而 `Change` 针对的是文件属性的变化，这些变化主要体现在 `inode` 中信息的变化，比如，权限的修改, 访问时间变化等。

# 修改文件时间

如何利用touch指令进行文件的时间修改？

## touch指令的介绍

`touch` 不仅可以创建文件，还可以对其进行时间的一些修改

```sh
$ touch --help
Usage: touch [OPTION]... FILE...
Update the access and modification times of each FILE to the current time.

A FILE argument that does not exist is created empty, unless -c or -h
is supplied.

A FILE argument string of - is handled specially and causes touch to
change the times of the file associated with standard output.

Mandatory arguments to long options are mandatory for short options too.
  -a                     change only the access time
  -c, --no-create        do not create any files
  -d, --date=STRING      parse STRING and use it instead of current time
  -f                     (ignored)
  -h, --no-dereference   affect each symbolic link instead of any referenced
                         file (useful only on systems that can change the
                         timestamps of a symlink)
  -m                     change only the modification time
  -r, --reference=FILE   use this file's times instead of current time
  -t STAMP               use [[CC]YY]MMDDhhmm[.ss] instead of current time
      --time=WORD        change the specified time:
                           WORD is access, atime, or use: equivalent to -a
                           WORD is modify or mtime: equivalent to -m
      --help     display this help and exit
      --version  output version information and exit

Note that the -d and -t options accept different time-date formats.
```

## 创建一个文件

```sh
$ touch 1.txt
$ ll
total 0
-rw-r--r-- 1 root root 0 Jun  1 15:13 1.txt
```

## 修改access time

使用当前时间
```sh
$ stat 1.txt
  File: ‘1.txt’
  Size: 0               Blocks: 0          IO Block: 4096   regular empty file
Device: fd00h/64768d    Inode: 51736618    Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2020-06-01 15:13:12.321000000 +0800
Modify: 2020-06-01 15:13:12.321000000 +0800
Change: 2020-06-01 15:13:12.321000000 +0800
 Birth: -

$ touch -a 1.txt
$ stat 1.txt
  File: ‘1.txt’
  Size: 0               Blocks: 0          IO Block: 4096   regular empty file
Device: fd00h/64768d    Inode: 51736618    Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2020-06-01 15:14:58.742000000 +0800
Modify: 2020-06-01 15:13:12.321000000 +0800
Change: 2020-06-01 15:14:58.742000000 +0800
 Birth: -
```

注意观察，上例中通过 `touch -a 1.txt` 修改了文件的访问时间。同时由于 `access time` 属于文件的属性，所以其对应的 `change time` 也发生了变化。

使用指定的日期

```sh
$ stat 1.txt
  File: ‘1.txt’
  Size: 0               Blocks: 0          IO Block: 4096   regular empty file
Device: fd00h/64768d    Inode: 51736618    Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2020-06-01 15:14:58.742000000 +0800
Modify: 2020-06-01 15:13:12.321000000 +0800
Change: 2020-06-01 15:14:58.742000000 +0800
 Birth: -

$ touch -d "05:55:55" 1.txt
$ stat 1.txt
  File: ‘1.txt’
  Size: 0               Blocks: 0          IO Block: 4096   regular empty file
Device: fd00h/64768d    Inode: 51736618    Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2020-06-01 05:55:55.000000000 +0800
Modify: 2020-06-01 05:55:55.000000000 +0800
Change: 2020-06-01 15:20:06.097000000 +0800
 Birth: -
```
这里通过 `touch -d ` 的方式修改时间，这里其实同时修改了 `access time` 和 `modify time`，另外由于文件属性发生了变化，所以文件的 `change time` 始终都是修改时的当前时间。

##　将文件1的时间设置成文件2的时间

```sh
$ stat 1.txt
  File: ‘1.txt’
  Size: 0               Blocks: 0          IO Block: 4096   regular empty file
Device: fd00h/64768d    Inode: 51736618    Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2020-06-01 05:55:55.000000000 +0800
Modify: 2020-06-01 05:55:55.000000000 +0800
Change: 2020-06-01 15:20:06.097000000 +0800
 Birth: -

$ touch 2.txt
$ stat 2.txt
  File: ‘2.txt’
  Size: 0               Blocks: 0          IO Block: 4096   regular empty file
Device: fd00h/64768d    Inode: 51736591    Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2020-06-01 15:27:16.871000000 +0800
Modify: 2020-06-01 15:27:16.871000000 +0800
Change: 2020-06-01 15:27:16.871000000 +0800
 Birth: -

$ touch -r 1.txt 2.txt
$ stat 2.txt
  File: ‘2.txt’
  Size: 0               Blocks: 0          IO Block: 4096   regular empty file
Device: fd00h/64768d    Inode: 51736591    Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2020-06-01 05:55:55.000000000 +0800
Modify: 2020-06-01 05:55:55.000000000 +0800
Change: 2020-06-01 15:27:38.317000000 +0800
 Birth: -
```

通过 `touch -r` 将文件 `1.txt` 的时间赋值给了 `2.txt`，`2.txt` 的属性变化后自动变化了 `change time`.

# 小结

1. 文件有三个时间相关的属性： `access` 访问时间, `modify` 内容修改时间, `change` 属性修改时间。
2. change 时间为系统时间，不可通过外部命令的方式修改或指定，只有在文件属性发生变化时，系统自动变更为属性变化时的系统时间。
3. 可以通过 `touch` 命令修改 `access` 和 `modify` 时间。其中 `touch -a` 仅修改 `access time`。 `touch -d` 同时修改 `access` 和 `modify` 时间。

# References

- [原文 如何查看文件的创建和修改时间, 皓皓松](https://blog.csdn.net/qq_31828515/article/details/62886112)
