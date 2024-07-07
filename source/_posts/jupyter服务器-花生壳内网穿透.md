---
title: jupyter服务器+花生壳内网穿透
date: 2024-07-07 15:15:20
tags: jupyter
---





## 背景



在现代数据科学和机器学习的工作中，Jupyter Notebook 已成为数据分析、可视化和代码共享的重要工具。实现一个远程Jupyter 服务器就可以通过浏览器远程访问服务器资源写代码跑实验了～～🚀



## 工作

+ 一台内网GPU服务器
+ 安装anaconda（自带jupyter）
+ 启动Jupyter 服务器
+ 安装安装花生壳（做内网穿透）



## jupyter 安装和启动

首先就是要有一台能跑实验的服务器，然后再次服务器上安装Anaconda。以ubuntu为例，先去<a href='https://www.anaconda.com/'>ubuntu官网</a> 下载一个***.sh 的安装脚本，把它复制到这台ubutnu 服务器上，然后进入这个文件所在的目录。

+ bash ./xxx.sh 开始安装，基本上一路回车就行。

+ 修改这个.bashrc  ，在.bashrc 末尾添加。

  ```
  export PATH="~/anaconda3/bin":$PATH
  source ~/anaconda3/bin/activate
  ```

  然后执行source ～/.bashrc

+ 输入conda 命令确认下是否安装成功 或者通过 ”which python“ 命令查看pthon 的安装目录，是否有一条在anaconda目录下面的。

+ 配置jupyter 服务，首先生成配置文件，终端输入：

  ```
  jupyter notebook --generate-config
  ```

  编辑生成的配置文件，启用远程访问：

  ```
  ~/.jupyter/jupyter_notebook_config.py
  ```

  修改一下几行

  ```
  # 启用IP地址监听
  c.NotebookApp.ip = '0.0.0.0'
  
  # 禁用自动打开浏览器
  c.NotebookApp.open_browser = False
  
  # 设置Jupyter Notebook的端口
  c.NotebookApp.port = 8888
  
  # 设置notebook目录
  c.NotebookApp.notebook_dir = '/path/to/your/notebooks'
  
  ```

  设置密码或令牌

  ```
  jupyter notebook password
  ```

  输入密码和确认

+ 然后就是启动Jupyter服务了，可以创建一个jupyter服务的文件夹，进入这个文件夹并且在命令行输入

  ```
  jupyter notebook
  ```

  即可运行jupyter notebook，一般在通过后台运行命令nohup命令运行。此时链接内网的浏览器可以在输入内网的ip地址进行访问了：

  <img src='https://test-1301661941.cos.ap-nanjing.myqcloud.com/%E8%BF%B7%E5%AE%AB.png'/>

  注意：第一次访问需要输入之前配置的密码。此时已完成jupyter的安装和启动



## 内网穿透

安装 花生壳，去花生壳官网下载x x x.deb的安装包，复制到ubutu 服务器上通过dpkg安装

```
sudo dpkg -i xxxx.deb
```

安装成功后 终端会打印一些信息，其中有一SN码，复制下来。然后浏览器输入 b.oray.com 进入花生壳的管理后台，自行注册账号登陆。

<img src='https://test-1301661941.cos.ap-nanjing.myqcloud.com/WX20240707-160632.png'/>

点击添加设备后填入之前复制的SN码，如果之前忘记记录，则在服务端输入 sudo phddns status  重新查看。

添加完成设备后：点击 内网穿透-> 添加映射

<img src='https://test-1301661941.cos.ap-nanjing.myqcloud.com/WX20240707-161211.png'/>

填写相应信息后点击去定即可通过花生壳的域名访问我们本地的jupyter服务器了。

<img src='https://test-1301661941.cos.ap-nanjing.myqcloud.com/jp.png'/>

