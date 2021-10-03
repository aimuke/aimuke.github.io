---
title: "第十四章、Linux 账号管理与 ACL 权限配置"
tags: [linux, user, group]
--- 

> 要登陆 Linux 系统一定要有账号与口令才行，否则怎么登陆，您说是吧？不过，不同的使用者应该要拥有不同的权限才行吧？我们还可以透过 `user/group` 的特殊权限配置， 来规范出不同的群组开发项目呢～在 Linux 的环境下，我们可以透过很多方式来限制用户能够使用的系统资源， 包括 [十一章、bash]() 提到的 `ulimit` 限制、还有特殊权限限制，如 `umask` 等等。 透过这些举动，我们可以规范出不同使用者的使用资源。另外，还记得系统管理员的账号吗？对！ 就是 `root` 。请问一下，除了 `root` 之外，是否可以有其他的系统管理员账号？ 为什么大家都要尽量避免使用数字型态的账号？如何修改用户相关的信息呢？这些我们都得要了解了解的！



# 1.Linux 的账号与群组
管理员的工作中，相当重要的一环是『`管理账号`』！因为整个系统都是你在管理的， 并且所有一般用户的账号申请，都必须要透过你的协助才行！所以你就必须要了解一下如何管理好一个服务器主机的账号！ 在管理 Linux 主机的账号时，我们必须先来了解一下 Linux 到底是如何辨别每一个使用者的！


## 1.1 使用者标识符： UID 与 GID

虽然我们登陆 Linux 主机的时候，输入的是我们的账号，但是其实 Linux 主机并不会直接认识你的『`账号名称`』的，他仅认识 `ID` (`ID` 就是一组号码)。 由于计算机仅认识 `0` 与 `1`，所以主机对于数字比较有概念的；至于账号只是为了让人们容易记忆而已。 而你的 `ID` 与账号的对应就在 `/etc/passwd` 当中。

> Tips:
>
> 如果你曾经在网络上下载过 `tarball` 类型的文件， 那么应该不难发现，在解压缩之后的文件中，文件拥有者的字段竟然显示『`不明的数字`』？奇怪吧？这没什么好奇怪的，因为 Linux 说实在话，他真的只认识代表你身份的号码而已！

那么到底有几种 `ID` 呢？还记得我们在 [第六章]() 内有提到过， 每一个文件都具有『`拥有人与拥有群组`』的属性吗？没错啦～每个登陆的使用者至少都会取得两个 `ID` ，一个是使用者 ID (User ID ，简称 `UID`)、一个是群组 ID (Group ID ，简称 `GID`)。

那么文件如何判别他的拥有者与群组呢？其实就是利用 `UID` 与 `GID`！每一个文件都会有所谓的拥有者 ID 与拥有群组 ID ，当我们有要显示文件属性的需求时，系统会依据 `/etc/passwd` 与 `/etc/group` 的内容， 找到 `UID / GID` 对应的账号与组名再显示出来！我们可以作个小实验，你可以用 `root` 的身份 `vi /etc/passwd` ，然后将你的一般身份的使用者的 `ID` 随便改一个号码，然后再到你的一般身份的目录下看看原先该账号拥有的文件，你会发现该文件的拥有人变成了 『`数字了`』呵呵！这样可以理解了吗？来看看底下的例子：

```sh
# 1. 先察看一下，系统里面有没有一个名为 dmtsai 的用户？
[root@www ~]# grep 'dmtsai' /etc/passwd
dmtsai:x:503:504::/home/dmtsai:/bin/bash   <==是有这个账号喔！
[root@www ~]# ll -d /home/dmtsai
drwx------ 4 dmtsai dmtsai 4096 Feb  6 18:25 /home/dmtsai
# 瞧一瞧，使用者的字段正是 dmtsai 本身喔！

# 2. 修改一下，将刚刚我们的 dmtsai 的 503 UID 改为 2000 看看：
[root@www ~]# vi /etc/passwd
....(前面省略)....
dmtsai:x:2000:504::/home/dmtsai:/bin/bash <==修改一下特殊字体部分，由 503 改过来
[root@www ~]# ll -d /home/dmtsai
drwx------ 4 503 dmtsai 4096 Feb  6 18:25 /home/dmtsai
# 很害怕吧！怎么变成 503 了？因为文件只会记录数字而已！
# 因为我们乱改，所以导致 503 找不到对应的账号，因此显示数字！

# 3. 记得将刚刚的 2000 改回来！
[root@www ~]# vi /etc/passwd
....(前面省略)....
dmtsai:x:503:504::/home/dmtsai:/bin/bash <==赶紧改回来！
```

你一定要了解的是，上面的例子仅是在说明 `UID` 与账号的对应性，在一部正常运行的 Linux 主机环境下，上面的动作不可随便进行， 这是因为系统上已经有很多的数据被创建存在了，随意修改系统上某些账号的 `UID` 很可能会导致某些程序无法进行，这将导致系统无法顺利运行的结果。 因为权限的问题啊！所以，了解了之后，请赶快回到 `/etc/passwd` 里面，将数字改回来喔！

> Tips:
> 
> 举例来说，如果上面的测试最后一个步骤没有将 `2000` 改回原本的 `UID` ，那么当 `dmtsai` 下次登陆时将没有办法进入自己的家目录！ 因为他的 `UID` 已经改为 `2000` ，但是他的家目录 (`/home/dmtsai`) 却记录的是 `503` ，由于权限是 `700` ， 因此他将无法进入原本的家目录！是否非常严重啊？

## 使用者账号
Linux 系统上面的用户如果需要登陆主机以取得 `shell` 的环境来工作时，他需要如何进行呢？ 首先，他必须要在计算机前面利用 `tty1~tty7` 的终端机提供的 `login` 接口，并输入账号与口令后才能够登陆。 如果是透过网络的话，那至少使用者就得要学习 `ssh` 这个功能了 (服务器篇再来谈)。 那么你输入账号口令后，系统帮你处理了什么呢？

1. 先找寻 `/etc/passwd` 里面是否有你输入的账号？如果没有则跳出，如果有的话则将该账号对应的 `UID` 与 `GID` (在 `/etc/group` 中) 读出来，另外，该账号的家目录与 `shell` 配置也一并读出；

2. 再来则是核对口令表啦！这时 Linux 会进入 `/etc/shadow` 里面找出对应的账号与 `UID`，然后核对一下你刚刚输入的口令与里头的口令是否相符？

3. 如果一切都 OK 的话，就进入 Shell 控管的阶段啰！

大致上的情况就像这样，所以当你要登陆你的 Linux 主机的时候，那个 `/etc/passwd` 与 `/etc/shadow` 就必须要让系统读取啦 (这也是很多攻击者会将特殊账号写到 `/etc/passwd` 里头去的缘故)，所以呢，如果你要备份 Linux 的系统的账号的话，那么这两个文件就一定需要备份才行呦！

由上面的流程我们也知道，跟使用者账号有关的有两个非常重要的文件，一个是管理使用者 `UID/GID` 重要参数的 `/etc/passwd` ，一个则是专门管理口令相关数据的 `/etc/shadow` 啰！那这两个文件的内容就非常值得进行研究啦！ 底下我们会简单的介绍这两个文件，详细的说明可以参考 man 5 passwd 及 man 5 shadow (注1)。


## /etc/passwd 文件结构

这个文件的构造是这样的：每一行都代表一个账号，有几行就代表有几个账号在你的系统中！ 不过需要特别留意的是，里头很多账号本来就是系统正常运行所必须要的，我们可以简称他为系统账号， 例如 `bin`, `daemon`, `adm`, `nobody` 等等，这些账号请不要随意的杀掉他呢！ 这个文件的内容有点像这样：

> Tips:
> 
> 鸟哥在接触 Linux 之前曾经碰过 Solaris 系统 (1999 年)，当时鸟哥啥也不清楚！由于『听说』Linux 上面的账号越复杂会导致系统越危险！所以鸟哥就将 `/etc/passwd` 上面的账号全部删除到只剩下 `root` 与鸟哥自己用的一般账号！结果你猜发生什么事？那就是....呼叫升阳的工程师来维护系统 @_@！糗到一个不行！大家不要学啊！	鸟

```sh
[root@www ~]# head -n 4 /etc/passwd
root:x:0:0:root:/root:/bin/bash  <==等一下做为底下说明用
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
```

我们先来看一下每个 Linux 系统都会有的第一行，就是 `root` 这个系统管理员那一行好了， 你可以明显的看出来，每一行使用『`:`』分隔开，共有七个咚咚，分别是：

### 账号名称
就是账号啦！用来对应 `UID` 的。例如 `root` 的 `UID` 对应就是 `0` (第三字段)；

### 口令
早期 Unix 系统的口令就是放在这字段上！但是因为这个文件的特性是所有的程序都能够读取，这样一来很容易造成口令数据被窃取， 因此后来就将这个字段的口令数据给他改放到 `/etc/shadow` 中了。所以这里你会看到一个『 `x` 』，呵呵！

### UID
这个就是使用者标识符啰！通常 Linux 对于 `UID` 有几个限制需要说给您了解一下：

|id 范围 |该 ID 使用者特性|
|:------|:---|
|0<br>(系统管理员)|	当 UID 是 0 时，代表这个账号是『系统管理员』！ 所以当你要让其他的账号名称也具有 root 的权限时，将该账号的 UID 改为 0 即可。 这也就是说，一部系统上面的系统管理员不见得只有 root 喔！ 不过，很不建议有多个账号的 UID 是 0 啦～|
|1~499<br>(系统账号)	|保留给系统使用的 ID，其实除了 0 之外，其他的 UID 权限与特性并没有不一样。默认 500 以下的数字让给系统作为保留账号只是一个习惯。<br><br>由于系统上面启动的服务希望使用较小的权限去运行，因此不希望使用 root 的身份去运行这些服务， 所以我们就得要提供这些运行中程序的拥有者账号才行。这些系统账号通常是不可登陆的， 所以才会有我们在第十一章提到的 /sbin/nologin 这个特殊的 shell 存在。<br><br>根据系统账号的由来，通常系统账号又约略被区分为两种：<br> 1~99：由 distributions 自行创建的系统账号；<br> 100~499：若用户有系统账号需求时，可以使用的账号 UID。
|500~65535<br>(可登陆账号)|	给一般使用者用的。事实上，目前的 linux 核心 (2.6.x 版)已经可以支持到 4294967295 (2^32-1) 这么大的 UID 号码喔！|

上面这样说明可以了解了吗？是的， `UID` 为 `0` 的时候，就是 `root` 呦！所以请特别留意一下你的 `/etc/passwd` 文件！

### GID
这个与 `/etc/group` 有关！其实 `/etc/group` 的观念与 `/etc/passwd` 差不多，只是他是用来规范组名与 `GID` 的对应而已！

### 用户信息说明栏
这个字段基本上并没有什么重要用途，只是用来解释这个账号的意义而已！不过，如果您提供使用 `finger` 的功能时， 这个字段可以提供很多的信息呢！本章后面的 `chfn` 命令会来解释这里的说明。

### 家目录
这是用户的家目录，以上面为例， `root` 的家目录在 `/root` ，所以当 `root` 登陆之后，就会立刻跑到 `/root` 目录里头啦！呵呵！ 如果你有个账号的使用空间特别的大，你想要将该账号的家目录移动到其他的硬盘去该怎么作？ 没有错！可以在这个字段进行修改呦！默认的用户家目录在 `/home/yourIDname`

### Shell

我们在[第十一章 BASH]() 提到很多次，当用户登陆系统后就会取得一个 Shell 来与系统的核心沟通以进行用户的操作任务。那为何默认 shell 会使用 `bash` 呢？就是在这个字段指定的啰！ 这里比较需要注意的是，有一个 shell 可以用来替代成让账号无法取得 shell 环境的登陆动作！那就是 `/sbin/nologin` 这个东西！这也可以用来制作纯 pop 邮件账号者的数据呢！

## /etc/shadow 文件结构

我们知道很多程序的运行都与权限有关，而权限与 `UID/GID` 有关！因此各程序当然需要读取 `/etc/passwd` 来了解不同账号的权限。 因此 `/etc/passwd` 的权限需配置为 `-rw-r--r--` 这样的情况， 虽然早期的口令也有加密过，但却放置到 `/etc/passwd` 的第二个字段上！这样一来很容易被有心人士所窃取的， 加密过的口令也能够透过暴力破解法去 try and error (试误) 找出来！

因为这样的关系，所以后来发展出将口令移动到 `/etc/shadow` 这个文件分隔开来的技术， 而且还加入很多的口令限制参数在 `/etc/shadow` 里头呢！在这里，我们先来了解一下这个文件的构造吧！ 鸟哥的 `/etc/shadow` 文件有点像这样：

```sh
[root@www ~]# head -n 4 /etc/shadow
root:$1$/30QpE5e$y9N/D0bh6rAACBEz.hqo00:14126:0:99999:7:::  <==底下说明用
bin:*:14126:0:99999:7:::
daemon:*:14126:0:99999:7:::
adm:*:14126:0:99999:7:::
```

基本上， `shadow` 同样以『`:`』作为分隔符，如果数一数，会发现共有九个字段啊，这九个字段的用途是这样的：

### 账号名称
由于口令也需要与账号对应啊～因此，这个文件的第一栏就是账号，必须要与 `/etc/passwd` 相同才行！

### 口令
这个字段内的数据才是真正的口令，而且是经过编码的口令 (加密) 啦！ 你只会看到有一些特殊符号的字母就是了！需要特别留意的是，虽然这些加密过的口令很难被解出来， 但是『很难』不等于『不会』，所以，这个文件的默认权限是『`-rw-------`』或者是『`-r--------`』，亦即只有 `root` 才可以读写就是了！你得随时注意，不要不小心更动了这个文件的权限呢！

另外，由于各种口令编码的技术不一样，因此不同的编码系统会造成这个字段的长度不相同。 举例来说，旧式的 `DES` 编码系统产生的口令长度就与目前惯用的 `MD5` 不同(注2)！ `MD5` 的口令长度明显的比较长些。由于固定的编码系统产生的口令长度必须一致，因此『当你让这个字段的长度改变后，该口令就会失效(算不出来)』。 很多软件透过这个功能，在此字段前加上 `!` 或 `*` 改变口令字段长度，就会让口令『`暂时失效`』了。

### 最近更动口令的日期
这个字段记录了『`更动口令那一天`』的日期，不过，很奇怪呀！在我的例子中怎么会是 `14126` 呢？呵呵，这个是因为计算 Linux 日期的时间是以 `1970 年 1 月 1 日` 作为 `1` 而累加的日期，1971 年 1 月 1 日则为 366 啦！ 得注意一下这个数据呦！上述的 `14126` 指的就是 `2008-09-04` 那一天啦！了解乎？ 而想要了解该日期可以使用本章后面 `chage` 命令的帮忙！至于想要知道某个日期的累积日数， 可使用如下的程序计算：

```sh
[root@www ~]# echo $(($(date --date="2008/09/04" +%s)/86400+1))
14126
```

上述命令中，`2008/09/04` 为你想要计算的日期， `86400` 为每一天的秒数， `%s` 为 `1970/01/01` 以来的累积总秒数。 由于 `bash` 仅支持整数，因此最终需要加上 `1` 补齐 `1970/01/01` 当天。

### 口令不可被更动的天数：(与第 3 字段相比)
第四个字段记录了：这个账号的口令在最近一次被更改后需要经过几天才可以再被变更！如果是 `0` 的话， 表示口令随时可以更动的意思。这的限制是为了怕口令被某些人一改再改而设计的！如果配置为 `20` 天的话，那么当你配置了口令之后， `20` 天之内都无法改变这个口令呦！

### 口令需要重新变更的天数：(与第 3 字段相比)
经常变更口令是个好习惯！为了强制要求用户变更口令，这个字段可以指定在最近一次更改口令后， 在多少天数内需要再次的变更口令才行。你必须要在这个天数内重新配置你的口令，否则这个账号的口令将会『变为过期特性』。 而如果像上面的 99999 (计算为 273 年) 的话，那就表示，呵呵，口令的变更没有强制性之意。

### 口令需要变更期限前的警告天数：(与第 5 字段相比)
当账号的口令有效期限快要到的时候 (第 5 字段)，系统会依据这个字段的配置，发出『警告』言论给这个账号，提醒他『再过 n 天你的口令就要过期了，请尽快重新配置你的口令呦！』，如上面的例子，则是口令到期之前的 7 天之内，系统会警告该用户。

### 口令过期后的账号宽限时间(口令失效日)：(与第 5 字段相比)
口令有效日期为『升级日期(第3字段)』+『重新变更日期(第5字段)』，过了该期限后用户依旧没有升级口令，那该口令就算过期了。 虽然口令过期但是该账号还是可以用来进行其他工作的，包括登陆系统取得 bash 。不过如果口令过期了， 那当你登陆系统时，系统会强制要求你必须要重新配置口令才能登陆继续使用喔，这就是口令过期特性。

那这个字段的功能是什么呢？是在口令过期几天后，如果使用者还是没有登陆更改口令，那么这个账号的口令将会『失效』， 亦即该账号再也无法使用该口令登陆了。要注意口令过期与口令失效并不相同。

### 账号失效日期
这个日期跟第三个字段一样，都是使用 1970 年以来的总日数配置。这个字段表示： 这个账号在此字段规定的日期之后，将无法再使用。 就是所谓的『账号失效』，此时不论你的口令是否有过期，这个『账号』都不能再被使用！ 这个字段会被使用通常应该是在『收费服务』的系统中，你可以规定一个日期让该账号不能再使用啦！

### 保留
最后一个字段是保留的，看以后有没有新功能加入。
举个例子来说好了，假如我的 `dmtsai` 这个用户的口令栏如下所示：

```sh
dmtsai:$1$vyUuj.eX$omt6lKJvMcIZHx4H7RI1V.:14299:5:60:7:5:14419:
```

这表示什么呢？先要注意的是 `14299` 是 `2009/02/24` 。所以 `dmtsai` 这个用户的口令相关意义是：

- 由于口令几乎仅能单向运算(由明码计算成为口令，无法由口令反推回明码)，因此由上表的数据我们`无法得知 dmstai 的实际口令明文`；

- 此账号最近一次更动口令的日期是 2009/02/24 (14299)；

- 能够再次修改口令的时间是 5 天以后，也就是 `2009/03/01` 以前 `dmtsai` 不能修改自己的口令；如果用户还是尝试要更动自己的口令，系统就会出现这样的信息：

    ```sh
    You must wait longer to change your password
    passwd: Authentication token manipulation error
    ```

    画面中告诉我们：你必须要等待更久的时间才能够变更口令之意啦！

- 由于口令过期日期定义为 60 天后，亦即累积日数为： `14299+60=14359`，经过计算得到此日数代表日期为 `2009/04/25`。 这表示：『使用者必须要在 `2009/03/01` 到 `2009/04/25` 之间的 60 天限制内去修改自己的口令，若 `2009/04/25` 之后还是没有变更口令时，该口令就宣告为过期』了！

- 警告日期设为 7 天，亦即是口令过期日前的 7 天，在本例中则代表 2009/04/19 ~ 2009/04/25 这七天。 如果用户一直没有更改口令，那么在这 7 天中，只要 dmtsai 登陆系统就会发现如下的信息：

    ```
    Warning: your password will expire in 5 days
    ```

- 如果该账号一直到 2009/04/25 都没有更改口令，那么口令就过期了。但是由于有 5 天的宽限天数， 因此 dmtsai 在 2009/04/30 前都还可以使用旧口令登陆主机。 不过登陆时会出现强制更改口令的情况，画面有点像底下这样：

    ```
    You are required to change your password immediately (password aged)
    WARNING: Your password has expired.
    You must change your password now and login again!
    Changing password for user dmtsai.
    Changing password for dmtsai
    (current) UNIX password:
    ```

    你必须要输入一次旧口令以及两次新口令后，才能够开始使用系统的各项资源。如果你是在 2009/04/30 以后尝试以 dmtsai 登陆的话，那么就会出现如下的错误信息且无法登陆，因为此时你的口令就失效去啦！

    ```
    Your account has expired; please contact your system administrator
    ```

- 如果使用者在 2009/04/25 以前变更过口令，那么第 3 个字段的那个 14299 的天数就会跟着改变，因此， 所有的限制日期也会跟着相对变动喔！^_^

- 无论使用者如何动作，到了 14419 (大约是 2009/07/24 左右) 该账号就失效了～

透过这样的说明，您应该会比较容易理解了吧？由于 shadow 有这样的重要性，因此可不能随意修改喔！ 但在某些情况底下你得要使用各种方法来处理这个文件的！举例来说，常常听到人家说：『我的口令忘记了』， 或者是『我的口令不晓得被谁改过，跟原先的不一样了』，这个时候怎么办？

- 一般用户的口令忘记了：这个最容易解决，请系统管理员帮忙， 他会重新配置好你的口令而不需要知道你的旧口令！利用 `root` 的身份使用 `passwd` 命令来处理即可。

- `root` 口令忘记了：这就麻烦了！因为你无法使用 `root` 的身份登陆了嘛！ 但我们知道 `root` 的口令在 `/etc/shadow` 当中，因此你可以使用各种可行的方法启动进入 Linux 再去修改。 例如重新启动进入单人维护模式([第二十章]())后，系统会主动的给予 `root` 权限的 `bash` 接口， 此时再以 `passwd` 修改口令即可；或以 Live CD 启动后挂载根目录去修改 `/etc/shadow`，将里面的 `root` 的口令字段清空， 再重新启动后 `root` 将不用口令即可登陆！登陆后再赶快以 `passwd` 命令去配置 `root` 口令即可。

> Tips:
> 
> 曾经听过一则笑话，某位老师主要是在教授 Linux 操作系统，但是他是兼任的老师，因此对于该系的计算机环境不熟。 由于当初安装该计算机教室 Linux 操作系统的人员已经离职且找不到联络方式了，也就是说 root 口令已经没有人晓得了！ 此时该老师就对学生说：『在 Linux 里面 root 口令不见了，我们只能重新安装』...感觉有点无力～ 又是个被 Windows 制约的人才！

# 关于群组： 有效与初始群组、groups, newgrp
认识了账号相关的两个文件 `/etc/passwd` 与 `/etc/shadow` 之后，你或许还是会觉得奇怪， 那么群组的配置文件在哪里？还有，在 `/etc/passwd` 的第四栏不是所谓的 `GID` 吗？那又是啥？ 呵呵～此时就需要了解 `/etc/group` 与 `/etc/gshadow` 啰～


## /etc/group 文件结构
这个文件就是在记录 `GID` 与组名的对应了～鸟哥测试机的 `/etc/group` 内容有点像这样：

```sh
[root@www ~]# head -n 4 /etc/group
root:x:0:root
bin:x:1:root,bin,daemon
daemon:x:2:root,bin,daemon
sys:x:3:root,bin,adm
```

这个文件每一行代表一个群组，也是以冒号『`:`』作为字段的分隔符，共分为四栏，每一字段的意义是：

### 组名
就是组名啦！

### 群组口令
通常不需要配置，这个配置通常是给『`群组管理员`』使用的，目前很少有这个机会配置群组管理员啦！ 同样的，口令已经移动到 `/etc/gshadow` 去，因此这个字段只会存在一个『`x`』而已；

### GID
就是群组的 `ID` 啊。我们 `/etc/passwd` 第四个字段使用的 `GID` 对应的群组名，就是由这里对应出来的！

### 此群组支持的账号名称
我们知道一个账号可以加入多个群组，那某个账号想要加入此群组时，将该账号填入这个字段即可。 举例来说，如果我想要让 `dmtsai` 也加入 `root` 这个群组，那么在第一行的最后面加上『`,dmtsai`』，注意不要有空格， 使成为『 `root:x:0:root,dmtsai` 』就可以啰～

谈完了 `/etc/passwd`, `/etc/shadow`, `/etc/group` 之后，我们可以使用一个简单的图示来了解一下 `UID / GID` 与口令之间的关系， 图示如下。其实重点是 `/etc/passwd` 啦，其他相关的数据都是根据这个文件的字段去找寻出来的。 下图中， `root` 的 `UID` 是 `0` ，而 `GID` 也是 `0` ，去找 `/etc/group` 可以知道 `GID` 为 `0` 时的组名就是 `root` 哩。 至于口令的寻找中，会找到 `/etc/shadow` 与 `/etc/passwd` 内同账号名称的那一行，就是口令相关数据啰。

![图 1.3.1 、账号相关文件之间的 UID/GID 与口令相关性示意图](http://cn.linux.vbird.org/linux_basic/0410accountmanager_files/id_link.gif)

至于在 `/etc/group` 比较重要的特色在于第四栏啦，因为每个使用者都可以拥有多个支持的群组，这就好比在学校念书的时候， 我们可以加入多个社团一样！ ^_^。不过这里你或许会觉得奇怪的，那就是：『`假如我同时加入多个群组，那么我在作业的时候，到底是以那个群组为准？`』 底下我们就来谈一谈这个『`有效群组`』的概念。

## 有效群组(effective group)与初始群组(initial group)
还记得每个使用者在他的 `/etc/passwd` 里面的第四栏有所谓的 `GID` 吧？那个 `GID` 就是所谓的『`初始群组 (initial group)` 』！也就是说，当用户一登陆系统，立刻就拥有这个群组的相关权限的意思。 举例来说，我们上面提到 `dmtsai` 这个使用者的 `/etc/passwd` 与 `/etc/group` 还有 `/etc/gshadow` 相关的内容如下：

```sh
[root@www ~]# usermod -G users dmtsai  <==先配置好次要群组
[root@www ~]# grep dmtsai /etc/passwd /etc/group /etc/gshadow
/etc/passwd:dmtsai:x:503:504::/home/dmtsai:/bin/bash
/etc/group:users:x:100:dmtsai  <==次要群组的配置
/etc/group:dmtsai:x:504:       <==因为是初始群组，所以第四字段不需要填入账号
/etc/gshadow:users:::dmtsai    <==次要群组的配置
/etc/gshadow:dmtsai:!::
```

仔细看到上面这个表格，在 `/etc/passwd` 里面， `dmtsai` 这个使用者所属的群组为 `GID=504` ，搜寻一下 `/etc/group` 得到 `504` 是那个名为 `dmtsai` 的群组啦！这就是 `initial group`。因为是初始群组， 使用者一登陆就会主动取得，不需要在 `/etc/group` 的第四个字段写入该账号的！

但是非 initial group 的其他群组可就不同了。举上面这个例子来说，我将 `dmtsai` 加入 `users` 这个群组当中，由于 `users` 这个群组并非是 `dmtsai` 的初始群组，因此， 我必须要在` /etc/group` 这个文件中，找到 `users` 那一行，并且将 `dmtsai` 这个账号加入第四栏， 这样 `dmtsai` 才能够加入 `users` 这个群组啊。

那么在这个例子当中，因为我的 `dmtsai` 账号同时支持 `dmtsai` 与 `users` 这两个群组， 因此，在读取/写入/运行文件时，针对群组部分，只要是 `users` 与 `dmtsai` 这两个群组拥有的功能， 我 `dmtsai` 这个使用者都能够拥有喔！这样瞭呼？不过，这是针对已经存在的文件而言， 如果今天我要创建一个新的文件或者是新的目录，请问一下，新文件的群组是 `dmtsai` 还是 `users` ？呵呵！这就得要检查一下当时的有效群组了 (`effective group`)。


## groups: 有效与支持群组的观察
如果我以 `dmtsai` 这个使用者的身份登陆后，该如何知道我所有支持的群组呢？ 很简单啊，直接输入 `groups` 就可以了！注意喔，是 `groups` 有加 `s` 呢！结果像这样：

```sh
[dmtsai@www ~]$ groups
dmtsai users
```

在这个输出的信息中，可知道 `dmtsai` 这个用户同时属于 `dmtsai` 及 `users` 这个两个群组，而且， 第一个输出的群组即为有效群组 (`effective group`) 了。 也就是说，我的有效群组为 `dmtsai` 啦～此时，如果我以 `touch` 去创建一个新档，例如： 『 `touch test` 』，那么这个文件的拥有者为 `dmtsai` ，而且群组也是 `dmtsai` 的啦。

```sh
[dmtsai@www ~]$ touch test
[dmtsai@www ~]$ ll
-rw-rw-r-- 1 dmtsai dmtsai 0 Feb 24 17:26 test
```

这样是否可以了解什么是有效群组了？通常有效群组的作用是在新建文件啦！那么有效群组是否能够变换？

## newgrp: 有效群组的切换
那么如何变更有效群组呢？就使用 `newgrp` 啊！不过使用 `newgrp` 是有限制的，那就是你想要切换的群组必须是你已经有支持的群组。举例来说， `dmtsai` 可以在 `dmtsai/users` 这两个群组间切换有效群组，但是 `dmtsai` 无法切换有效群组成为 `sshd` 啦！使用的方式如下：

```sh
[dmtsai@www ~]$ newgrp users
[dmtsai@www ~]$ groups
users dmtsai
[dmtsai@www ~]$ touch test2
[dmtsai@www ~]$ ll
-rw-rw-r-- 1 dmtsai dmtsai 0 Feb 24 17:26 test
-rw-r--r-- 1 dmtsai users  0 Feb 24 17:33 test2
```

此时， `dmtsai` 的有效群组就成为 `users` 了。 我们额外的来讨论一下 `newgrp` 这个命令，这个命令可以变更目前用户的有效群组， 而且是另外以一个 shell 来提供这个功能的喔，所以，以上面的例子来说， `dmtsai` 这个使用者目前是以另一个 shell 登陆的，而且新的 shell 给予 `dmtsai` 有效 `GID` 为 `users` 就是了。如果以图示来看就是如下所示：

![图 1.3.2 、newgrp 的运行示意图](http://cn.linux.vbird.org/linux_basic/0410accountmanager_files/newgrp.gif)

虽然用户的环境配置(例如环境变量等等其他数据)不会有影响，但是使用者的『`群组权限`』将会重新被计算。 但是需要注意，由于是新取得一个 shell ，因此如果你想要回到原本的环境中，请输入 `exit` 回到原本的 shell 喔！

既然如此，也就是说，只要我的用户有支持的群组就是能够切换成为有效群组！好了， 那么如何让一个账号加入不同的群组就是问题的所在啰。你要加入一个群组有两个方式，一个是透过系统管理员 (`root`) 利用 `usermod` 帮你加入，如果 `root` 太忙了而且你的系统有配置群组管理员，那么你可以透过群组管理员以 `gpasswd` 帮你加入他所管理的群组中！详细的作法留待下一小节再来介绍啰！

## /etc/gshadow
刚刚讲了很多关于『`有效群组`』的概念，另外，也提到 newg`rp 这个命令的用法， 但是，如果 `/etc/gshadow` 这个配置没有搞懂得话，那么 `newgrp` 是无法动作的呢！ 鸟哥测试机的 `/etc/gshadow` 的内容有点像这样：

```sh
[root@www ~]# head -n 4 /etc/gshadow
root:::root
bin:::root,bin,daemon
daemon:::root,bin,daemon
sys:::root,bin,adm
```

这个文件内同样还是使用冒号『`:`』来作为字段的分隔字符，而且你会发现，这个文件几乎与 `/etc/group` 一模一样啊！是这样没错～不过，要注意的大概就是第二个字段吧～第二个字段是口令栏， 如果口令栏上面是『!』时，表示该群组不具有群组管理员！至于第四个字段也就是支持的账号名称啰～ 这四个字段的意义为：

1. 组名
2. 口令栏，同样的，开头为 ! 表示无合法口令，所以无群组管理员
3. 群组管理员的账号 (相关信息在 gpasswd 中介绍)
4. 该群组的所属账号 (与 /etc/group 内容相同！)

以系统管理员的角度来说，这个 `gshadow` 最大的功能就是创建群组管理员啦！ 那么什么是群组管理员呢？由于系统上面的账号可能会很多，但是我们 `root` 可能平时太忙碌，所以当有使用者想要加入某些群组时， `root` 或许会没有空管理。此时如果能够创建群组管理员的话，那么`该群组管理员就能够将那个账号加入自己管理的群组中`！ 可以免去 root 的忙碌啦！不过，由于目前有类似 `sudo` 之类的工具， 所以这个群组管理员的功能已经很少使用了。我们会在后续的 gpasswd 中介绍这个实作。

# 账号管理
好啦！既然要管理账号，当然是由新增与移除使用者开始的啰～底下我们就分别来谈一谈如何新增、 移除与更改用户的相关信息吧～

## 新增与移除使用者： useradd, 相关配置文件, passwd, usermod, userdel
要如何在 Linux 的系统新增一个用户啊？呵呵～真是太简单了～我们登陆系统时会输入 (1)账号与 (2)口令， 所以创建一个可用的账号同样的也需要这两个数据。那账号可以使用 `useradd` 来新建用户，口令的给予则使用 `passwd` 这个命令！这两个命令下达方法如下：

```sh
useradd
[root@www ~]# useradd [-u UID] [-g 初始群组] [-G 次要群组] [-mM]\
>  [-c 说明栏] [-d 家目录绝对路径] [-s shell] 使用者账号名
选项与参数：
-u  ：后面接的是 UID ，是一组数字。直接指定一个特定的 UID 给这个账号；
-g  ：后面接的那个组名就是我们上面提到的 initial group 啦～
      该群组的 GID 会被放置到 /etc/passwd 的第四个字段内。
-G  ：后面接的组名则是这个账号还可以加入的群组。
      这个选项与参数会修改 /etc/group 内的相关数据喔！
-M  ：强制！不要创建用户家目录！(系统账号默认值)
-m  ：强制！要创建用户家目录！(一般账号默认值)
-c  ：这个就是 /etc/passwd 的第五栏的说明内容啦～可以随便我们配置的啦～
-d  ：指定某个目录成为家目录，而不要使用默认值。务必使用绝对路径！
-r  ：创建一个系统的账号，这个账号的 UID 会有限制 (参考 /etc/login.defs)
-s  ：后面接一个 shell ，若没有指定则默认是 /bin/bash 的啦～
-e  ：后面接一个日期，格式为『YYYY-MM-DD』此项目可写入 shadow 第八字段，
      亦即账号失效日的配置项目啰；
-f  ：后面接 shadow 的第七字段项目，指定口令是否会失效。0为立刻失效，
      -1 为永远不失效(口令只会过期而强制于登陆时重新配置而已。)
```

范例一：完全参考默认值创建一个用户，名称为 `vbird1`

```sh
[root@www ~]# useradd vbird1
[root@www ~]# ll -d /home/vbird1
drwx------ 4 vbird1 vbird1 4096 Feb 25 09:38 /home/vbird1
# 默认会创建用户家目录，且权限为 700 ！这是重点！

[root@www ~]# grep vbird1 /etc/passwd /etc/shadow /etc/group
/etc/passwd:vbird1:x:504:505::/home/vbird1:/bin/bash
/etc/shadow:vbird1:!!:14300:0:99999:7:::
/etc/group:vbird1:x:505:    <==默认会创建一个与账号一模一样的群组名
```

其实系统已经帮我们规定好非常多的默认值了，所以我们可以简单的使用『 `useradd` 账号 』来创建使用者即可。 CentOS 这些默认值主要会帮我们处理几个项目：

- 在 `/etc/passwd` 里面创建一行与账号相关的数据，包括创建 UID/GID/家目录等；
- 在 `/etc/shadow` 里面将此账号的口令相关参数填入，但是尚未有口令；
- 在 `/etc/group` 里面加入一个与账号名称一模一样的组名；
- 在 `/home` 底下创建一个与账号同名的目录作为用户家目录，且权限为 700

由于在 `/etc/shadow` 内仅会有口令参数而不会有加密过的口令数据，因此我们在创建使用者账号时， 还需要使用『 `passwd 账号` 』来给予口令才算是完成了用户创建的流程。如果由于特殊需求而需要改变使用者相关参数时， 就得要透过上述表格中的选项来进行创建了，参考底下的案例：

范例二：假设我已知道我的系统当中有个组名为 `users` ，且 `UID` `700` 并不存在，请用 `users` 为初始群组，以及 `uid` 为 `700` 来创建一个名为 `vbird2` 的账号
```sh
[root@www ~]# useradd -u 700 -g users vbird2
[root@www ~]# ll -d /home/vbird2
drwx------ 4 vbird2 users 4096 Feb 25 09:59 /home/vbird2

[root@www ~]# grep vbird2 /etc/passwd /etc/shadow /etc/group
/etc/passwd:vbird2:x:700:100::/home/vbird2:/bin/bash
/etc/shadow:vbird2:!!:14300:0:99999:7:::
# 看一下，UID 与 initial group 确实改变成我们需要的了！
```

在这个范例中，我们创建的是指定一个已经存在的群组作为使用者的初始群组，因为群组已经存在， 所以在 `/etc/group 里面就不会主动的创建与账号同名的群组了`！ 此外，我们也指定了特殊的 `UID` 来作为使用者的专属 `UID` 喔！了解了一般账号后，我们来瞧瞧那啥是系统账号 (`system account`) 吧！

范例三：创建一个系统账号，名称为 `vbird3`

```sh
[root@www ~]# useradd -r vbird3
[root@www ~]# ll -d /home/vbird3
ls: /home/vbird3: No such file or directory  <==不会主动创建家目录

[root@www ~]# grep vbird3 /etc/passwd /etc/shadow /etc/group
/etc/passwd:vbird3:x:100:103::/home/vbird3:/bin/bash
/etc/shadow:vbird3:!!:14300::::::
/etc/group:vbird3:x:103:
```

我们在谈到 `UID` 的时候曾经说过一般账号应该是 `500` 号以后，那用户自己创建的系统账号则一般是由 `100` 号以后起算的。 所以在这里我们加上 `-r` 这个选项以后，系统就会主动将账号与账号同名群组的 `UID/GID` 都指定小于 `500` 以下， 在本案例中则是使用 `100`(UID) 与 `103`(GID) 啰！此外，由于系统账号主要是用来进行运行系统所需服务的权限配置， 所以`系统账号默认都不会主动创建家目录的`！

由这几个范例我们也会知道，使用 `useradd` 创建使用者账号时，其实会更改不少地方，至少我们就知道底下几个文件：

- 用户账号与口令参数方面的文件：`/etc/passwd`, `/etc/shadow`
- 使用者群组相关方面的文件：/etc/group, /etc/gshadow
- 用户的家目录：/home/账号名称

那请教一下，你有没有想过，为何『 `useradd vbird1` 』会主动在 `/home/vbird1` 创建起用户的家目录？家目录内有什么数据且来自哪里？为何默认使用的是 `/bin/bash` 这个 shell ？为何口令字段已经都规范好了 (`0:99999:7` 那一串)？呵呵！这就得要说明一下 `useradd` 所使用的参考文件啰！


## useradd 参考档

其实 `useradd` 的默认值可以使用底下的方法呼叫出来：

```sh
[root@www ~]# useradd -D
GROUP=100		<==默认的群组
HOME=/home		<==默认的家目录所在目录
INACTIVE=-1		<==口令失效日，在 shadow 内的第 7 栏
EXPIRE=			<==账号失效日，在 shadow 内的第 8 栏
SHELL=/bin/bash		<==默认的 shell
SKEL=/etc/skel		<==用户家目录的内容数据参考目录
CREATE_MAIL_SPOOL=yes   <==是否主动帮使用者创建邮件信箱(mailbox)
```

这个数据其实是由 `/etc/default/useradd` 呼叫出来的！你可以自行用 `vim` 去观察该文件的内容。搭配上头刚刚谈过的范例一的运行结果，上面这些配置项目所造成的行为分别是：

- GROUP=100：新建账号的初始群组使用 GID 为 100 者

    系统上面 `GID` 为 `100` 者即是 `users` 这个群组，此配置项目指的就是让新设使用者账号的初始群组为 `users` 这一个的意思。 但是我们知道 CentOS 上面并不是这样的，在 CentOS 上面默认的群组为与账号名相同的群组。 举例来说， `vbird1` 的初始群组为 `vbird1` 。怎么会这样啊？这是因为针对群组的角度有两种不同的机制所致， 这两种机制分别是：

    - 私有群组机制：系统会创建一个与账号一样的群组给使用者作为初始群组。 这种群组的配置机制会比较有保密性，这是因为使用者都有自己的群组，而且家目录权限将会配置为 `700` (仅有自己可进入自己的家目录) 之故。使用这种机制将不会参考 `GROUP=100` 这个配置值。代表性的 distributions 有 RHEL, Fedora, CentOS 等；

    - 公共群组机制：就是以 `GROUP=100` 这个配置值作为新建账号的初始群组，因此每个账号都属于 `users` 这个群组， 且默认家目录通常的权限会是『 `drwxr-xr-x ... username users ...` 』，由于每个账号都属于 `users` 群组，因此大家都可以互相分享家目录内的数据之故。代表 `distributions` 如 SuSE等。

    由于我们的 CentOS 使用私有群组机制，因此这个配置项目是不会生效的！不要太紧张啊！

- HOME=/home：用户家目录的基准目录(basedir)

    用户的家目录通常是与账号同名的目录，这个目录将会摆放在此配置值的目录后。所以 vbird1 的家目录就会在 /home/vbird1/ 了！很容易理解吧！

- INACTIVE=-1：口令过期后是否会失效的配置值

    我们在 `shadow` 文件结构当中谈过，第七个字段的配置值将会影响到口令过期后， 在多久时间内还可使用旧口令登陆。这个项目就是在指定该日数啦！如果是 `0` 代表口令过期立刻失效， 如果是 `-1` 则是代表口令永远不会失效，如果是数字，如 `30` ，则代表过期 `30` 天后才失效。

- EXPIRE=：账号失效的日期

    就是 `shadow` 内的第八字段，你可以直接配置账号在哪个日期后就直接失效，而不理会口令的问题。 通常不会配置此项目，但如果是付费的会员制系统，或许这个字段可以配置喔！

- SHELL=/bin/bash：默认使用的 shell 程序文件名

    系统默认的 shell 就写在这里。假如你的系统为 mail server ，你希望每个账号都只能使用 email 的收发信件功能， 而不许用户登陆系统取得 shell ，那么可以将这里配置为 `/sbin/nologin` ，如此一来，新建的使用者默认就无法登陆！ 也免去后续使用 `usermod` 进行修改的手续！

- SKEL=/etc/skel：用户家目录参考基准目录

    这个咚咚就是指定用户家目录的参考基准目录啰～举我们的范例一为例， `vbird1` 家目录 `/home/vbird1` 内的各项数据，都是由 `/etc/skel` 所复制过去的～所以呢，未来如果我想要让新增使用者时，该用户的环境变量 `~/.bashrc` 就配置妥当的话，您可以到 `/etc/skel/.bashrc` 去编辑一下，也可以创建 `/etc/skel/www` 这个目录，那么未来新增使用者后，在他的家目录下就会有 `www` 那个目录了！这样瞭呼？

- CREATE_MAIL_SPOOL=yes：创建使用者的 mailbox

    你可以使用『 `ll /var/spool/mail/vbird1` 』看一下，会发现有这个文件的存在喔！这就是使用者的邮件信箱！

除了这些基本的账号配置值之外， `UID/GID` 还有口令参数又是在哪里参考的呢？那就得要看一下 `/etc/login.defs` 啦！ 这个文件的内容有点像底下这样：

```sh
MAIL_DIR        /var/spool/mail	<==用户默认邮件信箱放置目录

PASS_MAX_DAYS   99999	<==/etc/shadow 内的第 5 栏，多久需变更口令日数
PASS_MIN_DAYS   0	<==/etc/shadow 内的第 4 栏，多久不可重新配置口令日数
PASS_MIN_LEN    5	<==口令最短的字符长度，已被 pam 模块取代，失去效用！
PASS_WARN_AGE   7	<==/etc/shadow 内的第 6 栏，过期前会警告的日数

UID_MIN         500	<==使用者最小的 UID，意即小于 500 的 UID 为系统保留
UID_MAX       60000	<==使用者能够用的最大 UID
GID_MIN         500	<==使用者自定义组的最小 GID，小于 500 为系统保留
GID_MAX       60000	<==使用者自定义组的最大 GID

CREATE_HOME     yes	<==在不加 -M 及 -m 时，是否主动创建用户家目录？
UMASK           077     <==用户家目录创建的 umask ，因此权限会是 700
USERGROUPS_ENAB yes     <==使用 userdel 删除时，是否会删除初始群组
MD5_CRYPT_ENAB yes      <==口令是否经过 MD5 的加密机制处理
```

这个文件规范的数据则是如下所示：

- mailbox 所在目录：
    用户的默认 mailbox 文件放置的目录在 /var/spool/mail，所以 vbird1 的 mailbox 就是在 /var/spool/mail/vbird1 啰！

- shadow 口令第 4, 5, 6 字段内容：
    透过 PASS_MAX_DAYS 等等配置值来指定的！所以你知道为何默认的 /etc/shadow 内每一行都会有『 0:99999:7 』的存在了吗？^_^！不过要注意的是，由于目前我们登陆时改用 PAM 模块来进行口令检验，所以那个 PASS_MIN_LEN 是失效的！

- UID/GID 指定数值：
    虽然 Linux 核心支持的账号可高达 232 这么多个，不过一部主机要作出这么多账号在管理上也是很麻烦的！ 所以在这里就针对 UID/GID 的范围进行规范就是了。上表中的 UID_MIN 指的就是可登陆系统的一般账号的最小 UID ，至于 UID_MAX 则是最大 UID 之意。

    要注意的是，系统给予一个账号 UID 时，他是 (1)先参考 UID_MIN 配置值取得最小数值； (2)由 /etc/passwd 搜寻最大的 UID 数值， 将 (1) 与 (2) 相比，找出最大的那个再加一就是新账号的 UID 了。我们上面已经作出 UID 为 700 的 vbird2 ， 如果再使用『 useradd vbird4 』时，你猜 vbird4 的 UID 会是多少？答案是： 701 。 所以中间的 505~699 的号码就空下来啦！

    而如果我是想要创建系统用的账号，所以使用 useradd -r sysaccount 这个 -r 的选项时，就会找『比 500 小的最大的那个 UID + 1 』就是了。 ^_^

- 用户家目录配置值：
    为何我们系统默认会帮用户创建家目录？就是这个『CREATE_HOME = yes』的配置值啦！这个配置值会让你在使用 useradd 时， 主动加入『 -m 』这个产生家目录的选项啊！如果不想要创建用户家目录，就只能强制加上『 -M 』的选项在 useradd 命令运行时啦！至于创建家目录的权限配置呢？就透过 umask 这个配置值啊！因为是 077 的默认配置，因此用户家目录默认权限才会是『 drwx------ 』哩！

- 用户删除与口令配置值：
    使用『USERGROUPS_ENAB yes』这个配置值的功能是： 如果使用 userdel 去删除一个账号时，且该账号所属的初始群组已经没有人隶属于该群组了， 那么就删除掉该群组，举例来说，我们刚刚有创建 vbird4 这个账号，他会主动创建 vbird4 这个群组。 若 vbird4 这个群组并没有其他账号将他加入支持的情况下，若使用 userdel vbird4 时，该群组也会被删除的意思。 至于『MD5_CRYPT_ENAB yes』则表示使用 MD5 来加密口令明文，而不使用旧式的 DES(注2) 。

现在你知道啦，使用 `useradd` 这支程序在创建 Linux 上的账号时，至少会参考：

- /etc/default/useradd
- /etc/login.defs
- /etc/skel/*

这些文件，不过，最重要的其实是创建 `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow` 还有用户家目录就是了～所以，如果你了解整个系统运行的状态，也是可以手动直接修改这几个文件就是了。 OK！账号创建了，接下来处理一下用户的口令吧！


## passwd
刚刚我们讲到了，使用 useradd 创建了账号之后，在默认的情况下，该账号是暂时被封锁的， 也就是说，该账号是无法登陆的，你可以去瞧一瞧 /etc/shadow 内的第二个字段就晓得啰～ 那该如何是好？怕什么？直接给他配置新口令就好了嘛！对吧～配置口令就使用 passwd 啰！

```sh
[root@www ~]# passwd [--stdin]  <==所有人均可使用来改自己的口令
[root@www ~]# passwd [-l] [-u] [--stdin] [-S] \
>  [-n 日数] [-x 日数] [-w 日数] [-i 日期] 账号 <==root 功能
选项与参数：
--stdin ：可以透过来自前一个管线的数据，作为口令输入，对 shell script 有帮助！
-l  ：是 Lock 的意思，会将 /etc/shadow 第二栏最前面加上 ! 使口令失效；
-u  ：与 -l 相对，是 Unlock 的意思！
-S  ：列出口令相关参数，亦即 shadow 文件内的大部分信息。
-n  ：后面接天数，shadow 的第 4 字段，多久不可修改口令天数
-x  ：后面接天数，shadow 的第 5 字段，多久内必须要更动口令
-w  ：后面接天数，shadow 的第 6 字段，口令过期前的警告天数
-i  ：后面接『日期』，shadow 的第 7 字段，口令失效日期
```

范例一：请 root 给予 vbird2 口令

```sh
[root@www ~]# passwd vbird2
Changing password for user vbird2.
New UNIX password: <==这里直接输入新的口令，屏幕不会有任何反应
BAD PASSWORD: it is WAY too short <==口令太简单或过短的错误！
Retype new UNIX password:  <==再输入一次同样的口令
passwd: all authentication tokens updated successfully.  <==竟然还是成功修改了！
```

root 果然是最伟大的人物！当我们要给予用户口令时，透过 root 来配置即可。 root 可以配置各式各样的口令，系统几乎一定会接受！所以您瞧瞧，如同上面的范例一，明明鸟哥输入的口令太短了， 但是系统依旧可接受 vbird2 这样的口令配置。这个是 root 帮忙配置的结果，那如果是用户自己要改口令呢？ 包括 root 也是这样修改的喔！

范例二：用 vbird2 登陆后，修改 vbird2 自己的口令

```sh
[vbird2@www ~]$ passwd   <==后面没有加账号，就是改自己的口令！
Changing password for user vbird2.
Changing password for vbird2
(current) UNIX password: <==这里输入『原有的旧口令』
New UNIX password: <==这里输入新口令
BAD PASSWORD: it is based on a dictionary word <==口令检验不通过，请再想个新口令
New UNIX password: <==这里再想个来输入吧
Retype new UNIX password: <==通过口令验证！所以重复这个口令的输入
passwd: all authentication tokens updated successfully. <==有无成功看关键词
```

passwd 的使用真的要很注意，尤其是 root 先生啊！鸟哥在课堂上每次讲到这里，说是要帮自己的一般账号创建口令时， 有一小部分的学生就是会忘记加上账号，结果就变成改变 root 自己的口令，最后.... root 口令就这样不见去！唉～ 要帮一般账号创建口令需要使用『 passwd 账号 』的格式，使用『 passwd 』表示修改自己的口令！拜托！千万不要改错！

与 root 不同的是，一般账号在更改口令时需要先输入自己的旧口令 (亦即 current 那一行)，然后再输入新口令 (New 那一行)。 要注意的是，口令的规范是非常严格的，尤其新的 distributions 大多使用 PAM 模块来进行口令的检验，包括太短、 口令与账号相同、口令为字典常见字符串等，都会被 PAM 模块检查出来而拒绝修改口令，此时会再重复出现『 New 』这个关键词！ 那时请再想个新口令！若出现『 Retype 』才是你的口令被接受了！重复输入新口令并且看到『 successfully 』这个关键词时才是修改口令成功喔！

> Tips:
> 
> 与一般使用者不同的是， root 并不需要知道旧口令就能够帮用户或 root 自己创建新口令！ 但如此一来有困扰～就是如果你的亲密爱人老是告诉你『我的口令真难记，帮我配置简单一点的！』时， 千万不要妥协啊！这是为了系统安全...

为何用户要设订自己的口令会这么麻烦啊？这是因为口令的安全性啦！如果口令配置太简单， 一些有心人士就能够很简单的猜到你的口令，如此一来人家就可能使用你的一般账号登陆你的主机或使用其他主机资源， 对主机的维护会造成困扰的！所以新的 distributions 是使用较严格的 PAM 模块来管理口令，这个管理的机制写在 /etc/pam.d/passwd 当中。而该文件与口令有关的测试模块就是使用：pam_cracklib.so，这个模块会检验口令相关的信息， 并且取代 /etc/login.defs 内的 PASS_MIN_LEN 的配置啦！关于 PAM 我们在本章后面继续介绍，这里先谈一下， 理论上，你的口令最好符合如下要求：

- 口令不能与账号相同；
- 口令尽量不要选用字典里面会出现的字符串；
- 口令需要超过 8 个字符；
- 口令不要使用个人信息，如身份证、手机号码、其他电话号码等；
- 口令不要使用简单的关系式，如 1+1=2， Iamvbird 等；
- 口令尽量使用大小写字符、数字、特殊字符($,_,-等)的组合。

为了方便系统管理，新版的 passwd 还加入了很多创意选项喔！鸟哥个人认为最好用的大概就是这个『 --stdin 』了！ 举例来说，你想要帮 vbird2 变更口令成为 abc543CC ，可以这样下达命令呢！

**范例三：使用 standard input 创建用户的口令**

```sh
[root@www ~]# echo "abc543CC" | passwd --stdin vbird2
Changing password for user vbird2.
passwd: all authentication tokens updated successfully.
```

这个动作会直接升级用户的口令而不用再次的手动输入！好处是方便处理，缺点是这个口令会保留在命令中， 未来若系统被攻破，人家可以在 /root/.bash_history 找到这个口令呢！所以这个动作通常仅用在 shell script 的大量创建使用者账号当中！要注意的是，这个选项并不存在所有 distributions 版本中， 请使用 man passwd 确认你的 distribution 是否有支持此选项喔！

如果你想要让 vbird2 的口令具有相当的守则，举例来说你要让 vbird2 每 60 天需要变更口令， 口令过期后 10 天未使用就宣告口令失效，那该如何处理？

**范例四：管理 vbird2 的口令使具有 60 天变更、10 天口令失效的配置**

```sh
[root@www ~]# passwd -S vbird2
vbird2 PS 2009-02-26 0 99999 7 -1 (Password set, MD5 crypt.)
# 上面说明口令创建时间 (2009-02-26)、0 最小天数、99999 变更天数、7 警告日数
# 与口令不会失效 (-1) 。

[root@www ~]# passwd -x 60 -i 10 vbird2
[root@www ~]# passwd -S vbird2
vbird2 PS 2009-02-26 0 60 7 10 (Password set, MD5 crypt.)
```

那如果我想要让某个账号暂时无法使用口令登陆主机呢？举例来说， vbird2 这家伙最近老是胡乱在主机乱来， 所以我想要暂时让她无法登陆的话，最简单的方法就是让她的口令变成不合法 (shadow 第 2 字段长度变掉)！ 处理的方法就更简单的！

**范例五：让 vbird2 的账号失效，观察完毕后再让她失效**

```sh
[root@www ~]# passwd -l vbird2
[root@www ~]# passwd -S vbird2
vbird2 LK 2009-02-26 0 60 7 10 (Password locked.)
# 嘿嘿！状态变成『 LK, Lock 』了啦！无法登陆喔！
[root@www ~]# grep vbird2 /etc/shadow
vbird2:!!$1$50MnwNFq$oChX.0TPanCq7ecE4HYEi.:14301:0:60:7:10::
# 其实只是在这里加上 !! 而已！

[root@www ~]# passwd -u vbird2
[root@www ~]# grep vbird2 /etc/shadow
vbird2:$1$50MnwNFq$oChX.0TPanCq7ecE4HYEi.:14301:0:60:7:10::
# 口令字段恢复正常！
```

是否很有趣啊！您可以自行管理一下你的账号的口令相关参数喔！接下来让我们用更简单的方法来查阅口令参数喔！


## chage
除了使用 passwd -S 之外，有没有更详细的口令参数显示功能呢？有的！那就是 chage 了！ 他的用法如下：

```sh
[root@www ~]# chage [-ldEImMW] 账号名
选项与参数：
-l ：列出该账号的详细口令参数；
-d ：后面接日期，修改 shadow 第三字段(最近一次更改口令的日期)，格式 YYYY-MM-DD
-E ：后面接日期，修改 shadow 第八字段(账号失效日)，格式 YYYY-MM-DD
-I ：后面接天数，修改 shadow 第七字段(口令失效日期)
-m ：后面接天数，修改 shadow 第四字段(口令最短保留天数)
-M ：后面接天数，修改 shadow 第五字段(口令多久需要进行变更)
-W ：后面接天数，修改 shadow 第六字段(口令过期前警告日期)
```

**范例一：列出 vbird2 的详细口令参数**

```sh
[root@www ~]# chage -l vbird2
Last password change                               : Feb 26, 2009
Password expires                                   : Apr 27, 2009
Password inactive                                  : May 07, 2009
Account expires                                    : never
Minimum number of days between password change     : 0
Maximum number of days between password change     : 60
Number of days of warning before password expires  : 7
```

我们在 passwd 的介绍中谈到了处理 vbird2 这个账号的口令属性流程，使用 passwd -S 却无法看到很清楚的说明。如果使用 chage 那可就明白多了！如上表所示，我们可以清楚的知道 vbird2 的详细参数呢！ 如果想要修改其他的配置值，就自己参考上面的选项，或者自行 man chage 一下吧！^_^

chage 有一个功能很不错喔！如果你想要让『使用者在第一次登陆时， 强制她们一定要更改口令后才能够使用系统资源』，可以利用如下的方法来处理的！

**范例二：创建一个名为 agetest 的账号，该账号第一次登陆后使用默认口令，但必须要更改过口令后，使用新口令才能够登陆系统使用 bash 环境**

```sh
[root@www ~]# useradd agetest
[root@www ~]# echo "agetest" | passwd --stdin agetest
[root@www ~]# chage -d 0 agetest
# 此时此账号的口令创建时间会被改为 1970/1/1 ，所以会有问题！
```

**范例三：尝试以 agetest 登陆的情况**

```sh
You are required to change your password immediately (root enforced)
WARNING: Your password has expired.
You must change your password now and login again!
Changing password for user agetest.
Changing password for agetest
(current) UNIX password:  <==这个账号被强制要求必须要改口令！
```

非常有趣吧！你会发现 agetest 这个账号在第一次登陆时可以使用与账号同名的口令登陆， 但登陆时就会被要求立刻更改口令，更改口令完成后就会被踢出系统。再次登陆时就能够使用新口令登陆了！ 这个功能对学校老师非常有帮助！因为我们不想要知道学生的口令，那么在初次上课时就使用与学号相同的账号/口令给学生， 让她们登陆时自行配置她们的口令，如此一来就能够避免其他同学随意使用别人的账号，也能够保证学生知道如何更改自己的口令！


## usermod
所谓这『人有失手，马有乱蹄』，您说是吧？所以啰，当然有的时候会『不小心』在 useradd 的时候加入了错误的配置数据。或者是，在使用 useradd 后，发现某些地方还可以进行细部修改。 此时，当然我们可以直接到 /etc/passwd 或 /etc/shadow 去修改相对应字段的数据， 不过，Linux 也有提供相关的命令让大家来进行账号相关数据的微调呢～那就是 usermod 啰～

```sh
[root@www ~]# usermod [-cdegGlsuLU] username
选项与参数：
-c  ：后面接账号的说明，即 /etc/passwd 第五栏的说明栏，可以加入一些账号的说明。
-d  ：后面接账号的家目录，即修改 /etc/passwd 的第六栏；
-e  ：后面接日期，格式是 YYYY-MM-DD 也就是在 /etc/shadow 内的第八个字段数据啦！
-f  ：后面接天数，为 shadow 的第七字段。
-g  ：后面接初始群组，修改 /etc/passwd 的第四个字段，亦即是 GID 的字段！
-G  ：后面接次要群组，修改这个使用者能够支持的群组，修改的是 /etc/group 啰～
-a  ：与 -G 合用，可『添加次要群组的支持』而非『配置』喔！
-l  ：后面接账号名称。亦即是修改账号名称， /etc/passwd 的第一栏！
-s  ：后面接 Shell 的实际文件，例如 /bin/bash 或 /bin/csh 等等。
-u  ：后面接 UID 数字啦！即 /etc/passwd 第三栏的数据；
-L  ：暂时将用户的口令冻结，让他无法登陆。其实仅改 /etc/shadow 的口令栏。
-U  ：将 /etc/shadow 口令栏的 ! 拿掉，解冻啦！
```

如果你仔细的比对，会发现 usermod 的选项与 useradd 非常类似！ 这是因为 usermod 也是用来微调 useradd 添加的使用者参数嘛！不过 usermod 还是有新增的选项， 那就是 -L 与 -U ，不过这两个选项其实与 passwd 的 -l, -u 是相同的！而且也不见得会存在所有的 distribution 当中！接下来，让我们谈谈一些变更参数的实例吧！

**范例一：修改使用者 vbird2 的说明栏，加上『VBird's test』的说明。**

```sh
[root@www ~]# usermod -c "VBird's test" vbird2
[root@www ~]# grep vbird2 /etc/passwd
vbird2:x:700:100:VBird's test:/home/vbird2:/bin/bash
```

**范例二：用户 vbird2 口令在 2009/12/31 失效。**
```sh
[root@www ~]# usermod -e "2009-12-31" vbird2
[root@www ~]# grep vbird2 /etc/shadow
vbird2:$1$50MnwNFq$oChX.0TPanCq7ecE4HYEi.:14301:0:60:7:10:14609:
```

**范例三：我们创建 vbird3 这个系统账号时并没有给予家目录，请创建他的家目录**

```sh
[root@www ~]# ll -d ~vbird3
ls: /home/vbird3: No such file or directory  <==确认一下，确实没有家目录的存在！
[root@www ~]# cp -a /etc/skel /home/vbird3
[root@www ~]# chown -R vbird3:vbird3 /home/vbird3
[root@www ~]# chmod 700 /home/vbird3
[root@www ~]# ll -a ~vbird3
drwx------  4 vbird3 vbird3 4096 Sep  4 18:15 .  <==用户家目录权限
drwxr-xr-x 11 root   root   4096 Feb 26 11:45 ..
-rw-r--r--  1 vbird3 vbird3   33 May 25  2008 .bash_logout
-rw-r--r--  1 vbird3 vbird3  176 May 25  2008 .bash_profile
-rw-r--r--  1 vbird3 vbird3  124 May 25  2008 .bashrc
drwxr-xr-x  3 vbird3 vbird3 4096 Sep  4 18:11 .kde
drwxr-xr-x  4 vbird3 vbird3 4096 Sep  4 18:15 .mozilla
# 使用 chown -R 是为了连同家目录底下的用户/群组属性都一起变更的意思；
# 使用 chmod 没有 -R ，是因为我们仅要修改目录的权限而非内部文件的权限！
```

## userdel
这个功能就太简单了，目的在删除用户的相关数据，而用户的数据有：

- 用户账号/口令相关参数：/etc/passwd, /etc/shadow
- 使用者群组相关参数：/etc/group, /etc/gshadow
- 用户个人文件数据： /home/username, /var/spool/mail/username..

整个命令的语法非常简单：

```sh
[root@www ~]# userdel [-r] username
选项与参数：
-r  ：连同用户的家目录也一起删除
```

**范例一：删除 vbird2 ，连同家目录一起删除**
```sh
[root@www ~]# userdel -r vbird2
```

这个命令下达的时候要小心了！通常我们要移除一个账号的时候，你可以手动的将 /etc/passwd 与 /etc/shadow 里头的该账号取消即可！一般而言，如果该账号只是『暂时不激活』的话，那么将 /etc/shadow 里头账号失效日期 (第八字段) 配置为 0 就可以让该账号无法使用，但是所有跟该账号相关的数据都会留下来！ 使用 userdel 的时机通常是『你真的确定不要让该用户在主机上面使用任何数据了！』

另外，其实用户如果在系统上面操作过一阵子了，那么该用户其实在系统内可能会含有其他文件的。 举例来说，他的邮件信箱 (mailbox) 或者是例行性工作排程 (crontab, 十六章) 之类的文件。 所以，如果想要完整的将某个账号完整的移除，最好可以在下达 userdel -r username 之前， 先以『 find / -user username 』查出整个系统内属于 username 的文件，然后再加以删除吧！

## 图示用户功能
不论是 useradd/usermod/userdel ，那都是系统管理员所能够使用的命令， 如果我是一般身份使用者，那么我是否除了口令之外，就无法更改其他的数据呢？ 当然不是啦！这里我们介绍几个一般身份用户常用的账号数据变更与查询命令啰！


### finger
finger 的中文字面意义是：『手指』或者是『指纹』的意思。这个 finger 可以查阅很多用户相关的信息喔！ 大部分都是在 /etc/passwd 这个文件里面的信息啦！我们就先来检查检查用户信息吧！

```sh
[root@www ~]# finger [-s] username
选项与参数：
-s  ：仅列出用户的账号、全名、终端机代号与登陆时间等等；
-m  ：列出与后面接的账号相同者，而不是利用部分比对 (包括全名部分)
```

**范例一：观察 vbird1 的用户相关账号属性**
```sh
[root@www ~]# finger vbird1
Login: vbird1                           Name: (null)
Directory: /home/vbird1                 Shell: /bin/bash
Never logged in.
No mail.
No Plan.
```

由于 finger 类似指纹的功能，他会将用户的相关属性列出来！如上表所示，其实他列出来的几乎都是 /etc/passwd 文件里面的东西。列出的信息说明如下：

- Login：为使用者账号，亦即 /etc/passwd 内的第一字段；
- Name：为全名，亦即 /etc/passwd 内的第五字段(或称为批注)；
- Directory：就是家目录了；
- Shell：就是使用的 Shell 文件所在；
- Never logged in.：figner 还会调查用户登陆主机的情况喔！
- No mail.：调查 /var/spool/mail 当中的信箱数据；
- No Plan.：调查 ~vbird1/.plan 文件，并将该文件取出来说明！

不过是否能够查阅到 Mail 与 Plan 则与权限有关了！因为 Mail / Plan 都是与使用者自己的权限配置有关， root 当然可以查阅到用户的这些信息，但是 vbird1 就不见得能够查到 vbird3 的信息， 因为 /var/spool/mail/vbird3 与 /home/vbird3/ 的权限分别是 660, 700 ，那 vbird1 当然就无法查阅的到！ 这样解释可以理解吧？此外，我们可以创建自己想要运行的预定计划，当然，最多是给自己看的！可以这样做：

**范例二：利用 vbird1 创建自己的计划档**

```sh
[vbird1@www ~]$ echo "I will study Linux during this year." > ~/.plan
[vbird1@www ~]$ finger vbird1
Login: vbird1                           Name: (null)
Directory: /home/vbird1                 Shell: /bin/bash
Never logged in.
No mail.
Plan:
I will study Linux during this year.
```

**范例三：找出目前在系统上面登陆的用户与登陆时间**

```sh
[vbird1@www ~]$ finger
Login     Name       Tty      Idle  Login Time   Office     Office Phone
root      root       tty1           Feb 26 09:53
vbird1               tty2           Feb 26 15:21
```

在范例三当中，我们发现输出的信息还会有 Office, Office Phone 等信息，那这些信息要如何记录呢？ 底下我们会介绍 chfn 这个命令！来看看如何修改用户的 finger 数据吧！

### chfn

chfn 有点像是： change finger 的意思！这玩意的使用方法如下：

```sh
[root@www ~]# chfn [-foph] [账号名]
选项与参数：
-f  ：后面接完整的大名；
-o  ：您办公室的房间号码；
-p  ：办公室的电话号码；
-h  ：家里的电话号码！
```

**范例一：vbird1 自己更改一下自己的相关信息！**
```sh
[vbird1@www ~]$ chfn
Changing finger information for vbird1.
Password:                        <==确认身份，所以输入自己的口令
Name []: VBird Tsai test         <==输入你想要呈现的全名
Office []: Dic in Ksu. Tainan    <==办公室号码
Office Phone []: 06-2727175#356  <==办公室电话
Home Phone []: 06-1234567        <==家里电话号码

Finger information changed.
[vbird1@www ~]$ grep vbird1 /etc/passwd
vbird1:x:504:505:VBird Tsai test,Dic in Ksu. Tainan,06-2727175#356,06-1234567:
/home/vbird1:/bin/bash
# 其实就是改到第五个字段，该字段里面用多个『 , 』分隔就是了！

[vbird1@www ~]$ finger vbird1
Login: vbird1                           Name: VBird Tsai test
Directory: /home/vbird1                 Shell: /bin/bash
Office: Dic in Ksu. Tainan              Office Phone: 06-2727175#356
Home Phone: 06-1234567
On since Thu Feb 26 15:21 (CST) on tty2
No mail.
Plan:
I will study Linux during this year.
# 就是上面特殊字体呈现的那些地方是由 chfn 所修改出来的！
```

这个命令说实在的，除非是你的主机有很多的用户，否则倒真是用不着这个程序！这就有点像是 bbs 里头更改你『个人属性』的那一个数据啦！不过还是可以自己玩一玩！尤其是用来提醒自己相关数据啦！ ^_^


## chsh
这就是 change shell 的简写！使用方法就更简单了！

```sh
[vbird1@www ~]$ chsh [-ls]
选项与参数：
-l  ：列出目前系统上面可用的 shell ，其实就是 /etc/shells 的内容！
-s  ：配置修改自己的 Shell 啰
```

**范例一：用 vbird1 的身份列出系统上所有合法的 shell，并且指定 csh 为自己的 shell**

```sh
[vbird1@www ~]$ chsh -l
/bin/sh
/bin/bash
/sbin/nologin  <==所谓：合法不可登陆的 Shell 就是这玩意！
/bin/tcsh
/bin/csh       <==这就是 C shell 啦！
/bin/ksh
# 其实上面的信息就是我们在 bash 中谈到的 /etc/shells 啦！

[vbird1@www ~]$ chsh -s /bin/csh; grep vbird1 /etc/passwd
Changing shell for vbird1.
Password:  <==确认身份，请输入 vbird1 的口令
Shell changed.
vbird1:x:504:505:VBird Tsai test,Dic in Ksu. Tainan,06-2727175#356,06-1234567:
/home/vbird1:/bin/csh

[vbird1@www ~]$ chsh -s /bin/bash
# 测试完毕后，立刻改回来！

[vbird1@www ~]$ ll $(which chsh)
-rws--x--x 1 root root 19128 May 25  2008 /usr/bin/chsh
```

不论是 chfn 与 chsh ，都是能够让一般用户修改 /etc/passwd 这个系统文件的！所以你猜猜，这两个文件的权限是什么？ 一定是 SUID 的功能啦！看到这里，想到前面！ 这就是 Linux 的学习方法～ ^_^


### id
id 这个命令则可以查询某人或自己的相关 UID/GID 等等的信息，他的参数也不少，不过， 都不需要记～反正使用 id 就全部都列出啰～ ^_^

```sh
[root@www ~]# id [username]
```

**范例一：查阅 root 自己的相关 ID 信息！**
```sh
[root@www ~]# id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),
10(wheel) context=root:system_r:unconfined_t:SystemLow-SystemHigh
# 上面信息其实是同一行的数据！包括会显示 UID/GID 以及支持的所有群组！
# 至于后面那个 context=... 则是 SELinux 的内容，先不要理会他！
```

**范例二：查阅一下 vbird1 吧～**
```sh
[root@www ~]# id vbird1
uid=504(vbird1) gid=505(vbird1) groups=505(vbird1) context=root:system_r:
unconfined_t:SystemLow-SystemHigh

[root@www ~]# id vbird100
id: vbird100: No such user  <== id 这个命令也可以用来判断系统上面有无某账号！
```

## 新增与移除群组
OK！了解了账号的新增、删除、更动与查询后，再来我们可以聊一聊群组的相关内容了。 基本上，群组的内容都与这两个文件有关：/etc/group, /etc/gshadow。 群组的内容其实很简单，都是上面两个文件的新增、修改与移除而已， 不过，如果再加上有效群组的概念，那么 newgrp 与 gpasswd 则不可不知呢！


### groupadd
[root@www ~]# groupadd [-g gid] [-r] 组名
选项与参数：
-g  ：后面接某个特定的 GID ，用来直接给予某个 GID ～
-r  ：创建系统群组啦！与 /etc/login.defs 内的 GID_MIN 有关。

范例一：新建一个群组，名称为 group1
[root@www ~]# groupadd group1
[root@www ~]# grep group1 /etc/group /etc/gshadow
/etc/group:group1:x:702:
/etc/gshadow:group1:!::
# 群组的 GID 也是会由 500 以上最大 GID+1 来决定！
曾经有某些版本的教育训练手册谈到，为了让使用者的 UID/GID 成对，她们建议新建的与使用者私有群组无关的其他群组时，使用小于 500 以下的 GID 为宜。 也就是说，如果要创建群组的话，最好能够使用『 groupadd -r 群组名』的方式来创建啦！ 不过，这见仁见智啦！看你自己的抉择啰！


### groupmod
跟 usermod 类似的，这个命令仅是在进行 group 相关参数的修改而已。

[root@www ~]# groupmod [-g gid] [-n group_name] 群组名
选项与参数：
-g  ：修改既有的 GID 数字；
-n  ：修改既有的组名

范例一：将刚刚上个命令创建的 group1 名称改为 mygroup ， GID 为 201
[root@www ~]# groupmod -g 201 -n mygroup group1
[root@www ~]# grep mygroup /etc/group /etc/gshadow
/etc/group:mygroup:x:201:
/etc/gshadow:mygroup:!::
不过，还是那句老话，不要随意的更动 GID ，容易造成系统资源的错乱喔！


### groupdel
呼呼！ groupdel 自然就是在删除群组的啰～用法很简单：

[root@www ~]# groupdel [groupname]

范例一：将刚刚的 mygroup 删除！
[root@www ~]# groupdel mygroup

范例二：若要删除 vbird1 这个群组的话？
[root@www ~]# groupdel vbird1
groupdel: cannot remove user's primary group.
为什么 mygroup 可以删除，但是 vbird1 就不能删除呢？原因很简单，『有某个账号 (/etc/passwd) 的 initial group 使用该群组！』 如果查阅一下，你会发现在 /etc/passwd 内的 vbird1 第四栏的 GID 就是 /etc/group 内的 vbird1 那个群组的 GID ，所以啰，当然无法删除～否则 vbird1 这个用户登陆系统后， 就会找不到 GID ，那可是会造成很大的困扰的！那么如果硬要删除 vbird1 这个群组呢？ 你『必须要确认 /etc/passwd 内的账号没有任何人使用该群组作为 initial group 』才行喔！所以，你可以：

修改 vbird1 的 GID ，或者是：
删除 vbird1 这个使用者。

### gpasswd：群组管理员功能
如果系统管理员太忙碌了，导致某些账号想要加入某个项目时找不到人帮忙！这个时候可以创建『群组管理员』喔！ 什么是群组管理员呢？就是让某个群组具有一个管理员，这个群组管理员可以管理哪些账号可以加入/移出该群组！ 那要如何『创建一个群组管理员』呢？就得要透过 gpasswd 啰！

```sh
# 关于系统管理员(root)做的动作：
[root@www ~]# gpasswd groupname
[root@www ~]# gpasswd [-A user1,...] [-M user3,...] groupname
[root@www ~]# gpasswd [-rR] groupname
选项与参数：
    ：若没有任何参数时，表示给予 groupname 一个口令(/etc/gshadow)
-A  ：将 groupname 的主控权交由后面的使用者管理(该群组的管理员)
-M  ：将某些账号加入这个群组当中！
-r  ：将 groupname 的口令移除
-R  ：让 groupname 的口令栏失效

# 关于群组管理员(Group administrator)做的动作：
[someone@www ~]$ gpasswd [-ad] user groupname
选项与参数：
-a  ：将某位使用者加入到 groupname 这个群组当中！
-d  ：将某位使用者移除出 groupname 这个群组当中。

范例一：创建一个新群组，名称为 testgroup 且群组交由 vbird1 管理：
[root@www ~]# groupadd testgroup  <==先创建群组
[root@www ~]# gpasswd testgroup   <==给这个群组一个口令吧！
Changing the password for group testgroup
New Password:
Re-enter new password:
# 输入两次口令就对了！
[root@www ~]# gpasswd -A vbird1 testgroup  <==加入群组管理员为 vbird1
[root@www ~]# grep testgroup /etc/group /etc/gshadow
/etc/group:testgroup:x:702:
/etc/gshadow:testgroup:$1$I5ukIY1.$o5fmW.cOsc8.K.FHAFLWg0:vbird1:
# 很有趣吧！此时 vbird1 则拥有 testgroup 的主控权喔！身份有点像板主啦！

范例二：以 vbird1 登陆系统，并且让他加入 vbird1, vbird3 成为 testgroup 成员
[vbird1@www ~]$ id
uid=504(vbird1) gid=505(vbird1) groups=505(vbird1) ....
# 看得出来，vbird1 尚未加入 testgroup 群组喔！

[vbird1@www ~]$ gpasswd -a vbird1 testgroup
[vbird1@www ~]$ gpasswd -a vbird3 testgroup
[vbird1@www ~]$ grep testgroup /etc/group
testgroup:x:702:vbird1,vbird3
```

很有趣的一个小实验吧！我们可以让 testgroup 成为一个可以公开的群组，然后创建起群组管理员， 群组管理员可以有多个。在这个案例中，我将 vbird1 配置为 testgroup 的群组管理员，所以 vbird1 就可以自行添加群组成员啰～呼呼！然后，该群组成员就能够使用 newgrp 啰～

## 账号管理实例
账号管理不是随意建置几个账号就算了！有时候我们需要考虑到一部主机上面可能有多个账号在协同工作！ 举例来说，在大学任教时，我们学校的专题生是需要分组的，这些同一组的同学间必须要能够互相修改对方的数据文件， 但是同时这些同学又需要保留自己的私密数据，因此直接公开家目录是不适宜的。那该如何是好？ 为此，我们底下提供几个例子来让大家思考看看啰：

**任务一：单纯的完成上头交代的任务，假设我们需要的账号数据如下，你该如何实作？**

|账号名称|	账号全名|支持次要群组|是否可登陆主机|口令|
|:------|:--------|:----------|:------------|:----|
|myuser1	1st| user|	mygroup1|	可以|	password|
|myuser2	2nd| user|	mygroup1|	可以|	password|
|myuser3	3rd| user|	无额外支持|	不可以|	password|

处理的方法如下所示：

```sh
# 先处理账号相关属性的数据：
[root@www ~]# groupadd mygroup1
[root@www ~]# useradd -G mygroup1 -c "1st user" myuser1
[root@www ~]# useradd -G mygroup1 -c "2nd user" myuser2
[root@www ~]# useradd -c "3rd user" -s /sbin/nologin myuser3

# 再处理账号的口令相关属性的数据：
[root@www ~]# echo "password" | passwd --stdin myuser1
[root@www ~]# echo "password" | passwd --stdin myuser2
[root@www ~]# echo "password" | passwd --stdin myuser3
```

要注意的地方主要有：myuser1 与 myuser2 都有支持次要群组，但该群组不见得会存在，因此需要先手动创建他！ 然后 myuser3 是『不可登陆系统』的账号，因此需要使用 /sbin/nologin 这个 shell 来给予，这样该账号就无法登陆啰！ 这样是否理解啊！接下来再来讨论比较难一些的环境！如果是专题环境该如何制作？

**任务二：我的使用者 pro1, pro2, pro3 是同一个项目计划的开发人员，我想要让这三个用户在同一个目录底下工作， 但这三个用户还是拥有自己的家目录与基本的私有群组。假设我要让这个项目计划在 /srv/projecta 目录下开发， 可以如何进行？**

```sh
# 1. 假设这三个账号都尚未创建，可先创建一个名为 projecta 的群组，
#    再让这三个用户加入其次要群组的支持即可：
[root@www ~]# groupadd projecta
[root@www ~]# useradd -G projecta -c "projecta user" pro1
[root@www ~]# useradd -G projecta -c "projecta user" pro2
[root@www ~]# useradd -G projecta -c "projecta user" pro3
[root@www ~]# echo "password" | passwd --stdin pro1
[root@www ~]# echo "password" | passwd --stdin pro2
[root@www ~]# echo "password" | passwd --stdin pro3

# 2. 开始创建此项目的开发目录：
[root@www ~]# mkdir /srv/projecta
[root@www ~]# chgrp projecta /srv/projecta
[root@www ~]# chmod 2770 /srv/projecta
[root@www ~]# ll -d /srv/projecta
drwxrws--- 2 root projecta 4096 Feb 27 11:29 /srv/projecta
```

由于此项目计划只能够给 pro1, pro2, pro3 三个人使用，所以 /srv/projecta 的权限配置一定要正确才行！ 所以该目录群组一定是 projecta ，但是权限怎么会是 2770 呢还记得第七章谈到的 SGID 吧？为了让三个使用者能够互相修改对方的文件， 这个 SGID 是必须要存在的喔！如果连这里都能够理解，嘿嘿！您的账号管理已经有一定程度的概念啰！ ^_^

但接下来有个困扰的问题发生了！假如任务一的 myuser1 是 projecta 这个项目的助理，他需要这个项目的内容， 但是他『不可以修改』项目目录内的任何数据！那该如何是好？你或许可以这样做：

- 将 myuser1 加入 projecta 这个群组的支持，但是这样会让 myuser1 具有完整的 /srv/projecta 的权限， myuser1 是可以删除该目录下的任何数据的！这样是有问题的；
- 将 /srv/projecta 的权限改为 2775 ，让 myuser1 可以进入查阅数据。但此时会发生所有其他人均可进入该目录查阅的困扰！ 这也不是我们要的环境。

真要命！传统的 Linux 权限无法针对某个个人配置专属的权限吗？其实是可以啦！接下来我们就来谈谈这个功能吧！

# 主机的细部权限规划：ACL 的使用
从第六章开始，我们就一直强调 Linux 的权限概念是非常重要的！ 但是传统的权限仅有三种身份 (owner, group, others) 搭配三种权限 (r,w,x) 而已，并没有办法单纯的针对某一个使用者或某一个群组来配置特定的权限需求，例如前一小节最后的那个任务！ 此时就得要使用 ACL 这个机制啦！这玩意挺有趣的，底下我们就来谈一谈：


## 什么是 ACL
ACL 是 Access Control List 的缩写，主要的目的是在提供传统的 owner,group,others 的 read,write,execute 权限之外的细部权限配置。ACL 可以针对单一使用者，单一文件或目录来进行 r,w,x 的权限规范，对于需要特殊权限的使用状况非常有帮助。

那 ACL 主要可以针对哪些方面来控制权限呢？他主要可以针对几个项目：

- 使用者 (user)：可以针对使用者来配置权限；
- 群组 (group)：针对群组为对象来配置其权限；
- 默认属性 (mask)：还可以针对在该目录下在创建新文件/目录时，规范新数据的默认权限；

好了，再来看看如何让你的文件系统可以支持 ACL 吧！

## 如何启动 ACL
由于 ACL 是传统的 Unix-like 操作系统权限的额外支持项目，因此要使用 ACL 必须要有文件系统的支持才行。目前绝大部分的文件系统都有支持 ACL 的功能，包括 ReiserFS, EXT2/EXT3, JFS, XFS 等等。在我们的 CentOS 5.x 当中，默认使用 Ext3 是启动 ACL 支持的！至于察看你的文件系统是否支持 ACL 可以这样看：

```sh
[root@www ~]# mount  <==直接查阅挂载参数的功能
/dev/hda2 on / type ext3 (rw)
/dev/hda3 on /home type ext3 (rw)
# 其他项目鸟哥都将他省略了！假设我们只要看这两个装置。但没有看到 acl 喔！

[root@www ~]# dumpe2fs -h /dev/hda2  <==由 superblock 内容去查询
....(前面省略)....
Default mount options:    user_xattr acl
....(后面省略)....
```

由 mount 单纯去查阅不见得可以看到实际的项目，由于目前新的 distributions 常常会主动加入某些默认功能， 如上表所示，其实 CentOS 5.x 在默认的情况下 (Default mount options:) 就帮你加入 acl 的支持了！ 那如果你的系统默认不会帮你加上 acl 的支持呢？那你可以这样做：

```sh
[root@www ~]# mount -o remount,acl /
[root@www ~]# mount
/dev/hda2 on / type ext3 (rw,acl)
# 这样就加入了！但是如果想要每次启动都生效，那就这样做：

[root@www ~]# vi /etc/fstab
LABEL=/1   /   ext3    defaults,acl    1 1
```

如果你不确定或者是不会使用 dumpe2fs 观察你的文件系统，那么建议直接将上述的 /etc/fstab 里面的内容修改一下即可！

## ACL 的配置技巧： getfacl, setfacl
好了，让你的 filesystem 启动 ACL 支持后，接下来该如何配置与观察 ACL 呢？ 很简单，利用这两个命令就可以了：

- getfacl：取得某个文件/目录的 ACL 配置项目；
- setfacl：配置某个目录/文件的 ACL 规范。

先让我们来瞧一瞧 setfacl 如何使用吧！


### setfacl 命令用法
```sh
[root@www ~]# setfacl [-bkRd] [{-m|-x} acl参数] 目标文件名
选项与参数：
-m ：配置后续的 acl 参数给文件使用，不可与 -x 合用；
-x ：删除后续的 acl 参数，不可与 -m 合用；
-b ：移除所有的 ACL 配置参数；
-k ：移除默认的 ACL 参数，关于所谓的『默认』参数于后续范例中介绍；
-R ：递归配置 acl ，亦即包括次目录都会被配置起来；
-d ：配置『默认 acl 参数』的意思！只对目录有效，在该目录新建的数据会引用此默认值
```
上面谈到的是 acl 的选项功能，那么如何配置 ACL 的特殊权限呢？特殊权限的配置方法有很多， 我们先来谈谈最常见的，就是针对单一使用者的配置方式：

```sh
# 1. 针对特定使用者的方式：
# 配置规范：『 u:[使用者账号列表]:[rwx] 』，例如针对 vbird1 的权限规范 rx ：
[root@www ~]# touch acl_test1
[root@www ~]# ll acl_test1
-rw-r--r-- 1 root root 0 Feb 27 13:28 acl_test1
[root@www ~]# setfacl -m u:vbird1:rx acl_test1
[root@www ~]# ll acl_test1
-rw-r-xr--+ 1 root root 0 Feb 27 13:28 acl_test1
# 权限部分多了个 + ，且与原本的权限 (644) 看起来差异很大！但要如何查阅呢？

[root@www ~]# setfacl -m u::rwx acl_test1
[root@www ~]# ll acl_test1
-rwxr-xr--+ 1 root root 0 Feb 27 13:28 acl_test1
# 无使用者列表，代表配置该文件拥有者，所以上面显示 root 的权限成为 rwx 了！
```
上述动作为最简单的 ACL 配置，利用『 u:使用者:权限 』的方式来配置的啦！配置前请加上 -m 这个选项。 如果一个文件配置了 ACL 参数后，他的权限部分就会多出一个 + 号了！但是此时你看到的权限与实际权限可能就会有点误差！ 那要如何观察呢？就透过 getfacl 吧！


### getfacl 命令用法
```sh
[root@www ~]# getfacl filename
选项与参数：
getfacl 的选项几乎与 setfacl 相同！所以鸟哥这里就免去了选项的说明啊！

# 请列出刚刚我们配置的 acl_test1 的权限内容：
[root@www ~]# getfacl acl_test1
# file: acl_test1   <==说明档名而已！
# owner: root       <==说明此文件的拥有者，亦即 ll 看到的第三使用者字段
# group: root       <==此文件的所属群组，亦即 ll 看到的第四群组字段
user::rwx           <==使用者列表栏是空的，代表文件拥有者的权限
user:vbird1:r-x     <==针对 vbird1 的权限配置为 rx ，与拥有者并不同！
group::r--          <==针对文件群组的权限配置仅有 r 
mask::r-x           <==此文件默认的有效权限 (mask)
other::r--          <==其他人拥有的权限啰！
上面的数据非常容易查阅吧？显示的数据前面加上 # 的，代表这个文件的默认属性，包括文件名、文件拥有者与文件所属群组。 底下出现的 user, group, mask, other 则是属于不同使用者、群组与有效权限(mask)的配置值。 以上面的结果来看，我们刚刚配置的 vbird1 对于这个文件具有 r 与 x 的权限啦！这样看的懂吗？ 如果看的懂的话，接下来让我们在测试其他类型的 setfacl 配置吧！

# 2. 针对特定群组的方式：
# 配置规范：『 g:[群组列表]:[rwx] 』，例如针对 mygroup1 的权限规范 rx ：
[root@www ~]# setfacl -m g:mygroup1:rx acl_test1
[root@www ~]# getfacl acl_test1
# file: acl_test1
# owner: root
# group: root
user::rwx
user:vbird1:r-x
group::r--
group:mygroup1:r-x  <==这里就是新增的部分！多了这个群组的权限配置！
mask::r-x
other::r--
```

基本上，群组与使用者的配置并没有什么太大的差异啦！如上表所示，非常容易了解意义。不过，你应该会觉得奇怪的是， 那个 mask 是什么东西啊？其实他有点像是『有效权限』的意思！他的意义是： 使用者或群组所配置的权限必须要存在于 mask 的权限配置范围内才会生效，此即『有效权限 (effective permission)』 我们举个例子来看，如下所示：

```sh
# 3. 针对有效权限 mask 的配置方式：
# 配置规范：『 m:[rwx] 』，例如针对刚刚的文件规范为仅有 r ：
[root@www ~]# setfacl -m m:r acl_test1
[root@www ~]# getfacl acl_test1
# file: acl_test1
# owner: root
# group: root
user::rwx
user:vbird1:r-x        #effective:r-- <==vbird1+mask均存在者，仅有 r 而已！
group::r--
group:mygroup1:r-x     #effective:r--
mask::r--
other::r--
```

您瞧，vbird1 与 mask 的集合发现仅有 r 存在，因此 vbird1 仅具有 r 的权限而已，并不存在 x 权限！这就是 mask 的功能了！我们可以透过使用 mask 来规范最大允许的权限，就能够避免不小心开放某些权限给其他使用者或群组了。 不过，通常鸟哥都是将 mask 配置为 rwx 啦！然后再分别依据不同的使用者/群组去规范她们的权限就是了。

###  例题
将前一小节任务二中 /srv/projecta 这个目录，让 myuser1 可以进入查阅，但 myuser1 不具有修改的权力。
答：
由于 myuser1 是独立的使用者与群组，而 /srv 是附属于 / 之下的，因此 /srv 已经具有 acl 的功能。 透过如下的配置即可搞定：

```sh
# 1. 先测试看看，使用 myuser1 能否进入该目录？
[myuser1@www ~]$ cd /srv/projecta
-bash: cd: /srv/projecta: Permission denied  <==确实不可进入！

# 2. 开始用 root 的身份来配置一下该目录的权限吧！
[root@www ~]# setfacl -m u:myuser1:rx /srv/projecta
[root@www ~]# getfacl /srv/projecta
# file: srv/projecta
# owner: root
# group: projecta
user::rwx
user:myuser1:r-x  <==还是要看看有没有配置成功喔！
group::rwx
mask::rwx
other::---

# 3. 还是得要使用 myuser1 去测试看看结果！
[myuser1@www ~]$ cd /srv/projecta
[myuser1@www projecta]$ ll -a
drwxrws---+ 2 root projecta 4096 Feb 27 11:29 .  <==确实可以查询档名
drwxr-xr-x  4 root root     4096 Feb 27 11:29 ..

[myuser1@www projecta]$ touch testing
touch: cannot touch `testing': Permission denied <==确实不可以写入！
```

请注意，上述的 1, 3 步骤使用 myuser1 的身份，2步骤才是使用 root 去配置的！

上面的配置我们就完成了之前任务二的后续需求喔！这么简单呢！接下来让我们来测试一下，如果我用 root 或者是 pro1 的身份去 /srv/projecta 添加文件或目录时，该文件或目录是否能够具有 ACL 的配置？ 意思就是说，ACL 的权限配置是否能够被次目录所『继承？』先试看看：

```sh
[root@www ~]# cd /srv/projecta
[root@www ~]# touch abc1
[root@www ~]# mkdir abc2
[root@www ~]# ll -d abc*
-rw-r--r-- 1 root projecta    0 Feb 27 14:37 abc1
drwxr-sr-x 2 root projecta 4096 Feb 27 14:37 abc2
```

你可以明显的发现，权限后面都没有 + ，代表这个 acl 属性并没有继承喔！如果你想要让 acl 在目录底下的数据都有继承的功能，那就得如下这样做了！

```sh
# 4. 针对默认权限的配置方式：
# 配置规范：『 d:[ug]:使用者列表:[rwx] 』

# 让 myuser1 在 /srv/projecta 底下一直具有 rx 的默认权限！
[root@www ~]# setfacl -m d:u:myuser1:rx /srv/projecta
[root@www ~]# getfacl /srv/projecta
# file: srv/projecta
# owner: root
# group: projecta
user::rwx
user:myuser1:r-x
group::rwx
mask::rwx
other::---
default:user::rwx
default:user:myuser1:r-x
default:group::rwx
default:mask::rwx
default:other::---

[root@www ~]# cd /srv/projecta
[root@www projecta]# touch zzz1
[root@www projecta]# mkdir zzz2
[root@www projecta]# ll -d zzz*
-rw-rw----+ 1 root projecta    0 Feb 27 14:57 zzz1
drwxrws---+ 2 root projecta 4096 Feb 27 14:57 zzz2
# 看吧！确实有继承喔！然后我们使用 getfacl 再次确认看看！

[root@www projecta]# getfacl zzz2
# file: zzz2
# owner: root
# group: projecta
user::rwx
user:myuser1:r-x
group::rwx
mask::rwx
other::---
default:user::rwx
default:user:myuser1:r-x
default:group::rwx
default:mask::rwx
default:other::---
```

透过这个『针对目录来配置的默认 ACL 权限配置值』的项目，我们可以让这些属性继承到次目录底下呢！ 非常方便啊！那如果想要让 ACL 的属性全部消失又要如何处理？透过『 setfacl -b 檔名 』即可啦！ 太简单了！鸟哥就不另外介绍了！请自行测试测试吧！

# 使用者身份切换
什么？在 Linux 系统当中还要作身份的变换？这是为啥？可能有底下几个原因啦！

- 使用一般账号：系统平日操作的好习惯
    事实上，为了安全的缘故，一些老人家都会建议你，尽量以一般身份使用者来操作 Linux 的日常作业！等到需要配置系统环境时， 才变换身份成为 root 来进行系统管理，相对比较安全啦！避免作错一些严重的命令，例如恐怖的『 rm -rf / 』(千万作不得！)

- 用较低权限启动系统服务
    相对于系统安全，有的时候，我们必须要以某些系统账号来进行程序的运行。 举例来说， Linux 主机上面的一套软件，名称为 apache ，我们可以额外创建一个名为 apache 的用户来启动 apache 软件啊，如此一来，如果这个程序被攻破，至少系统还不至于就损毁了～

- 软件本身的限制
    在远古时代的 telnet 程序中，该程序默认是不许使用 root 的身份登陆的，telnet 会判断登陆者的 UID， 若 UID 为 0 的话，那就直接拒绝登陆了。所以，你只能使用一般使用者来登陆 Linux 服务器。 此外， ssh (注3) 也可以配置拒绝 root 登陆喔！那如果你有系统配置需求该如何是好啊？就变换身份啊！

由于上述考虑，所以我们都是使用一般账号登陆系统的，等有需要进行系统维护或软件升级时才转为 root 的身份来动作。 那如何让一般使用者转变身份成为 root 呢？主要有两种方式喔：

- 以『 su - 』直接将身份变成 root 即可，但是这个命令却需要 root 的口令，也就是说，如果你要以 su 变成 root 的话，你的一般使用者就必须要有 root 的口令才行；

- 以『 sudo 命令 』运行 root 的命令串，由于 sudo 需要事先配置妥当，且 sudo 需要输入用户自己的口令， 因此多人共管同一部主机时， sudo 要比 su 来的好喔！至少 root 口令不会流出去！

底下我们就来说一说 `su` 跟 `sudo` 的用法啦！

## su

`su` 是最简单的身份切换命令了，他可以进行任何身份的切换唷！方法如下：

```sh
[root@www ~]# su [-lm] [-c 命令] [username]
选项与参数：
-   ：单纯使用 - 如『 su - 』代表使用 login-shell 的变量文件读取方式来登陆系统；
      若使用者名称没有加上去，则代表切换为 root 的身份。
-l  ：与 - 类似，但后面需要加欲切换的使用者账号！也是 login-shell 的方式。
-m  ：-m 与 -p 是一样的，表示『使用目前的环境配置，而不读取新使用者的配置文件』
-c  ：仅进行一次命令，所以 -c 后面可以加上命令喔！
```

上表的解释当中有出现之前[第十一章]()谈过的 login-shell 配置文件读取方式，如果你忘记那是啥东西， 请先回去第十一章瞧瞧再回来吧！这个 `su` 的用法当中，有没有加上那个减号『 - 』差很多喔！ 因为涉及 `login-shell` 与 `non-login shell` 的变量读取方法。这里让我们以一个小例子来说明吧！

范例一：假设你原本是 vbird1 的身份，想要使用 non-login shell 的方式变成 root

```sh
[vbird1@www ~]$ su       <==注意提示字符，是 vbird1 的身份喔！
Password:                <==这里输入 root 的口令喔！
[root@www vbird1]# id    <==提示字符的目录是 vbird1 喔！
uid=0(root) gid=0(root) groups=0(root),1(bin),...   <==确实是 root 的身份！
[root@www vbird1]# env | grep 'vbird1'
USER=vbird1
PATH=/usr/local/bin:/bin:/usr/bin:/home/vbird1/bin  <==这个影响最大！
MAIL=/var/spool/mail/vbird1                         <==收到的 mailbox 是 vbird1
PWD=/home/vbird1                                    <==并非 root 的家目录
LOGNAME=vbird1
# 虽然你的 UID 已经是具有 root 的身份，但是看到上面的输出信息吗？
# 还是有一堆变量为原本 vbird1 的身份，所以很多数据还是无法直接利用。
[root@www vbird1]# exit   <==这样可以离开 su 的环境！
```

单纯使用『 `su` 』切换成为 `root` 的身份，读取的变量配置方式为 `non-login shell` 的方式，这种方式很多原本的变量不会被改变， 尤其是我们之前谈过很多次的 `PATH` 这个变量，由于没有改变成为 `root` 的环境 (一堆 `/sbin`, `/usr/sbin` 等目录都没有被包含进来)， 因此很多 `root` 惯用的命令就只能使用绝对路径来运行咯。其他的还有 MAIL 这个变量，你输入 mail 时， 收到的邮件竟然还是 `vbird1` 的，而不是 `root` 本身的邮件！是否觉得很奇怪啊！所以切换身份时，请务必使用如下的范例二：

范例二：使用 login shell 的方式切换为 root 的身份并观察变量

```sh
[vbird1@www ~]$ su -
Password:   <==这里输入 root 的口令喔！
[root@www ~]# env | grep root
USER=root
MAIL=/var/spool/mail/root
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin
PWD=/root
HOME=/root
LOGNAME=root
# 了解差异了吧？下次变换成为 root 时，记得最好使用 su - 喔！
[root@www ~]# exit   <==这样可以离开 su 的环境！
```

上述的作法是让使用者的身份变成 `root` 并开始操作系统，如果想要离开 `root` 的身份则得要利用 `exit` 离开才行。 那我如果只是想要运行『一个只有 `root` 才能进行的命令，且运行完毕就恢复原本的身份』呢？那就可以加上 `-c` 这个选项啰！ 请参考底下范例三！


```sh
范例三：vbird1 想要运行『 head -n 3 /etc/shadow 』一次，且已知 root 口令
[vbird1@www ~]$ head -n 3 /etc/shadow
head: cannot open `/etc/shadow' for reading: Permission denied
[vbird1@www ~]$ su - -c "head -n 3 /etc/shadow"
Password: <==这里输入 root 的口令喔！
root:$1$/30QpEWEBEZXRD0bh6rAABCEQD.BAH0:14126:0:99999:7:::
bin:*:14126:0:99999:7:::
daemon:*:14126:0:99999:7:::
[vbird1@www ~]$ <==注意看，身份还是 vbird1 喔！继续使用旧的身份进行系统操作！
```

由于 `/etc/shadow` 权限的关系，该文件仅有 `root` 可以查阅。为了查阅该文件，所以我们必须要使用 `root` 的身份工作。 但我只想要进行一次该命令而已，此时就使用类似上面的语法吧！好，那接下来，如果我是 `root` 或者是其他人， 想要变更成为某些特殊账号，可以使用如下的方法来切换喔！

```sh
范例四：原本是 vbird1 这个使用者，想要变换身份成为 dmtsai 时？
[vbird1@www ~]$ su -l dmtsai
Password: <==这里输入 dmtsai 的口令喔！
[dmtsai@www ~]$ su -
Password: <==这里输入 root 的口令喔！
[root@www ~]# id sshd
uid=74(sshd) gid=74(sshd) groups=74(sshd) ... <==确实有存在此人
[root@www ~]# su -l sshd
This account is currently not available.      <==竟然说此人无法切换？
[root@www ~]# finger sshd
Login: sshd                             Name: Privilege-separated SSH
Directory: /var/empty/sshd              Shell: /sbin/nologin
[root@www ~]# exit    <==离开第二次的 su 
[dmtsai@www ~]$ exit  <==离开第一次的 su 
[vbird1@www ~]$ exit  <==这才是最初的环境！ 
```

`su` 就这样简单的介绍完毕，总结一下他的用法是这样的：

若要完整的切换到新使用者的环境，必须要使用『 `su - username` 』或『 `su -l username` 』， 才会连同 `PATH/USER/MAIL` 等变量都转成新用户的环境；

如果仅想要运行一次 `root` 的命令，可以利用『 `su - -c "命令串"` 』的方式来处理；

使用 `root` 切换成为任何使用者时，并不需要输入新用户的口令；

虽然使用 `su` 很方便啦，不过缺点是，当我的主机是多人共管的环境时，如果大家都要使用 `su` 来切换成为 `root` 的身份，那么不就每个人都得要知道 `root` 的口令，这样口令太多人知道可能会流出去， 很不妥当呢！怎办？透过 `sudo` 来处理即可！

## sudo
相对于 su 需要了解新切换的用户口令 (常常是需要 root 的口令)， sudo 的运行则仅需要自己的口令即可！ 甚至可以配置不需要口令即可运行 sudo 呢！由于 sudo 可以让你以其他用户的身份运行命令 (通常是使用 root 的身份来运行命令)，因此并非所有人都能够运行 sudo ， 而是仅有规范到 /etc/sudoers 内的用户才能够运行 sudo 这个命令喔！说的这么神奇，底下就来瞧瞧那 sudo 如何使用？

### sudo 的命令用法

由于一开始系统默认仅有 root 可以运行 sudo ，因此底下的范例我们先以 root 的身份来运行，等到谈到 visudo 时，再以一般使用者来讨论其他 sudo 的用法吧！ sudo 的语法如下：

```sh
[root@www ~]# sudo [-b] [-u 新使用者账号]
选项与参数：
-b  ：将后续的命令放到背景中让系统自行运行，而不与目前的 shell 产生影响
-u  ：后面可以接欲切换的使用者，若无此项则代表切换身份为 root 。
```

```sh
范例一：你想要以 sshd 的身份在 /tmp 底下创建一个名为 mysshd 的文件
[root@www ~]# sudo -u sshd touch /tmp/mysshd
[root@www ~]# ll /tmp/mysshd
-rw-r--r-- 1 sshd sshd 0 Feb 28 17:42 /tmp/mysshd
# 特别留意，这个文件的权限是由 sshd 所创建的情况喔！
```

```sh
范例二：你想要以 vbird1 的身份创建 ~vbird1/www 并于其中创建 index.html 文件
[root@www ~]# sudo -u vbird1 sh -c "mkdir ~vbird1/www; cd ~vbird1/www; \
>  echo 'This is index.html file' > index.html"
[root@www ~]# ll -a ~vbird1/www
drwxr-xr-x 2 vbird1 vbird1 4096 Feb 28 17:51 .
drwx------ 5 vbird1 vbird1 4096 Feb 28 17:51 ..
-rw-r--r-- 1 vbird1 vbird1   24 Feb 28 17:51 index.html
# 要注意，创建者的身份是 vbird1 ，且我们使用 sh -c "一串命令" 来运行的！
```

`sudo` 可以让你切换身份来进行某项任务，例如上面的两个范例。范例一中，我们的 root 使用 sshd 的权限去进行某项任务！ 要注意，因为我们无法使用『 `su - sshd` 』去切换系统账号 (因为系统账号的 shell 是 `/sbin/nologin`)， 这个时候 `sudo` 真是他 `X` 的好用了！立刻以 sshd 的权限在 `/tmp` 底下创建文件！查阅一下文件权限你就了解意义啦！ 至于范例二则更使用多重命令串 (透过分号 `;` 来延续命令进行)，使用 `sh -c` 的方法来运行一连串的命令， 如此真是好方便！

但是 `sudo` 默认仅有 `root` 能使用啊！为什么呢？因为 `sudo` 的运行是这样的流程：

当用户运行 sudo 时，系统于 /etc/sudoers 文件中搜寻该使用者是否有运行 sudo 的权限；
若使用者具有可运行 sudo 的权限后，便让使用者『输入用户自己的口令』来确认；

若口令输入成功，便开始进行 sudo 后续接的命令(但 root 运行 sudo 时，不需要输入口令)；
若欲切换的身份与运行者身份相同，那也不需要输入口令。

所以说，sudo 运行的重点是：『能否使用 sudo 必须要看 /etc/sudoers 的配置值， 而可使用 sudo 者是透过输入用户自己的口令来运行后续的命令串』喔！由于能否使用与 /etc/sudoers 有关， 所以我们当然要去编辑 sudoers 文件啦！不过，因为该文件的内容是有一定的规范的，因此直接使用 vi 去编辑是不好的。 此时，我们得要透过 visudo 去修改这个文件喔！


### visudo 与 /etc/sudoers

从上面的说明我们可以知道，除了 root 之外的其他账号，若想要使用 sudo 运行属于 root 的权限命令，则 root 需要先使用 visudo 去修改 /etc/sudoers ，让该账号能够使用全部或部分的 root 命令功能。为什么要使用 visudo 呢？这是因为 /etc/sudoers 是有配置语法的，如果配置错误那会造成无法使用 sudo 命令的不良后果。因此才会使用 visudo 去修改， 并在结束离开修改画面时，系统会去检验 /etc/sudoers 的语法就是了。

一般来说，visudo 的配置方式有几种简单的方法喔，底下我们以几个简单的例子来分别说明：

- 单一用户可进行 root 所有命令，与 sudoers 文件语法：

    假如我们要让 vbird1 这个账号可以使用 root 的任何命令，那么可以简单的这样进行修改即可：

    ```sh
    [root@www ~]# visudo
    ....(前面省略)....
    root    ALL=(ALL)       ALL  <==找到这一行，大约在 76 行左右
    vbird1  ALL=(ALL)       ALL  <==这一行是你要新增的！
    ....(前面省略)....
    ```

    有趣吧！其实 visudo 只是利用 vi 将 /etc/sudoers 文件呼叫出来进行修改而已，所以这个文件就是 /etc/sudoers 啦！ 这个文件的配置其实很简单，如上面所示，如果你找到 76 行 (有 root 配置的那行) 左右，看到的数据就是：

    ```
    使用者账号  登陆者的来源主机名=(可切换的身份)  可下达的命令
    root                         ALL=(ALL)           ALL   <==这是默认值
    ```

    上面这一行的四个组件意义是：

    - 系统的哪个账号可以使用 sudo 这个命令的意思，默认为 root 这个账号；
    - 当这个账号由哪部主机联机到本 Linux 主机，意思是这个账号可能是由哪一部网络主机联机过来的， 这个配置值可以指定客户端计算机(信任用户的意思)。默认值 root 可来自任何一部网络主机
    - 这个账号可以切换成什么身份来下达后续的命令，默认 root 可以切换成任何人；
    - 可用该身份下达什么命令？这个命令请务必使用绝对路径撰写。 默认 root 可以切换任何身份且进行任何命令之意。

    那个 ALL 是特殊的关键词，代表任何身份、主机或命令的意思。所以，我想让 vbird1 可以进行任何身份的任何命令， 就如同上表特殊字体写的那样，其实就是复制上述默认值那一行，再将 root 改成 vbird1 即可啊！ 此时『vbird1 不论来自哪部主机登陆，他可以变换身份成为任何人，且可以进行系统上面的任何命令』之意。 修改完请储存后离开 vi，并以 vbird1 登陆系统后，进行如下的测试看看：

    ```sh
    [vbird1@www ~]$ tail -n 1 /etc/shadow  <==注意！身份是 vbird1
    tail: cannot open `/etc/shadow' for reading: Permission denied
    # 因为不是 root 嘛！所以当然不能查询 /etc/shadow

    [vbird1@www ~]$ sudo tail -n 1 /etc/shadow <==透过 sudo

    We trust you have received the usual lecture from the local System
    Administrator. It usually boils down to these three things:

        #1) Respect the privacy of others.  <==这里仅是一些说明与警示项目
        #2) Think before you type.
        #3) With great power comes great responsibility.

    Password: <==注意啊！这里输入的是『 vbird1 自己的口令 』
    pro3:$1$GfinyJgZ$9J8IdrBXXMwZIauANg7tW0:14302:0:99999:7::: 
    # 看！vbird1 竟然可以查询 shadow ！
    ```

    注意到了吧！vbird1 输入自己的口令就能够运行 root 的命令！所以，系统管理员当然要了解 vbird1 这个用户的『操守』才行！否则随便配置一个用户，他恶搞系统怎办？另外，一个一个配置太麻烦了， 能不能使用群组的方式来配置呢？参考底下的方式吧。

- 利用群组以及免口令的功能处理 `visudo`

    我们在本章前面曾经创建过 pro1, pro2, pro3 ，这三个用户能否透过群组的功能让这三个人可以管理系统？ 可以的，而且很简单！同样我们使用实际案例来说明：

    ```sh
    [root@www ~]# visudo  <==同样的，请使用 root 先配置
    ....(前面省略)....
    %wheel     ALL=(ALL)    ALL <==大约在 84 行左右，请将这行的 # 拿掉！
    # 在最左边加上 % ，代表后面接的是一个『群组』之意！改完请储存后离开

    [root@www ~]# usermod -a -G wheel pro1 <==将 pro1 加入 wheel 的支持
    ```

    上面的配置值会造成『任何加入 wheel 这个群组的使用者，就能够使用 sudo 切换任何身份来操作任何命令』的意思。 你当然可以将 wheel 换成你自己想要的群组名。接下来，请分别切换身份成为 pro1 及 pro2 试看看 sudo 的运行。

    ```sh
    [pro1@www ~]$ sudo tail -n 1 /etc/shadow <==注意身份是 pro1
    ....(前面省略)....
    Password:  <==输入 pro1 的口令喔！
    pro3:$1$GfinyJgZ$9J8IdrBXXMwZIauANg7tW0:14302:0:99999:7:::

    [pro2@www ~]$ sudo tail -n 1 /etc/shadow <==注意身份是 pro2
    Password:
    pro2 is not in the sudoers file.  This incident will be reported.
    # 仔细看错误信息他是说这个 pro2 不在 /etc/sudoers 的配置中！
    ```

    这样理解群组了吧？如果你想要让 pro3 也支持这个 sudo 的话，不需要重新使用 visudo ，只要利用 usermod 去修改 pro3 的群组支持，让 wheel 也支持 pro3 的话，那他就能够进行 sudo 啰！ 简单吧！不过，既然我们都信任这些 sudo 的用户了，能否提供『不需要口令即可使用 sudo 』呢？ 就透过如下的方式：

    ```sh
    [root@www ~]# visudo  <==同样的，请使用 root 先配置
    ....(前面省略)....
    %wheel     ALL=(ALL)   NOPASSWD: ALL <==大约在 87 行左右，请将 # 拿掉！
    # 在最左边加上 % ，代表后面接的是一个『群组』之意！改完请储存后离开
    ```

    重点是那个 NOPASSWD 啦！该关键词是免除口令输入的意思喔！

- 有限制的命令操作：

    上面两点都会让使用者能够利用 root 的身份进行任何事情！这样总是不太好～如果我想要让用户仅能够进行部分系统任务， 比方说，系统上面的 myuser1 仅能够帮 root 修改其他用户的口令时，亦即『当使用者仅能使用 passwd 这个命令帮忙 root 修改其他用户的口令』时，你该如何撰写呢？可以这样做：

    ```sh
    [root@www ~]# visudo  <==注意是 root 身份
    myuser1	ALL=(root)  /usr/bin/passwd  <==最后命令务必用绝对路径
    ```

    上面的配置值指的是『myuser1 可以切换成为 root 使用 passwd 这个命令』的意思。其中要注意的是： 命令字段必须要填写绝对路径才行！否则 visudo 会出现语法错误的状况发生！ 此外，上面的配置是有问题的！我们使用底下的命令操作来让您了解：
    
    ```sh
    [myuser1@www ~]$ sudo passwd myuser3  <==注意，身份是 myuser1
    Password:  <==输入 myuser1 的口令
    Changing password for user myuser3. <==底下改的是 myuser3 的口令喔！这样是正确的
    New UNIX password:
    Retype new UNIX password:
    passwd: all authentication tokens updated successfully.

    [myuser1@www ~]$ sudo passwd
    Changing password for user root.  <==见鬼！怎么会去改 root 的口令？
    ```
    
    恐怖啊！我们竟然让 root 的口令被 myuser3 给改变了！下次 root 回来竟无法登陆系统...欲哭无泪～怎办？ 所以我们必须要限制用户的命令参数！修改的方法为将上述的那行改一改先：
    
    ```sh
    [root@www ~]# visudo  <==注意是 root 身份
    myuser1	ALL=(root)  !/usr/bin/passwd, /usr/bin/passwd [A-Za-z]*, \
                        !/usr/bin/passwd root
    ```
    
    由于屏幕一行写不完，我将这行写成两行，所以上面第一行最后加上反斜杠啰。加上惊叹号『 ! 』代表『不可运行』的意思。 因此上面这一行会变成：可以运行『 passwd 任意字符』，但是『 passwd 』与『 passwd root 』这两个命令例外！ 如此一来 myuser1 就无法改变 root 的口令了！这样这位使用者可以具有 root 的能力帮助你修改其他用户的口令， 而且也不能随意改变 root 的口令！很有用处的！

- 透过别名建置 visudo：

    如上述第三点，如果我有 15 个用户需要加入刚刚的管理员行列，那么我是否要将上述那长长的配置写入 15 行啊？ 而且如果想要修改命令或者是新增命令时，那我每行都需要重新配置，很麻烦ㄟ！有没有更简单的方式？ 是有的！透过别名即可！我们 visudo 的别名可以是『命令别名、帐户别名、主机别名』等。不过这里我们仅介绍帐户别名， 其他的配置值有兴趣的话，可以自行玩玩！

    假设我的 pro1, pro2, pro3 与 myuser1, myuser2 要加入上述的口令管理员的 sudo 列表中， 那我可以创立一个帐户别名称为 ADMPW 的名称，然后将这个名称处理一下即可。处理的方式如下：

    ```sh
    [root@www ~]# visudo  <==注意是 root 身份
    User_Alias ADMPW = pro1, pro2, pro3, myuser1, myuser2
    Cmnd_Alias ADMPWCOM = !/usr/bin/passwd, /usr/bin/passwd [A-Za-z]*, \
                        !/usr/bin/passwd root
    ADMPW   ALL=(root)  ADMPWCOM
    ```

    我透过 User_Alias 创建出一个新账号，这个账号名称一定要使用大写字符来处理，包括 Cmnd_Alias(命令别名)、Host_Alias(来源主机名别名) 都需要使用大写字符的！这个 ADMPW 代表后面接的那些实际账号。 而该账号能够进行的命令就如同 ADMPWCOM 后面所指定的那样！上表最后一行则写入这两个别名 (账号与命令别名)， 未来要修改时，我只要修改 User_Alias 以及 Cmnd_Alias 这两行即可！配置方面会比较简单有弹性喔！

- sudo 的时间间隔问题：

    或许您已经发现了，那就是，如果我使用同一个账号在短时间内重复操作 sudo 来运行命令的话， 在第二次运行 sudo 时，并不需要输入自己的口令！sudo 还是会正确的运行喔！为什么呢？ 第一次运行 sudo 需要输入口令，是担心由于用户暂时离开座位，但有人跑来你的座位使用你的账号操作系统之故。 所以需要你输入一次口令重新确认一次身份。

    两次运行 sudo 的间隔在五分钟内，那么再次运行 sudo 时就不需要再次输入口令了， 这是因为系统相信你在五分钟内不会离开你的作业，所以运行 sudo 的是同一个人！呼呼！真是很人性化的设计啊～ ^_^。不过如果两次 sudo 操作的间隔超过 5 分钟，那就得要重新输入一次你的口令了 (注4)

    另外要注意的是，因为使用一般账号时，理论上不会使用到 /sbin, /usr/sbin 等目录内的命令，所以 $PATH 变量不会含有这些目录，因此很多管理命令需要使用绝对路径来下达比较妥当喔！

- `sudo` 搭配 `su` 的使用方式：

    很多时候我们需要大量运行很多 root 的工作，所以一直使用 `sudo` 觉得很烦ㄟ！那有没有办法使用 `sudo` 搭配 `su` ， 一口气将身份转为 `root` ，而且还用用户自己的口令来变成 `root` 呢？是有的！而且方法简单的会让你想笑！ 我们创建一个 `ADMINS` 帐户别名，然后这样做：
    
    ```sh
    [root@www ~]# visudo
    User_Alias  ADMINS = pro1, pro2, pro3, myuser1
    ADMINS ALL=(root)  /bin/su -
    ```

    接下来，上述的 `pro1`, `pro2`, `pro3`, `myuser1` 这四个人，只要输入『 `sudo su -` 』并且输入『`自己的口令`』后， 立刻变成 `root` 的身份！不但 `root` 口令不会外流，用户的管理也变的非常方便！ 这也是实务上面多人共管一部主机时常常使用的技巧呢！这样管理确实方便，不过还是要强调一下大前提， 那就是『这些你加入的使用者，全部都是你能够信任的用户』！

# 使用者的特殊 shell 与 PAM 模块

我们前面一直谈到的大多是一般身份用户与系统管理员 (`root`) 的相关操作， 而且大多是讨论关于可登陆系统的账号来说。那么换个角度想，如果我今天想要创建的， 是一个『仅能使用 mail server 相关邮件服务的账号，而该账号并不能登陆 Linux 主机』呢？如果不能给予该账号一个口令，那么该账号就无法使用系统的各项资源，当然也包括 mail 的资源， 而如果给予一个口令，那么该账号就可能可以登陆 Linux 主机啊！呵呵～伤脑筋吧～ 所以，底下让我们来谈一谈这些有趣的话题啰！

另外，在本章之前谈到过 /etc/login.defs 文件中，关于口令长度应该默认是 5 个字符串长度，但是我们上面也谈到，该配置值已经被 PAM 模块所取代了，那么 PAM 是什么？为什么他可以影响我们使用者的登陆呢？这里也要来谈谈的！


## 特殊的 shell, /sbin/nologin
在本章一开头的 passwd 文件结构里面我们就谈过系统账号这玩意儿，这玩意儿的 shell 就是使用 `/sbin/nologin` ，重点在于系统账号是不需要登陆的！所以我们就给他这个无法登陆的合法 shell。 使用了这个 shell 的用户即使有了口令，你想要登陆时他也无法登陆，因为会出现如下的信息喔：

```sh
This account is currently not available.
```

我们所谓的『无法登陆』指的仅是：『这个使用者无法使用 bash 或其他 shell 来登陆系统』而已， 并不是说这个账号就无法使用其他的系统资源喔！ 举例来说，各个系统账号，打印作业由 lp 这个账号在管理， WWW 服务由 apache 这个账号在管理， 他们都可以进行系统程序的工作，但是『就是无法登陆主机』而已啦！^_^

换个角度来想，如果我的 Linux 主机提供的是邮件服务，所以说，在这部 Linux 主机上面的账号， 其实大部分都是用来收受主机的信件而已，并不需要登陆主机的呢！ 这个时候，我们就可以考虑让单纯使用 mail 的账号以 `/sbin/nologin` 做为他们的 shell ， 这样，最起码当我的主机被尝试想要登陆系统以取得 shell 环境时，可以拒绝该账号呢！

另外，如果我想要让某个具有 `/sbin/nologin` 的使用者知道，他们不能登陆主机时， 其实我可以创建『 `/etc/nologin.txt` 』这个文件， 并且在这个文件内说明不能登陆的原因，那么下次当这个用户想要登陆系统时， 屏幕上出现的就会是 `/etc/nologin.txt` 这个文件的内容，而不是默认的内容了！

**例题：**

当使用者尝试利用纯 mail 账号 (例如 myuser3) 时，利用 `/etc/nologin.txt` 告知用户不要利用该账号登陆系统。

**答：**

直接以 vi 编辑该文件，内容可以是这样：
```sh
[root@www ~]# vi /etc/nologin.txt
This account is system account or mail account.
Please DO NOT use this account to login my Linux server.
想要测试时，可以使用 myuser3 (此账号的 shell 是 /sbin/nologin) 来测试看看！
[root@www ~]# su - myuser3
This account is system account or mail account.
Please DO NOT use this account to login my Linux server.
[root@www ~]#
```

结果会发现与原本的默认信息不一样喔！ ^_^

## PAM 模块简介

在过去，我们想要对一个使用者进行认证 (authentication)，得要要求用户输入账号口令， 然后透过自行撰写的程序来判断该账号口令是否正确。也因为如此，我们常常得使用不同的机制来判断账号口令， 所以搞的一部主机上面拥有多个各别的认证系统，也造成账号口令可能不同步的验证问题！ 为了解决这个问题因此有了 PAM (Pluggable Authentication Modules, 嵌入式模块) 的机制！

PAM 可以说是一套应用程序编程接口 (Application Programming Interface, API)，他提供了一连串的验证机制，只要使用者将验证阶段的需求告知 PAM 后， PAM 就能够回报使用者验证的结果 (成功或失败)。由于 PAM 仅是一套验证的机制，又可以提供给其他程序所呼叫引用，因此不论你使用什么程序，都可以使用 PAM 来进行验证，如此一来，就能够让账号口令或者是其他方式的验证具有一致的结果！也让程序设计师方便处理验证的问题喔！ (注5)

![图 5.2.1、 PAM 模块与其他程序的相关性](http://cn.linux.vbird.org/linux_basic/0410accountmanager_files/pam-1.gif)

如上述的图示， PAM 是一个独立的 API 存在，只要任何程序有需求时，可以向 PAM 发出验证要求的通知， PAM 经过一连串的验证后，将验证的结果回报给该程序，然后该程序就能够利用验证的结果来进行可登陆或显示其他无法使用的信息。 这也就是说，你可以在写程序的时候将 PAM 模块的功能加入，就能够利用 PAM 的验证功能啰。 因此目前很多程序都会利用 PAM 喔！所以我们才要来学习他啊！

PAM 用来进行验证的数据称为模块 (Modules)，每个 PAM 模块的功能都不太相同。举例来说， 还记得我们在本章使用 passwd 命令时，如果随便输入字典上面找的到的字符串， passwd 就会回报错误信息了！这是为什么呢？这就是 PAM 的 pam_cracklib.so 模块的功能！他能够判断该口令是否在字典里面！ 并回报给口令修改程序，此时就能够了解你的口令强度了。

所以，当你有任何需要判断是否在字典当中的口令字符串时，就可以使用 pam_cracklib.so 这个模块来验证！ 并根据验证的回报结果来撰写你的程序呢！这样说，可以理解 PAM 的功能了吧？没错！ PAM 的模块也是很重要的一环！

## PAM 模块配置语法
PAM 藉由一个与程序相同文件名的配置文件来进行一连串的认证分析需求。我们同样以 passwd 这个命令的呼叫 PAM 来说明好了。 当你运行 passwd 后，这支程序呼叫 PAM 的流程是：

1. 用户开始运行 /usr/bin/passwd 这支程序，并输入口令；
2. passwd 呼叫 PAM 模块进行验证；
3. PAM 模块会到 /etc/pam.d/ 找寻与程序 (passwd) 同名的配置文件；
4. 依据 /etc/pam.d/passwd 内的配置，引用相关的 PAM 模块逐步进行验证分析；
5. 将验证结果 (成功、失败以及其他信息) 回传给 passwd 这支程序；
6. passwd 这支程序会根据 PAM 回传的结果决定下一个动作 (重新输入新口令或者通过验证！)

从上头的说明，我们会知道重点其实是 /etc/pam.d/ 里面的配置文件，以及配置文件所呼叫的 PAM 模块进行的验证工作！ 既然一直谈到 passwd 这个口令修改命令，那我们就来看看 /etc/pam.d/passwd 这个配置文件的内容是怎样吧！

```sh
[root@www ~]# cat /etc/pam.d/passwd
#%PAM-1.0  <==PAM版本的说明而已！
auth       include      system-auth <==每一行都是一个验证的过程
account    include      system-auth
password   include      system-auth
验证类别   控制标准     PAM 模块与该模块的参数
```

在这个配置文件当中，除了第一行宣告 PAM 版本之外，其他任何『 # 』开头的都是批注，而每一行都是一个独立的验证流程， 每一行可以区分为三个字段，分别是验证类别(type)、控制标准(flag)、PAM的模块与该模块的参数。 底下我们先来谈谈验证类别与控制标准这两项数据吧！

> Tips:
> 
> 你会发现在我们上面的表格当中出现的是『 include (包括) 』这个关键词，他代表的是『请呼叫后面的文件来作为这个类别的验证』， 所以，上述的每一行都要重复呼叫 /etc/pam.d/system-auth 那个文件来进行验证的意思！

### 第一个字段：验证类别 (Type)
验证类别主要分为四种，分别说明如下：

- auth 是 authentication (认证) 的缩写，所以这种类别主要用来检验使用者的身份验证，这种类别通常是需要口令来检验的， 所以后续接的模块是用来检验用户的身份。

- account (账号) 则大部分是在进行 authorization (授权)，这种类别则主要在检验使用者是否具有正确的权限， 举例来说，当你使用一个过期的口令来登陆时，当然就无法正确的登陆了。

- session 是会议期间的意思，所以 session 管理的就是使用者在这次登陆 (或使用这个命令) 期间，PAM 所给予的环境配置。 这个类别通常用在记录用户登陆与注销时的信息！例如，如果你常常使用 su 或者是 sudo 命令的话， 那么应该可以在 /var/log/secure 里面发现很多关于 pam 的说明，而且记载的数据是『session open, session close』的信息！

- password 就是口令嘛！所以这种类别主要在提供验证的修订工作，举例来说，就是修改/变更口令啦！

这四个验证的类型通常是有顺序的，不过也有例外就是了。 会有顺序的原因是，(1)我们总是得要先验证身份 (auth) 后， (2)系统才能够藉由用户的身份给予适当的授权与权限配置 (account)，而且(3)登陆与注销期间的环境才需要配置， 也才需要记录登陆与注销的信息 (session)。如果在运行期间需要口令修订时，(4)才给予 password 的类别。这样说起来， 自然是需要有点顺序吧！

### 第二个字段：验证的控制旗标 (control flag)

那么『验证的控制旗标(control flag)』又是什么？简单的说，他就是『验证通过的标准』啦！ 这个字段在管控该验证的放行方式，主要也分为四种控制方式：

- required 此验证若成功则带有 success (成功) 的标志，若失败则带有 failure 的标志，但不论成功或失败都会继续后续的验证流程。 由于后续的验证流程可以继续进行，因此相当有利于数据的登录 (log) ，这也是 PAM 最常使用 required 的原因。

- requisite 若验证失败则立刻回报原程序 failure 的标志，并终止后续的验证流程。若验证成功则带有 success 的标志并继续后续的验证流程。 这个项目与 required 最大的差异，就在于失败的时候还要不要继续验证下去？由于 requisite 是失败就终止， 因此失败时所产生的 PAM 信息就无法透过后续的模块来记录了。

- sufficient 若验证成功则立刻回传 success 给原程序，并终止后续的验证流程；若验证失败则带有 failure 标志并继续后续的验证流程。 这玩意儿与 requisits 刚好相反！

- optional 这个模块控件目大多是在显示信息而已，并不是用在验证方面的。
如果将这些控制旗标以图示的方式配合成功与否的条件绘图，会有点像底下这样：

![图 5.3.1、 PAM 控制旗标所造成的回报流程](http://cn.linux.vbird.org/linux_basic/0410accountmanager_files/pam-2.gif)

程序运行过程中遇到验证时才会去呼叫 PAM ，而 PAM 验证又分很多类型与控制，不同的控制旗标所回报的信息并不相同。 如上图所示， requisite 失败就回报了并不会继续，而 sufficient 则是成功就回报了也不会继续。 至于验证结束后所回报的信息通常是『succes 或 failure 』而已，后续的流程还需要该程序的判断来继续运行才行。

## 常用模块简介
谈完了配置文件的语法后，现在让我们来查阅一下 CentOS 5.x 提供的 PAM 默认文件的内容是啥吧！ 由于我们常常需要透过各种方式登陆 (login) 系统，因此就来看看登陆所需要的 PAM 流程为何：

```sh
[root@www ~]# cat /etc/pam.d/login
#%PAM-1.0
auth [user_unknown=ignore success=ok ignore=ignore default=bad] pam_securetty.so
auth       include      system-auth
account    required     pam_nologin.so
account    include      system-auth
password   include      system-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    include      system-auth
session    required     pam_loginuid.so
session    optional     pam_console.so
# pam_selinux.so open should only be followed by sessions...
session    required     pam_selinux.so open
session    optional     pam_keyinit.so force revoke
# 我们可以看到，其实 login 也呼叫多次的 system-auth ，所以底下列出该配置文件

[root@www ~]# cat /etc/pam.d/system-auth
#%PAM-1.0
# This file is auto-generated.
# User changes will be destroyed the next time authconfig is run.
auth     required     pam_env.so
auth     sufficient   pam_unix.so nullok try_first_pass
auth     requisite    pam_succeed_if.so uid >= 500 quiet
auth     required     pam_deny.so

account  required     pam_unix.so
account  sufficient   pam_succeed_if.so uid < 500 quiet
account  required     pam_permit.so

password requisite    pam_cracklib.so try_first_pass retry=3
password sufficient   pam_unix.so md5 shadow nullok try_first_pass use_authtok
password required     pam_deny.so

session  optional     pam_keyinit.so revoke
session  required     pam_limits.so
session  [success=1 default=ignore] pam_succeed_if.so service in crond quiet \
                      use_uid
session  required     pam_unix.so
```

上面这个表格当中使用到非常多的 PAM 模块，每个模块的功能都不太相同，详细的模块情报可以在你的系统中找到：

- /etc/pam.d/*：每个程序个别的 PAM 配置文件；
- /lib/security/*：PAM 模块文件的实际放置目录；
- /etc/security/*：其他 PAM 环境的配置文件；
- /usr/share/doc/pam-*/：详细的 PAM 说明文件。

例如鸟哥使用未 update 过的 CentOS 5.2 ，pam_nologin 说明文件档在： /usr/share/doc/pam-0.99.6.2/txts/README.pam_nologin。你可以自行查阅一下该模块的功能。 鸟哥这里仅简单介绍几个较常使用的模块，详细的信息还得要您努力查阅参考书呢！ ^_^

pam_securetty.so：
限制系统管理员 (root) 只能够从安全的 (secure) 终端机登陆；那什么是终端机？例如 tty1, tty2 等就是传统的终端机装置名称。那么安全的终端机配置呢？ 就写在 /etc/securetty 这个文件中。你可以查阅一下该文件， 就知道为什么 root 可以从 tty1~tty7 登陆，但却无法透过 telnet 登陆 Linux 主机了！

pam_nologin.so：
这个模块可以限制一般用户是否能够登陆主机之用。当 /etc/nologin 这个文件存在时，则所有一般使用者均无法再登陆系统了！若 /etc/nologin 存在，则一般使用者在登陆时， 在他们的终端机上会将该文件的内容显示出来！所以，正常的情况下，这个文件应该是不能存在系统中的。 但这个模块对 root 以及已经登陆系统中的一般账号并没有影响。

pam_selinux.so：
SELinux 是个针对程序来进行细部管理权限的功能，SELinux 这玩意儿我们会在第十七章的时候再来详细谈论。由于 SELinux 会影响到用户运行程序的权限，因此我们利用 PAM 模块，将 SELinux 暂时关闭，等到验证通过后， 再予以启动！

pam_console.so：
当系统出现某些问题，或者是某些时刻你需要使用特殊的终端接口 (例如 RS232 之类的终端联机设备) 登陆主机时， 这个模块可以帮助处理一些文件权限的问题，让使用者可以透过特殊终端接口 (console) 顺利的登陆系统。

pam_loginuid.so：
我们知道系统账号与一般账号的 UID 是不同的！一般账号 UID 均大于 500 才合理。 因此，为了验证使用者的 UID 真的是我们所需要的数值，可以使用这个模块来进行规范！

pam_env.so：
用来配置环境变量的一个模块，如果你有需要额外的环境变量配置，可以参考 /etc/security/pam_env.conf 这个文件的详细说明。

pam_unix.so：
这是个很复杂且重要的模块，这个模块可以用在验证阶段的认证功能，可以用在授权阶段的账号许可证管理， 可以用在会议阶段的登录文件记录等，甚至也可以用在口令升级阶段的检验！非常丰富的功能！ 这个模块在早期使用得相当频繁喔！

pam_cracklib.so：
可以用来检验口令的强度！包括口令是否在字典中，口令输入几次都失败就断掉此次联机等功能，都是这模块提供的！ 这玩意儿很重要！

pam_limits.so：
还记得我们在十一章谈到的 ulimit 吗？ 其实那就是这个模块提供的能力！还有更多细部的配置可以参考： /etc/security/limits.conf 内的说明。
了解了这些模块的大致功能后，言归正传，讨论一下 login 的 PAM 验证机制流程是这样的：

验证阶段 (auth)：首先，(a)会先经过 pam_securetty.so 判断，如果使用者是 root 时，则会参考 /etc/securetty 的配置； 接下来(b)经过 pam_env.so 配置额外的环境变量；再(c)透过 pam_unix.so 检验口令，若通过则回报 login 程序；若不通过则(d)继续往下以 pam_succeed_if.so 判断 UID 是否大于 500 ，若小于 500则回报失败，否则再往下 (e)以 pam_deny.so 拒绝联机。

授权阶段 (account)：(a)先以 pam_nologin.so 判断 /etc/nologin 是否存在，若存在则不许一般使用者登陆； (b)接下来以 pam_unix 进行账号管理，再以 (c) pam_succeed_if.so 判断 UID 是否小于 500 ，若小于 500 则不记录登录信息。(d)最后以 pam_permit.so 允许该账号登陆。

口令阶段 (password)：(a)先以 pam_cracklib.so 配置口令仅能尝试错误 3 次；(b)接下来以 pam_unix.so 透过 md5, shadow 等功能进行口令检验，若通过则回报 login 程序，若不通过则 (c)以 pam_deny.so 拒绝登陆。

会议阶段 (session)：(a)先以 pam_selinux.so 暂时关闭 SELinux；(b)使用 pam_limits.so 配置好用户能够操作的系统资源； (c)登陆成功后开始记录相关信息在登录文件中； (d)以 pam_loginuid.so 规范不同的 UID 权限；(e)开启 pam_selinux.so 的功能。
总之，就是依据验证类别 (type) 来看，然后先由 login 的配置值去查阅，如果出现『 include system-auth 』 就转到 system-auth 文件中的相同类别，去取得额外的验证流程就是了。然后再到下一个验证类别，最终将所有的验证跑完！ 就结束这次的 PAM 验证啦！

经过这样的验证流程，现在你知道为啥 /etc/nologin 存在会有问题，也会知道为何你使用一些远程联机机制时， 老是无法使用 root 登陆的问题了吧？没错！这都是 PAM 模块提供的功能啦！

例题：
为什么 root 无法以 telnet 直接登陆系统，但是却能够使用 ssh 直接登陆？
答：
一般来说， telnet 会引用 login 的 PAM 模块，而 login 的验证阶段会有 /etc/securetty 的限制！ 由于远程联机属于 pts/n (n 为数字) 的动态终端机接口装置名称，并没有写入到 /etc/securetty ， 因此 root 无法以 telnet 登陆远程主机。至于 ssh 使用的是 /etc/pam.d/sshd 这个模块， 你可以查阅一下该模块，由于该模块的验证阶段并没有加入 pam_securetty ，因此就没有 /etc/securetty 的限制！故可以从远程直接联机到服务器端。

另外，关于 telnet 与 ssh 的细部说明，请参考鸟哥的 Linux 私房菜服务器篇

## 其他相关文件
除了前一小节谈到的 /etc/securetty 会影响到 root 可登陆的安全终端机， /etc/nologin 会影响到一般使用者是否能够登陆的功能之外，我们也知道 PAM 相关的配置文件在 /etc/pam.d ， 说明文件在 /usr/share/doc/pam-(版本) ，模块实际在 /lib/security/ 。那么还有没有相关的 PAM 文件呢？ 是有的，主要都在 /etc/security 这个目录内！我们底下介绍几个可能会用到的配置文件喔！


limits.conf
我们在第十一章谈到的 ulimit 功能中， 除了修改使用者的 ~/.bashrc 配置文件之外，其实系统管理员可以统一藉由 PAM 来管理的！ 那就是 /etc/security/limits.conf 这个文件的配置了。这个文件的配置很简单，你可以自行参考一下该文件内容。 我们这里仅作个简单的介绍：

范例一：vbird1 这个用户只能创建 100MB 的文件，且大于 90MB 会警告
[root@www ~]# vi /etc/security/limits.conf
vbird1	soft		fsize		 90000
vbird1	hard		fsize		100000
#账号   限制依据	限制项目 	限制值
# 第一字段为账号，或者是群组！若为群组则前面需要加上 @ ，例如 @projecta
# 第二字段为限制的依据，是严格(hard)，还是仅为警告(soft)；
# 第三字段为相关限制，此例中限制文件容量，
# 第四字段为限制的值，在此例中单位为 KB。
# 若以 vbird1 登陆后，进行如下的操作则会有相关的限制出现！

[vbird1@www ~]$ ulimit -a
....(前面省略)....
file size               (blocks, -f) 90000
....(后面省略)....

[vbird1@www ~]$ dd if=/dev/zero of=test bs=1M count=110
File size limit exceeded
[vbird1@www ~]$ ll -k test
-rw-rw-r-- 1 vbird1 vbird1 90000 Mar  4 11:30 test
# 果然有限制到了

范例二：限制 pro1 这个群组，每次仅能有一个用户登陆系统 (maxlogins)
[root@www ~]# vi /etc/security/limits.conf
@pro1   hard   maxlogins   1
# 如果要使用群组功能的话，这个功能似乎对初始群组才有效喔！
# 而如果你尝试多个 pro1 的登陆时，第二个以后就无法登陆了。
# 而且在 /var/log/secure 文件中还会出现如下的信息：
# pam_limits(login:session): Too many logins (max 1) for pro1
这个文件挺有趣的，而且是配置完成就生效了，你不用重新启动任何服务的！ 但是 PAM 有个特殊的地方，由于他是在程序呼叫时才予以配置的，因此你修改完成的数据， 对于已登陆系统中的用户是没有效果的，要等他再次登陆时才会生效喔！另外， 上述的配置请在测试完成后立刻批注掉，否则下次这两个使用者登陆就会发生些许问题啦！ ^_^


/var/log/secure, /var/log/messages
如果发生任何无法登陆或者是产生一些你无法预期的错误时，由于 PAM 模块都会将数据记载在 /var/log/secure 当中，所以发生了问题请务必到该文件内去查询一下问题点！举例来说， 我们在 limits.conf 的介绍内的范例二，就有谈到多重登陆的错误可以到 /var/log/secure 内查阅了！ 这样你也就知道为何第二个 pro1 无法登陆啦！^_^

# Linux 主机上的用户信息传递
谈了这么多的账号问题，总是该要谈一谈，那么如何针对系统上面的用户进行查询吧？ 想几个状态，如果你在 Linux 上面操作时，刚好有其他的用户也登陆主机，你想要跟他对谈，该如何是好？ 你想要知道某个账号的相关信息，该如何查阅？呼呼！底下我们就来聊一聊～


## 查询使用者： w, who, last, lastlog
如何查询一个用户的相关数据呢？这还不简单，我们之前就提过了 id, finger 等命令了，都可以让您了解到一个用户的相关信息啦！那么想要知道使用者到底啥时候登陆呢？ 最简单可以使用 last 检查啊！这个玩意儿我们也在 第十一章 bash 提过了， 您可以自行前往参考啊！简单的很。

Tips:
早期的 Red Hat 系统的版本中， last 仅会列出当月的登陆者信息，不过在我们的 CentOS 5.x 版以后， last 可以列出从系统创建之后到目前为止的所有登陆者信息！这是因为登录档轮替的配置不同所致。 详细的说明可以参考后续的第十九章登录档简介。	鸟哥的图示
那如果你想要知道目前已登陆在系统上面的用户呢？可以透过 w 或 who 来查询喔！如下范例所示：

[root@www ~]# w
 13:13:56 up 13:00,  1 user,  load average: 0.08, 0.02, 0.01
USER   TTY    FROM            LOGIN@   IDLE   JCPU   PCPU WHAT
root   pts/1  192.168.1.100   11:04    0.00s  0.36s  0.00s -bash
vbird1 pts/2  192.168.1.100   13:15    0.00s  0.06s  0.02s w
# 第一行显示目前的时间、启动 (up) 多久，几个用户在系统上平均负载等；
# 第二行只是各个项目的说明，
# 第三行以后，每行代表一个使用者。如上所示，root 登陆并取得终端机名 pts/1 之意。

[root@www ~]# who
root     pts/1        2009-03-04 11:04 (192.168.1.100)
vbird1   pts/2        2009-03-04 13:15 (192.168.1.100)
另外，如果您想要知道每个账号的最近登陆的时间，则可以使用 lastlog 这个命令喔！ lastlog 会去读取 /var/log/lastlog 文件，结果将数据输出如下表：

[root@www ~]# lastlog
Username    Port   From           Latest
root        pts/1  192.168.1.100  Wed Mar  4 11:04:22 +0800 2009
bin                                        **Never logged in**
....(中间省略)....
vbird1      pts/2  192.168.1.100  Wed Mar  4 13:15:56 +0800 2009
....(以下省略)....
这样就能够知道每个账号的最近登陆的时间啰～ ^_^

## 使用者对谈： write, mesg, wall
那么我是否可以跟系统上面的用户谈天说地呢？当然可以啦！利用 write 这个命令即可。 write 可以直接将信息传给接收者啰！举例来说，我们的 Linux 目前有 vbird1 与 root 两个人在在线， 我的 root 要跟 vbird1 讲话，可以这样做：

[root@www ~]# write 使用者账号 [用户所在终端接口]

[root@www ~]# who
root     pts/1    2009-03-04 11:04 (192.168.1.100)
vbird1   pts/2    2009-03-04 13:15 (192.168.1.100)  <==有看到 vbird1 在在线

[root@www ~]# write vbird1 pts/2
Hello, there:
Please don't do anything wrong...  <==这两行是 root 写的信息！
# 结束时，请按下 [crtl]-d 来结束输入。此时在 vbird1 的画面中，会出现：

Message from root@www.vbird.tsai on pts/1 at 13:23 ...
Hello, there:
Please don't do anything wrong...
EOF
怪怪～立刻会有信息响应给 vbird1 ！不过......当时 vbird1 正在查数据，哇！ 这些信息会立刻打断 vbird1 原本的工作喔！所以，如果 vbird1 这个人不想要接受任何信息，直接下达这个动作：

[vbird1@www ~]$ mesg n
[vbird1@www ~]$ mesg
is n
不过，这个 mesg 的功能对 root 传送来的信息没有抵挡的能力！所以如果是 root 传送信息， vbird1 还是得要收下。 但是如果 root 的 mesg 是 n 的，那么 vbird1 写给 root 的信息会变这样：

[vbird1@www ~]$ write root
write: root has messages disabled
了解乎？如果想要解开的话，再次下达『 mesg y 』就好啦！想要知道目前的 mesg 状态，直接下达『 mesg 』即可！瞭呼？ 相对于 write 是仅针对一个使用者来传『简讯』，我们还可以『对所有系统上面的用户传送简讯 (广播)』哩～ 如何下达？用 wall 即可啊！他的语法也是很简单的喔！

[root@www ~]# wall "I will shutdown my linux server..."
然后你就会发现所有的人都会收到这个简讯呢！

## 使用者邮件信箱： mail
使用 wall, write 毕竟要等到使用者在在线才能够进行，有没有其他方式来联络啊？ 不是说每个 Linux 主机上面的用户都具有一个 mailbox 吗？ 我们可否寄信给使用者啊！呵呵！当然可以啊！我们可以寄、收 mailbox 内的信件呢！ 一般来说， mailbox 都会放置在 /var/spool/mail 里面，一个账号一个 mailbox (文件)。 举例来说，我的 vbird1 就具有 /var/spool/mail/vbird1 这个 mailbox 喔！

那么我该如何寄出信件呢？就直接使用 mail 这个命令即可！这个命令的用法很简单的，直接这样下达：『 mail username@localhost -s "邮件标题" 』即可！ 一般来说，如果是寄给本机上的使用者，基本上，连『 @localhost 』都不用写啦！ 举例来说，我以 root 寄信给 vbird1 ，信件标题是『 nice to meet you 』，则：

[root@www ~]# mail vbird1 -s "nice to meet you"
Hello, D.M. Tsai
Nice to meet you in the network.
You are so nice.  byebye!
.    <==这里很重要喔，结束时，最后一行输入小数点 . 即可！
Cc:  <==这里是所谓的『副本』，不需要寄给其他人，所以直接 [Enter]
[root@www ~]#  <==出现提示字符，表示输入完毕了！
如此一来，你就已经寄出一封信给 vbird1 这位使用者啰，而且，该信件标题为： nice to meet you，信件内容就如同上面提到的。不过，你或许会觉得 mail 这个程序不好用～ 因为在信件编写的过程中，如果写错字而按下 Enter 进入次行，前一行的数据很难删除ㄟ！ 那怎么办？没关系啦！我们使用数据流重导向啊！呵呵！利用那个小于的符号 ( < ) 就可以达到取代键盘输入的要求了。也就是说，你可以先用 vi 将信件内容编好， 然后再以 mail vbird1 -s "nice to meet you" < filename 来将文件内容传输即可。

例题：
请将你的家目录下的环境变量文件 (~/.bashrc) 寄给自己！
答：
mail -s "bashrc file content" vbird < ~/.bashrc

刚刚上面提到的是关于『寄信』的问题，那么如果是要收信呢？呵呵！同样的使用 mail 啊！ 假设我以 vbird1 的身份登陆主机，然后输入 mail 后，会得到什么？

[vbird1@www ~]$ mail
Mail version 8.1 6/6/93.  Type ? for help.
"/var/spool/mail/vbird1": 1 message 1 new
>N  1 root@www.vbird.tsai   Wed Mar  4 13:36  18/663   "nice to meet you"
&  <==这里可以输入很多的命令，如果要查阅，输入 ? 即可！
在 mail 当中的提示字符是 & 符号喔，别搞错了～输入 mail 之后，我可以看到我有一封信件， 这封信件的前面那个 > 代表目前处理的信件，而在大于符号的左边那个 N 代表该封信件尚未读过， 如果我想要知道这个 mail 内部的命令有哪些，可以在 & 之后输入『 ? 』，就可以看到如下的画面：

& ?
    Mail   Commands
t <message list>                type messages
n                               goto and type next message
e <message list>                edit messages
f <message list>                give head lines of messages
d <message list>                delete messages
s <message list> file           append messages to file
u <message list>                undelete messages
R <message list>                reply to message senders
r <message list>                reply to message senders and all recipients
pre <message list>              make messages go back to /usr/spool/mail
m <user list>                   mail to specific users
q                               quit, saving unresolved messages in mbox
x                               quit, do not remove system mailbox
h                               print out active message headers
!                               shell escape
cd [directory]                  chdir to directory or home if none given
<message list> 指的是每封邮件的左边那个数字啦！而几个比较常见的命令是：

命令	意义
h	列出信件标头；如果要查阅 40 封信件左右的信件标头，可以输入『 h 40 』
d	删除后续接的信件号码，删除单封是『 d10 』，删除 20~40 封则为『 d20-40 』。 不过，这个动作要生效的话，必须要配合 q 这个命令才行(参考底下说明)！
s	将信件储存成文件。例如我要将第 5 封信件的内容存成 ~/mail.file:『s 5 ~/mail.file』
x	或者输入 exit 都可以。这个是『不作任何动作离开 mail 程序』的意思。 不论你刚刚删除了什么信件，或者读过什么，使用 exit 都会直接离开 mail，所以刚刚进行的删除与阅读工作都会无效。 如果您只是查阅一下邮件而已的话，一般来说，建议使用这个离开啦！除非你真的要删除某些信件。
q	相对于 exit 是不动作离开， q 则会进行两项动作： 1. 将刚刚删除的信件移出 mailbox 之外； 2. 将刚刚有阅读过的信件存入 ~/mbox ，且移出 mailbox 之外。鸟哥通常不很喜欢使用 q 离开， 因为，很容易忘记读过什么咚咚～导致信件给他移出 mailbox 说～

由于读过的信件若使用『 q 』来离开 mail 时，会将该信件移动到 ~/mbox 中，所以你可以这样想象： /var/spool/mail/vbird1 为 vbird1 的『新件匣』，而 /home/vbird1/mbox 则为『收件匣』的意思。 那如何读取 /home/vbird1/mbox 呢？就使用『mail -f /home/vbird1/mbox』即可。

# 手动新增使用者
一般来说，我们不很建议大家使用手动的方式来新增使用者，为什么呢？ 因为使用者的创建涉及到 GID/UID 等权限的关系，而且，与文件/目录的权限也有关系， 使用 useradd 可以帮我们自动配置好 UID/GID 家目录以及家目录相关的权限配置， 但是，手动来添加的时候，有可能会忘东忘西，结果导致一些困扰的发生。

不过，要了解整个系统，最好还是手动来修改过比较好，至少我们的账号问题可以完全依照自己的意思去修订， 而不必迁就于系统的默认值啊！但是，还是要告诫一下朋友们，要手动配置账号时， 您必须要真的很了解自己在作什么，尤其是与权限有关的配置方面喔！好吧！底下就让我们来玩一玩啰～ ^_^


## 一些检查工具
既然要手动修改账号的相关配置文件，那么一些检查群组、账号相关的命令就不可不知道啊～ 尤其是那个口令转换的 pwconv 及 pwuconv 这两个玩意～可重要的很呢！底下我们稍微介绍一下这些命令吧！


pwck
pwck 这个命令在检查 /etc/passwd 这个账号配置文件内的信息，与实际的家目录是否存在等信息， 还可以比对 /etc/passwd /etc/shadow 的信息是否一致，另外，如果 /etc/passwd 内的数据字段错误时，会提示使用者修订。 一般来说，我只是利用这个玩意儿来检查我的输入是否正确就是了。

[root@www ~]# pwck
user adm: directory /var/adm does not exist
user uucp: directory /var/spool/uucp does not exist
user gopher: directory /var/gopher does not exist
瞧！上面仅是告知我，这些账号并没有家目录，由于那些账号绝大部分都是系统账号， 确实也不需要家目录的，所以，那是『正常的错误！』呵呵！不理他。 ^_^。 相对应的群组检查可以使用 grpck 这个命令的啦！


pwconv
这个命令主要的目的是在『将 /etc/passwd 内的账号与口令，移动到 /etc/shadow 当中！』 早期的 Unix 系统当中并没有 /etc/shadow 呢，所以，用户的登陆口令早期是在 /etc/passwd 的第二栏，后来为了系统安全，才将口令数据移动到 /etc/shadow 内的。使用 pwconv 后，可以：

比对 /etc/passwd 及 /etc/shadow ，若 /etc/passwd 内存在的账号并没有对应的 /etc/shadow 口令时，则 pwconv 会去 /etc/login.defs 取用相关的口令数据，并创建该账号的 /etc/shadow 数据；

若 /etc/passwd 内存在加密后的口令数据时，则 pwconv 会将该口令栏移动到 /etc/shadow 内，并将原本的 /etc/passwd 内相对应的口令栏变成 x ！
一般来说，如果您正常使用 useradd 添加使用者时，使用 pwconv 并不会有任何的动作，因为 /etc/passwd 与 /etc/shadow 并不会有上述两点问题啊！ ^_^。不过，如果手动配置账号，这个 pwconv 就很重要啰！


pwunconv
相对于 pwconv ， pwunconv 则是『将 /etc/shadow 内的口令栏数据写回 /etc/passwd 当中， 并且删除 /etc/shadow 文件。』这个命令说实在的，最好不要使用啦！ 因为他会将你的 /etc/shadow 删除喔！如果你忘记备份，又不会使用 pwconv 的话，粉严重呢！


chpasswd
chpasswd 是个挺有趣的命令，他可以『读入未加密前的口令，并且经过加密后， 将加密后的口令写入 /etc/shadow 当中。』这个命令很常被使用在大量建置账号的情况中喔！ 他可以由 Standard input 读入数据，每笔数据的格式是『 username:password 』。 举例来说，我的系统当中有个用户账号为 dmtsai ，我想要升级他的口令 (update) ， 假如他的口令是 abcdefg 的话，那么我可以这样做：

[root@www ~]# echo "dmtsai:abcdefg" | chpasswd -m
神奇吧！这样就可以升级了呢！在默认的情况中， chpasswd 使用的是 DES 加密方法来加密， 我们可以使用 chpasswd -m 来使用 CentOS 5.x 默认的 MD5 加密方法。这个命令虽然已经很好用了，不过 CentOS 5.x 其实已经提供了『 passwd --stdin 』的选项，老实说，这个 chpasswd 可以不必使用了。但考虑其他版本不见得会提供 --stdin 给 passwd 这个命令，所以您还是得要了解一下这个命令用途！

## 特殊账号，如纯数字账号的手工创建
在我们了解了 UID/GID 与账号的关系之后，基本上，您应该了解了，为啥我们不建议使用纯数字的账号了！因为很多时候，系统会搞不清楚那组数字是『账号』还是『 UID 』，这不是很好啦～也因此，在早期某些版本底下，是没有办法使用数字来创建账号的。例如在 Red Hat 9 的环境中， 使用『 useradd 1234 』他会显示『 useradd: invalid user name '1234' 』了解了吗？

Tips:
在较新的 distribution 当中，纯数字的账号已经可以被 useradd 创建了。不过鸟哥还是非常不建议使用纯数字账号。 例如在 setfacl 的配置值中，若使用『 setfacl -m u:501:rwx filename 』那个 501 代表的是 UID 还是账号？ 因为 setfacl 的配置是支持使用 UID 或账号的，纯数字账号很容易造成系统的误解！	鸟哥的图示
不过，有的时候，长官的命令难为啊 @_@ 有时还是得要创建这方面的账号的，那该如何是好？ 呵呵！当然可以手动来创建这样的账号啦！不过，为了系统安全起见，鸟哥还是不建议使用纯数字的账号的啦！ 因此，底下的范例当中，我们使用手动的方式来创建一个名为 normaluser 的账号， 而且这个账号属于 normalgroup 这个群组。OK！那么整个步骤该如何是好呢？ 由前面的说明来看，您应该了解了账号与群组是与 /etc/group, /etc/shadow, /etc/passwd, /etc/gshadow 有关，因此，整个动作是这样的：

先创建所需要的群组 ( vi /etc/group )；
将 /etc/group 与 /etc/gshadow 同步化 ( grpconv )；
创建账号的各个属性 ( vi /etc/passwd )；
将 /etc/passwd 与 /etc/shadow 同步化 ( pwconv )；
创建该账号的口令 ( passwd accountname )；
创建用户家目录 ( cp -a /etc/skel /home/accountname )；
更改用户家目录的属性 ( chown -R accountname.group /home/accountname )。
够简单的咯吧！让我们来玩一玩啰～
1. 创建群组 normalgroup ，假设 520 这个 GID 没有被使用！并且同步化 gshadow
[root@www ~]# vi /etc/group
# 在最后一行加入底下这一行！
normalgroup:x:520:
[root@www ~]# grpconv
[root@www ~]# grep 'normalgroup' /etc/group /etc/gshadow
/etc/group:normalgroup:x:520:
/etc/gshadow:normalgroup:x::
# 最后确定 /etc/group, /etc/gshadow 都存在这个群组才行！搞定群组啰！

2. 创建 normaluser 这个账号，假设 UID 700 没被使用掉！
[root@www ~]# vi /etc/passwd
# 在最后一行加入底下这一行！
normaluser:x:700:520::/home/normaluser:/bin/bash

3. 同步化口令，并且创建该用户的口令
[root@www ~]# pwconv
[root@www ~]# grep 'normaluser' /etc/passwd /etc/shadow
/etc/passwd:normaluser:x:700:520::/home/normaluser:/bin/bash
/etc/shadow:normaluser:x:14307:0:99999:7:::
# 确定 /etc/passwd, /etc/shadow 都含有 normaluser 的信息了！但是口令还不对～
[root@www ~]# passwd normaluser
Changing password for user normaluser.
New UNIX password:
Retype new UNIX password:
passwd: all authentication tokens updated successfully.

4. 创建用户家目录，并且修订权限！
[root@www ~]# cp -a /etc/skel /home/normaluser
[root@www ~]# chown -R normaluser:normalgroup /home/normaluser
[root@www ~]# chmod 700 /home/normaluser
别怀疑！这样就搞定了一个账号的配置了！ 从此以后，你可以创建任何名称的账号啰～不过，还是不建议您配置一些很怪很怪的账号名称啦！

## 大量建置账号范本(适用 passwd --stdin 选项)
由于 CentOS 5.x 的 passwd 已经提供了 --stdin 的功能，因此如果我们可以提供账号口令的话， 那么就能够很简单的建置起我们的账号口令了。底下鸟哥制作一个简单的 script 来运行新增用户的功能喔！

[root@www ~]# vi account1.sh
#!/bin/bash
# 这支程序用来创建新增账号，功能有：
# 1. 检查 account1.txt 是否存在，并将该文件内的账号取出；
# 2. 创建上述文件的账号；
# 3. 将上述账号的口令修订成为『强制第一次进入需要修改口令』的格式。
# 2009/03/04    VBird
export PATH=/bin:/sbin:/usr/bin:/usr/sbin

# 检查 account1.txt 是否存在
if [ ! -f account1.txt ]; then
        echo "所需要的账号文件不存在，请创建 account1.txt ，每行一个账号名称"
        exit 1
fi

usernames=$(cat account1.txt)

for username in $usernames
do
        useradd $username                         <==新增账号
        echo $username | passwd --stdin $username <==与账号相同的口令
        chage -d 0 $username                      <==强制登陆修改口令
done
接下来只要创建 account1.txt 这个文件即可！鸟哥创建这个文件里面共有十行，你可以自行创建该文件！ 内容每一行一个账号。注意，最终的结果会是每个账号具有与账号相同的口令，且初次登陆后， 必须要重新配置口令后才能够再次登陆使用系统资源！

[root@www ~]# vi account1.txt
std01
std02
std03
std04
std05
std06
std07
std08
std09
std10

[root@www ~]# sh account1.sh
Changing password for user std01.
passwd: all authentication tokens updated successfully.
....(后面省略)....
这支简单的脚本你可以在按如下的连结下载：

http://cn.linux.vbird.org/linux_basic/0410accountmanager/account1.sh
另外，鸟哥的 script 是在 zh_TW.big5 的语系下创建的，如果你需要转成万国码 (utf8) 的编码格式， 请下载上述文件后，利用第十章谈到的 iconv 来处理语系的问题！

## 大量建置账号的范例(适用于连续数字，如学号)
前一小节的内容已经可以满足很多朋友的账号建置方法了，不过，某些时候上述的 script 还是很麻烦！ 因为需要手动编辑 account1.txt 嘛！如果是类似学校单位这种学号非常类似的账号时，有没有更快的方案？ 此外，如果需要每个班级同属于一个群组，不同班级的群组不同，又该如何建置？这是比较麻烦啦！

目前很多网站都有提供大量创建账号的工具，例如台南县网中心的卧龙大师：

http://news.ols3.net/techdoc/old/howtouse_cmpwd101.htm
提供的好用的 cmpwd 程序，但是小三大师的程序仅供学术单位使用，一般个人是无权使用的(参考上述连结的授权)。 不过，其实我们也可以利用简单的 script 来帮我们达成喔！例如底下这支程序， 他的运行结果与小三大师提供的程序差不多啦～但是因为我是直接以 useradd 来新增的， 所以，即使不了解 UID ，也是可以适用的啦～整支程序的特色是：

默认不允许使用纯数字方式创建账号；
可加入年级来区分账号；
可配置账号的起始号码与账号数量；
有两种口令创建方式，可以与账号相同或程序自行以随机数创建口令文件。
运行方法也简单的要命～请自行参考的啦！不再多说～使用时请注意，不要在公家使用的主机上面进行测试，因为..... 这支程序会大量创建账号嘛！^_^

#!/bin/bash
#
# 这支程序主要在帮您创建大量的账号之用，更多的使用方法请参考：
# http://cn.linux.vbird.org/linux_basic/0410accountmanager.php#manual_amount
#
# 本程序为鸟哥自行开发，在 CentOS 5.x 上使用没有问题，
# 但不保证绝不会发生错误！使用时，请自行负担风险～
#
# History:
# 2005/09/05    VBird   刚刚才写完，使用看看先～
# 2009/03/04    VBird   加入一些语系的修改与说明，修改口令产生方式 (用 openssl)
export LANG=zh_TW.big5
export PATH=/sbin:/usr/sbin:/bin:/usr/bin
accountfile="user.passwd"

# 1. 进行账号相关的输入先！
echo ""
echo "例如我们昆山四技的学号为： 4960c001 到 4960c060 ，那么："
echo "账号开头代码为         ：4"
echo "账号层级或年级为       ：960c"
echo "号码数字位数为(001~060)：3"
echo "账号开始号码为         ：1"
echo "账号数量为             ：60"
echo ""
read -p "账号开头代码 ( Input title name, ex> std )======> " username_start
read -p "账号层级或年级 ( Input degree, ex> 1 or enter )=> " username_degree
read -p "号码部分的数字位数 ( Input \# of digital )======> " nu_nu
read -p "起始号码 ( Input start number, ex> 520 )========> " nu_start
read -p "账号数量 ( Input amount of users, ex> 100 )=====> " nu_amount
read -p "口令标准 1) 与账号相同 2)随机数自定义 ==============> " pwm
if [ "$username_start" == "" ]; then
        echo "没有输入开头的代码，不给你运行哩！" ; exit 1
fi
# 判断数字系统
testing0=$(echo $nu_nu     | grep '[^0-9]' )
testing1=$(echo $nu_amount | grep '[^0-9]' )
testing2=$(echo $nu_start  | grep '[^0-9]' )
if [ "$testing0" != "" -o "$testing1" != "" -o "$testing2" != "" ]; then
        echo "输入的号码不对啦！有非为数字的内容！" ; exit 1
fi
if [ "$pwm" != "1" ]; then
        pwm="2"
fi

# 2. 开始输出账号与口令文件！
[ -f "$accountfile" ] && mv $accountfile "$accountfile"$(date +%Y%m%d)
nu_end=$(($nu_start+$nu_amount-1))
for (( i=$nu_start; i<=$nu_end; i++ ))
do
        nu_len=${#i}
        if [ $nu_nu -lt $nu_len ]; then
                echo "数值的位数($i->$nu_len)已经比你配置的位数($nu_nu)还大！"
                echo "程序无法继续"
                exit 1
        fi
        nu_diff=$(( $nu_nu - $nu_len ))
        if [ "$nu_diff" != "0" ]; then
                nu_nn=0000000000
                nu_nn=${nu_nn:1:$nu_diff}
        fi
        account=${username_start}${username_degree}${nu_nn}${i}
        if [ "$pwm" == "1" ]; then
                password="$account"
        else
                password=$(openssl rand -base64 6)
        fi
        echo "$account":"$password" | tee -a "$accountfile"
done

# 3. 开始创建账号与口令！
cat "$accountfile" | cut -d':' -f1 | xargs -n 1 useradd -m
chpasswd < "$accountfile"
pwconv
echo "OK！创建完成！"
如果有需要创建同一班级具有同一群组的话，可以先使用 groupadd 创建群组后， 将该群组加入『 cat "$accountfile" | cut -d':' -f1 | xargs -n 1 useradd -m -g groupname 』那行！这支脚本可以在底下连结下载：

http://cn.linux.vbird.org/linux_basic/0410accountmanager/account2.sh
如果仅是测试而已，想要将刚刚创建的使用者整个删除，则可以使用如下的脚本来进行删除！

[root@www ~]# vi delaccount2.sh
#!/bin/bash
usernames=$(cat user.passwd | cut -d ':' -f 1)
for username in $usernames
do
	echo "userdel -r $username"
	userdel -r $username
done
[root@www ~]# sh delaccount2.sh
总之，账号管理是很重要的！希望上面的说明能够对大家有点帮助啦！

# 重点回顾
Linux 操作系统上面，关于账号与群组，其实记录的是 UID/GID 的数字而已；
使用者的账号/群组与 UID/GID 的对应，参考 /etc/passwd 及 /etc/group 两个文件
/etc/passwd 文件结构以冒号隔开，共分为七个字段，分别是『账号名称、口令、UID、GID、全名、家目录、shell』
UID 只有 0 与非为 0 两种，非为 0 则为一般账号。一般账号又分为系统账号 (1~499) 即可登陆者账号 (大于 500)
账号的口令已经移动到 /etc/shadow 文件中，该文件权限为仅有 root 可以更动。该文件分为九个字段，内容为『 账号名称、加密口令、口令更动日期、口令最小可变动日期、口令最大需变动日期、口令过期前警告日数、口令失效天数、 账号失效日、保留未使用』
使用者可以支持多个群组，其中在新建文件时会影响新文件群组者，为有效群组。而写入 /etc/passwd 的第四个字段者， 称为初始群组。
与使用者创建、更改参数、删除有关的命令为：useradd, usermod, userdel等，口令创建则为 passwd；
与群组创建、修改、删除有关的命令为：groupadd, groupmod, groupdel 等；
群组的观察与有效群组的切换分别为：groups 及 newgrp 命令；
useradd 命令作用参考的文件有： /etc/default/useradd, /etc/login.defs, /etc/skel/ 等等
观察用户详细的口令参数，可以使用『 chage -l 账号 』来处理；
用户自行修改参数的命令有： chsh, chfn 等，观察命令则有： id, finger 等
ACL 可进行单一个人或群组的权限管理，但 ACL 的启动需要有文件系统的支持；
ACL 的配置可使用 setfacl ，查阅则使用 getfacl ；
身份切换可使用 su ，亦可使用 sudo ，但使用 sudo 者，必须先以 visudo 配置可使用的命令；
PAM 模块可进行某些程序的验证程序！与 PAM 模块有关的配置文件位于 /etc/pam.d/* 及 /etc/security/*
系统上面账号登陆情况的查询，可使用 w, who, last, lastlog 等；
在线与使用者交谈可使用 write, wall，脱机状态下可使用 mail 传送邮件！
# 本章习题
情境模拟题一：想将本服务器的账号分开管理，分为单纯邮件使用，与可登陆系统账号两种。其中若为纯邮件账号时， 将该账号加入 mail 为初始群组，且此账号不可使用 bash 等 shell 登陆系统。若为可登陆账号时， 将该账号加入 youcan 这个次要群组。

目标：了解 /sbin/nologin 的用途；
前提：可自行观察使用者是否已经创建等问题；
需求：需已了解 useradd, groupadd 等命令的用法；

解决方案如下：

预先察看一下两个群组是否存在？
[root@www ~]# grep mail /etc/group
[root@www ~]# grep youcan /etc/group
[root@www ~]# groupadd youcan
可发现 youcan 尚未被创建，因此如上表所示，我们主动去创建这个群组啰。

开始创建三个邮件账号，此账号名称为 pop1, pop2, pop3 ，且口令与账号相同。可使用如下的程序来处理：
[root@www ~]# vim popuser.sh
#!/bin/bash
for username in pop1 pop2 pop3
do
	useradd -g mail -s /sbin/nologin -M $username
	echo $username | passwd --stdin $username
done
[root@www ~]# sh popuser.sh

开始创建一般账号，只是这些一般账号必须要能够登陆，并且需要使用次要群组的支持！所以：
[root@www ~]# vim loginuser.sh
#!/bin/bash
for username in youlog1 youlog2 youlog3
do
	useradd -G youcan -s /bin/bash -m $username
	echo $username | passwd --stdin $username
done
[root@www ~]# sh loginuser.sh

这样就将账号分开管理了！非常简单吧！
简答题部分
root 的 UID 与 GID 是多少？而基于这个理由，我要让 test 这个账号具有 root 的权限，应该怎么作？
root 的 UID 与 GID 均为 0 ，所以要让 test 变成 root 的权限，那么就将 /etc/passwd 里面， test 的 UID 与 GID 字段变成 0 即可！
假设我是一个系统管理员，我有一个用户最近不乖，所以我想暂时将他的账号停掉， 让他近期无法进行任何动作，等到未来他乖一点之后，我再将他的账号激活，请问：我可以怎么作比较好？？
由于这个账号是暂时失效的，所以不能使用 userdel 来删除，否则很麻烦！那么应该如何配置呢？再回去瞧一瞧 /etc/shadow 的架构，可以知道有这几个可使用的方法：
将 /etc/passwd 的 shell 字段写成 /sbin/nologin ，即可让该账号暂时无法登陆主机；
将 /etc/shadow 内的口令字段，添加一个 * 号在最前面，这样该账号亦无法登陆！
将 /etc/shadow 的第八个字段关于账号取消日期的那个，配置小于目前日期的数字，那么他就无法登陆系统了！
我在使用 useradd 的时候，新增的账号里面的 UID, GID 还有其他相关的口令控制，都是在哪几个文件里面配置的？
在 /etc/login.defs 还有 /etc/default/useradd 里面规定好的！
我希望我在配置每个账号的时候( 使用 useradd )，默认情况中，他们的家目录就含有一个名称为 www 的子目录，我应该怎么作比较好？
由于使用 useradd 的时候，会自动以 /etc/skel 做为默认的家目录，所以，我可以在 /etc/skel 里面新添加一个名称为 www 的目录即可！
简单说明系统账号与一般用户账号的差别？
一般而言，为了让系统能够顺利以较小的权限运行，系统会有很多账号， 例如 mail, bin, adm 等等。而为了确保这些账号能够在系统上面具有独一无二的权限， 一般来说 Linux 都会保留一些 UID 给系统使用。在 CentOS 5.x 上面，小于 500 以下的账号 (UID) 即是所谓的 System account。
简单说明，为何 CentOS 5.x 创建使用者时，他会主动的帮使用者创建一个群组，而不是使用 /etc/default/useradd 的配置？
不同的 linux distributions 对于使用者 group 的创建机制并不相同。主要的机制分为：
Public group schemes: 用户将会直接给予一个系统指定的群组，一般来说即是 users ， 可以 SuSE Server 9 为代表；
Private group schemes: 系统会创建一个与账号一样的组名！以 CentOS 5.x 为例！
如何创建一个使用者名称 alex, 他所属群组为 alexgroup, 预计使用 csh, 他的全名为 "Alex Tsai"， 且他还得要加入 users 群组当中！
groupadd alexgroup
useradd -c "Alex Tsai" -g alexgroup -G users -m alex
务必先创建群组，才能够创建使用者喔！
由于种种因素，导致你的用户家目录以后都需要被放置到 /account 这个目录下。 请问，我该如何作，可以让使用 useradd 时，默认的家目录就指向 /account ？
最简单的方法，编辑 /etc/default/useradd ，将里头的 HOME=/home 改成 HOME=/account 即可。
我想要让 dmtsai 这个使用者，加入 vbird1, vbird2, vbird3 这三个群组，且不影响 dmtsai 原本已经支持的次要群组时，该如何动作？
usermod -a -G vbird1,vbird2,vbird3 dmtsai

# Reference

- [原文 - 第十四章、Linux 账号管理与 ACL 权限配置](http://cn.linux.vbird.org/linux_basic/0410accountmanager.php#normal_user)

- 注1：最完整与详细的口令文件说明，可参考各 distribution 内部的 man page。 本文中以 CentOS 5.x 的『 man 5 passwd 』及『 man 5 shadow 』的内容说明；
- 注2：MD5与DES均为加密的机制，详细的解释可参考维基百科的说明：[MD5](http://zh.wikipedia.org/wiki/MD5), [DES](http://en.wikipedia.org/wiki/Data_Encryption_Standard)
在早期的 Linux 版本中，主要使用 DES 加密算法，近期则使用 MD5 作为默认算法。
- 注3：telnet 与 ssh 都是可以由远程用户主机联机到 Linux 服务器的一种机制！详细数据可查询鸟站文章： [远程联机服务器](http://cn.linux.vbird.org/linux_server/0310telnetssh.php)
- 注4：详细的说明请参考 man sudo ，然后以 5 作为关键词搜寻看看即可了解。
- 注5：详细的 PAM 说明可以参考如下连结：[维基百科](http://en.wikipedia.org/wiki/Pluggable_Authentication_Modules), [Linux-PAM网页](http://www.kernel.org/pub/linux/libs/pam/)
