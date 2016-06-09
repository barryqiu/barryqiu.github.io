---
layout: post
title: Restful Api的访问控制方法
categories: web技术
---
最近在做的两个项目，都需要使用Restful Api，接口的安全性和访问控制便成为一个问题。

# 1 现有解决方案

看了一下别家的API访问控制办法，在stackoverflow上Simone Carletti 提出了两种解决方案：

1.使用一个基础HTTP认证，GitHub使用的就是这种方法，发送给指定URL的请求应该包含认证信息

```
http://api.example.com/resource/id
with basic authentication
username: token
password: the api key
```

这种方式的缺点十分明显，就是针对HTTP类型的请求，所有的信息都会暴露在外面，很容易就被破解掉，Github使用这种，是因为Github全站HTTPS了。

2.将API Token作为一个查询参数传递过去，发送给指定URL的请求如下

```
http://api.example.com/resource/id?token=api_key
```

这种方式的缺点类似上面那种方法。
回答中提出了还有三种方法——[原始链接](http://stackoverflow.com/questions/4968009/api-design-http-basic-authentication-vs-api-token) ：

- using an API key in the header (e.g. 'Authorization: Token MY_API_KEY') instead of as a url param. 在HTTP头里面存储token

- 亚马逊的解决方案，加入了签名的TOKEN，即在客户端用密钥和id签名出一个token，然后传递token，到服务器端再次签名，这样由于密钥并没有在网络上传输，基本安全的，同时亚马逊还使用了时间参数防止重放攻击，[链接](http://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html)

- OAUTH认证，这种需要辅以用户账号系统，通常涉及到服务器，客户端和用户三方的认证，即用户对这次请求进行认证。

研究新浪的API访问控制，发现它使用的是AccessToken，有两种方式来使用该AccessToken：

* API请求 URL 的后面加上一个AccessToken

* Http头里面加一个字段AccessToken=xxx

这种AccessToken是写死在程序里面的，在每次请求的时候附带上，对于这种AccessToekn新浪那边有过期时间，过期之后就无法再使用了。

很明显这种方式是不安全的，一旦别人获取到这个AccessToekn 就可以伪装身份使用该API，这个访问控制就形同虚设了。但是新浪也知道这一点，所以利用这种方式使用的接口都是较为基础的接口，高级一点的接口需要使用Oauth 2.0进行二次认证的访问控制。

# 2 我的解决方案

在我的项目中，没有像新浪微博账号这种的用户管理系统，做OAtuth认证不太合适，通过参考网上大神的资料，大致想了下面这种访问控制的办法。

1、为每个应用颁发一个账号（user）和密码（password），相当于（公钥和私钥）。

2、服务器后台存储该账号和密码（密文存储 MD5加密）。

3、应用端在通过HTTP请求该接口的时候，需要在HTTP HEADER 附带下面几个字段:

```
时间: date=unix时间戳
签名: sign=sign
```

该sign的生成算法是这样的：

```
String sign = HTTPMETHOD（GET/POST）+ api_uri（API的访问URI）+date（即上面的UNIX时间戳）+length（发送body的数据长度）+password（后台颁发的密码）;
sign = MD5(sign);
sign = user+":"+sign;
```

4、后台服务器在接收到请求后：

- 比对HTTP头中的时间戳，比对服务器时间，如果超过某个阈值，则拒绝访问，同时返回请校准你的应用时间。

- 如果没有超过时间阈值，则从sign中取出user然后在数据库中查找对应的password，然后同样根据上面的sign生成算法，来生成sign进行身份认证，认证成功则执行API，失败则返回认证失败。

# 3 参考文献

[1] [stackoverflow:api-design-http-basic-authentication-vs-api-token](http://stackoverflow.com/questions/4968009/api-design-http-basic-authentication-vs-api-token) 

[2] [aws Rest Authentication](http://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html)

[3] [api-双方认证探讨](http://xiezhenye.com/2013/03/api-%E5%8F%8C%E6%96%B9%E8%AE%A4%E8%AF%81%E6%8E%A2%E8%AE%A8.html)

    
