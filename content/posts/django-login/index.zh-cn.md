---
weight: 14
title: "Django 实现登录后跳转"
date: 2019-08-29T16:56:04+08:00
lastmod: 2019-08-29T16:56:04+08:00
draft: false
author: "JackBai"
authorLink: "https://www.geekby.cn"
description: "这篇文章介绍了怎样在Django中实现登录后跳转."
featuredImage: "https://cdn.jsdelivr.net/gh/jackbai233/image-hosting@master/20211024/django-login.303wk0edalg0.png"

tags: ["Django"]
categories: ["Python", "后端"]

lightgallery: true
---

这篇文章介绍了怎样在Django中实现登录后跳转.

<!--more-->

## 说明
> 实现网页登录后跳转应该分为两类：即*登录成功后跳转*和*登录失败再次登录成功后跳转*。参考网上内容，基本都只实现了第一类。而没有实现第二类。
## 实现
为了能让登录失败后再次登录成功后还能实现跳转。我这里采用了笨办法， 即：**无论登录成功与否，都将跳转链接在前后端进行传递** ,这样跳转链接就不会在登录失败后消失。不多说，上代码
### 后端 views.py
```python
from django.shortcuts import render, redirect

def login(request):
    # 当前端点击登录按钮时，提交数据到后端，进入该POST方法
    if request.method == "POST":
        # 获取用户名和密码
        username = request.POST.get("username")
        passwd = request.POST.get("password")
        # 在前端传回时也将跳转链接传回来
        next_url = request.POST.get("next_url")

        # 对用户进行验证，假设正确的用户名密码为"aaa", "123"
        if username == 'aaa' and passwd == '123':
            # 判断用户一开始是不是从login页面进入的
            # 如果跳转链接不为空并且跳转页面不是登出页面，则登录成功后跳转，否则直接进入主页
            if next_url and next_url != "/logout/":
                response = redirect(next_url)
            else:
                response = redirect("/index/")
            return response
        # 若用户名或密码失败,则将提示语与跳转链接继续传递到前端
        else:
            error_msg = "用户名或密码不正确，请重新尝试"
            return render(request, "app/login.html", {
                'login_error_msg': error_msg,
                'next_url': next_url,
            })
    # 若没有进入post方法，则说明是用户刚进入到登录页面。用户访问链接形如下面这样：
    # http://host:port/login/?next=/next_url/
    # 拿到跳转链接
    next_url = request.GET.get("next", '')

    # 直接将跳转链接也传递到后端
    return render(request, "autotest/login.html", {
        'next_url': next_url,
    })
```

### 前端页面 login.html

```html
   <form action="{% url 'login' %}" method="post">
                 <h1>请使用xxx登录</h1>
                 <div>
                   <input id="user" type="text" class="form-control" name="username" placeholder="账户" required="" />
                 </div>
                 <div>
                   <input id="pwd" type="password" class="form-control" name="password" placeholder="密码" required="" />
                 </div>
       // 注意这里多了一个input。它用来保存跳转链接，以便每次点击登录按钮时将跳转链接传递回后端
       // 通过display：none属性将该input元素隐藏起来
                   <div style="display: none;">
                       <input id="next" type="text" name="next_url" value="{{ next_url }}" />
                   </div>
       // 判断是否有错误提示，若有则显示
                   {% if login_error_msg %}
                       <div id="error-msg">
                           <span style="color: rgba(255,53,49,0.8); font-family: cursive;">{{ login_error_msg }}</span>
                       </div>
                   {% endif %}
                 <div>
                     <button type="submit" class="btn btn-default" style="float: initial; margin-right: 60px">登录</button>
                 </div>
   </form>
```

## 总结
其实这种实现方式就是让跳转链接在前后端交互中不损失掉。当然也可以在前端不用form元素，直接用ajax的post形式，然后让跳转在前端的ajax逻辑中执行。