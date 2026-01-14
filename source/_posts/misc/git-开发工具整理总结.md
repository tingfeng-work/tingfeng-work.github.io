---
title: git 开发工具整理总结
date: 2026-1-14 14:20
categories:
  - 杂记
tags:
  - git
  - 总结
toc: true
toc_number: true

---

> 本文是在学习过程中，零星对 git 的学习的总结与汇总。

## 什么是 Git

git 是基于 c 语言的分布式版本控制系统，与之对应的是集中式版本控制系统。这两者的不同：

* 集中式版本控制系统：版本库存放在中央服务器，必须联网才能从中央服务器拉取最新版本，推送新版本到中央服务器。
* 分布式版本控制系统：没有中央服务器，每个人的电脑都有一个完整的版本库。多人协作的实现是通过将各自修改的内容推送给对方。实际应用中，通常用一台电脑充当“中央服务器”，这个服务器只用来交换大家的修改信息，不存放版本库。

git 的优势不单单是不必联网，还有强大的分支管理

--------------

## 版本库

版本库又名仓库（Repository），可以理解为一个目录，这个目录中的所有文件受 git 管理，每个文件的增删改，git 都能跟踪，以便根据需求还原。

创建版本库非常简单，在需要交给 git 管理的目录下，执行 `git init` 命令即可。

> 需要注意的是，目录名尽量不要包含中文。

所有的版本控制系统，其实都只是跟踪文本文件的改动，例如 txt 文件、程序代码等等，不能监控图片、视频这种二进制文件（只能监控大小的改变，内容的改变不能监控）。

### 添加文件到版本库

在 git 管理的目录下，通过 `git add <filename>` 添加文件到版本库，再通过 `git commit -m "msg"` 命令把文件提交到仓库。`git status` 命令可以查看仓库当前状态，会提示修改过的文件，以及准备提交的文件（add 的文件）。`git diff` 命令可以对比文件哪些地方变化了。

注意添加和提交是两个操作。

### 小结（目前最常用命令）

* `git init`：将当前目录交给 git 管理
* `git add <filename>`：将指定文件添加到版本库中
* `git commit -m "msg"`：提交本次修改，msg 是对本次提交的说明
* `git status`：查看仓库当前状态，哪些文件被修改了，哪些文件添加到版本库中但是还没有提交
* `git diff`：查看文件修改内容

----------------

## 时光穿梭

### 版本回退

同一仓库，经过多次提交后，难免记不清哪个版本修改了什么内容，所以在进行版本回退前，可以通过 `git log` 命令查看历史记录，可以看到包含了每次提交的 commit id：364... 以及谁提交的，以及日期和每次提交的说明。

![1](D:/develop/tingfeng-work.github.io/source/_posts/misc/assets/1.png)

此外，我们还需要当前版本的信息，head 指向的就是当前版本，然后就可以定位需要回退的版本，通过 `git reset` 命令进行回退

```bash
git reset --hard HEAD^
```

^ 表示上个版本，多个版本可用 ~， HEAD~100 表示回退 100  个版本。

--hard 参数表示回退上个版本的已提交状态，--soft 表示回退到上个版本的未提交状态，--mixed 回到上个版本已添加但是为提交的状态。

也可以通过 `git reset --hard f35ed` 回退到具体版本，或者穿梭到未来的版本，注意这里 commit id 不用写全，只要让 git 能够找到唯一的版本即可。`git reflog` 命令显示 git 记录的每一次命令，通过这个可用查看版本号。

> git 的版本回退速度非常快，因为 git 内部有个指向当前版本的 HEAD 指针，回退版本时，仅仅是将 HEAD 指向指定版本

#### 小结

* `git reset --hard HEAD^/指定版本号`：回退到当前版本的上个版本或回退到指定版本号的版本
* `git log` 与 `git reflog` ：命令查看历史记录，来找到需要回退的版本或版本号

### 工作区与暂存区

工作区，就是电脑中实际操作的目录。

版本库，是工作区中的隐藏目录 .git，其中存储 git 进行版本控制的很多信息，最重要的就是暂存区（stage）和 git 自动创建的 master 分支，以及指向 master 分支的 HEAD 指针。

![2](D:/develop/tingfeng-work.github.io/source/_posts/misc/assets/2.png)

* `git add` 就是将**文件修改**添加到暂存区
* `git commit` 就是将暂存区的所有内容添加到当前分支

> 可以理解为：需要提交的文件修改通通放到暂存区，然后，一次性提交暂存区的所有修改。

### 管理修改

git 管理的是对文件的修改而非文件，怎么理解？例如，在文件中新增一个 'a' 字符，然后 `git add` 后又将字符改为了 'b'，暂存区中只知道你将新增了一个字符 'a'，而不知道它改为了 'b'，这是一个很好的证明 git 管理的是修改而非文件本身。

### 撤销修改

撤销修改有三种情形：

* 只是对工作区进行了修改，还没有 add 与 commit：直接通过 `git checkout --filename` 撤销修改
* 对工作区的修改，添加到了暂存区：通过 `git reset HEAD <file>` 命令回到上一个版本，然后通过  `git checkout --filename` 撤销修改。 

* 对工作区的修改不仅添加到了缓存区，还提交到了分支当中：`git reset HEAD^` 进行版本回退。

> git checkout --filename：让文件回到最近一次 add 或 commit 时的状态

### 文件的删除

仓库中的文件的删除也是一次修改操作，git 会记录下来。如果想在 git 仓库中也实现文件的永久删除，可以用 `git rm filename` 命令，也可以用 `git add filename` 将这次修改提交到暂存区，然后再提交。

如果要撤销这次删除操作，只需要用到上一节介绍的 `git checkout --filename` 命令

> 这里需要注意的就是：没有被添加到仓库中的文件，git 是无法控制的

----------

## 远程仓库

以上的功能，集中版本控制系统也能做到。真正体现 git 优势的是远程仓库，简单来说，就是将本地仓库托管到服务器上每天 24 小时开机，也可以从服务器中拉取别人的仓库，还能将提交推送到服务器上。

GitHub 就是提供 Git 仓库托管服务的，你需要告诉远程仓库你的身份信息，谁都可以修改你的远程仓库。本地Git仓库和 GitHub 仓库之间的传输是通过 SSH 加密的，所以需要先设置 SSH 密钥。

### 添加远程库

在 github 上创建好远程仓库后，需要将它与本地仓库关联：

```bash
git remote add origin git@github.com:tingfeng-worK/project.git
```

如果没有设置 SSH 密钥这一步，本地推送就推送不到远程仓库，相当于没有登录。命令中的 origin 是默认的远程仓库名，将本地仓库的内容推送到远程仓库，用命令 `git push`，实际上是将当前分支 master 推送到远程库，第一次推送时需要加上 `-u` 参数，它表示 git 不但会把本地的 master 分支推送到远程库中新的 master 分支，还会将它们关联起来，以后推送或拉取时就可以简化命令。

```bash
git push -u origin master
```

### 从远程库克隆

上述添加远程库是将本地已有的仓库添加并关联到远程仓库，而将远程仓库拉取到本地用到的 git 命令是：

```bash
git clone git@github.com:tingfeng-worK/project.git # ssh 协议

git clone https://github.com/tingfeng-work/project.git # https 协议
```

> Git 支持多协议，可以使用 https，但是使用 ssh 最快

------------

## 分支管理

Git 的分支管理是它的优势之一，无论创建、切换和删除分支，Git在1秒钟之内就能完成！无论你的版本库是1个文件还是1万个文件。

分支管理就好比在一个项目中，你与队友并行在不同的分支上开发不同的功能，在这个分支上你想提交就提交，不会影响主分支，在功能完成后，将分支合并到主分支上，项目就同时具备了你俩开发的功能。

### 分支的创建与合并

每次提交，git 都会 将它们串联为一条时间线，这个时间线就是一个分支，目前为止用到的都是 git 默认为我们常见的主分支 master，HEAD 严格来说不是指向提交，而是指向 master，而 master 指向每一次提交，如图所示：

![3](D:/develop/tingfeng-work.github.io/source/_posts/misc/assets/3.png)

每次提交，master 分支都会向前移动一次，HEAD 指针也随之移动。

当我们创建新的分支 dev 时，相当于创建了一个新的时间线，Git 会创建一个 dev 指针，指向 master 相同的提交，再把 HEAD 指向 dev，就表示当前分支在 dev 上：

![4](D:/develop/tingfeng-work.github.io/source/_posts/misc/assets/4.png)

这也解释了为什么 git 新建分支很快，它只是新建了一个指针，同时更改了 HEAD 的指向。现在对工作区的修改提交之后，就是 dev 指针移动，而 master 指针不变了：

![5](D:/develop/tingfeng-work.github.io/source/_posts/misc/assets/5.png)

在`dev`上的工作完成后，就可以把`dev`合并到`master`上。这种单时间线的合并最简单，就是直接把`master`指向`dev`的当前提交，就完成了合并。相关指令：

* `git branch`：查看有哪些分支
* `git branch <name>` ：新建分支
* `git checkout <name>` 或 `git switch <name>`：切换分支
* `git checkout -b <name>` 或 `git switch -c <name>`：创建并切换到新建的分支
* `git merger <name>`：合并某分支到当前分支
* `git branch -d <name>`：删除分支

### 冲突解决

上面描述最简单的单时间线的分支合并，考虑合并两个时间线的分支，这种情况下，git 无法执行快速合并，只能试着把各自的修改合并起来：

![6](D:/develop/tingfeng-work.github.io/source/_posts/misc/assets/6.png)

但是，这种合并可能会有冲突，假设这两次提交针对文件的同一地方进行了修改，合并分支时就会产生冲突，这也很好理解，git 不知道到底要怎么修改，所以必须手动解决冲突后再提交，也就是明确告诉 git 要怎么改。手动解决后，再合并就会变为：

![7](D:/develop/tingfeng-work.github.io/source/_posts/misc/assets/7.png)

可以看到这种合并会创建一次新的提交，这也很好理解，因为这是对文件的一次修改。

* `git log --graph --pretty=oneline --abbrev-commit`：可以看到分支合并情况。

### 多人协作

* 从远程仓库拉取分支进行本地修改
* 修改后从本地推送分支，如果推送失败，说明远程仓库中已经有人对操作 1 中拉取的分支进行了修改，并提交到远程仓库了，这时需要需要进行合并。
* 如果合并失败，就在本地解决冲突，然后再提交。

这个过程会用到的指令：

* `git remote -v`：查看远程仓库信息

* `git pull`：拉取并合并远程基于当前分支的提交，这个命令执行的前提是远程仓库上的分支与当前分支建立了联系。
* `git branch --set-upstream-to <branch-name> origin/<branch-name>`：将远程仓库中的分支与本地分支建立联系
* `git push origin <branch-name>`：从本地推送分支到远程仓库
* `git checkout -b branch-name origin/branch-name`

### Rebase

将本地未 push 的分叉提交历史整理为一条分支（一条直线）

--------------------

## 标签

标签就相当于一个有名字的不会动的指针，指向某一个 commit，方便进行版本控制。如果没有标签，想要跳转到指定的版本，需要通过 git log 找 commit 的 id，而且 id 还很长，所以标签就起到了快速找到指定版本的作用。

### 创建标签

切换到需要打标签的提交，通过 `git tag <name>`，就实现打标签了，也可以指定 commit id 的方式打标签，同时指定 `git tag -a <name> -m "msg"` 参数可以对创建标签进行说明。`git tag` 查看所有标签，`git show <tagname>`查看指定标签。

> 注意：标签总是和某个commit挂钩。如果这个commit既出现在master分支，又出现在dev分支，那么在这两个分支上都可以看到这个标签。

### 操作标签

- 命令`git push origin <tagname>`可以推送一个本地标签；
- 命令`git push origin --tags`可以推送全部未推送过的本地标签；
- 命令`git tag -d <tagname>`可以删除一个本地标签；
- 命令`git push origin :refs/tags/<tagname>`可以删除一个远程标签。

----------

## GitHub 开源项目

在 GitHub 上，利用 Git 强大的克隆和分支功能，可以实现自由参与各种开源项目了。

对于开源项目，先 fork 到自己的远程仓库，再从自己的远程仓库中克隆到本地仓库。对本地仓库的修改，再提交到自己的远程仓库，如果希望自己的修改被项目方接受，需要在 GitHub 上发起 pull request，最终取决于对方是否接受。

------------

## 学习 Git 的网站

https://learngitbranching.js.org/?locale=zh_CN



