---
layout: post
category : GIT
title: '将SVN仓库迁移到GIT仓库'
tagline: ""
tags : [GIT]
---
* auto-gen TOC:
{:toc}

# 将SVN仓库迁移到GIT仓库
* 近期在部门内推广使用git，新项目已经全部使用git来作为代码版本库工具
* 这周打算将旧项目的svn仓库也迁移到git仓库，统一版本库工具
* 因为希望迁移后，还是能保留版本库内的提交日志等信息，以及需要把所有分支都对应迁移到git仓库
* 为了完成这个目标以及减少迁移的工作量，查阅了一些资料，以及参考了外面一些团队的实践，最后采用了git-svn工具，并将迁移过程整理成自动化脚本
* 参考资料：[Git 与其他系统 - 迁移到 Git](https://git-scm.com/book/zh/v2/Git-%E4%B8%8E%E5%85%B6%E4%BB%96%E7%B3%BB%E7%BB%9F-%E8%BF%81%E7%A7%BB%E5%88%B0-Git)

<!--break-->

### 脚本代码

```
#!/bin/bash

SVN_REPO=$1 ##svn仓库
GIT_REPO=$2 ##git仓库

help(){
    echo "使用语法: $0 svn仓库地址 目标git仓库地址"
    echo "eg:bash $0 https://svn.xxx.com/xxx https://git.xxx.com/git/xxx.git"
}

if [ "$SVN_REPO" == "" ] ; then
    echo "svn仓库地址不能为空"
    help
    exit
fi

if [ "$GIT_REPO" == "" ] ; then
    echo "git仓库地址不能为空"
    help
    exit
fi

##因为我本地是使用mac环境，Unix和linux中的sed有点不一样，所以我在本地装了gsed工具
GLOBAL_OS=$(uname)      ##操作系统

if [ "$GLOBAL_OS" == "Linux" ] ; then
    SED=sed
else
    SED=gsed
fi

REPO_NAME=$(echo $SVN_REPO  | awk -F "/" '{print $NF}')
IGNORE="/node_modules
/public/storage
/public/web
/vendor
/.idea
Homestead.json
Homestead.yaml
.env
.idea
.DS_Store
composer.lock
tags
npm-debug.log
/release
/deps
"

EMPTY_IGNORE="
*
!.gitignore
"

##这里要注意的是，-T参数对应的是svn仓库中的trunk命名规则 -b对应的是分支目录的命名规则，如果你的svn仓库中不是这样命名的，请把这个参数改成你仓库中实际对应的值
git svn clone $SVN_REPO --no-metadata -T trunk -b branch -t tags $REPO_NAME
cd $REPO_NAME
cp -Rf .git/refs/remotes/origin/tags/* .git/refs/tags/
rm -Rf .git/refs/remotes/origin/tags

cp -Rf .git/refs/remotes/origin/* .git/refs/heads/
rm -Rf .git/refs/remotes/origin

##这里我把svn中的所有分支都放到了heads
cp -Rf .git/refs/remotes/* .git/refs/heads/
rm -Rf .git/refs/remotes

##将trunk分支重命名成master
mv .git/refs/heads/trunk .git/refs/heads/master

##对每个分支初始化一些具体业务
for branch in $(ls .git/refs/heads/)
do
    git checkout ${branch}
    ##处理一些业务
    echo "${IGNORE}" > .gitignore
    rm -Rf deps
    mkdir -p www/template_c
    echo "${EMPTY_IGNORE}" > www/template_c/.gitignore
    rm -f composer.lock
    $SED -i "/mcweb\//{s/mcweb/mcweb-deps/g}" composer.json
    git add .
    git commit . -m'【开发】初始化一些具体业务'
    echo $branch
done


git remote add origin $GIT_REPO
git checkout master
##因为本地和远程是两个不同的项目，要把两个不同的项目合并，git需要添加一句代码，在git pull，这句代码是在git 2.9.2版本发生的，最新的版本需要添加--allow-unrelated-histories
git pull origin master --allow-unrelated-histories
git push origin --all

```

### 迁移前，svn仓库的分支结构
![](/images/post/14930268442268.jpg)
### 迁移后，git仓库的分支结构
![](/images/post/14930269454241.jpg)


