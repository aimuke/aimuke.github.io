---
title: "linux 环境变量"
tags: [linux, env, set, export]
---

# 变量分类

linux 中变量分两种：

* **用户变量**：对用户生效的变量，同一个用户登录的时候都有效， 使用 `env` 访问
* **shell 变量**：当前shell 中的变量，包含了shell 对应用户的变量， 使用 `set` 访问

**shell 导出的用户变量**：将 shell 中定义的变量导出为用户变量，使用export 导出后可以使用 `env` 访问

用户变量有两种设置，当前用户变量和所有用户的默认变量。

- 所有用户的默认变量在 `/etc/profile` 中设置
- 当前用户的变量在当前用户的主目录下进行设置 `~/bash*` 相关文件（如 `.bash\_profile` ）

# set env export区别

* `set`: 显示\(设置\)shell变量 包括的私有变量以及用户变量，不同类的shell有不同的私有变量 bash,ksh,csh每中shell私有变量都不一样
* `env`: 显示\(设置\)用户变量变量
* `export`: 显示\(设置\)当前导出成用户变量的shell变量。

举个例子来讲:

```bash
[oracle@zhou3 ~]$ aaa=bbb --shell变量设定   
[oracle@zhou3 ~]$ echo $aaa    
bbb   
[oracle@zhou3 ~]$ env| grep aaa --设置完当前用户变量并没有   
[oracle@zhou3 ~]$ set| grep aaa  --shell变量有   
aaa=bbb   
[oracle@zhou3 ~]$ export| grep aaa --这个指的export也没导出，导出变量也没有   
[oracle@zhou3 ~]$ export aaa   --那么用export 导出一下   
[oracle@zhou3 ~]$ env| grep aaa  --发现用户变量内存在了   
aaa=bbb
```

**总结:** 

- linux 分 shell变量\( `set` \)，用户变量\( `env` \)， shell变量包含用户变量;
- `export` 是一种命令工具，是显示那些通过 `export` 命令把shell变量中包含的用户变量导入给用户变量的那些变量.

#  变量加载时机

最根本的设置、更改变量的配置文件

* `~/.bash_profile`  用户登录时被读取，其中包含的命令被执行
* `~/.bashrc`  启动新的shell时被读取，并执行
* `~/.bash_logout`  shell 登录退出时被读取

此外，shell（这里指bash）的初始化过程是这样的：

1. bash检查文件 `/etc/profile` 是否存在, 存在就读取该文件，否则跳过
2. bash检查主目录下的文件 `.bash\_profile` 是否存在, 存在就读取該文件，否则跳过
3. bash检查主目录下的 `.bash\_login` 是否存在， 存在就读取该文件，否则跳过
4. bash检查主目录下的文件 `.profile` 是否存在， 存在就读取该文件，否则跳过。

这些步骤都执行完后，就出现提示符了， ksh 默认提示符是 `$`.

> 修改配置文件中的用户变量后，需要重新加载一遍对应的脚本数据才会生效。比如: 修改了 /`.bash\_profile` 中的 PATH ，需要重新执行一下  `source ~/.bash_profile` 后才会生效

# 删除环境变量

使用 `unset` 命令来清除环境变量，注意 `set`， `env`，  `export` 设置的变量，都可以用 `unset` 来清除的

```bash
清除环境变量的值用unset命令。如果未指定值，则该变量值将被 设为NULL。示
例如下：  
$ export TEST="Test..." #增加一个环境变量TEST  
$ env|grep TEST #此命令有输入，证明环境变量TEST已经存在了  
TEST=Test...  
$ unset $TEST #删除环境变量TEST  
$ env|grep TEST #此命令没有输出，证明环境变量TEST已经不存在了
```

# 设置只读变量

```bash
使用了readonly命令的话，变量就不可以被修改或清除了。示例如下：
$ export TEST="Test..." #增加一个环境变量TEST
$ readonly TEST #将环境变量TEST设为只读
$ unset TEST #会发现此变量不能被删除
-bash: unset: TEST: cannot unset: readonly variable
$ TEST="New" #会发现此也变量不能被修改
-bash: TEST: readonly variable
```

# 常见的shell变量

* PATH 包含了一系列由冒号分隔开的目录，系统就从这些目录里寻找可执行文件。如果你输入的可执行文件（例如ls、rc-update或者emerge） 不在这些目录中，系统就无法执行它（除非你输入这个命令的完整路径，如/bin/ls）。 
* `ROOTPATH` 功能和PATH相同，但它只罗列出超级用户（root）键入命令时所需检查的目录。
* `LDPATH` 包含了一系列用冒号隔开的目录，动态链接器将在这些目录里查找库文件。
* `MANPATH` 包含了一系列用冒号隔开的目录，命令man会在这些目录里搜索man页面。 
* `INFODIR` 包含了一系列用冒号隔开的目录，命令info将在这些目录里搜索info页面。 
* `PAGER` 包含了浏览文件内容的程序的路径（例如less或者more）。 
* `EDITOR` 包含了修改文件内容的程序（文件编辑器）的路径（比如nano或者vi）。 
* `KDEDIRS` 包含了一系列用冒号隔开的目录，里面放的是KDE相关的资料。
* `CONFIG\_PROTECT` 包含了一系列用空格隔开的目录，它们在更新的时候会被Portage保护起来。
* `CONFIG\_PROTECT\_MASK` 包含了一系列用空格隔开的目录，它们在更新的时候不会被Portage保护起来。

* `PATH`：决定了shell将到哪些目录中寻找命令或程序
* `HOME`：当前用户主目录
* `MAIL`：是指当前用户的邮件存放目录。
* `SHELL`：是指当前用户用的是哪种Shell。
* `HISTSIZE`：是指保存历史命令记录的条数
* `LOGNAME`：是指当前用户的登录名。
* `HOSTNAME`：是指主机的名称，许多应用程序如果要用到主机名的话，通常是从这个环境变量中来取得的。
* `LANG/LANGUGE`：是和语言相关的环境变量，使用多种语言的用户可以修改此环境变量。
* `PS1`：是基本提示符，对于root用户是\#，对于普通用户是$。
* `PS2`：是附属提示符，默认是“&gt;”。可以通过修改此环境变量来修改当前的命令符，比如下列命令会将提示符修改成字符串“Hello,My NewPrompt :\) ”。

```bash
# PS1=" Hello,My NewPrompt :) "
```

# References

* [shell环境变量以及set,env,export的区别](https://blog.csdn.net/longxibendi/article/details/6125075?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.control) , 凤凰舞者 qq:578989855 

