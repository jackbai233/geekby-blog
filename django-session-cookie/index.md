# Django 用Session和Cookie分别实现记住用户登录状态


由于http协议的请求是无状态的。故为了让用户在浏览器中再次访问该服务端时，他的登录状态能够保留（也可翻译为该用户访问这个服务端其他网页时不需再重复进行用户认证）。我们可以采用Cookie或Session这两种方式来让浏览器记住用户。

<!--more-->

## Cookie与Session说明与实现
### Cookie
#### 说明
Cookie是一段小信息(数据格式一般是类似key-value的键值对)，由服务器生成，并发送给浏览器让浏览器保存(保存时间由服务端定夺)。当浏览器下次访问该服务端时，会将它保存的Cookie再发给服务器，从而让服务器根据Cookie知道是哪个浏览器或用户在访问它。（由于浏览器遵从的协议，它不会把该服务器的Cookie发送给另一个不同host的服务器）。
#### Django中实现Cookie
```python
from django.shortcuts import render, redirect

# 设置cookie
"""
key: cookie的名字
value: cookie对应的值
max_age: cookie过期的时间
"""
response.set_cookie(key, value, max_age)
# 为了安全，有时候我们会调用下面的函数来给cookie加盐
response.set_signed_cookie(key,value,salt='加密盐',...)

# 获取cookie 
request.COOKIES.get(key)
request.get_signed_cookie(key, salt="加密盐", default=None)

# 删除cookie
reponse.delete_cookie(key)
```

下面就是具体的代码实现了
views.py
```python
# 编写装饰器检查用户是否登录
def check_login(func):
    def inner(request, *args, **kwargs):
        next_url = request.get_full_path()
        # 假设设置的cookie的key为login，value为yes
        if request.get_signed_cookie("login", salt="SSS", default=None) == 'yes':
            # 已经登录的用户，则放行
            return func(request, *args, **kwargs)
        else:
            # 没有登录的用户，跳转到登录页面
            return redirect(f"/login?next={next_url}")

    return inner


# 编写用户登录页面的控制函数
@csrf_exempt
def login(request):
    if request.method == "POST":
        username = request.POST.get("username")
        passwd = request.POST.get("password")
        next_url = request.POST.get("next_url")

        # 对用户进行验证，假设用户名为：aaa, 密码为123
        if username === 'aaa' and passwd == '123':
            # 执行其他逻辑操作，例如保存用户信息到数据库等
            # print(f'next_url={next_url}')
            # 登录成功后跳转,否则直接回到主页面
            if next_url and next_url != "/logout/":
                response = redirect(next_url)
            else:
                response = redirect("/index/")
            # 若登录成功，则设置cookie，加盐值可自己定义取，这里定义12小时后cookie过期
            response.set_signed_cookie("login", 'yes', salt="SSS", max_age=60*60*12)
            return response
        else:
            # 登录失败，则返回失败提示到登录页面
            error_msg = '登录验证失败，请重新尝试'
            return render(request, "app/login.html", {
                'login_error_msg': error_msg,
                'next_url': next_url,
            })
    # 用户刚进入登录页面时，获取到跳转链接，并保存
    next_url = request.GET.get("next", '')
    return render(request, "app/login.html", {
        'next_url': next_url
    })


# 登出页面
def logout(request):
    rep = redirect("/login/")
    # 删除用户浏览器上之前设置的cookie
    rep.delete_cookie('login')
    return rep


# 给主页添加登录权限认证
@check_login
def index(request):
    return render(request, "app/index.html")
```
由上面看出，其实就是在第一次用户登录成功时，设置cookie，用户访问其他页面时进行cookie验证，用户登出时删除cookie。另外附上前端的login.html部分代码

```html
            <form action="{% url 'login' %}" method="post">
              <h1>请使xx账户登录</h1>
              <div>
                <input id="user" type="text" class="form-control" name="username" placeholder="账户" required="" />
              </div>
              <div>
                <input id="pwd" type="password" class="form-control" name="password" placeholder="密码" required="" />
              </div>
                <div style="display: none;">
                    <input id="next" type="text" name="next_url" value="{{ next_url }}" />
                </div>
                {% if login_error_msg %}
                    <div id="error-msg">
                        <span style="color: rgba(255,53,49,0.8); font-family: cursive;">{{ login_error_msg }}</span>
                    </div>
                {% endif %}
              <div>
                  <button type="submit" class="btn btn-default" style="float: initial; margin-left: 0px">登录</button>
              </div>
            </form>
```



### Session
#### Session说明
Session则是为了保证用户信息的安全，将这些信息保存到服务端进行验证的一种方式。但它却依赖于cookie。具体的过程是：服务端给每个客户端(即浏览器)设置一个cookie（从上面的cookie我们知道，cookie是一种”key, value“形式的数据，这个cookie的value是服务端随机生成的一段但唯一的值）。
当客户端下次访问该服务端时，它将cookie传递给服务端，服务端得到cookie，根据该cookie的value去服务端的Session数据库中找到该value对应的用户信息。（Django中在应用的setting.py中配置Session数据库）。
根据以上描述，我们知道Session把用户的敏感信息都保存到了服务端数据库中，这样具有较高的安全性。

#### Django中Session的实现
```python
# 设置session数据, key是字符串，value可以是任何值
request.session[key] = value
# 获取 session
request.session.get[key]
# 删除 session中的某个数据
del request.session[key]
# 清空session中的所有数据
request.session.delete()
```
下面就是具体的代码实现了：
首先就是设置保存session的数据库了。这个在setting.py中配置：(注意我这里数据库用的mongodb，并使用了django_mongoengine库；关于这个配置请根据自己使用的数据库进行选择，具体配置可参考[官方教程](https://docs.djangoproject.com/zh-hans/2.2/ref/settings/#std:setting-SESSION_ENGINE))

```python
SESSION_ENGINE = 'django_mongoengine.sessions'
SESSION_SERIALIZER = 'django_mongoengine.sessions.BSONSerializer'
```

views.py

```python
# 编写装饰器检查用户是否登录
def check_login(func):
    def inner(request, *args, **kwargs):
        next_url = request.get_full_path()
        # 获取session判断用户是否已登录
        if request.session.get('is_login'):
            # 已经登录的用户...
            return func(request, *args, **kwargs)
        else:
            # 没有登录的用户，跳转刚到登录页面
            return redirect(f"/login?next={next_url}")

    return inner


@csrf_exempt
def login(request):
    if request.method == "POST":
        username = request.POST.get("username")
        passwd = request.POST.get("password")
        next_url = request.POST.get("next_url")
        # 若是有记住密码功能
        # remember_sign = request.POST.get("check_remember")
        # print(remember_sign)

        # 对用户进行验证
        if username == 'aaa' and passwd == '123':
            # 进行逻辑处理，比如保存用户与密码到数据库

            # 若要使用记住密码功能，可保存用户名、密码到session
            # request.session['user_info'] = {
                # 'username': username,
                # 'password': passwd
            # }
            request.session['is_login'] = True
            # 判断是否勾选了记住密码的复选框
            # if remember_sign == 'on':
            #   request.session['is_remember'] = True
            # else:
                # request.session['is_remember'] = False

            # print(f'next_url={next_url}')
            if next_url and next_url != "/logout/":
                response = redirect(next_url)
            else:
                response = redirect("/index/")
            return response
        else:
            error_msg = '登录验证失败，请重新尝试'
            return render(request, "app/login.html", {
                'login_error_msg': error_msg,
                'next_url': next_url,
            })
    next_url = request.GET.get("next", '')
    # 检查是否勾选了记住密码功能
    # password, check_value = '', ''
    # user_session = request.session.get('user_info', {})
    # username = user_session.get('username', '')
    # print(user_session)
    #if request.session.get('is_remember'):
    #    password = user_session.get('password', '')
    #    check_value = 'checked'
    # print(username, password)
    return render(request, "app/login.html", {
        'next_url': next_url,
        # 'user': username,
        # 'password': password,
        # 'check_value': check_value
    })


def logout(request):
    rep = redirect("/login/")
    # request.session.delete()
    # 登出，则删除掉session中的某条数据
    if 'is_login' in request.session:
        del request.session['is_login']
    return rep


@check_login
def index(request):
    return render(request, "autotest/index.html")
```
另附login.html部分代码：
```html
            <form action="{% url 'login' %}" method="post">
              <h1>请使xxx账户登录</h1>
              <div>
                <input id="user" type="text" class="form-control" name="username" placeholder="用户" required="" value="{{ user }}" />
              </div>
              <div>
                <input id="pwd" type="password" class="form-control" name="password" placeholder="密码" required="" value="{{ password }}" />
              </div>
                <div style="display: none;">
                    <input id="next" type="text" name="next_url" value="{{ next_url }}" />
                </div>
                {% if login_error_msg %}
                    <div id="error-msg">
                        <span style="color: rgba(255,53,49,0.8); font-family: cursive;">{{ login_error_msg }}</span>
                    </div>
                {% endif %}
                // 若设置了记住密码功能
               // <div style="float: left">
               //     <input id="rmb-me" type="checkbox" name="check_remember" {{ check_value }}/>记住密码
               // </div>
              <div>
                  <button type="submit" class="btn btn-default" style="float: initial; margin-right: 60px">登录</button>
              </div>
            </form>
```
总的来看，session也是利用了cookie，通过cookie生成的value的唯一性，从而在后端数据库session表中找到这value对应的数据。session的用法可以保存更多的用户信息，并使这些信息不易被暴露。
## 总结
session和cookie都能实现记住用户登录状态的功能，如果为了安全起见，还是使用session更合适



参考文章：[https://blog.csdn.net/qq_34755081/article/details/82808537](https://blog.csdn.net/qq_34755081/article/details/82808537)


---

> 作者: [JackBai](https://github.com/jackbai233)  
> URL: https://www.geekby.cn/django-session-cookie/  

