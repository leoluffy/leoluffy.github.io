---
layout: post
category : GIT
title: '使用git pages发布在线文档'
tagline: ""
tags : [GIT]
---
* auto-gen TOC:
{:toc}


安装gitbook工具

```
yuyongjiadeMacBook-Pro:dev-guide-doc yuyongjia$ sudo cnpm install -g gitbook
yuyongjiadeMacBook-Pro:dev-guide-doc yuyongjia$ gitbook
You need to install "gitbook-cli" to have access to the gitbook command anywhere on your system.
If you've installed this package globally, you need to uninstall it.
>> Run "npm uninstall -g gitbook" then "npm install -g gitbook-cli"
yuyongjiadeMacBook-Pro:dev-guide-doc yuyongjia$ sudo cnpm install -g gitbook-cli 
yuyongjiadeMacBook-Pro:dev-guide-doc yuyongjia$ gitbook -V
CLI version: 2.3.0
Installing GitBook 3.2.2

```

* 创建一个文档仓库，并clone到本地  
* 我的仓库内容包含了一个git目录，git目录下有一份git开发流程文档

```
yuyongjiadeMacBook-Pro:dev-guide-doc yuyongjia$ git clone https://git.xxx.com/git/web-doc/dev-guide-doc.git
yuyongjiadeMacBook-Pro:dev-guide-doc yuyongjia$ ls
README.md		git
yuyongjiadeMacBook-Pro:dev-guide-doc yuyongjia$ ls git/
git-flow.md  
```

gitbook init初始化

```
yuyongjiadeMacBook-Pro:dev-guide-doc yuyongjia$ ls
README.md		git
yuyongjiadeMacBook-Pro:dev-guide-doc yuyongjia$ gitbook init
info: create git/README.md 
info: create SUMMARY.md 
info: initialization is finished 

```
* 成功初始化之后，会在工作目录生成SUMMARY.md和README.md文件  
* 这里git是我放文档的子目录，看起来也会自动在每个子目录下创建README文件  
* SUMMARY.md是书籍的目录结构   
* README.md 文件是书籍的介绍  

编辑SUMMARY.md

```
# Summary

* [Introduction](README.md)
* [git相关](git/README.md)
	* [git开发流程梳理](git/git-flow.md)
```

本地预览效果

```
yuyongjiadeMacBook-Pro:dev-guide-doc yuyongjia$ gitbook serve
Live reload server started on port: 35729
Press CTRL+C to quit ...

info: 7 plugins are installed 
info: loading plugin "livereload"... OK 
info: loading plugin "highlight"... OK 
info: loading plugin "search"... OK 
info: loading plugin "lunr"... OK 
info: loading plugin "sharing"... OK 
info: loading plugin "fontsettings"... OK 
info: loading plugin "theme-default"... OK 
info: found 2 pages 
info: found 0 asset files 
info: >> generation finished with success in 0.8s ! 

Starting server ...
Serving book on http://localhost:4000

```
![](/images/14925742083710.jpg)

完成文档编写后，执行构建

```
yuyongjiadeMacBook-Pro:dev-guide-doc yuyongjia$ gitbook build
info: 7 plugins are installed 
info: 6 explicitly listed 
info: loading plugin "highlight"... OK 
info: loading plugin "search"... OK 
info: loading plugin "lunr"... OK 
info: loading plugin "sharing"... OK 
info: loading plugin "fontsettings"... OK 
info: loading plugin "theme-default"... OK 
info: found 2 pages 
info: found 0 asset files 
info: >> generation finished with success in 0.6s ! 
```

构建后生成的文件在_book文件夹里面

```
yuyongjiadeMacBook-Pro:dev-guide-doc yuyongjia$ ls
README.md	SUMMARY.md	_book		git
yuyongjiadeMacBook-Pro:dev-guide-doc yuyongjia$ cd _book/
git/               gitbook/           index.html         search_index.json
```
添加.gitignore 文件，忽略_book目录

```
vim .gitignore 
##添加以下内容
_book
```

将_book发布到gh-pages分支

```
##创建一个空白的文档目录
git checkout --orphan gh-pages
##将_book目录的内容拷贝到gh-pages分支的根目录
cp -rf _book/* .
##添加所有文件
git add *
git commit -m "【开发】提交pages文件"
git push origin gh-pages

```
访问在线文档：https://git.xxx.com/web-doc/dev-guide-doc/pages/  
![](/images/14925826306077.jpg)

