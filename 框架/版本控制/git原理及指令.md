# git原理及指令



## 基本用法

![image-20200116155557949](https://tva1.sinaimg.cn/large/006tNbRwgy1gayghy7e39j30h308jaae.jpg)

上面的四条命令在工作目录、暂存目录(也叫做索引)和仓库之间复制文件。

- `git add files` 把当前文件放入暂存区域。files传递'.'则是全部文件
- `git commit` 给暂存区域生成快照并提交。
- `git reset -- files` 用来撤销最后一次`git add *files*`，你也可以用`git reset` 撤销所有暂存区域文件。
- `git checkout -- *files*` 把文件从暂存区域复制到工作目录，用来丢弃本地修改。

你可以用 `git reset -p`, `git checkout -p`, or `git add -p`进入交互模式。

也可以跳过暂存区域直接从仓库取出文件或者直接提交代码。



![image-20200116155842408](https://tva1.sinaimg.cn/large/006tNbRwgy1gaygksnuq4j30mo081aag.jpg)

- `git commit -a `相当于运行 `git add` 把所有当前目录下的文件加入暂存区域再运行。`git commit`.
- `git commit files` 进行一次包含最后一次提交加上工作目录中文件快照的提交。并且文件被添加到暂存区域。
- `git checkout HEAD -- files` 回滚到复制最后一次提交。





## 约定

后文中以下面的形式使用图片。

![image-20200116160323203](https://tva1.sinaimg.cn/large/006tNbRwgy1gaygpnrqjbj30m40ar75b.jpg)

绿色的5位字符表示提交的ID，分别指向父节点。分支用橘色显示，分别指向特定的提交。当前分支由附在其上的*HEAD*标识。 

这张图片里显示最后5次提交，*ed489*是最新提交。 *master*分支指向此次提交，另一个*maint*分支指向祖父提交节点。







## 分支

- 查看本地分支： `git branch`
- 查看本地及远程分支： `git branch -a`
- 创建分支，比如创建test分支： `git branch test`
- 创建并且切换到指定分支： `git checkout -b test`
- 切换到指定分支，比如test分支： `git checkout test`
- 把test分支提交到远程仓库 `git push origin test`
- 删除指定分支，比如删除test分支： `git branch -d test`
- 合并分支，比如把当前分支合并test分支上： `git merge `
- 远程新建了一个分支，本地没有该分支，使用：`git checkout --track origin/branch_name`进行关联，这时本地会新建一个分支名叫 branch_name ，会自动跟踪远程的同名分支 branch_name。
- 如果本地新建了一个分支 branch_name，但是在远程没有。这时候 push 和 pull 指令就无法确定该跟踪谁，一般来说我们都会使其跟踪远程同名分支，所以可以利用 git push --set-upstream origin branch_name ，这样就可以自动在远程创建一个 branch_name 分支，然后本地分支会 track 该分支。后面再对该分支使用 push 和 pull 就自动同步。`git push --set-upstream origin branch_name`
- 把分支推送到远程仓库：`git push origin test`  然后远程仓库及本地就会有名为test的分支了。





## 文件的修改

- 改乱了工作区某个文件的内容，想直接丢弃工作区的修改时（场景1），用命令`git checkout -- file`
- 不但改乱了工作区某个文件的内容，还添加到了暂存区时，想丢弃修改，分两步，第一步用命令`git reset HEAD file`，就回到了场景1，第二步按场景1操作。



## 暂存文件

- `git stash` 可用来暂存当前正在进行的工作， 比如想pull 最新代码， 又不想加新commit， 或者另外一种情况，为了fix 一个紧急的bug,  先stash, 使返回到自己上一个commit, 改完bug之后再stash pop, 继续原来的工作。
- `git stash pop` 恢复改动。如果你要恢复的是最近的一次改动，git stash pop即可，我用这个用的最多。如果有多次stash操作，那就通过git stash list查看stash列表，从中选择你想要pop的stash，运行命令git stash pop stash@{id}或者 git stash apply stash@{id}即可。这方面网上的资料挺多的。



## 忽略文件

1、通过定义在项目目录下定义.gitignore文件，把需要忽略的内容写在路面，例如：

```
.idea
target
```
就是忽略掉.idea及target文件夹内的文件

2、上述方法如果失效了，可能是文件原本就被跟踪了，解决方法是删除缓存，参考：https://www.cnblogs.com/youyoui/p/8337147.html





## 代码回滚

### 远程代码回滚

假如有问题的代码提交到了远程，可以使用下面方式强制回滚

1、`git reset --hard commit_id`     退到/进到 指定commit的sha码
2、`git push origin huangy_0829  --force`  强制推送到远程分支



### 本地代码回滚

假如你想丢弃你在本地的所有改动与提交，可以到服务器上获取最新的版本历史，并将你本地主分支指向它：
`git fetch origin`
`git reset --hard origin/master`



### 利用缓冲区回滚本地修改

把文件从暂存区域复制到工作目录，用来丢弃本地修改。

`git checkout -- files`









## 标签

为软件发布创建标签是推荐的。这个概念早已存在，在 SVN 中也有。你可以执行如下命令创建一个叫做 *1.0.0* 的标签：

`git tag 1.0.0 1b2e1d63ff`

*1b2e1d63ff* 是你想要标记的提交 ID 的前 10 位字符。可以使用下列命令获取提交 ID：

`git log`





## 查看代码区别

可以利用diff查看代码的区别

![image-20200116160450235](https://tva1.sinaimg.cn/large/006tNbRwgy1gaygr63z0jj30k10bcmy2.jpg)

- `git diff maint`表示当前分支的代码和maint分支的代码进行比较，有哪些区别





## Commit

下面讲讲Commit原理：

提交时，git用暂存区域的文件创建一个新的提交，并把此时的节点设为父节点。然后把当前分支指向新的提交节点。下图中，当前分支是*master*。 在运行命令之前，*master*指向*ed489*，提交后，*master*指向新的节点*f0cec*并以*ed489*作为父节点。

![image-20200116161047039](https://tva1.sinaimg.cn/large/006tNbRwgy1gaygxcxq9oj30je0axaap.jpg)

即便当前分支是某次提交的祖父节点，git会同样操作。下图中，在*master*分支的祖父节点*maint*分支进行一次提交，生成了*1800b*。 这样，*maint*分支就不再是*master*分支的祖父节点。此时，合并merge是必须的。

![image-20200116161406908](https://tva1.sinaimg.cn/large/006tNbRwgy1gayh0tn115j30hf0bdwf4.jpg)

如果想更改一次提交，使用 `git commit --amend`。git会使用与当前提交相同的父节点进行一次新提交，旧的提交会被取消。



## Checkout

checkout命令用于从历史提交（或者暂存区域）中拷贝文件到工作目录，也可用于切换分支。

当给定某个文件名（或者打开-p选项，或者文件名和-p选项同时打开）时，git会从指定的提交中拷贝文件到暂存区域和工作目录。比如，`git checkout HEAD~ foo.c`会将提交节点*HEAD~*(即当前提交节点的父节点)中的`foo.c`复制到工作目录并且加到暂存区域中。（如果命令中没有指定提交节点，则会从暂存区域中拷贝内容。）注意当前分支不会发生变化。

![image-20200116162053296](https://tva1.sinaimg.cn/large/006tNbRwgy1gayh7v9vzsj30gv0b7t99.jpg)

当不指定文件名，而是给出一个（本地）分支时，那么*HEAD*标识会移动到那个分支（也就是说，我们“切换”到那个分支了），然后暂存区域和工作目录中的内容会和*HEAD*对应的提交节点一致。新提交节点（下图中的a47c3）中的所有文件都会被复制（到暂存区域和工作目录中）；只存在于老的提交节点（ed489）中的文件会被删除；不属于上述两者的文件会被忽略，不受影响。

![image-20200116162255876](https://tva1.sinaimg.cn/large/006tNbRwgy1gayha02vi5j30h10aqt9d.jpg)

如果既没有指定文件名，也没有指定分支名，而是一个标签、远程分支、SHA-1值或者是像*master~3*类似的东西，就得到一个匿名分支，称作*detached HEAD*（被分离的*HEAD*标识）。这样可以很方便地在历史版本之间互相切换。比如说你想要编译1.6.6.1版本的git，你可以运行`git checkout v1.6.6.1`（这是一个标签，而非分支名），编译，安装，然后切换回另一个分支，比如说`git checkout master`。然而，当提交操作涉及到“分离的HEAD”时，其行为会略有不同，详情见在下面。

![image-20200116162331834](https://tva1.sinaimg.cn/large/006tNbRwgy1gayhamia4kj30hg0b5aao.jpg)



**HEAD标识处于分离状态时的提交操作**

当*HEAD*处于分离状态（不依附于任一分支）时，提交操作可以正常进行，但是不会更新任何已命名的分支。(你可以认为这是在更新一个匿名分支。)

![image-20200116162530607](https://tva1.sinaimg.cn/large/006tNbRwgy1gayhcoockpj30hn0awt9b.jpg)

一旦此后你切换到别的分支，比如说*master*，那么这个提交节点（可能）再也不会被引用到，然后就会被丢弃掉了。注意这个命令之后就不会有东西引用*2eecb*。

![image-20200116162602220](https://tva1.sinaimg.cn/large/006tNbRwgy1gayhd87v87j30ho0ay0tf.jpg)

但是，如果你想保存这个状态，可以用命令`git checkout -b *name*`来创建一个新的分支。

![image-20200116162631922](https://tva1.sinaimg.cn/large/006tNbRwgy1gayhdqxj4hj30gj0bi0tc.jpg)



## Reset

reset命令把当前分支指向另一个位置，并且有选择的变动工作目录和索引。也用来在从历史仓库中复制文件到索引，而不动工作目录。

如果不给选项，那么当前分支指向到那个提交。如果用`--hard`选项，那么工作目录也更新，如果用`--soft`选项，那么都不变。

![image-20200116162752919](https://tva1.sinaimg.cn/large/006tNbRwgy1gayhf5k6e9j30gy0b2gmb.jpg)

如果没有给出提交点的版本号，那么默认用*HEAD*。这样，分支指向不变，但是索引会回滚到最后一次提交，如果用`--hard`选项，工作目录也同样。

![image-20200116162859184](https://tva1.sinaimg.cn/large/006tNbRwgy1gayhgar9lnj30hv0az3z3.jpg)



如果给了文件名(或者 `-p`选项), 那么工作效果和带文件名的[checkout](http://marklodato.github.io/visual-git-guide/index-zh-cn.html#checkout)差不多，除了索引被更新。

![image-20200116162928647](https://tva1.sinaimg.cn/large/006tNbRwgy1gayhgtgvmzj30h00ba0t8.jpg)





## Merge

merge 命令把不同分支合并起来。合并前，索引必须和当前提交相同。如果另一个分支是当前提交的祖父节点，那么合并命令将什么也不做。 另一种情况是如果当前提交是另一个分支的祖父节点，就导致*fast-forward*合并（指向只是简单的移动，并生成一个新的提交）。

![image-20200116163024588](https://tva1.sinaimg.cn/large/006tNbRwgy1gayhhs4iyaj30il0br3z7.jpg)

否则就是一次真正的合并。默认把当前提交(*ed489* 如下所示)和另一个提交(*33104*)以及他们的共同祖父节点(*b325c*)进行一次[三方合并](http://en.wikipedia.org/wiki/Three-way_merge)。结果是先保存当前目录和索引，然后和父节点*33104*一起做一次新提交。

![image-20200116163313601](https://tva1.sinaimg.cn/large/006tNbRwgy1gayhkph5atj30kx0bht9u.jpg)









## 参考

[git简明指南](https://www.runoob.com/manual/git-guide/)

[图解git](http://marklodato.github.io/visual-git-guide/index-zh-cn.html)

