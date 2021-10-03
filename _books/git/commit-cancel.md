---
title: "撤销修改"
tags: [git]
--- 

# 撤销工作区修改[checkout]

本地修改了一堆文件(并没有使用 `git add` 到暂存区)，想放弃修改。

单个文件/文件夹：

```sh
$ git checkout -- filename
```

所有文件/文件夹：

```sh
$ git checkout .
```

命令 `git checkout -- filename` 意思就是，把 `filename` 文件在工作区的修改全部撤销，这里有两种情况：

- `readme.txt` 自修改后还没有被放到暂存区，现在，撤销修改就回到和版本库一模一样的状态；

- `readme.txt` 已经添加到暂存区后，又作了修改，现在，撤销修改就回到添加到暂存区后的状态。

总之，就是让这个文件回到最近一次 `git commit` 或 `git add` 时的状态。

# 撤销工作区新增文件

本地新增了一堆文件(并没有git add到暂存区)，想放弃修改。

单个文件/文件夹：

```sh
$ rm filename / rm dir -rf
```

所有文件/文件夹：

```sh
$ git clean -xdf
```

> 删除新增的文件，如果文件已经已经 `git add` 到暂存区，并不会删除！


# 撤销暂存区修改[reset]

本地修改/新增了一堆文件，已经 `git add` 到暂存区，想放弃修改。

单个文件/文件夹：

```sh
$ git reset HEAD filename
```

所有文件/文件夹：

```sh
$ git reset HEAD .
```

# 撤销已commit修改[reset]

本地通过 `git add & git commit` 之后，想要撤销此次 `commit`, 可以使用 `git reset id`

这个 `id` 是你想要回到的那个节点，可以通过 `git log` 查看，可以只选前6位

```sh
$ git reset commit_id
```

> 撤销之后，你所做的已经 `commit` 的修改还在工作区！`git reset id` 只是把暂存区的修改撤销掉（`unstage`），重新放回工作区：

```sh
$ git reset --hard commit_id
```

> 撤销之后，你所做的已经 `commit` 的修改将会清除，仍在工作区/暂存区的代码也将会清除！


`git reset` 命令既可以回退版本，也可以把暂存区的修改回退到工作区。当我们用 `HEAD` 时，表示最新的版本。

# 撤销已经push的修改[revert]

在push代码以后,又使用 `reset --hard <commit...>` 回退代码到某个版本之前,但是这样会有一个问题,你线上的代码没有变,线上 `commit,stage` 都没有变,当你把本地代码修改完提交的时候你会发现权是冲突..... 所以,这种情况你要使用下面的方式

```
git revert commitid
```

`git revert commitid` 用于反转提交,执行命令时要求工作树必须是干净的.其是用一个新提交来消除一个历史提交所做的任何修改.

看似达到的效果是一样的,其实完全不同.

- `git revert` 是用一次新的commit来回滚之前的commit， `git reset` 是直接删除指定的commit

- 如果在日后现有分支和历史分支需要 `merge` 的时候, 因为 `git revert` 是用一次逆向的commit“中和”之前的提交，因此日后合并老的branch时，导致这部分改变不会再次出现，但是 `git reset` 是之间把某些commit在某个branch上删除，因而和老的branch再次merge时，这些被回滚的commit应该还会被引入(因为在没有删除的分支上还有，merge的时候依然会被合入)。

- reset 是在正常的commit历史中,删除了指定的commit,这时 HEAD 是向后移动了,而 revert 是在正常的commit历史中再commit一次,只不过是反向提交,他的 HEAD 是一直向前的.

# reset 三种模式

`git reset commitid` 用于撤销已经提交的修改。由于git中将代码分成了三个分区 `工作空间`, `stage`, `已提交库`。所以撤销也对应了三种级别的撤销。

## --soft

回退 `版本库` 信息到某个版本。不涉及 `工作空间` 和 `stage` 的回退,如果还需要提交,直接 `commit` 即可.


## --mixed

将 `stage` 和 `已提交库` 中的代码回退到 `commitid` 对应的版本。 工作空间中的代码不变，保持当前工作空间中的代码。

```
git reset 默认是 --mixed 模式 
git reset --mixed  等价于  git reset
```

## --hard

三个空间的代码全都回退到 `commitid` 对应的版本状态。

# 小结

- 修改了工作区某个文件的内容，想直接丢弃工作区的修改时，用命令 `git checkout -- file`。

- 修改了工作区某个文件的内容，还添加到了暂存区时，想丢弃修改，分两步，第一步用命令 `git reset HEAD <file>` ，就回到了场景1，第二步按场景1操作。

- 已经commit了，想要撤销本次提交，`git reset commitid`

- 已经 push 到远程repo需要撤销，只能用一次反向提交来抵消以前的修改，`git revert commitid`

# Reference

- [原文 - 撤销修改](https://www.liaoxuefeng.com/wiki/896043488029600/897889638509536)


- [git放弃修改&放弃增加文件](https://blog.csdn.net/ustccw/article/details/79068547)

- [git中reset与revert的区别](https://www.jianshu.com/p/0e1fe709dd97)
