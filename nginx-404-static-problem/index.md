# nginx子路径部署遇到404问题


本文介绍了当在nginx下通过子路径部署服务时，遇到找不到静态资源，报404错误问题的解决方法。
<!--more-->

## 问题背景
一般想通过多个一个nginx，利用多个子路径部署多个服务时，虽然服务可以正常起来，但当访问这些页面时，页面中的css、js静态资源文件会找不到，报`404`错误。
并且你会发现，浏览器查找的这些资源文件的跟自己预估的不符(资源文件实际在子路径对应的目录里，而浏览器却是到根路径下寻找)。

## 解决方式
一般遇到该问题，是因为子路径目录下的index.html中配置的资源(js、img)路径是都是针对nginx的根目录去寻找的，并不是去子路径下寻找。这种问题的解决办法有两种：
#### 1. nginx中配置根据`正则规则`过滤路径，使这些特殊路径去子路径下去查找

``` nginx
	  location ~* \.(js|css|gif|ico|bmp|png|jpg|jpeg|flv|swf|xap|woff)$ {
	      root        /path/to/static;
	  }
```
#### 2. 将index.html和其他页面中的资源链接路径进行更改，加上子路径
这种一般一些进行二次打包的项目，如VitePress会告诉怎么配置(见这里:[App Configs | VitePress (vuejs.org)](https://vitepress.vuejs.org/config/app-configs#base))，然后使打包生成的文件中链接加上子路径。
此时nginx中的配置方法见下：

a. 多加一个`location`：

```nginx
    # 将打包后的文件和资源放到/sur/share/nginx/html/docs/目录下
    location /docs/ {
        root /usr/share/nginx/html;
        index index.html;
        autoindex on;
    }
```


b. 多加一个`server`，使用方向代理：

```nginx
    # 该location放在一个server里面
    location /docs/ {
        proxy_pass http://localhost:8082/;
        #    # 需要添加的代码
        # proxy_set_header X-Real-IP $remote_addr;

        # proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # proxy_set_header Host $host;
        # proxy_set_header X-NginX-Proxy true;
    }

    # 新加一个server，作为反向代理
    server {
        listen 8082;
        server_name  localhost;

        location / {
            # 该路径可以自己进行定义
            root /usr/share/nginx/docs;
            index index.html;
        }
    }
```

## 参考
[问题解决1：nginx反向代理丢失js、css问题 - 简书 (jianshu.com)](https://www.jianshu.com/p/f0cdbc691c85)

---

> 作者: geekby  
> URL: https://www.geekby.cn/nginx-404-static-problem/  

