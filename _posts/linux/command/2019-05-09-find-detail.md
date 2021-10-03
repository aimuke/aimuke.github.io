---
title: "每天一个linux命令（22）：find 命令的参数详解"
tags: [linux, command, find]
---

find一些常用参数的一些常用实例和一些具体用法和注意事项。

# 使用name选项

文件名选项是`find`命令最常用的选项，要么单独使用该选项，要么和其他选项一起使用。  

可以使用某种文件名模式来匹配文件，记住要用引号将文件名模式引起来。  

想要在当前目录及子目录中查找所有的`*.log`文件，可以用： 

```sh
find . -name "*.log"   
```

想要的当前目录及子目录中查找文件名以一个大写字母开头的文件，可以用：  
```sh
find . -name "[A-Z]*"  
```

如果想在当前目录查找文件名以一个个小写字母开头，最后是4到9加上.log结束的文件：  
```
find . -name "[a-z]*[4-9].log" -print
```

# 用perm选项

按照文件权限模式用`-perm`选项,按文件权限模式来查找文件的话。 

如在当前目录下查找文件权限位为755的文件，即文件属主可以读、写、执行，其他用户可以读、执行的文件，可以用：  
```sh
[root@localhost test]# find . -perm 755 -print
```

还有一种表达方法：数字前面要加一个横杠-，表示匹配权限高于指定值的文件。
> 比如`-042` 表示文件的权限必须第一位大于`0`,第二位必须大于`4`, 第三位必须大于`2`。
> 
> 如`741`是不满足的，因为第一位权限不够，`242`是满足的

# 忽略目录

## 忽略单个目录
如果在查找文件时希望忽略某个目录，因为你知道那个目录中没有你所要查找的文件，那么可以使用-prune选项来指出需要忽略的目录。在使用`-prune`选项时要当心，因为如果你同时使用了`-depth`选项，那么`-prune`选项就会被`find`命令忽略。如果希望在`test`目录下查找文件，但不希望在`test/dtest`目录下查找，可以用：  

```sh
bobo@ubuntu:~/test$ find .
.
./dtest
./dtest/test2.txt
./dtest/test1.txt
./test3.txt
bobo@ubuntu:~/test$ find . -path ./dtest -prune -o -print
.
./test3.txt
bobo@ubuntu:~/test$
```

```sh
[xxxxxxxxxxxxx] ➤ tree
.
+--- dir1
|   +--- sub1.txt
+--- dir2
|   +--- sub2.json
|   +--- sub2.txt
+--- test1.txt
+--- test2.txt
+--- test3.json
[xxxxxxxxxxxxxx] ➤ find . -path ./dir1 -prune -o -name "*.txt"
./dir1
./dir2/sub2.txt
./test1.txt
./test2.txt

```


```sh
find [-path ..] [expression] 
```

在路径列表的后面的是表达式 

`-path "test" -prune -o -print` 是 `-path "test" -a -prune -o -print` 的简写表达式按顺序求值, `-a` 和 `-o` 都是短路求值，与 shell 的 `&&` 和 `||` 类似。

如果 `-path "test"` 为真，则求值 `-prune` , `-prune` 返回真，与逻辑表达式为真；否则不求值 `-prune`，与逻辑表达式为假。

如果 `-path "test" -a -prune` 为假，则求值 `-print` ，`-print`返回真，或逻辑表达式为真；否则不求值 `-print`，或逻辑表达式为真。 

这个表达式组合特例可以用伪码写为:

```sh
if -path "test" then  
	-prune  
else  
	-print  
```

## 避开多个目录

```sh
find test \( -path test/test4 -o -path test/test3 \) -prune -o -print 
```

圆括号表示表达式的结合。  `\` 表示引用，即指示 `shell` 不对后面的字符作特殊解释，而留给 `find` 命令去解释其意义。  

```sh
find test \(-path test/test4 -o -path test/test3 \) -prune -o -name "*.log" -print
```

# 使用user和nouser选项：

按文件属主查找文件：

```sh
find /etc -user peida  
```

为了查找属主帐户已经被删除的文件，可以使用`-nouser`选项。在`/home`目录下查找所有的这类文件

```sh
find /home -nouser
```

这样就能够找到那些属主在`/etc/passwd`文件中没有有效帐户的文件。在使用`-nouser`选项时，不必给出用户名； find命令能够为你完成相应的工作。

同理查找 `group`和`nogroup`选项

# 按照更改或访问时间查找

如果希望按照更改时间来查找文件，可以使用mtime,atime或ctime选项。如果系统突然没有可用空间了，很有可能某一个文件的长度在此期间增长迅速，这时就可以用mtime选项来查找这样的文件。  

用减号-来限定更改时间在距今n日以内的文件，而用加号+来限定更改时间在距今n日以前的文件。  

希望在系统根目录下查找更改时间在5日以内的文件，可以用：

```sh
find / -mtime -5 -print
```
为了在/var/adm目录下查找更改时间在3日以前的文件，可以用:
```sh
find /var/adm -mtime +3 -print
```

# 查找比某个文件新或旧的文件

如果希望查找更改时间比某个文件新但比另一个文件旧的所有文件，可以使用`-newer`选项。

它的一般形式为：  
```sh
newest_file_name ! oldest_file_name  
```
其中，`!`是逻辑非符号。  

实例1：查找更改时间比文件log2012.log新但比文件log2017.log旧的文件

```sh
find -newer log2012.log ! -newer log2017.log
```

实例2：查找更改时间在比log2012.log文件新的文件  
```sh
find . -newer log2012.log -print
```

# 使用type选项

实例1：在/etc目录下查找所有的目录  

```sh
find /etc -type d -print  
```
实例2：在当前目录下查找除目录以外的所有类型的文件  
```sh
find . ! -type d -print  
```

# 使用size选项

可以按照文件长度来查找文件，这里所指的文件长度既可以用块（block）来计量，也可以用字节来计量。以字节计量文件长度的表达形式为`N c`；以块计量文件长度只用数字表示即可。  

在按照文件长度查找文件时，一般使用这种以字节表示的文件长度，在查看文件系统的大小，因为这时使用块来计量更容易转换。  

实例1：在当前目录下查找文件长度大于1 M字节的文件  

```sh
find . -size +1000000c -print
```
实例2：在/home/apache目录下查找文件长度恰好为100字节的文件:  
```sh
find /home/apache -size 100c -print  
```
实例3：在当前目录下查找长度超过10块的文件（一块等于512字节） 
```sh
find . -size +10 -print
```
# 使用depth选项

在使用find命令时，可能希望先匹配所有的文件，再在子目录中查找。使用depth选项就可以使find命令这样做。这样做的一个原因就是，当在使用find命令向磁带上备份文件系统时，希望首先备份所有的文件，其次再备份子目录中的文件。  

实例1：find命令从文件系统的根目录开始，查找一个名为CON.FILE的文件。   

```sh
find / -name "CON.FILE" -depth -print
```

它将首先匹配所有的文件然后再进入子目录中查找

# 使用mount选项

在当前的文件系统中查找文件（不进入其他文件系统），可以使用find命令的mount选项。

从当前目录开始查找位于本文件系统中文件名以XC结尾的文件  

```sh
find . -name "*.XC" -mount -print 
```

# 参考文献

- [每天一个linux命令（21）：find命令之xargs](http://www.cnblogs.com/peida/archive/2012/11/16/2773289.html)
