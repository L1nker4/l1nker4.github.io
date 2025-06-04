# Git命令记录




### 1. 基本操作



#### 基本命令

```
mkdir notes
cd notes

# 初始化一个git仓库 
git init

echo &#34;my notes&#34; &gt; readme.md

# 查看状态
git status

# 查看修改内容
git diff readme.md

# 添加文件到仓库
git add readme.md

# 提交更改
git commit -m &#34;add readme&#34;

# 查看提交日志
git log

# 回退到上一个版本
git reset --hard HEAD^

# HEAD: 当前版本
# HEAD^: 上一个版本
# HEAD~100: 前一百个版本
# 58a65b: 指定版本
```



#### 工作区与暂存区

```
|-------|            |-------|               |-------|
| 工作区 | -- add --&gt; | 暂存区 | -- commit --&gt; | 主分支 |
|-------|            |-------|               |-------|


# 查看工作区与暂存区文件的差异
git diff readme.md

# 查看工作区和版本库最新版本的差异
git diff HEAD -- readme.md

# 放弃工作区的修改(未执行 git add 命令)
git checkout -- readme.md

# 撤销暂存区（已经执行了 git add 命令）
git reset HEAD readme.md
```



#### 删除文件

```
# 删除文件
rm test.txt
git add test.txt
git commit -m &#34;rm test.txt&#34;

# 直接从版本库中删除文件
git rm test.txt
git commit -m &#34;git rm test.txt&#34;

# 还原文件
git reset --hard CommitID
```



### 2. 远程仓库



#### Github仓库

```
# 在本地创建一个 gitlearn 目录，并进入
mkdir gitlearn
cd gitlearn

# 使用命令 git init 初始化一个库
git init

# 添加远程库 origin
git remote add origin git@github.com:l1nker4/gitlearn.git

# 添加 readme.md 文件
echo &#34;git learn is very easy!&#34; &gt; readme.md

# 添加文件并提交到仓库
git add readme.md
git commit -m &#34;add readme&#34;

# 执行向远程仓库 origin master 分支推送更新（-u 参数用于关联远程仓库）
git push -u origin master
```





#### 更新远程仓库

```
# 查看当前分支的关联远程仓库
git branch -vv

# 查看远程仓库的 URL
git remote -v

# 重新设置远程仓库的 URL
git remote set-url origin git@github.com:l1nker4/gitlearn.git

# 将当前修改更新到远程 master 分支
git push origin master
```





### 3.分支管理



#### 创建与合并分支

```
# 查看当前分支
git branch 

# 基于当前分支创建新分支
git branch dev

# 切换分支
git checkout dev

# 创建并切换分支
git checkout -b dev

# 切换到 master 分支
git checkout master

# 合并指定分支到当前分支(合并 dev 分支到 master 分支)
git merge dev 

# 删除分支
git branch -d dev
```





#### 冲突管理

在 master 分支上修改 readme.md，新增一行内容。提交修改。 在 dev 分支上修改 readme.md，也新增一行内容。提交修改。

然后切换到 master 分支，将 dev 分支的内容合并。

```
git merge dev
```



会有冲突提醒。打开 readme.md:

```
&lt;&lt;&lt;&lt;&lt;&lt;&lt; HEAD
Creating a new branch is quick &amp; simple.
=======
Creating a new branch is quick AND simple.
&gt;&gt;&gt;&gt;&gt;&gt;&gt; feature1
```





然后修改内容。再次提交。

```
# 查看分支合并情况
git log --graph  --pretty=oneline --abbrev-commit

# 工作现场“储藏”
git stash

# 查看储藏的你内容
git stash list

# 恢复储藏的内容，并删除
git stash pop

# 也可以指定恢复储藏的内容
git stash apply stash@{0}

# 删除储藏的内容
git stash drop
```



#### 多人协作

```
# 查看远程库的信息
git remote
git remote -v

# 推送分支(推送本地分支到远程分支 master 上)
git push origin master 

# 创建本地分支
git checkout -b dev origin/dev

# 设置本地 dev 分支与远程 origin/dev 分支的链接
git branch --set-upstream dev origin/dev

# 拉取分支
git pull
```





### 4.命令表

```
git add .                                     # 增加当前子目录下所有更改过的文件至index
git add FILENAME                              # 添加一个文件到git index

git blame FILENAME                            # 显示文件每一行的修改记录(提交ID, 修改人员)

git branch                                    # 显示本地分支
git branch -a                                 # 显示所有分支
git branch -r                                 # 查看远程所有分支
git branch -vv                                # 查看本地和远程的关联关系
git branch branch_0.1 master                  # 从主分支master创建branch_0.1分支
git branch --contains 50089                   # 显示包含提交50089的分支
git branch -D hotfixes/BJVEP933               # 强制删除分支hotfixes/BJVEP933
git branch -d hotfixes/BJVEP933               # 删除分支hotfixes/BJVEP933
git branch -D master develop                  # 删除本地库develop
git branch -m branch_0.1 branch_1.0           # 将branch_0.1重命名为branch_1.0
git branch -m master master_copy              # 本地分支改名
git branch --merged                           # 显示所有已合并到当前分支的分支
git branch --no-merged                        # 显示所有未合并到当前分支的分支
git branch -r -d branch_remote_name           # 删除远程branch

git checkout -- README                        # 检出head版本的README文件（可用于修改错误回退）
git checkout -b dev                           # 建立一个新的本地分支dev
git checkout -b devel origin/develop          # 从远程分支develop创建新本地分支devel并检出
git checkout -b master master_copy            # 上面的完整版
git checkout -b master_copy                   # 从当前分支创建新分支master_copy并检出
git checkout branch_1.0/master                # 切换到branch_1.0/master分支
git checkout dev                              # 切换到本地dev分支
git checkout features/performance             # 检出已存在的features/performance分支
git checkout --track hotfixes/BJVEP933        # 检出远程分支hotfixes/BJVEP933并创建本地跟踪分支
git checkout --track origin/dev               # 切换到远程dev分支
git checkout v2.0                             # 检出版本v2.0
git cherry-pick ff44785404a8e                 # 合并提交ff44785404a8e的修改

git clone GIT_REPO_LINK                       # 从服务器上将代码给拉下来

git commit -am &#39;xxx&#39;                          # 将add和commit合为一步
git commit -m &#39;xxx&#39;                           # 提交
git commit -a -m                              # add/commit
git commit -a -v                              # 一般提交命令
git commit --amend -m &#39;xxx&#39;                   # 合并上一次提交（用于反复修改）
git commit -v                                 # 当你用－v参数的时候可以看commit的差异

git clean -df                                 # 清理临时文件

git config --global color.branch auto
git config --global color.diff auto
git config --global color.interactive auto
git config --global color.status auto
git config --global color.ui true             # git status等命令自动着色
git config --global user.email &#34;xxx@xxx.com&#34;  # 配置邮件
git config --global user.name &#34;xxx&#34;           # 配置用户名
git config --list                             # 看所有用户
git config --system                           # 配置所有用户公用配置

git diff                                      # 显示所有未添加至index的变更
git diff HEAD^                                # 比较与上一个版本的差异
git diff COMMITID1 COMMITID2 --name-only      # 显示两个节点之间的所有差异文件名
git diff --cached                             # 显示所有已添加index但还未commit的变更
git diff --staged                             # 同上
git diff HEAD -- ./lib                        # 比较与HEAD版本lib目录的差异
git diff origin/master..master                # 比较远程分支master上有本地分支master上没有的
git diff origin/master..master --stat         # 只显示差异的文件，不显示具体内容

git fetch                                     # 获取所有远程分支（不更新本地分支，另需merge）
git fetch --prune                             # 获取所有原创分支并清除服务器上已删掉的分支

git format-patch  CMTID                       # 生成所有的补丁(CMTID, HEAD]  
git am 0001-Fix1.patch                        # 按生成顺序0001,0002应用补丁即可

git fsck
git gc
git grep &#34;delete from&#34;                        # 文件中搜索文本“delete from”
git grep -e &#39;#define&#39; --and -e SORT_DIRENT
git init                                      # 初始化本地git仓库（创建新仓库）

git log                                       # 显示提交日志
git log -1                                    # 显示1行日志 -n为n行
git log -5
git log --stat                                # 显示提交日志及相关变动文件
git log v2.0                                  # 显示v2.0的日志

git ls-files                                  # 列出git index包含的文件
git ls-tree HEAD                              # 内部命令：显示某个git对象

git merge origin/dev                          # 将分支dev与当前分支进行合并
git merge origin/master                       # 合并远程master分支至当前分支

git mv README README2                         # 重命名文件README为README2

git pull                                      # 本地与服务器端同步
git pull origin master                        # 获取远程分支master并merge到当前分支
git push remote_repo branch_name              # 将本地分支推送到服务器上去。
git push --force                              # 推送会失败，可用强制推送

git push origin :branch_remote_name
git push origin :hotfixes/BJVEP933            # 删除远程仓库的hotfixes/BJVEP933分支
git push origin master                        # 将当前分支push到远程master分支
git push origin master:hb-dev                 # 将本地库与服务器上的库进行关联
git push --tags                               # 把所有tag推送到远程仓库

git rebase upstream/branch                    # 合并分支
git rebase -i 408700955                       # 合并多次提交为一次提交

git reflog                                    # 显示所有提交，包括孤立节点

git remote add upstream  URL                  # 增加远程定义（用于push/pull/fetch）
git remote show origin                        # 显示远程库origin里的资源
git remote show                               # 查看远程库

git reset --hard HEAD                         # 将当前版本重置为HEAD（
git revert dfb02e6e4                          # 撤销提交 dfb02e6e4
git rev-parse v2.0                            # 内部命令：显示某个ref对于的SHA1 HASH

git rm [file name]                            # 删除一个文件
git rm a.a                                    # 移除文件(从暂存区和工作区中删除)
git rm --cached a.a                           # 移除文件(只从暂存区中删除)
git rm -f a.a                                 # 强行移除修改后文件(从暂存区和工作区中删除)
git rm -r *                                   # 递归删除
git rm xxx                                    # 删除index中的文件
git rm FILENAME                               # 从git中删除指定文件

git show                                      # 等价于 git show HEAD
git show CMTID                                # 显示某个提交的详细内容
git show CMTID --name-only
git show HEAD                                 # 显示HEAD提交日志
git show HEAD@{5}
git show HEAD^                                # 显示HEAD的父（上一个版本）的提交日志 
git show HEAD~3
git show master@{yesterday}                   # 显示master分支昨天的状态
git show -s --pretty=raw CMTID
git show v2.0                                 # 显示v2.0的日志及详细内容
git show-branch                               # 图示当前分支历史
git show-branch --all                         # 图示所有分支历史

git stash                                     # 暂存当前修改，将所有至为HEAD状态
git stash apply stash@{0}                     # 应用第一次暂存
git stash list                                # 查看所有暂存
git stash pop                                 # 将文件从临时空间pop下来
git stash push                                # 将文件给push到一个临时空间中
git stash show -p stash@{0}                   # 参考第一次暂存
git status                                    # 查看当前版本状态（是否修改）

git tag                                       # 显示已存在的tag
git tag -a v2.0 -m &#39;xxx&#39;                      # 增加v2.0的tag

git whatchanged                               # 显示提交历史对应的文件修改

git update-index --assume-unchanged file      # 忽略修改，git status不会显示
git update-index --no-assume-unchanged file   # 添加被忽略文件，git status会显示
```



---

> Author:   
> URL: http://localhost:1313/posts/%E5%B7%A5%E5%85%B7/git-note/  

