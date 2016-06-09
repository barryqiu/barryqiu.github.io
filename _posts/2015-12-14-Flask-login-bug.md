---
title: Flask Login的一个小坑
layout: post
categories: 填坑记录
---

这两天在做服务器迁移，要将一个Flask搭建的网站以及配套的后台接口程序移植到一台新的机器上，按照上一篇博客的内容配置好了服务器环境并且将代码git clone下来顺利运行起来了，但是一进入到登陆界面或者注册界面之后，就会报错，错误如下：

```bash
Traceback (most recent call last):
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/gunicorn/workers/sync.py", line 130, in handle
    self.handle_request(listener, req, client, addr)
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/gunicorn/workers/sync.py", line 171, in handle_request
    respiter = self.wsgi(environ, resp.start_response)
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/flask/app.py", line 1836, in __call__
    return self.wsgi_app(environ, start_response)
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/flask/app.py", line 1820, in wsgi_app
    response = self.make_response(self.handle_exception(e))
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/flask/app.py", line 1403, in handle_exception
    reraise(exc_type, exc_value, tb)
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/flask/app.py", line 1817, in wsgi_app
    response = self.full_dispatch_request()
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/flask/app.py", line 1477, in full_dispatch_request
    rv = self.handle_user_exception(e)
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/flask/app.py", line 1381, in handle_user_exception
    reraise(exc_type, exc_value, tb)
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/flask/app.py", line 1475, in full_dispatch_request
    rv = self.dispatch_request()
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/flask/app.py", line 1461, in dispatch_request
    return self.view_functions[rule.endpoint](**req.view_args)
  File "/opt/myapp/app/auth/views.py", line 19, in register
    return render_template('auth/register.html', form=form)
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/flask/templating.py", line 128, in render_template
    context, ctx.app)
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/flask/templating.py", line 110, in _render
    rv = template.render(context)
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/jinja2/environment.py", line 989, in render
    return self.environment.handle_exception(exc_info, True)
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/jinja2/environment.py", line 754, in handle_exception
    reraise(exc_type, exc_value, tb)
  File "/opt/myapp/app/templates/auth/register.html", line 2, in top-level template code
     import "bootstrap/wtf.html" as wtf 
  File "/opt/myapp/app/templates/base.html", line 1, in top-level template code
     extends "bootstrap/base.html
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/flask_bootstrap/templates/bootstrap/base.html", line 1, in top-level template code
     block doc 
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/flask_bootstrap/templates/bootstrap/base.html", line 4, in block "doc"
    - block html
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/flask_bootstrap/templates/bootstrap/base.html", line 20, in block "html"
     block body -
  File "/usr/local/qqwebserver/python/bin/myapp/lib/python2.7/site-packages/flask_bootstrap/templates/bootstrap/base.html", line 21, in block "body"
     block navbar 
  File "/opt/myapp/app/templates/base.html", line 33, in block "navbar"
    if current_user.is_authenticated()
AttributeError: 'bool' object has no attribute '__call__'
```

第一反应是我擦，在原来服务器上运行的好好的程序，为啥到这就出错了，依赖软件啥的都安装过了，看这个错误类型抱的是bool类型的对象没有属性__call__，看起来是把对象当成函数调用了，难道是Flask-login升级后把内置函数变成属性了？先看看软件版本吧，

新服务器

```bash
pip list
Flask-Login (0.3.2)
```

旧服务器

```bash
pip list
Flask-Login (0.2.11)
```

新旧服务器的Flask-login差了一个大版本号，果断先把新服务器的卸载了然后安装对应版本的软件

```bash
pip uninstall  Flask-Login
pip install  Flask-Login==0.2.11
```

然后再重新载入运行，果断好了。。


源码部分

下面是0.2.11版本的user对象的源码：

```python
class UserMixin(object):
    '''
    This provides default implementations for the methods that Flask-Login
    expects user objects to have.
    '''
    def is_active(self):
        return True

    def is_authenticated(self):
        return True

    def is_anonymous(self):
        return False

    def get_id(self):
        try:
            return unicode(self.id)
        except AttributeError:
            raise NotImplementedError('No `id` attribute - override `get_id`')

    def __eq__(self, other):
        '''
        Checks the equality of two `UserMixin` objects using `get_id`.
        '''
        if isinstance(other, UserMixin):
            return self.get_id() == other.get_id()
        return NotImplemented

    def __ne__(self, other):
        '''
        Checks the inequality of two `UserMixin` objects using `get_id`.
        '''
        equal = self.__eq__(other)
        if equal is NotImplemented:
            return NotImplemented
        return not equal

    if sys.version_info[0] != 2:  # pragma: no cover
        # Python 3 implicitly set __hash__ to None if we override __eq__
        # We set it back to its default implementation
        __hash__ = object.__hash__
```

下面这是0.3.2版本的源码

```python
class UserMixin(object):
    '''
    This provides default implementations for the methods that Flask-Login
    expects user objects to have.
    '''
    @property
    def is_active(self):
        return True

    @property
    def is_authenticated(self):
        return True

    @property
    def is_anonymous(self):
        return False

    def get_id(self):
        try:
            return unicode(self.id)
        except AttributeError:
            raise NotImplementedError('No `id` attribute - override `get_id`')

    def __eq__(self, other):
        '''
        Checks the equality of two `UserMixin` objects using `get_id`.
        '''
        if isinstance(other, UserMixin):
            return self.get_id() == other.get_id()
        return NotImplemented

    def __ne__(self, other):
        '''
        Checks the inequality of two `UserMixin` objects using `get_id`.
        '''
        equal = self.__eq__(other)
        if equal is NotImplemented:
            return NotImplemented
        return not equal

    if sys.version_info[0] != 2:  # pragma: no cover
        # Python 3 implicitly set __hash__ to None if we override __eq__
        # We set it back to its default implementation
        __hash__ = object.__hash__
```

可以看出来3版本相对于2版本，添加了一个@property的修饰器，导致了is\_authenticated()的调用方式无法成功，而我的项目代码中都是用的这种语法，至此真相大白。

总结：

* 2版本和3版本的is\_authenticated类型不同，2版本的取值方式是user.is\_authenticated()，3版本变成了user.is\_authenticated。
* python的服务程序迁移一定注意软件版本


