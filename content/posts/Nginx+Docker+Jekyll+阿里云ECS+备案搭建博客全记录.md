---
title: "Nginx+Docker+Jekyll+阿里云ECS+备案搭建博客全记录"
date: 2019-08-28 00:00:00.0
draft: false
type: "post"
showTableOfContents: true
tags: ["misc"]
---

---
layout:     post
title:      "Nginx+Docker+Jekyll+阿里云ECS+备案搭建博客全记录"
date:       2019-08-28
author:     "马赛琦"
tags:
    - Docker
    - Jekyll
    - Nginx
---

> “记录本站搭建全过程，为想要自己搭建博客的朋友提供一个参考”

## 前言与准备

### Docker

Docker是一个很方便的跨平台应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的镜像中，然后发布到任何流行的Linux或Windows机器上。Docker中各个`容器`质检各自是一个“沙箱”，彼此隔离，但其也提供了丰富的容器之间互联的方式使得各个容器质检能够既保持独立，又能彼此交互。Docker也提供了`挂载`这一功能，开发者可以将一些容器的数据挂载到宿主机的指定位置实现数据层面的宿主机与容器质检的互联。

本文假设你已经安装了Docker并已知晓基础的使用方法，如果你之前没有使用过Docker，你需要首先自行了解一些Docker的基本概念：`容器`，`镜像`，`挂载`。此外，你还需要一个已经安装好的Docker环境，这一点笔者不复赘述，百度中教程还是挺多的，笔者更推荐官网文档，都是一些很基础的单词，应该大体可以理解。

### Nginx

Nginx是一个是一个高性能的HTTP和反向代理web服务器，本文的博客将依托Nginx进行访问。

### Jekyll

Jekyll是一个优秀的静态博客网站生成器。Jekyll被广泛用于很多程序员的博客，很多Jekyll博客都是搭建在Github Pages上，通过一些简单的命令就能够快速将markdown的文件转化为对应的静态网页文件并部署上传到Github Pages中去。

虽然Jekyll最广泛的使用方法还是部署到Github Pages中，但是Github Pages不利于百度等国内搜索引擎SEO优化(Github官宣禁止百度爬虫，认为不安全)。当然，国内很多优秀的代码管理平台，比如笔者之前使用的Coding，也支持`Pages`服务，也是一个不错的选择。

笔者本身手头上恰好有阿里云的服务器，再加上`Pages`服务本身限制还是挺多的，如果是自己的服务器很多地方就更加方便，因此就开始尝试自己搭建环境进行部署。

与Jekyll相比较的有一个叫做Hexo的静态博客系统，总体上和jekyll相似，大家可以按照喜好选用，本文以Jekyll为例展开讲解。

## 让我们开始吧

### 安装Docker(ubantu为例)

[官网文档](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

除此以外，笔者也推荐安装完顺便安装一下Portainer，这个是一个很方便的Docker图形化管理工具，当然这不是本文的必需品。

### 安装Nginx到Docker中  

**这里第一次创建实例的目的其实是为了拿默认的配置文件，然后我们就可以剥离出来放在宿主机里做配置，你如果之前已经有Nginx的主配置文件，也可以省去第一次创建实例的一步。**

- 拉取Nginx镜像
````
docker pull nginx
````

- 创建挂载目录(分别对应日志目录、配置目录、网站文件目录)
````
mkdir /home/nginx/log
mkdir /home/nginx/config
mkdir /home/nginx/html
````

- 启动一个Nginx容器实例
````
docker run --name nginx-test -p 80:80 -d nginx
````

- 查看实例的短id
````
docker ps
````

- 取出默认的配置文件复制到宿主机目录中
````
docker cp 短id:/etc/nginx/nginx.conf /home/nginx/
````

- 移除刚刚创建的Nginx实例
````
docker rm -fv 短id/实例名
````

- 重新创建一个拥有挂载目录的，能够自动重启的Nginx实例
````
docker run —name docker_nginx -d -p 80:80 -p 443:443\
—restart always\
-v /home/nginx/log:/var/log/nginx \
-v /home/nginx/config:/etc/nginx/conf.d \
-v /home/nginx/nginx.conf:/etc/nginx/nginx.conf \
-v /home/nginx/html:/usr/share/nginx/html nginx
````

- 至此Nginx就算配置安装完成了，可以尝试访问一下ip，应该有`Welcome to nginx`的提示

### 安装Jekyll环境(本地电脑安装即可)

- 首先你需要安装Ruby环境，这一步因为系统环境不一样可能安装步骤也不一样，Mac系统可以[参考这里](https://www.jianshu.com/p/c073e6fc01f5)，其他用户百度查询一下应该还是有很多教程的
- 安装完Ruby环境后便可以安装Jekyll了，目前[官网](https://jekyllrb.com/)提供的安装命令是：
````
 gem install bundler jekyll
````

- 至此，Jekyll就安装完了，你可以输入一下命令创建一个简单的Jekyll博客进行预览(一条一条输入)：
````
jekyll new my-awesome-site
cd my-awesome-site
jekyll server
````

- 访问127.0.0.1:4000就可以查看创建的博客了

### 选用Jekyll主题

笔者这里使用的是Hux(黄轩)先生制作的Jekyll主题，在此也感谢Hux的心血付出。

主题的GitHub地址：[点击此处](https://github.com/Huxpro/huxpro.github.io)

### 生成静态博客文件并部署到Nginx中

- 输入一下命令生成静态文件：
````
jekyll build
````

- 你的博客目录下应该会生成一个文件夹`_site`，将此文件夹下所有内容上传到服务器中的`/home/nginx/html`中(之前配置的Nginx网站目录)
- 访问你的服务器ip，大功告成

### 后续备案

#### 工信备案
笔者使用的阿里云ECS，备案流程相对简单，直接在阿里云官网点击备案，按照流程一步一步来就可以了

#### 公安备案

网上在线提交相关材料就可以，需要注意一个问题，截止目前官网上传图片的方式还是flash的方式，可能会出现无法上传等问题，笔者后来是在一台Windows电脑下使用Firefox浏览器并用手机热点进行联网才上传成功的

## 总结

这次博客从开始计划到备案完成总共花了大概一周时间(博客部署和备案并行)。
关于备案，笔者第一次备案，工信部审核大概花了5天时间，公安部审核花了2天时间。