---
layout: post
category : SVN
title: 'SVN设置忽略目录/文件'
tagline: ""
tags : [SVN]
---
* auto-gen TOC:
{:toc}

先设置SVN默认的编辑器为vim   

```
vim ~/.bash_profile
export SVN_EDITOR=vim
source ~/.bash_profile
```

忽略template_c下的所有文件

```
svn propedit svn:ignore template_c

```
在弹出来的编辑器中输入通配符：

```
*
```   
保存退出，会看到有一行提示：

```
Set new value for property 'svn:ignore' on 'template_c'
```
设置忽略前

```
svn st
?       trunk/www/template_c/2973e489aa12f44656c4d83500c534ef1348439d.file.login.html.php
?       trunk/www/template_c/491825a777cdca6a976daa6f58c920c7e1eda13e.file.index.html.php

```
设置忽略后

```
svn st
M      template_c
```

