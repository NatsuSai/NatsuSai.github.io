---
title: Nginx基础——Rewrite规则
date: 2018/12/30 15:13:00
categories: 
    - 编程
tags: 
    - Nginx
    - 转载
comments: true
thumbnail: /gallery/machi.png
---
&emsp;&emsp;rewrite是nginx一个特别重要的指令，该指令可以使用正则表达式改写URI。可以指定一个或多个rewrite指令，按顺序匹配。

<!--more-->
## 正则匹配规则
{% blockquote %}
~  区分大小写匹配  
~* 不区分大小写匹配  
!~ 和 !~* 区分大小写不匹配及不区分大小写不匹配  
{% endblockquote %}

## 文件及目录匹配

{% blockquote %}
-f和!-f 判断是否存在文件  
-d和!-d 判断是否存在目录  
-e和!-e 判断是否存在文件或目录  
-x和!-x 判断文件是否可执行
{% endblockquote %}

## rewrite基本语法

{% codeblock %}
set  
if  
return
break
rewrite
{% endcodeblock %}

### break指令  
{% blockquote %}
使用范围：server，location，if;  
{% endblockquote %} 

中断当前相同作用域的其他nginx配置。 

### if指令  
{% blockquote %}
使用范围：server，location  
{% endblockquote %} 

检查一个条件是否符合。If指令不支持嵌套，不支持多个条件&&和||处理。

### return指令
{% blockquote %}
格式：return code ;  
使用范围：server，location，if;
{% endblockquote %} 

结束规则的执行并返回状态码给客户端。

### set指令
{% blockquote %}
使用环境：server，location，if  
{% endblockquote %} 

定义一个变量，并给变量赋值。变量的值可以为文本、变量或者变量的组合。
{% codeblock lang:bash %}
set $var "hello world"
{% endcodeblock %}

## rewrite指令格式
    rewrite regex replacement [flag]
flag标志位有四种：
{% blockquote %}
break：停止rewrite检测,也就是说当含有break flag的rewrite语句被执行时,该语句就是rewrite的最终结果。   
last：停止rewrite检测,但是跟break有本质的不同,last的语句不一定是最终结果。  
redirect：返回302临时重定向，一般用于重定向到完整的URL(包含http:部分)   
permanent：返回301永久重定向，一般用于重定向到完整的URL(包含http:部分) 
{% endblockquote %} 

## 应用实例
当访问的文件和目录不存在时，重定向到某个php文件

{% codeblock %}
if( !-e $request_filename )
{
    rewrite ^/(.*)$ index.php last;
}
{% endcodeblock %}
    
目录对换 /123456/xxxx ====> /xxxx?id=123456

{% codeblock %}
rewrite ^/(\d+)/(.+)/  /$2?id=$1 last;
{% endcodeblock %}
    
如果客户端使用的是IE浏览器，则重定向到/ie目录下

{% codeblock %}
if( $http_user_agent ~ MSIE)
{
    rewrite ^(.*)$ /ie/$1 break;
}
{% endcodeblock %}
    
禁止访问以/data开头的文件

{% codeblock %}
location ~ ^/data
{
    deny all;
}
{% endcodeblock %}
    
禁止访问以.sh，.flv，.mp3为文件后缀名的文件

{% codeblock %}
location ~ .*\.(sh|flv|mp3)$
{
    return 403;
}
{% endcodeblock %}
    
设置某些类型文件的浏览器缓存时间

{% codeblock %}
location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$
{
    expires 30d;
}
{% endcodeblock %}
    
文件反盗链并设置过期时间

{% codeblock %}
location ~*^.+\.(jpg|jpeg|gif|png|swf|rar|zip|css|js)$ 
{
    valid_referers none blocked *.linuxidc.com*.linuxidc.net localhost 208.97.167.194;
    if ($invalid_referer) {
        rewrite ^/ http://img.linuxidc.net/leech.gif;
        return 412;
        break;
    }
    access_log  off;
    root /opt/lampp/htdocs/web;
    expires 3d;
    break;
}
{% endcodeblock %}
    
将多级目录下的文件转成一个文件，增强seo效果
  
{% codeblock %}  
/job-123-456-789.html 指向/job/123/456/789.html

rewrite^/job-([0-9]+)-([0-9]+)-([0-9]+)\.html$ /job/$1/$2/jobshow_$3.html last;
{% endcodeblock %}
    
域名跳转

{% codeblock %}
server
{
    listen 80;
    server_name jump.linuxidc.com;
    index index.html index.htm index.php;
    root /opt/lampp/htdocs/www;
    rewrite ^/ http://www.linuxidc.com/;
    access_log off;
}
{% endcodeblock %}
    
多域名转向

{% codeblock %}
server_name www.linuxidc.comwww.linuxidc.net;
index index.html index.htm index.php;
root  /opt/lampp/htdocs;
if ($host ~ "linuxidc\.net") {
    rewrite ^(.*) http://www.linuxidc.com$1permanent;
}
{% endcodeblock %}
