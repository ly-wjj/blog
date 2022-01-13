

+++

author = "lcomedy"
title = "手把手教你快速发布博客网站文章"
date = "2022-01-13"
description = "快速发布文章到博客网站"
tags = [
    "blog",
    "hugo",
    "pratice",
    "cicd",
]
categories = [
    "pratice",
]
series = ["博客网站"]
aliases = ["搭建博客网站"]

+++

我之前介绍过如何搭建一套自己的博客网站，现在我想发快速发布一遍文章到自己的博客上，怎么办呢？

## 前期准备

1. github账号
2. markdown编辑器
3. git工具

## 原理

1. 上传文件
2. 触发action
3. 获取文件
4. 渲染文件

## 设计分析

![image-20220113233547143](images/image-20220113233547143.png)教程

### webhook-server

我们使用python3，快速编写一个webhook的服务端，用于接受github的webhook请求。

```shell
## 服务器上安装flask
pip3 install flask
```

1. 编辑gen_secret.py，执行python3 gen_secret.py，得到一个对应的secret

   ```python
   # -*- coding=utf8 -*-
   import os
   import base64
   import random
   import time
   import hashlib
   
   # 方法一
   tmp = os.urandom(44)
   secret_key = base64.b64encode(tmp)
   print(secret_key)
   ```

2. 编写webhook-server.py

   ```python
   import hmac
   import os
   from flask import Flask, request, jsonify
   
   app = Flask(__name__)
   app.debug = False
   ## 定义sercret,可以通过脚本生成一个secret
   github_secret = 'xxxxxxxx'
   
   def encryption(data):
       key = github_secret.encode('utf-8')
       obj = hmac.new(key, msg=data, digestmod='sha1')
       return obj.hexdigest()
   
   @app.route('/api/v1/webhook/deploy', methods=['POST'])
   def post_data():
       post_data = request.data
       token = encryption(post_data)
       # 认证签名是否有效
       signature = request.headers.get('X-Hub-Signature', '').split('=')[-1]
       if signature != token:
           return "token is not vaild", 401
       os.system('sh /opt/webhook/deploy.sh')
       return jsonify({"status": 200})
   
   if __name__ == '__main__':
       app.run(port=8989)
   ```

3. 编写一个deploy.sh，主要是如下作用：

   1. 拉取存储博客文章的代码仓库

   2. 将文章移动到对应的hugo目录下

   3. 使用hugo命令渲染出新的前端文件

   ```shell
   #!/bin/bash
   git_path="/data/blog"
   ## 填写自己的github仓库地址
   git_url="xxxx"
   blog_path="/data/hugo-blog/lcomedy/content"
   backup_path="/data/backup"
   
   if [ ! -d "$git_path" ];then
     git clone $git_url $git_path
   fi
   cd $git_path && git pull --force
   if [ ! -d "$backup_path" ];then
     mkdir $backup_path
   fi
   rm -rf $backup_path/*
   mv -f $blog_path/* $backup_path
   cp -rf $git_path/* $blog_path/
   cd $blog_path/../ && hugo --baseUrl="/" -t  coder
   ```

4. 配置nginx，在blog.lcomedy.tech的域名下，这个之前上一篇文化文章提到过，/etc/nginx/conf.d/hugo.conf中进行配置。

   ```nginx
     location = /api/v1/webhook/deploy {
       	proxy_pass http://127.0.0.1:8989;
         proxy_set_header X-Real-IP $remote_addr;
         proxy_set_header Host $host;
         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
     }
   ```

5. 配置webhook的service,vim /lib/systemd/system/webhook.service

   ```
   [Unit]
   Description=webhook server daemon
   After=multi-user.target
   
   [Service]
   Type=simple
   ExecStart=/usr/bin/python3 /opt/deploy/server.py
   ExecReload=/bin/kill -HUP $MAINPID
   KillMode=process
   Restart=on-abort
   ExecStop=/bin/kill $MAINPID
   
   [Install]
   WantedBy=multi-user.target
   ```

6. 最终启动webhook服务，重启nginx

   ```shell
   systemctl daemon-reload
   systemctl enable webhook
   systemctl start webhook
   docker restart nginx
   ```

### 配置gihub

#### 创建blog的代码仓库

  在自己的github中创建一个blog的仓库。

#### 配置访问权限

1. 访问setting

   ![image-20220113225254617](images/image-20220113225254617.png)

2. 设置deploy.key

   ![image-20220113225354395](images/image-20220113225354395.png)

3. 将公钥拷贝到github中

#### 配置webhook

1. github的对应的仓库里，选择setting，配置webhooks

   ![image-20220113225531048](images/image-20220113225531048.png)

2. 输入webhook定义的地址，以及secret

3. 选择Just the `push` event.

### 发布文章

前面按步骤配置完成后，进行如下操作

1. git clone 仓库地址
2. 创建posts目录
3. 在posts目录下，进行文章的保存
4. git push 到远程仓库

通过git push触发webhook，最终将文章渲染到博客网站。大功告成！

