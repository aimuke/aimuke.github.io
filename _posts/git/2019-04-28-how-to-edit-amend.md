---
title: "git如何编辑amend信息"
tags: ["git", "amend", "editor"]

---

通过命令行amend提交信息时，没有反应，不知道应该如何退出，界面如下

```sh
Create 2019-04-26-details.md

# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
#
# Author:    Bob <aaa@126.com>
# Date:      Fri Apr 26 17:33:21 2019 +0800
#
# On branch master
# Your branch is up-to-date with 'origin/master'.
#
# Changes to be committed:
#       new file:   _posts/linux/2019-04-26-details.md
#
# Changes not staged for commit:
#       modified:   _config.yml
#       deleted:    guide/_posts/2013-12-24-categories.md
#       deleted:    guide/index.html
#
# Untracked files:
#       _guides/
#       guide.html
#       run.sh
#



                     [ Read 39 lines ]
^G Get Help  ^O Write Out ^W Where Is  ^K Cut Text  ^J Justify   ^C Cur Pos
^X Exit      ^R Read File ^\ Replace   ^U Uncut Text^T To Spell  ^_ Go To Line

```

# 答案
这个是使用`nano`进行编辑提交的页面，退出方法为：

`Ctrl + X`然后输入`y`再然后回车，就可以退出了

如果你想把默认编辑器换成别的, 在GIT配置中设置 `core.editor`: 

```sh
git config --global core.editor "vim"
```

# stackoverflow
## just for git
If you want to set the editor only for Git, do either (**you don’t need both**):

Set core.editor in your Git config: 
```sh
git config --global core.editor "vim"
```
Set the GIT_EDITOR environment variable: 
```sh
export GIT_EDITOR=vim
```
## set for all
If you want to set the editor for Git and also other programs, set the standardized VISUAL and EDITOR environment variables*:
```sh
export VISUAL=vim
export EDITOR="$VISUAL"
```
* Setting both is not necessarily needed, but some programs may not use the more-correct VISUAL. See VISUAL vs. EDITOR.

For Sublime Text: Add this to the .gitconfig. The --wait is important. (it allows to type text in sublime and will wait for save/close event.
```
[core]
    editor = 'subl' --wait
```
'subl' can be replaced by the full path of the executable but is usually available when correctly installed.


# 参考文献

- [Ubuntu git提交出现这个界面怎么退出](https://segmentfault.com/q/1010000002490510)

- [How do I make Git use the editor of my choice for commits?](https://stackoverflow.com/questions/2596805/how-do-i-make-git-use-the-editor-of-my-choice-for-commits)
