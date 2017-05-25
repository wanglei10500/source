---
title: git操作
tags:
 - git
categories: 经验分享
---
### git远程分支重命名
在git中重命名远程分支，步骤为先删除远程分支，然后重命名本地分支，再重新提交一个远程分支。
示例把devel分支重命名为develop分支：

```
$ git branch -av 列出本地分支和远程分支 显示hash值
```

删除远程分支

```
$ git push --delete origin devel
```

重命名本地分支

```
$ git branch -m devel develop
```

推送本地分支

```
$ git push origin develop
```
### 同步远程分支到本地
```
$ git fetch origin
```

### 创建本地分支 跟踪远程分支
```
$ git branch -av
$ git checkout -b V2.5.0 origin/V2.5.0
```

### 手动合并代码 解决冲突
```
Step 1. Update the repo and checkout the branch we are going to merge

git fetch origin
git checkout -b dev-v2.5.0-layne origin/dev-v2.5.0-layne
Step 2. Merge the branch and push the changes to GitLab

git checkout dev-v2.5.0
git merge --no-ff dev-v2.5.0-layne
git push origin dev-v2.5.0
```

### 删除本地分支
```
git branch -D
```
