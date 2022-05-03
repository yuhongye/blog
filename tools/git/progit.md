这是git笔记，已经使用git大概有5年了，这次是系统复习一下，只记录自己认为重要的和未掌握的部分。

# 0 起步

### 1 直接记录快照，而非差异比较

#### 1. 1 其他版本管理系统是记录差异

![其他版本管理系统](/Users/caoxiaoyong/Documents/blog/tools/git/images/deltas.png)

#### 1.2 git对全部文件制作一个快照并保存这个快照的索引

![git保存快照索引](/Users/caoxiaoyong/Documents/blog/tools/git/images/snapshots.png)

每当有改动时，git对全部文件制作一个快照并__保存这个快照的索引__。为了高效，如果某个文件没有更改，git不会重新存储该文件，而是保留一个指向之前文件的链接。git对待数据更像是一个快照流。

Todo: 当一个文件有更新时，两次保存的文件是否会使用压缩算法?

### 2 git一般只添加数据

 Git 的操作几乎只往 Git 数据库中增加数据。 很难让 Git 执行任何不可逆操作，或者让它以任何方式清除数据。

### 3 配置

git有三种级别的配置:

1. `/etc/gitconfig` : 为系统上每个用户及他们的仓库的通用配置，当使用`--system`选项时会从此文件读取变量。
2. `~/.gitconfig 或 ~/.config/git/config`：只针对当前用户，当使用`--global`选项时从此文件读取变量
3. `current repo/.git/config` ：只针对当前仓库

优先级从低到高: `系统级 << 用户级 << 仓库`

```git
1. 设置变量
git config --global user.name "John Doe"
git config --global user.email johndoe@example.com
git config user.name "Local_John"

2.读取变量
git config user.name // Local_john
git config --global user.name "John Doe"

3.展示配置
git config --list
```

基本配置

```git
[alias]
	co = checkout
	ci = commit
	st = status
	last = log -1 HEAD
[user]
	email = caoxyemail@163.com
	name = 喻红叶
[core]
	editor = vim

[pull]
	rebase = true
[filter "lfs"]
	clean = git-lfs clean -- %f
	smudge = git-lfs smudge -- %f
	process = git-lfs filter-process
	required = true
```

# 1 基础

### 基本命令

##### git clone 

`git clone url`会在本地创建一个同名的仓库，`git clone url new_name`会在本地创建一个名为`new_name`的仓库

##### git add: 添加内容到下一次提交中

这是一个多功能命令:

1. 开始跟踪新文件
2. 把已追踪的文件放到暂存区(下次commit提交的区域)
3. 合并时把有冲突的文件标记为已解决

##### .gitignore

git使用glob模式，不同于shell中的正则。

下面解释 `.gitignore` 中的知识点:

```git
# 1. 所有已 # 开头的行都是注释，会被忽略掉

# 2. 不追踪 .a 文件
*.a

# 3. !表示取反，虽然忽略 .a 文件，但是需要追踪 lib.a
!lib.a

# 4. 模式匹配以 / 开头防止递归: 仅忽略当前目录下的 TODO 目录，不忽略子目录的 TODO 目录
/TODO

# 5. 模式匹配以 / 结尾指定目录: 忽略 build/ 目录下的所有文件
build/

# 6. 仅忽略 doc/ 目录下的 txt 文件，不忽略其子目录的 txt 文件
doc/*.txt

# 7. 递归地 doc/ 目录下所有的 pdf 文件
doc/**/*.pdf
```

##### git diff

`git diff`展示的是未暂存的修改，如果想看已暂存未提交的修改使用命令: `git diff --staged`

##### git commit

`git commit -a `会把所有已追踪并且本次已修改的文件一起提交了，省了一次`git add`

##### git rm

1. 仅使用文件系统的`rm`工具，则相当于改了文件内容，仍未到暂存区
2.  `git rm`命令来记录此次移除文件的操作，下次提交时该文件就不再纳入版本管理了
3. 如果某个文件已经被追踪了，并且本次还被修改了，则需要使用 `-f`强制删除选项
4. `git rm`同时会把磁盘上的文件删除，如果仅仅只是想从git追踪中删除，但是在磁盘上行不删除使用`--cached`选项: `git rm --cached README`
5. `git rm`后面可以列出文件或者目录的名字，也可以使用`glob`模式:

```git
# 1. 删除所有的 .log 文件，注意 * 之前有反斜杠\，因为 git 有自己的文件模式扩展匹配方式
git rm log/\*.log

# 删除所有以 ~ 结尾的文件
git rm \*~
```

##### git mv

`git mv README.md README`相当于如下三条命令:

1. `mv README.md README`
2. `git rm README.md`
3. `git add README`

##### 取消暂存操作

假设当前状态如下：

```git
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	modified:   a
```

如果想取消暂存 a 的操作，根据上面提示可以使用如下命令：`git restore --staged a`。

在之前的版本中使用`git reset HEAD a`，这样 `reset`有了多个含义，因此还是使用`restore好一些`

##### 撤销对文件的修改

假设当前状态如下:

```git
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   a

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	b
```

注意`b`没有被git追踪。如果想撤销对`a`的修改，可以使用如下命令：`git restore a`. 不能对`b`使用`restore`操作，说白了`b`没有被git追踪，只能使用文件系统命令。

在之前的版本中使用`git checkout -- a`，这样`checkout`的语义就混淆了。



