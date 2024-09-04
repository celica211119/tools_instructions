# 0.安装配置
## 查看git版本
```
git -v
```
## 设置git信息
```
git config --global user.name <YourName> 设置用户名
git config --global user.email <YourEmail> 设置邮箱
git config --global --list 查看配置信息

--local：当前仓库生效（缺省）
--global：所有仓库生效
--system：所有用户生效
包含空格时需要使用双引号
```

# 1.初始化仓库
## 当前目录初始化为仓库
```
git init
```
## 在当前目录下创建并初始化仓库
```
git init <YourRepoName>
```
## 在当前目录拉取并初始化仓库
```
git clone <YourRepoUrl>
```

# 2.本地提交
```
git add 工作区文件同步到暂存区
git rm 删除工作区和暂存区的文件
git rm --cached 删除暂存区的文件
git ls-files 查看暂存区内容
git commit 提交暂存区所有修改到本地仓库
git commit -m <CommitMessage> 提交暂存区所有修改到本地仓库并添加提交信息
git status 查看本地仓库当前状态
git log 查看提交记录
git log --oneline 仅显示id和提交信息

add：可使用通配符、文件夹作为参数
commit：仅提交暂存区，不检查工作区内容
```

# 3.版本回滚
```
git reset --soft <CommitHash> 回退至指定版本
git reflog 查看版本历史

--soft：工作区和暂存区保留当前状态
--hard：工作区和暂存区回退至指定版本
--mixed：工作区保留当前状态，缓存区回退至指定版本（缺省）
```

# 4.文件内容比对
```
git diff 比对工作区与缓存区的差异
git diff HEAD 比对工作区与当前版本库的差异
git diff --cached 比对缓存区与当前版本库的差异
git diff <CommitHash1> <CommitHash2> 比对两个版本的差异
git diff <BranchName1> <BranchName2> 比对两个分支的差异
git diff <CommitHash1> <CommitHash2> <FileName> 比对两个版本的文件
```

# 5.远程推送与拉取
```
git remote add <RepoAlias> <YourRepoUrl> 添加远程仓库地址
git push -u <RepoAlias> <BranchName> 推送本地仓库修改到远程的同名分支
git push -u <RepoAlias> <OriginName>:<LocalName> 推送本地仓库修改到远程仓库的指定分支
git pull 拉取并合并远程仓库名为origin的main分支
git pull -u <RepoAlias> <OriginName>:<LocalName> 拉取远程仓库的更新并合并指定分支
git fetch <RepoAlias> 拉取远程仓库的更新
```
