---
title: 关于PHP错误日志踩过的一些坑
date: 2017-08-19 11:41:04
categories: PHP
tags: 
- log
- php
- error
---


对于线上的项目来说，错误日志和访问日志是至关重要的。学会如何分析日志找出问题是一个必备技能。本文就谈谈关于PHP的错误日志那些事。
<!-- more -->


###### phpinfo() 中 Local Value（局部变量）Master Value（主变量） 的区别

>首先说说这两个东西，
>phpinfo() 的很多部分有两个Column：Local Value和Master Value
>1. Master Value是PHP.ini文件中的内容.
>2.Local value 是当前目录中的设置，这个值会覆盖Master Value中对应的值
>由于WEB Sever Config或.htaccess的设置,或程序中ini_set()的设置,当前目录中的设置会不同于PHP.ini文 件中的设置
>PS：Apache的配置文件中可以重写php.ini的设置，可能在conf/httpd.conf，也可能在>conf.d/***.conf中，一般在conf.d/php.conf中

理解上面两个东东了再来说说关于错误日志的配置问题


>nginx是一个web服务器，因此nginx的access日志只有对访问页面的记录，不会有php 的 error log信息。
nginx把对php的请求发给php-fpm fastcgi进程来处理，默认的php-fpm只会输出php-fpm的错误信息，在php-fpm的errors log里也看不到php的errorlog
原因是php-fpm的配置文件php-fpm.conf中默认是关闭worker进程的错误输出，直接把他们重定向到/dev/null,所以我们在nginx的error log 和php-fpm的errorlog都看不到php的错误日志。
调试起来就很痛苦了。解决nginx下php-fpm不记录php错误日志的办法:
>
>1.修改php-fpm.conf中配置 没有则增加
`catch_workers_output = yes //记录错误日志到php-fpm中` 
`error_log = log/error_log //错误日志存放位置`
>
>2.修改php.ini中配置，没有则增加
>`log_errors = On`
>`error_log = "/usr/local/lnmp/php/var/log/error_log"`
>`error_reporting=E_ALL&~E_NOTICE`
>
>
>3.重启php-fpm，当PHP执行错误时就能看到错误日志在`"/usr/local/lnmp/php/var/log/error_log"`中了
>
>请注意：
>
>###### 这里是坑点：
>
>1. php-fpm.conf 中的php_admin_value[error_log] 参数 会覆盖php.ini中的 error_log 参数,所以确保你在phpinfo()中看到的最终error_log文件具有可写权限并且没有设置php_admin_value[error_log] 参数，否则错误日志会输出到php-fpm的错误日志里
>
>2.找不到php.ini位置，使用php的phpinfo()结果查看
>也可以在命令行用这个命令查看ini位置：
>`php -i | grep php.ini`
>
>查看error_log位置
>`php -i | grep error_log`
>
>###### 这里又是一个需要注意的地方：如何将php的错误日志输出到nginx的错误日志里
>在PHP 5.3.8及之前的版本中，通过FastCGI运行的PHP，在用户访问时出现错误，会首先写入到PHP的errorlog中，如果PHP的errorlog无法写入，则会将错误内容返回给FastCGI接口，然后nginx在收到FastCGI的错误返回后记录到了nginx的errorlog中
>在PHP 5.3.9及之后的版本中，出现错误后PHP只尝试写入PHP的errorlog中，如果失败则不会再返回到FastCGI了，错误日志会输出到php-fpm的错误日志里。所以如果想把php错误日志输出到nginx错误日志，需要使用php5.3.8之前的版本，并且配置文件中php的error_log对于php worker进程不可写



最后再强调一次：
如果错误日志没有写入到文件,查看www用户对`php_admin_value[error_log]`的路径是否有写入权限

`php_flag` 修改`php.ini`中的配置 开关形式On或Off 可以被`ini_set`修改
`php_value` 修改`php.ini`中的配置 value形式 可以被`ini_set`修改
`php_admin_flag` 修改`php.ini`中的配置 开关形式On或Off 不可以被`ini_set`修改
`php_admin_value` 修改`php.ini`中的配置 value形式 不可以被`ini_set`修改