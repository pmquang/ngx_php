ngx_php
======
[![Build Status](https://travis-ci.org/rryqszq4/ngx_php.svg?branch=master)](https://travis-ci.org/rryqszq4/ngx_php) 
[![Gitter](https://badges.gitter.im/rryqszq4/ngx_php.svg)](https://gitter.im/rryqszq4/ngx_php?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)
[![GitHub release](https://img.shields.io/github/release/rryqszq4/ngx_php.svg)](https://github.com/rryqszq4/ngx_php/releases/latest)
[![license](https://img.shields.io/badge/license-BSD--2--Clause-blue.svg)](https://github.com/rryqszq4/ngx_php/blob/master/LICENSE)

[ngx_php](https://github.com/rryqszq4/ngx_php)是嵌入php脚本的nginx模块。你也可以称它为nginx-php5-module。

概述
----
我很有幸经历了apache的mod_php时代、nginx + php-fpm时代，之前写的代码都是以阻塞方式运行的。最近几年异步、
非阻塞这些词在行业内非常热，特别是当我接触openresty的时候，我完全被折服的同时也反思了下，为什么不能把php嵌入nginx里面呢？
借鉴了openresty的思想，我打算实现我的想法，于是ngx_php诞生了。搭载nginx之上借助nginx中的subrequest和upstream等模块，
ngx_php可以实现非阻塞io通信，并且代码是至上而下顺序书写，避免了繁琐的回调写法。性能相比较php-fpm会高出几个量级。

特性
--------
* 支持加载php.ini配置文件  
可以在nginx的配置文件中加载php的配置文件
* 支持原生php的全局变量$_GET, $_POST, $_COOKIE, $_SERVER, $_FILES, $_SESSION...
* 支持运行php代码与文件  
可以在nginx的配置文件中书写代码，也可以在配置文件中加载代码
* 支持RFC 1867文件上传协议
* 支持php错误输出
* 支持加载与运行PECL扩展  
遗憾的是部分扩展是阻塞方式的，可以正常运行但在ngx_php中并不能换来性能的提升
* 支持nginx的API在php中调用  
利用php扩展封装了一些nginx的底层接口，方便在php中调用

环境
-----------
- PHP 5.3.* ~ PHP 5.6.*
- nginx-1.4.7 ~ nginx-1.11.4

安装
-------
- 安装php  
需要编译php，并且需要开启线程安全和编译动态共享库
```sh
$ wget http://php.net/distributions/php-5.3.29.tar.gz
$ tar xf php-5.3.29.tar.gz
$ cd php-5.3.29
$ ./configure --prefix=/path/to/php \
$             --enable-maintainer-zts \
$             --enable-embed
$ make && make install
```

- 安装ngx_php  
编译完php就可以开始安装ngx_php，需要重新编译nginx并加入ngx_php模块
```sh
$ git clone https://github.com/rryqszq4/ngx_php.git

$ wget 'http://nginx.org/download/nginx-1.6.3.tar.gz'
$ tar xf nginx-1.6.3.tar.gz
$ cd nginx-1.6.3

$ export PHP_BIN=/path/to/php/bin
$ export PHP_INC=/path/to/php/include/php
$ export PHP_LIB=/path/to/php/lib

$ ./configure --user=www --group=www \
$             --prefix=/path/to/nginx \
$             --with-ld-opt="-Wl,-rpath,$PHP_LIB" \
$             --add-module=/path/to/ngx_php/dev/ngx_devel_kit \
$             --add-module=/path/to/ngx_php
$ make && make install
```

摘要
--------

```nginx
user www www;
worker_processes  4;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    keepalive_timeout  65;
    
    client_max_body_size 10m;   
    client_body_buffer_size 4096k;

    php_ini_path /usr/local/php/etc/php.ini;

    server {
        listen       80;
        server_name  localhost;
    
        location /php {
            content_by_php '
                echo "hello ngx_php";
            ';
        }
    }
}
```

典型框架配置
--------------

**wordpress (多入口模式):**

```nginx
server {
    listen 80;
    server_name wordpress-sample.com;
    
    location ~ \.php$ {
        root   /home/www/wordpress;
        content_by_php "
            require_once('/home/www/wordpress'.$_SERVER['DOCUMENT_URI']);
        ";
    }
}
```

**yaf & yii (单一入口模式):**

```nginx
server {
    listen 80;
    server_name yaf-sample.com;
    access_log  logs/yaf-sample.com.access.log;

    root /home/www/yaf-sample;
    index index.php index.html;
    
    location /favicon.ico {
        log_not_found off;
    }

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        content_by_php '
            header("Content-Type: text/html;charset=UTF-8");
            require_once("/home/www/yaf-sample/index.php");
        ';
    }
}  
```

测试
----
使用perl语言开发的[Test::Nginx](https://github.com/openresty/test-nginx)的测试模块进行测试, 用来发现ngx_php在开发与使用中存在的问题与缺陷。 

```sh
cd /path/to/ngx_php
export PATH=/path/to/nginx/sbin:$PATH
prove -r t
```

Test result:

```sh
t/001-hello.t ........... ok
t/002-ini.t ............. ok
t/003-_GET.t ............ ok
t/004-_POST.t ........... ok
t/005-_SERVER.t ......... ok
t/006-_COOKIE.t ......... ok
t/007-_FILES.t .......... ok
t/008-error.t ........... ok
t/009-session.t ......... ok
t/100-ngx_socket_tcp.t .. ok
All tests successful.
Files=10, Tests=22,  3 wallclock secs ( 0.04 usr  0.01 sys +  0.94 cusr  0.27 csys =  1.26 CPU)
Result: PASS
```

nginx指令
----------
* [php_ini_path](#php_ini_path)
* [init_by_php](#init_by_php)
* [init_by_php_file](#init_by_php_file)
* [rewrite_by_php](#rewrite_by_php)
* [rewrite_by_php_file](#rewrite_by_php_file)
* [access_by_php](#access_by_php)
* [access_by_php_file](#access_by_php_file)
* [content_by_php](#content_by_php)
* [content_by_php_file](#content_by_php_file)
* [content_thread_by_php](#content_thread_by_php)
* [content_thread_by_php_file](#content_thread_by_php_file)
* [set_by_php](#set_by_php)
* [set_run_by_php](#set_run_by_php)
* [set_by_php_file](#set_by_php_file)
* [set_run_by_php_file](#set_run_by_php_file)

php_ini_path
------------
**syntax:** *php_ini_path &lt;php.ini file path&gt;*  
**context:** *http*  
**phase:** *loading-config*  
加载php配置文件

```nginx
php_ini_path /usr/local/php/etc/php.ini;
```

init_by_php
-----------
**syntax:** *init_by_php &lt;php script code&gt;*  

**context:** *http*  

**phase:** *loading-config*

init_by_php_file
----------------
**syntax:** *init_by_php_file &lt;php script file&gt;*  

**context:** *http*  

**phase:** *loading-config*

rewrite_by_php
--------------
**syntax:** *rewrite_by_php &lt;php script code&gt;*  

**context:** *http, server, location, location if*  

**phase:** *rewrite*  

nginx的rewrite阶段运行php代码。

rewrite_by_php_file
-------------------
**syntax:** *rewrite_by_php_file &lt;php script file&gt;*  

**context:** *http, server, location, location if*  

**phase:** *rewrite*

access_by_php
-------------
**syntax:** *access_by_php &lt;php script code&gt;*  

**context:** *http, server, location, location if*  

**phase:** *access*  

nginx的access阶段运行php代码。

access_by_php_file
------------------
**syntax:** *access_by_php_file &lt;php script file&gt;*  

**context:** *http, server, location, location if*  

**phase:** *access*

content_by_php
--------------
**syntax:** *content_by_php &lt;php script code&gt;*  

**context:** *http, server, location, location if*  

**phase:** *content*  

ngx_php核心处理阶段，可以执行php代码，但是这个指令被设计为以阻塞方式运行php代码，因此不要使用此指令做io操作。
```nginx
location /content_by_php {    
    content_by_php "
        header('Content-Type: text/html;charset=UTF-8');
    
        echo phpinfo();
    ";
        
}
```

content_by_php_file
-------------------
**syntax:** *content_by_php_file &lt;php script file&gt;*  

**context:** *http, server, location, location if*  

**phase:** *content*  

ngx_php核心处理阶段，可以执行php文件，但是这个指令被设计为以阻塞方式执行php文件，因此不要使用此指令做io操作。
```nginx
location /content_by_php_file {
        content_by_php_file /home/www/index.php;
}
```

content_thread_by_php
---------------------
**syntax:** *content_thread_by_php &lt;php script code&gt;*  

**context:** *http, server, location, location if*  

**phase:** *content*  

ngx_php核心处理阶段，可以执行php代码，底层使用nginx异步机制＋多线程实现以非阻塞方式运行php代码。
```nginx
location /content_thread_by_php {
    content_sync_by_php "
        echo 'hello world';

        $res = ngx_location::capture('/list=s_sh000001');
        var_dump($res);
        
        $capture_multi = array(
                            '/list=s_sh000001',
                            '/list=s_sh000001',
                            '/list=s_sh000001'
                    );
        $res = ngx_location::capture_multi($capture_multi);
        var_dump($res);
        
        $res = ngx_location::capture('/list=s_sh000001');
        var_dump($res);
        
        $res = ngx_location::capture('/list=s_sh000001');
        #var_dump($res);
    ";
}

location /list {
    proxy_pass http://hq.sinajs.cn;
    proxy_set_header Accept-Encoding "";
}
```

content_thread_by_php_file
--------------------------
**syntax:** *content_thread_by_php_file &lt;php script file&gt;*  

**context:** *http, server, location, location if*  

**phase:** *content*  

ngx_php核心处理阶段，可以执行php文件，底层使用nginx异步机制＋多线程实现以非阻塞方式运行php文件。

set_by_php
----------
**syntax:** *set_by_php &lt;php script code&gt;*  

**context:** *server, server if, location, location if*  

**phase:** *content*

set_run_by_php
--------------
**syntax:** *set_run_by_php &lt;php script code&gt;*  

**context:** *server, server if, location, location if*  

**phase:** *content*

set_by_php_file
---------------
**syntax:** *set_by_php_file &lt;php script file&gt;*  

**context:** *server, server if, location, location if*  

**phase:** *content*

set_run_by_php_file
-------------------
**syntax:** *set_run_by_php_file &lt;php script file&gt;*  

**context:** *server, server if, location, location if*  

**phase:** *content*


Nginx的php接口
-------------
* [ngx_location::capture_async](#ngx_locationcapture_async)
* [ngx_location::capture_multi_async](#ngx_locationcapture_multi_async)
* [ngx_location::capture](#ngx_locationcapture)
* [ngx_location::capture_multi](#ngx_locationcapture_multi)
* [ngx_socket_tcp::__construct](#ngx_socket_tcp__construct)
* [ngx_socket_tcp::connect](#ngx_socket_tcpconnect)
* [ngx_socket_tcp::send](#ngx_socket_tcpsend)
* [ngx_socket_tcp::receive](#ngx_socket_tcpreceive)
* [ngx_socket_tcp::close](#ngx_socket_tcpclose)
* [ngx_socket_tcp::settimeout](#ngx_socket_tcpsettimeout)
* [ngx_log::error](#ngx_logerror)

ngx_location::capture_async
---------------------------
**syntax:** *ngx_location::capture_async(string $uri, mixed $closure)*  

**context:** *content_async_by_php*  

借助nginx底层强大的subrequest，实现php完全非阻塞的异步代码调用

```php
ngx_location::capture_async('/foo', function($callback = 'callback'){
    echo $callback;
});
```

ngx_location::capture_multi_async
---------------------------------
**syntax:** *ngx_location::capture_multi_async(array $uri, mixed $closure)*  

**context:** *content_async_by_php*  

和ngx_location::capture_async相似，但是可以支持完全非阻塞的并行异步代码调用

```php
$capture_multi = array(
    '/foo',
    '/bar',
    '/baz'
);
ngx_location::capture_multi_async($capture_multi, function($callback = 'callback'){
    var_dump($callback);
});
```

ngx_location::capture
---------------------
**syntax:** *ngx_location::capture(string $uri)*  

**context:** *content_thread_by_php* *content_sync_by_php*  

借助nginx底层强大的subrequest，实现php完全非阻塞调用

```php
$result = ngx_location::capture('/foo');
echo $result;
```

ngx_location::capture_multi
---------------------------
**syntax:** *ngx_location::capture_multi(array $uri)*  

**context:** *content_thread_by_php* *content_sync_by_php*  

和ngx_location::capture相似，但是可以支持完全非阻塞的并行调用

```php
$capture_multi = array(
    '/foo',
    '/bar',
    '/baz'
);
$result = ngx_location::capture_multi($capture_multi);
var_dump($result);
```

ngx_socket_tcp::__construct
---------------------------
**syntax:** *ngx_socket_tcp::__construct()*  

**context:** *content_thread_by_php* *content_sync_by_php*  

```php
$tcpsock = new ngx_socket_tcp();
```

ngx_socket_tcp::connect
---------------------------
**syntax:** *ngx_socket_tcp::connect(string $host, int $port)*  

**context:** *content_thread_by_php* *content_sync_by_php*  

resolver 8.8.8.8;

```php
$tcpsock = new ngx_socket_tcp();
$tcpsock->connect('127.0.0.1',11211));
```

ngx_socket_tcp::send
---------------------------
**syntax:** *ngx_socket_tcp::send(string $buf)*  

**context:** *content_thread_by_php* *content_sync_by_php*  

```php
$tcpsock = new ngx_socket_tcp();
$tcpsock->connect('127.0.0.1',11211));
$tcpsock->send('stats\r\n');
```

ngx_socket_tcp::receive
---------------------------
**syntax:** *ngx_socket_tcp::receive()*  

**context:** *content_thread_by_php* *content_sync_by_php*  

```php
$tcpsock = new ngx_socket_tcp();
$tcpsock->connect('127.0.0.1',11211));
$tcpsock->send('stats\r\n');
$result = $tcpsock->receive();
var_dump($result);
$tcpsock->close();
```

ngx_socket_tcp::close
---------------------------
**syntax:** *ngx_socket_tcp::close()*  

**context:** *content_thread_by_php*  *content_sync_by_php*  

ngx_socket_tcp::settimeout
---------------------------
**syntax:** *ngx_socket_tcp::settimeout(int time)*  

**context:** *content_thread_by_php*  *content_sync_by_php*  

ngx_log::error
--------------
**syntax:** *ngx_log::error(int level, string log)* 

**context:** *content_thread_by_php* *content_sync_by_php* 

Nginx的错误日志等级，在php中的实现。
* ngx_log::STDERR
* ngx_log::EMERG
* ngx_log::ALERT
* ngx_log::CRIT
* ngx_log::ERR
* ngx_log::WARN
* ngx_log::NOTICE
* ngx_log::INFO
* ngx_log::DEBUG

```php
ngx_log::error(ngx_log::ERR, "test");

/*
 2016/10/06 22:10:19 [error] 51402#0: *1 test while reading response header from upstream, client: 192.168.80.1, 
 server: localhost, request: "GET /_mysql HTTP/1.1", upstream: "127.0.0.1:3306", host: "192.168.80.140"
*/
```

问题
--------
[issues #6](https://github.com/rryqszq4/ngx_php/issues/6) - 注意在 php-5.3.29 中, libxml2 2.7.6 不是线程安全的. 可以尝试在安装php阶段，禁用xml.
```sh
./configure --prefix=/usr/local/php5329 \
            --with-config-file-path=/usr/local/php5329/etc \
            --with-iconv=/usr/local/libiconv \
            --disable-xml \
            --disable-libxml \
            --disable-dom \
            --disable-simplexml \
            --disable-xmlreader \
            --disable-xmlwriter \
            --without-pear \
            --enable-maintainer-zts  \
            --enable-embed
```

拷贝与授权
---------------------
Copyright (c) 2016, rryqszq4 <ngxphp@gmail.com>
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this
  list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice,
  this list of conditions and the following disclaimer in the documentation
  and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
