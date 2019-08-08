---
title: 编译PHP时添加PDO扩展时遇到的小坑
date: 2019/8/1 10:47
categories: 后端
tag: [PHP,MySQL]
toc: true
---

昨天入职新公司，在新机器上部署环境。
直接配了PHP 7.3.7 + MySQL 8.0.17，编译的过程不是很复杂，不过在配置编译参数 `--with-pdo-mysql` 的时候却出现了下面的问题：
1. 生成 `configure` 脚本没有异常。
2. make 编译过程中出现了 `ext/pdo_mysql/php_pdo_mysql_int.h:144:2: error: unknown type name 'my_bool'` 的问题。

起初看到这个异常还以为 `.h` 文件缺少了头文件之类的问题，在自行加上 `#include <stdbool.h>` 后依然报错。
再往后想一下，官方的 `.h` 文件肯定不会有什么问题，一定是参数写错了，回头去看 `--with-pdo-mysql` 参数后面的内容。
> `--with-pdo-mysql=/path/mysql` 
距离上一次编译已经是很久以前了，自从用了 `brew` 之后经常偷懒（惭愧）。
印象中就记得直接指定 `mysql` 的目录就可以了。

经过查阅，发现这个参数的具体内容在 `php5.3` 的时候就已经引入了 `mysqlnd` 。在 `php5.4` 之后的版本已经作为默认配置项了。
> 以前的这种参数方式相当于是使用 `PHP` 通过 `MySQL` 的 `libmysql` 库来实现，这个库是用C/C++编写的，虽然一直以来 `PHP` 通过 `libmysql` 访问数据库性能也一直很好，但是却无法利用PHP本身的很多特性。
>
> mysqlnd提供了和Zend引擎高度的集成性，更加快速的执行速度，更少的内存消耗，利用了PHP的Stream API，以及客户端缓存机制。由于mysqlnd是透过Zend引擎，因此提供更多高级特性，以及有效利用Zend进行加速。

综上所述，无论是从性能还是编译过程，都有了很大的提升，所以放弃以前的编译参数的方式自然也是情理之中了。

顺便记录一下目前自己用得到的 `configure` 脚本：

```SH
./configure --prefix=/usr/local/php --with-config-file-path=/usr/local/php/etc --enable-fpm --with-fpm-user=nintenichi --with-fpm-group=staff --enable-bcmath --enable-ctype --enable-json --enable-mbstring --with-openssl=/usr/local/Cellar/openssl/1.0.2s/ --with-pdo-mysql=mysqlnd --enable-tokenizer --enable-xml --with-iconv=/usr/local/Cellar/libiconv/1.16/ --enable-zip --with-libzip=/usr/local/Cellar/libzip/1.5.2 --with-zlib-dir=/usr/local/Cellar/zlib/1.2.11
```

其中的一些扩展库使用 `brew` 安装，省去了使用 `xcode` 出现各种各样奇怪问题的情况。
