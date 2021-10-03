---
title: "回车和换行"
tags: [linux, windows]
--- 

# 来源

今天，我总算搞清楚"回车"（`carriage return CR`）和"换行"（`line feed LF`）这两个概念的来历和区别了。

在计算机还没有出现之前，有一种叫做电传打字机（Teletype Model 33）的玩意，每秒钟可以打10个字符。但是它有一个问题，就是打完一行换行的时候，要用去0.2秒，正好可以打两个字符。要是在这0.2秒里面，又有新的字符传过来，那么这个字符将丢失。

于是，研制人员想了个办法解决这个问题，就是在每行后面加两个表示结束的字符。

- 一个叫做"回车"，告诉打字机把打印头定位在左边界；
- 另一个叫做"换行"，告诉打字机把纸向下移一行。

这就是"换行"和"回车"的来历，从它们的英语名字上也可以看出一二。

后来，计算机发明了，这两个概念也就被般到了计算机上。那时，存储器很贵，一些科学家认为在每行结尾加两个字符太浪费了，加一个就可以。于是，就出现了分歧。

- Unix系统里，每行结尾只有"<换行>"，即"`\n`"；
- Windows系统里面，每行结尾是"<回车><换行>"，即"`\r\n`"；
- Mac系统里，每行结尾是"<回车>"。

一个直接后果是，Unix/Mac系统下的文件在Windows里打开的话，所有文字会变成一行；而Windows里的文件在Unix/Mac下打开的话，在每行的结尾可能会多出一个 `^M` 符号。

**attilax 评论补充**

> 这个说法我有点儿补充:"回车"和"换行"是来源机械英文打字机的，电传打字机那个是后来的.
>
> - "车"指的是纸车,带着纸一起左右移动的模块。 当开始打第一个字之前，要把纸车拉到最右边，上紧弹簧。随着打字， 弹簧把纸车拉回去，每当打完一行后，纸车就完全收回去了，所以叫回车。
> - 换行的概念就是: 打字机左边有个"把手 "，往下 扳动一下,纸会上移一行。

# 操作系统文件换行符

在ASCII中存在这样两个字符 `CR`（编码为 `13` ）和 `LF`（编码为 `10` ），在编程中我们一般称其分别为'`\r`'和'`\n`'。他们被用来作为换行标志，但在不同系统中换行标志又不一样。下面是不同操作系统采用不同的换行符：

- Unix和类Unix（如Linux）：换行符采用 `\n`
- Windows和MS-DOS：换行符采用 `\r\n`
- Mac OS X之前的系统：换行符采用 `\r`
- Mac OS X：换行符采用 `\n`

# Linux中查看换行符

在Linux中查看换行符的方法应该有很多种，这里介绍两种比较常用的方法。

- 使用 `cat  -A [Filename]` 查看，如下图所示，看到的为一个Windows形式的换行符， `\r` 对应符号 `^M` ，`\n` 对应符号 `$` .

    ```sh
    cat -A test.txt
    this is the first line^M$
    this is the 2nd line^M$
    this is 3rd linel
    ```

- 使用vi编辑器查看，然后使用 `set list` 命令显示特殊字符：

    ```sh
    this is the first line$
    this is the 2nd line$
    this is 3rd line$
    ```

咦，细心的朋友发现了，怎么 `^M` 还是没显示出来，这里也是给大家提个醒，用VI的二进制模式（ `vi -b [FileName]` ）打开，才能够显示出 `^M`：

# Windows换行符转换为Linux格式

下面介绍三种方法，选择哪一种看自己喜好，当然你也可以选择第x种，^_^。

- 使用VI: 使用 `VI` 普通模式打开文件，然后运行命令 `set ff=unix` 则可以将Windows 换行符转换为Linux换行符，简单吧！命令中 `ff` 的全称为 `file encoding`。

- 使用命令 `dos2unix`，如下所示

    ```sh
    [root@localhost test]# dos2unix gggggggg.txt 
    dos2unix: converting file gggggggg.txt to UNIX format ...
    ```

- 使用 `sed` 命令删除 `\r` 字符: 

    ```sh
    [root@localhost test]# sed -i 's/\r//g' gggggggg.txt
    ```

# Reference

- [原文 - 回车和换行](https://www.ruanyifeng.com/blog/2006/04/post_213.html)

- [Windows文件换行符转Linux换行符](https://blog.csdn.net/CJF_iceKing/article/details/47836201)
