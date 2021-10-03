---
title: "Linux动态链接"
tags: [ldd, "动态链接"]
--- 

[原文地址](https://www.jianshu.com/p/ea9f4a6b136d), [linjinhe](https://www.jianshu.com/u/96141c0d5c5c)

# 问题

曾经不止一次遇到过这样的情况：从机器A拷贝一个二进制文件到另一台机器B，两台机器的操作系统版本一样，可是在机器A能正常运行，在机器B却提示错误。最常见的就是提示动态链接库找不到，如：

```
./test: error while loading shared libraries: libstdc++.so.6: cannot open shared object file: No such file or directory
```

其实就是说，找不到动态链接库 `libstdc++.so.6`。

最近又有一次碰到类似的问题，所以顺便把动态链接库的基本原理了解了一遍。

# 静态链接

静态链接库，在Linux下文件名后缀为 `.a` ，如 `libstdc++.a` 。在编译链接时直接将目标代码加入可执行程序。

# 动态链接

动态链接库，在Linux下是 `.so` 文件，在编译链接时只需要记录需要链接的号，运行程序时才会进行真正的“链接”，所以称为“动态链接”。如果同一台机器上有多个服务使用同一个动态链接库，则只需要加载一份到内存中共享。因此， `动态链接库` 也称 `共享库`。

# 命名规则

动态链接库与应用程序之间的真正链接是在应用程序运行时，因此很容易出现开发环境和运行环境的动态链接库不兼容或缺失的情况。

Linux通过规定动态链接库的版本命名规则来管理兼容性问题。

Linux规定动态链接库的文件名规则比如如下：

```
​ libname.so.x.y.z
```

- `lib` ：统一前缀。
- `so` ：统一后缀。
- `name` ：库名，如libstdc++.so.6.0.21的name就是stdc++。
- `x` ：主版本号。表示库有重大升级，不同主版本号的库之间是不兼容的。如libstdc++.so.6.0.21的主版本号是6。
- `y` ：次版本号。表示库的增量升级，如增加一些新的接口。在主版本号相同的情况下，高的次版本号向后兼容低的次版本号。如libstdc++.so.6.0.21的次版本号是0。
- `z` ：发布版本号。表示库的优化、bugfix等。相同的主次版本号，不同的发布版本号的库之间完全兼容。如libstdc++.so.6.0.21的发布版本号是21。
另外，Linux下的一个动态链接库会有下面三个名字：

```
libstdc++.so -> libstdc++.so.6.0.21*
libstdc++.so.6 -> libstdc++.so.6.0.21*
libstdc++.so.6.0.21*
```

- `libstdc++.so` ：linker name，程序编译链接时如果依赖了共享库，链接器只认不带任何版本的共享库。如果存在多个同名（上面命名规则中的name）动态链接库，linker name会指向最新的一个。
- `libstdc++.so.6` ：SO_NAME， 程序运行时会按照这个名称去找真正的库文件。也就是说，ELF可执行文件中保存的动态库名就是SO_NAME。如果存在多个同一主版本号的动态链接库，SO_NAME会指向最新的一个。
- `libstdc++.so.6.0.21` ：real name，这是动态链接库的真正名称。

## 相关路径
/lib：最关键和基础的动态链接库。

/usr/lib：关键的动态链接库。

/usr/local/lib：第三方动态链接库。

由/etc/ld.so.conf配置文件指定的目录。

## ldconfig

动态链接器不可能在每次查找动态链接库都去遍历所有动态链接库的目录，这样速度太慢了。因此，在系统启动时会通过 `ldconfig` 为动态链接库生成 `SO_NAME` 和 `/etc/ld.so.cache` 存放系统动态链接库的路径信息，加速动态链接库的查找。

程序启动查找动态链接库的路径顺序如下：

1. 由LD_LIBRARY_PATH指定的路径。
2. 由路径缓存文件/etc/ld.so.cache指定的路径。
3. 默认共享库目录，先/usr/lib，然后/lib。

注意，安装动态链接库后，需要重启系统或运行 `ldconfig` 生成 `SO_NAME` 和刷新 `/etc/ld.so.cache` 文件。

## ldd

通过 `ldd elf_file` 可以查看ELF文件依赖哪些动态链接库，如

```sh
$ ldd test
linux-vdso.so.1 =>  (0x00007ffc89b46000)
libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f6e20ec7000)
libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f6e20bc1000)
libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f6e209ab000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f6e205e3000)
/lib64/ld-linux-x86-64.so.2 (0x00007f6e211cb000)
```

linux-vdso.so.1是内核提供的一个动态链接库，所以这里只有一个内存地址。

/lib64/ld-linux-x86-64.so.2是一个动态链接库的绝对路径。

# References

- [原文地址 Linux动态链接](https://www.jianshu.com/p/ea9f4a6b136d), [linjinhehttps://www.jianshu.com/u/96141c0d5c5c]()
