---
title: "Git: 如何修复gerrit merge conflict状态"
tags: [git, merge, conflict, rebase]
list_number: n
---

A本地修改了某个文件File，B本地也修改了这个文件File，他们都先后 `git push` 到了 `gerrit` 上，这个时候 `reviewer` 无论先进谁的提交， `gerrit` 上另一笔提交的状态都会显示 `merge conflict` ，那如何更新这一笔 `change` ，而不是 `Abandon` 然后再提一个 `change` 呢，假设A的提交了先进了，要修改B的 `change` 。

# 1 先同步服务器代码

```sh
git remote update
```

# 2 rebase

```sh
git rebase remote_branch
```

会出现如下提示：

```
First, rewinding head to replay your work on top of it...
Applying: your_patch
Using index info to reconstruct a base tree...
M     your_modified_file
Falling back to patching base and 3-way merge...
Auto-merging your_modified_file
CONFLICT (content): Merge conflict in your_modified_file
Recorded preimage for 'your_modified_file'
Failed to merge in the changes.
Patch failed at 0001 your_patch
The copy of the patch that failed is found in:
   /your_prj_path/.git/rebase-apply/patch

When you have resolved this problem, run "git rebase --continue".
If you prefer to skip this patch, run "git rebase --skip" instead.
To check out the original branch and stop rebasing, run "git rebase --abort".
```

这个时候 `git log` 就能看到 A 的提交。

`git status` 能看到此时正在 `rebasing` ,要你修复冲突后再运行 `continue rebase` :

```sh
rebase in progress; onto 5ac5126
You are currently rebasing.
  (fix conflicts and then run "git rebase --continue")
  (use "git rebase --skip" to skip this patch)
  (use "git rebase --abort" to check out the original branch)

Unmerged paths:
  (use "git reset HEAD <file>..." to unstage)
  (use "git add <file>..." to mark resolution)

    both modified:      your_conflict_file

no changes added to commit (use "git add" and/or "git commit -a")
```

## 修改冲突文件
打开冲突文件，能发现A和B的提交都在里面，手动编辑这个文件让他保持A的提交内容，然后再加上B的修改。

## 重新添加修改后的文件

```sh
git add .
```

`add` 后会提示冲突修复了。

```sh
rebase in progress; onto 5ac5126
You are currently rebasing.
  (all conflicts fixed: run "git rebase --continue")

Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

    modified:   your_file
```

## 继续rebase

```sh
git rebase --continue
```

成功后会提示：

```sh
Applying: your_patch
Recorded resolution for 'your_modified_file'.
```

再 `git log` 方法B的提交在A的后面。

# 提交
push后gerrit这一笔提交status的merge conflict已经没有了，History里会显示：

```sh
Uploaded patch set 2.
OK, Done.
```

# References

- [原文 Git: 如何修复gerrit merge conflict状态](http://tjtech.me/how-to-fix-merge-conflict-on-gerrit.html)
