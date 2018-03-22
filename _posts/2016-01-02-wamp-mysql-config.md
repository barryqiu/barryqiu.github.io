---
title: wamp的mysql配置
layout: post
categories: 数据库
---
wamp是windows上一组非常方便的amp（apache，mysql，php）工具套装，我也经常使用，但是在正式使用之前mysql可能需要我们进行一下简单的配置，这里对配置的过程进行下面的记录。

# 修改字符编码为UTF8
在[client]下面添加：

```
default-character-set=utf8
```

在[mysqld]下面添加：

```
character-set-server = utf8
```

这样就可以了，通过重启service之后在控制台中输入下面的命令查看结果

```
show variable like '%char%'
```

# 设置root密码
wamp安装之后默认的root密码是空，需要对它进行重新设置：

* 进入MYSQL控制台,提示输入密码，因为没有密码直接按回车键进入。
* 输入“USE mysql;”然后回车，输入

```sql
update user set password=password('123456') where user='root'; 
FLUSH PRIVILEGES;
```
然后回车;返回信息：
Query OK, 0 rows affected (0.00 sec)
Rows matched: 2 Changed: 0 Warnings: 0

如果是5.7版本的mysql，原来user里的password字段已经变更为authentication_string,所以使用上述语句更新会提示“password 字段不存在，所以上述设置应该修改为：

```
UPDATE mysql.USER SET authentication_string = PASSWORD ('123456'), password_expired = 'N' WHERE USER = 'root';
FLUSH PRIVILEGES;
```

* 重启mysql服务;

* 再次进入控制台时即可以输入新设置的密码了。

这样mysql登录与密码是没问题了，但是使用phpmyadmin登录不了，找到phpmyadmin安装目录，一般是在wamp/apps/phpmyadmin/这个位置，打开config.inc.php，编辑如下几行：

```
$cfg['Servers'][$i]['auth_type'] = 'http';    
//由config修改为http$cfg['Servers'][$i]['user'] = 'root';
$cfg['Servers'][$i]['password'] = 'newpassword';
$cfg['Servers'][$i]['AllowNoPassword'] = false;
```

* 重新启动wamp，按照刚才设置的密码登陆即可。
