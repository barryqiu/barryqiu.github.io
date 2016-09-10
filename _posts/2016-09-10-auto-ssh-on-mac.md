---
title: Mac下自动SSH远程服务器的脚本
layout: post
categories: 运维
---

今年跟了我3年的 X230 屏幕中间开了一条亮线,一怒之下直接换了一个 MBP ,用的都挺好,但是在链接远程服务器时进行运维和开发的时候不是很方便,之前用 win 的时候可以使用 Xshell 或者 SecurtCRT 保存了我常用的几台服务器的账户和密码,需要访问哪台直接点击就连接上了,但是 mac 没有这么好用的工具,肿么办呢,就想到写一个脚本来帮我读取远程主机的用户和密码,然后通过输入一个命令就可以很方便的链接上各台机器.

通常我们登录远程主机的时候都会输入 

```
ssh user@ip
```

然后如果是第一次登录会出现提示是否保存 RSA key, 选择 yes 之后会提示输入密码,输入密码就可以登录主机了.由于这中间有个交互的过程,bash shell虽然功能很强大,但是不能实现有交互功能的多机器之间的操作,例如ssh和ftp.所以我使用了 expect 类型的脚本,expect是一种能够按照脚本内容里面设定的方式与交互式程序进行“会话”的程序。根据脚本内容，Expect可以知道程序会提示或反馈什么内容以及 什么是正确的应答。它是一种可以提供“分支和嵌套结构”来引导程序流程的解释型脚本语言。具体的代码如下:

```
#!/usr/bin/expect

#Usage sshsudologin.expect <host> <ssh user> <ssh password> 

set timeout 60

spawn ssh [lindex $argv 1]@[lindex $argv 0]

expect "yes/no" { 
	send "yes\r"
	expect "*?assword" { send "[lindex $argv 2]\r" }
	} "*?assword" { send "[lindex $argv 2]\r" }
interact
```

该脚本可以通过命令行参数传入host user 和password来实现登录远程主机,

但是每次都要手动输入host,user和password比较麻烦可以通过把这些信息保存在一个文件中,然后用另外一个脚本读取文件内容再调用上面的脚本,该脚本如下:


```
#!/bin/bash

name=$(cat /etc/goto.ini | awk '{printf $1 " "}')

if [ "$1" = "-h" ] || [ "$1" = "-help" ]
then
    echo "the first args is host alias"
    echo "    There are several to choose[ "$name"]"
    exit 0
fi

if [ -z "$1" ]
then
    echo "    please input -help for more information."
    exit 0
fi


config=$(cat /etc/goto.ini |grep $1)

ip=`echo "$config" | awk '{print $2}'`
user=`echo "$config" | awk '{print $3}'`
password=`echo "$config" | awk '{print $4}'`
sshlogin.expect $ip $user $password

```

保存主机信息的示例如下:

```
name1,ip1,user1,password1
name2,ip2,user2,password2
```

然后把上面的脚本加入到系统环境变量 PATH 中,就可以在任意位置通过输入一行命令, `脚本名 namex`, 这里namex是主机的别名,这里专门取一个别名是为了方便输入,不用每次输入IP这么麻烦,例如别名可以为 `online`,`test`等