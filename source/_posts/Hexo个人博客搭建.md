---
title: Hexo个人博客搭建+自动部署
date: 2024-06-10 13:14:18
tags: Blog
categories: Blog
cover: https://test-1301661941.cos.ap-nanjing.myqcloud.com/501718002288_.pic.jpg
---
## 个人博客搭建

近期通过Hexo 实现了一个 个人博客，用来分享和记录一写日常。下面将详细介绍下整个搭建过程。

### 准备工作

* 安装并跑通一个本地Hexo
* 新建一个GitHub仓库将本地的Hexo项目push到仓库
* 一个腾讯云服务器，通过webhook服务来实现GitHub后的自动部署，配置Nginx对端口的映射
* 申请域名，ssl证书

> 下面将详细介绍怎么实现

### 安装并跑通一个本地Hexo

+ 安装Node.js和Git：Hexo需要Node.js和Git来运行。如果你还没有安装，请先安装它们。你可以从它们的官方网站下载并按照说明进行安装。
  具体安装方法可以参考：<a href='https://hexo.io/zh-cn/docs/'>Hexo 文档</a>
+ 根据文档安装和启动后，访问项目地址后即可看到一个hello world默认主题的页面（此时已经完成本地项目），至于主题的更换此处不在说明。


### 新建一个Github仓库并且将刚本地的Hexo项目push到仓库中

新建一个Github仓库后将本地的Hexo项目push上去后，可以得到一个这样结构的GitHub仓库：

<img src='https://test-1301661941.cos.ap-nanjing.myqcloud.com/github.png'/>
  
此时完成仓库的构建，在此之后可以通过本地编写 .md 文档后push到次仓库中。下面将介绍在每次Push后如何自动部署到自己的腾讯云服务器上。

### 云服务器申请

一腾讯云为例，去腾讯云官网租一个自己经济能力范围内的服务器即可，如下图所示，如果有学生优惠的情况下大概在100¥以内能租个一年。
<img src='https://test-1301661941.cos.ap-nanjing.myqcloud.com/tengxunyun.png'/>

租完服务器后至于怎么链接，本文不在叙述，如不回使用linux的同学可花1-3天去b站学习下常用的基础命令。

此时有了一台云服务器后，要对此服务器的Git、node 环境进行搭建。搭建完成后即可在云服务器上拉Git仓库中的代码到服务器上，然后在服务器上运行起来并且通过浏览器中输入服务器Ip+hexo项目的端口号访问此项目。此时项目已经实现了外网的部署，其他人就可以通过你的服务器ip+端口号访问了。
<img src='https://test-1301661941.cos.ap-nanjing.myqcloud.com/ip%2Bport.png'/>

接下来是实现github 提供的 webhook在本地项目每次push后来通知到我们的云服务器，云服务器收到github发来的post请求后重新在云服务器中Hexo项目地址中重新通过git拉项目代码，重新执行Hexo 打包命名等。此过程需要我们配置github webhook向我们的服务器发送push通知。具体操作如下：

* 在github仓库中点击settings
<img src='https://test-1301661941.cos.ap-nanjing.myqcloud.com/git1.png'/>

* 然后点击webhooks
<img src='https://test-1301661941.cos.ap-nanjing.myqcloud.com/git2.png'/>

* add webhook 添加webhook
<img src='https://test-1301661941.cos.ap-nanjing.myqcloud.com/git3.png'/>

* 填写相关信息后 点击Add webHook 确认
<img src='https://test-1301661941.cos.ap-nanjing.myqcloud.com/git4.png'/>


至此github 的webhook 完成。此时我们每次在本地编写完成 .md 文章后 push到GitHub 仓库后，githbu 就会向我们刚刚填写的那个web hooks地址中发送一个post请求。那个地址是我们的云服务器，所以我们要在我们的云服器上启动一个webhook服务来接收和处理这个请求。具体实现我们使用puthon来实现一个简单的后端服务：

```python
from flask import Flask, request, abort
import subprocess

hexo_project_path = '/home/ubuntu/HexoBlog'
app = Flask(__name__)

@app.route('/webhook', methods=['POST'])
def webhook():
    # 验证请求是否来自 GitHub

    if request.headers.get('X-GitHub-Event') == 'push':
        payload = request.json
        # 检查仓库名称
        if payload['repository']['full_name'] == 'zhangtuo723/HexoBlog':
            # 拉取最新代码并执行部署命令
            subprocess.run(['git', 'pull', 'origin', 'main'],cwd=hexo_project_path)
            subprocess.run(['hexo', 'clean'],cwd=hexo_project_path)
            subprocess.run(['hexo', 'generate'],cwd=hexo_project_path)
            subprocess.run(['hexo', 'deploy'],cwd=hexo_project_path)
            return 'Deployed Hexo successfully', 200
    # 请求不是来自 GitHub 或者仓库名称不匹配，返回 400 错误
    abort(400)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3000)

```

其目的是收到/webhook 发来的请求，如果是push时间，则进行项目的自动部署。其流程为：重新拉仓库代码，以及执行 hexo 相关启动项目代码。

到此我们完成和项目的自动部署，整个交互流程为：
* 我们本地编写.md文章
* push到github仓库
* github自动通过配置的webhook 向我们的云服务器发送一个post请求
* 我们的服务器收到请求后重新拉仓库最新资源代码，然后执行hexo项目运行


### 申请域名和ssl 以及配置Nginx

完成以上配置后我们的项目即可通过云服务器的ip地址+端口访问了，为了方便访问所以要去申请一个域名来和我们的服务器地址绑定，将申请的域名映射到服务器的80和443端口，然后我们在服务器上通过Nginx 将这个个端口转发哦到我们的具体的hexo 服务地址和端口上。

域名的申请便宜一点的几块钱即可申请1年。以腾讯云为例，申请完成后在控制台将域名映射到我们的云服务器地址
<img src='https://test-1301661941.cos.ap-nanjing.myqcloud.com/nds.png' />

如果要使用443端口，也就是https服务的话，要额外的申请ssl证书，然后将这个证书上传到我们的云服务器，并且在nginx配置哪里倒入证书和key。nginx 具体配置如下：

``` nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://43.140.197.245:4000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        add_header Cache-Control "public, max-age=60";
    }
}


server {
    listen 443 ssl;
    server_name _;
    ssl_certificate /home/ubuntu/tuoz.vip_nginx/tuoz.vip_bundle.crt;
    ssl_certificate_key /home/ubuntu/tuoz.vip_nginx/tuoz.vip.key;
    location / {
        proxy_pass http://43.140.197.245:4000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    
        add_header Cache-Control "public, max-age=60";
        }
 }

```

其中：
+ ssl_certificate /home/ubuntu/tuoz.vip_nginx/tuoz.vip_bundle.crt;
+ ssl_certificate_key /home/ubuntu/tuoz.vip_nginx/tuoz.vip.key;
  
为ssl证书的地址。至此完成整个项目的搭建和部署

体验demo：<a href="https://www.tuoz.vip"> 我的博客🚀 </a>

### 总结

完成了Hexo的构建，掌握了一些项目编写到自动部署的一些基础技术。

🍠奥利给！！