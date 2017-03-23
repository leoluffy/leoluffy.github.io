---
layout: post
category : PHP
title: 'Mac下安装NMP环境'
tagline: ""
tags : [PHP]
---
* auto-gen TOC:
{:toc}

## 安装 brew

Brew 是 Mac 下面的包管理工具，通过 Github 托管适合 Mac 的编译配置以及 Patch，可以方便的安装开发工具。 Mac 自带ruby 所以安装起来很方便，同时它也会自动把git也给你装上。 [跳转到brew官方网站](http://brew.sh)


```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

#### homebrew的常用命令 

1. brew update #更新可安装包的最新信息，建议每次安装前都运行下  
2. brew search pkg_name #搜索相关的包信息  
3. brew install pkg_name #安装包  

## 安装nginx  
  
#### 使用brew安装nginx

```
brew install nginx --with-http2

```

#### ngnix配置
配置目录：/usr/local/etc/nginx/nginx.conf

```
user  www www;
worker_processes auto;
error_log  /Users/yuyongjia/logs/nginx/nginx_error.log  crit;
##Specifies the value for maximum file descriptors that can be opened by this process.
worker_rlimit_nofile 51200;

events
{
    ##use epoll;
    worker_connections 51200;
}

http
{
    include       mime.types;
    default_type  application/octet-stream;


    server_names_hash_bucket_size 128;
    #client_header_buffer_size 32k;
    #large_client_header_buffers 4 32k;
    #client_max_body_size 8m;

    server_tokens off;
    expires       1h;
    sendfile on;
    keepalive_timeout 60;
    tcp_nodelay on;
    error_page   404  /;

    fastcgi_connect_timeout 20;
    fastcgi_send_timeout 30;
    fastcgi_read_timeout 60;
    fastcgi_buffer_size 64k;
    fastcgi_buffers 4 64k;
    fastcgi_busy_buffers_size 128k;
    fastcgi_temp_file_write_size 128k;
    fastcgi_temp_path /dev/shm;

    gzip on;    
    gzip_disable "MSIE [1-6].";
    gzip_min_length  2048;
    gzip_buffers     4 16k;
    gzip_http_version 1.1;
    gzip_types  text/plain  text/css  application/xml application/x-javascript ;

    log_format  access  '$request_time - $remote_user $remote_addr [$time_iso8601] "$request" '
                                        '$status $body_bytes_sent "$http_referer" '
                                        '"$http_user_agent" $host $server_addr';

    access_log off;

#################  include  ###################

    include vhost/*.conf;
}

```
#### 启动nginx

```
sudo nginx
sudo nginx -t
sudo nginx -s reload
```
#### 自定义nginx启动命令

```
vim /etc/profile 
alias nginx_start="sudo nginx"
alias nginx_reload="sudo nginx -s reload"
source /etc/profile 
```
## 安装PHP
#### 使用brew安装php
添加brew的PHP扩展库：

```
brew tap homebrew/dupes
brew tap homebrew/php
brew update
```
这里搜索的是全部php版本  

```
 brew search php 
```
安装PHP7我们就这样搜  

```
brew search php7
```
我这里选择homebrew/php/php70安装   

```
brew install homebrew/php/php70
```

安装完毕后，有输出如下信息：  

```
The php.ini file can be found in:
    /usr/local/etc/php/7.0/php.ini

✩✩✩✩ Extensions ✩✩✩✩

If you are having issues with custom extension compiling, ensure that
you are using the brew version, by placing /usr/local/bin before /usr/sbin in your PATH:

      PATH="/usr/local/bin:$PATH"

PHP70 Extensions will always be compiled against this PHP. Please install them
using --without-homebrew-php to enable compiling against system PHP.

✩✩✩✩ PHP CLI ✩✩✩✩

If you wish to swap the PHP you use on the command line, you should add the following to ~/.bashrc,
~/.zshrc, ~/.profile or your shell's equivalent configuration file:

      export PATH="$(brew --prefix homebrew/php/php70)/bin:$PATH"

✩✩✩✩ FPM ✩✩✩✩

To launch php-fpm on startup:
    mkdir -p ~/Library/LaunchAgents
    cp /usr/local/opt/php70/homebrew.mxcl.php70.plist ~/Library/LaunchAgents/
    launchctl load -w ~/Library/LaunchAgents/homebrew.mxcl.php70.plist

The control script is located at /usr/local/opt/php70/sbin/php70-fpm

OS X 10.8 and newer come with php-fpm pre-installed, to ensure you are using the brew version you need to make sure /usr/local/sbin is before /usr/sbin in your PATH:

  PATH="/usr/local/sbin:$PATH"

You may also need to edit the plist to use the correct "UserName".

Please note that the plist was called 'homebrew-php.josegonzalez.php70.plist' in old versions
of this formula.

With the release of macOS Sierra the Apache module is now not built by default. If you want to build it on your system
you have to install php with the --with-apache option. See  brew options php70  for more details.

To have launchd start homebrew/php/php70 now and restart at login:
  brew services start homebrew/php/php70
==> Summary
🍺  /usr/local/Cellar/php70/7.0.12_5: 332 files, 38.4M
```

执行：  

```
yuyongjiadeMacBook-Pro:local yuyongjia$ php -v
PHP 5.5.36 (cli) (built: May 29 2016 01:07:06) 
Copyright (c) 1997-2015 The PHP Group
Zend Engine v2.5.0, Copyright (c) 1998-2015 Zend Technologies
yuyongjiadeMacBook-Pro:local yuyongjia$ PATH="/usr/local/bin:$PATH"
yuyongjiadeMacBook-Pro:local yuyongjia$ php -v
PHP 7.0.12 (cli) (built: Oct 14 2016 09:55:03) ( NTS )
Copyright (c) 1997-2016 The PHP Group
Zend Engine v3.0.0, Copyright (c) 1998-2016 Zend Technologies

PATH="/usr/local/sbin:$PATH"

```
#### 自定义php-fpm的启动命令  

```
sudo php-fpm -D # 启动
sudo killall php-fpm # 关闭
vim /etc/profile 
alias fpm_start="sudo php-fpm -D"
alias fpm_stop="sudo killall php-fpm"
alias fpm_restart='fpm_stop && fpm_start'
source /etc/profile 

```

## 安装mysql
#### 使用brew安装mysql
```
brew install mysql
chown -R mysql:mysql /usr/local/var/mysql
chmod -R 755 /usr/local/var/mysql
```
#### 自定义mysql的启动命令

```
vim /etc/profile 
alias mysql_start="sudo mysql.server start"
alias mysql_stop="sudo mysql.server stop"
alias mysql_restart="mysql_stop && mysql_start"
source /etc/profile 

```
#### 自定义进入mysql命令

```
vim /etc/profile 
alias con_mysql="mysql -uroot -p密码"
source /etc/profile 
```
## 安装composer

```
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php 
mv composer.phar /usr/local/bin/composer
##查看当前composer版本
composer -V
```
#### 使用composer中国镜像
```
##修改composer的全局配置文件，全局生效
composer config -g repo.packagist composer https://packagist.phpcomposer.com
```

## 安装memcached软件
#### 使用brew安装memcached

```
brew install memcached
```
#### 设置自定义启动和关闭命令
```
vim /etc/profile 
alias memcached_start="/usr/local/opt/memcached/bin/memcached -d -m 64 -c 4096 -p 11210 -u www -t 10 && /usr/local/opt/memcached/bin/memcached -d -m 256 -c 4096 -p 11211 -u www -t 10"
alias memcached_stop="pkill -9 memcached"
source /etc/profile 
```
## 安装php7.0 memcached扩展
#### 使用brew安装memcached扩展
```
brew install --HEAD homebrew/php/php70-memcached
```
#### 设置将session存储到memcached

```
session.save_handler = memcached
session.save_path = "127.0.0.1:11210"
```