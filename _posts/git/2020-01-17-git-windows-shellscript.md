---
title: "Windows下git修改文件权限"
tags: [linux, git, windows, shell]
---

**原创I爱吃鱼 最后发布于2017-08-15 12:02:06**

我们在Windows下做开发时会写在linux上运行的脚本，这个脚本默认是没有执行权限的，可以使用git修改文件权限后提交。其实可以直接在linux上直接修改文件权限，但是如果代码部署在很多服务器上的话就需要手动修改每一个服务器上的文件权限，并且还会使服务器的版本和远程库版本不一致。

1. 进入文件所在目录，打开Git Bash
2. 查看文件权限 `git ls-tree HEAD`

    ```sh
    $ git ls-tree HEAD
    100644 blob ff398380c2c8a1dad17ce00a307223fb57fcba24    test.sh
    ```

    `100755` 表示文件有执行权限

3. 修改文件的权限

    ```sh
    $ git update-index --chmod=+x test.sh
    $ git status
    On branch master
    Changes to be committed:
    (use "git reset HEAD <file>..." to unstage)

            modified:   test.sh
    ```

4. 提交修改 git commit -m “revise permission access”，并推送

    ```sh
    $ git add .
    $ git commit -m "add runnable right"
    [master de7f2b6] add runnable right
    1 file changed, 0 insertions(+), 0 deletions(-)
    mode change 100644 => 100755 test.sh
    ```

5. 再次查看文件权限，发现权限修改成功
    ```sh
    $ git ls-tree HEAD
    100755 blob ff398380c2c8a1dad17ce00a307223fb57fcba24    test.sh
    ```

6. linux上拉取修改，文件有执行权限

# References

- [原文 - Windows下git修改文件权限](https://blog.csdn.net/u012061554/article/details/77186824)
