[TOC]

##### 创建仓库

```makefile
git init
```

##### 添加文件到仓库中

```makefile
# 将文件添加至暂存区
git add filename
# 将文件添加到本地仓库
git commit -m "comment"
# 将文件保存到远程仓库
git push
```

##### 查看当前分支的状态

```makefile
git status
```

##### 查看文件改动

```makefile
git diff filename
```

##### 查看历史记录

```makefile
git log
git log --pretty=oneline
```

##### 版本回退

```makefile
# 回退至上一个版本
git reset --hard HEAD^
# 回退至上上个版本
git reset --hard HEAD^^
# 回退到前100个版本
git reset --hard HEAD~100
# 通过版本号回退
git reset --hard 版本号
```

##### 查看版本号

```makefile
git reflog
```

##### git撤销文件修改

1. 手动进行操作，恢复文件
2. 直接回退版本
3. git checkout --filename 会将文件中进行了修改但是并没有添加到暂存区中的操作撤销

```makefile
git checkout --filename
```

##### git删除文件

- 直接 rm filename 删除文件，然后 git commit
- 直接回退版本

##### git恢复文件（只对在工作区删除，但还没有 commit 的文件有效）

```makefile
git checkout -- filename
```

##### 创建ssh密钥

```makefile
ssh-keygen -t rsa –C "email"
```

##### 把本地仓库与远程仓库关联起来

首先在github 上创建一个空的仓库（假设在本地已经有了一个仓库）

```makefile
# 指明远程仓库
git remote add origin 远程仓库地址
# 推送 main 分支到远程仓库
git push -u origin main
# 上述操作完成之后，不再需要 -u 参数
git push origin main
```

##### 克隆远程仓库

```makefile
git clone 远程仓库地址
```

##### 创建分支与合并分支

```makefile
# 创建分支并切换
git checkout -b branchname
# 将指定分支合并到当前分支
git merge branchname
# 禁用 fast-forward 模式
git merge –no-ff -m "comment" branchname
```

##### 删除分支

```
git branch -d branchname
```

##### 将当前的工作线程隐藏起来，即使当前没有提交，也可以进行分支切换

```makefile
git stash
git stash list
git stash apply + git stash drop
git stash pop
```

##### 将远程的分支到本地

```makefile
git checkout -b origin/branchname
```

##### 拉取远程仓库分支

```makefile
git pull
```

##### 建立本地分支与远程分支的链接

```makefile
git branch --set-upstream local_branchname origin/remote_branchname
```

