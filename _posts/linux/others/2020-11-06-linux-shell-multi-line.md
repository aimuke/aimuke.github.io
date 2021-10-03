---
title: "shellscript 中操作多行字符串变量"
tags: [shell, script, shellscript, bash, variable, multiline, string]
---

当我们在 shell 的 bash 里操作多行内容的字符串,我们往往会想到普通的字符串处理办法 例如:

```text
string="Hello linux"
echo $string
```

其实 bash 提供了一个非常好的解决办法,就是 “Multi-line”

# 变量注入

e.g. 包含变量

```sh
cat > myfile.txt <<EOF
this file has $variable $names $inside
EOF

# 注入文档到 myfile.txt
cat myfile.txt
#输入:
#this file has

variable="ONE"
names="TWO"
inside="expanded variables"

cat > myfile.txt <<EOF
this file has $variable $names $inside
EOF

#print out the content of myfile.txt
cat myfile.txt
#输入:
#this file has ONE TWO expanded variables
```

# 不注入变量值

PS: 引用符号 "EOF" 决定是否需要输入变量

```sh
cat > myfile.txt <<"EOF"
this file has $variable $dollar $name $inside
EOF

cat myfile.txt
#得到
#this file has $variable $dollar $name $inside
```

转义 dollar "$" 符号,bash将取消变量的解析

```sh
cat > myfile.txt <<EOF
this file has $variable \$dollar \$name \$inside
EOF

cat myfile.txt
# 得到
# this file has $variable $dollar $name $inside
```

# 将多行文本赋值到变量

例1: 变量注入

```sh
read -d '' stringvar <<-"_EOF_"

all the leading dollars in the $variable $name are $retained

_EOF_
# 输入变量
echo $stringvar;
# all the leading dollars in the $variable $name are $retained
```

例2:直接定义，换行符被删除,引号含义变化

```sh
VARIABLE1="<?xml version="1.0" encoding='UTF-8'?>
<report>
  <img src="a-vs-b.jpg"/>
  <caption>Thus is a future post on Multi Line Strings in bash
  <date>1511</date>-<date>1512</date>.</caption>
</report>"

echo $VARIABLE1
# <?xml version=1.0 encoding='UTF-8'?> <report> <img src=a-vs-b.jpg/> <caption>Thus is a future post on Multi Line Strings in bash <date>1511</date>-<date>1512</date>.</caption> </report>
```

> 注意： 定义变量的时候，version 后面的 1.0 是有引号的，输出后，引号没了

例3:使用cat 原样保留了原定义的数据

```sh
VARIABLE2=$(cat <<EOF
<?xml version="1.0" encoding='UTF-8'?>
<report>
  <img src="a-vs-b.jpg"/>
  <caption>Thus is a future post on Multi Line Strings in bash
  <date>1511</date>-<date>1512</date>.</caption>
</report>
EOF
)
echo $VARIABLE2
#<?xml version="1.0" encoding='UTF-8'?> <report> <img src="a-vs-b.jpg"/> <caption>Thus is a future post on Multi Line Strings in bash <date>1511</date>-<date>1512</date>.</caption> </report>
```

同上，输出内容与原定义内容相同

```sh
VARABLE3=`cat <<EOF
<?xml version="1.0" encoding='UTF-8'?>
<report>
  <img src="a-vs-b.jpg"/>
  <caption>Thus is a future post on Multi Line Strings in bash
  <date>1511</date>-<date>1512</date>.</caption>
</report>
EOF`
```

例8:

```sh
tee aa.txt << EOF
echo "Hello World 20314"
EOF

cat aa.txt
#echo "Hello World 20314"
```

例如10:

```sh
sudo sh -c "cat > /aaa.txt" <<"EOT"
this text gets saved as sudo - $10 - ten dollars ...
EOT

cat /aaa.txt
#this text gets saved as sudo - $10 - ten dollars ...
```

例11:

```sh
cat << "EOF" | sudo tee /aaa.txt
let's count
$one
two
$three
four

EOF

cat /aaa.txt
#let's count
#$one
#two
#$three
#four
```

关于 tee

```sh
> tee –help
Usage: tee [OPTION]… [FILE]…
Copy standard input to each FILE, and also to standard output.
-a, –append append to the given FILEs, do not overwrite
-i, –ignore-interrupts ignore interrupt signals
–help display this help and exit
–version output version information and exit
If a FILE is -, copy again to standard output.
Report tee bugs to bug-coreutils@gnu.org
GNU coreutils home page:
General help using GNU software:
For complete documentation, run: info coreutils ‘tee invocation’
```

# References

* [原文 bash – 操作多行字符串变](https://whatua.com/2018/02/24/unix-bash-%E6%93%8D%E4%BD%9C%E5%A4%9A%E8%A1%8C%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8F%98%E9%87%8F/) ， [WHOMM](https://whatua.com/author/mxroot/)

