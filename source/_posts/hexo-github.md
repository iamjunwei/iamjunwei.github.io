---
title: hexo+github搭建自己的博客
date: 2016-03-04 13:42:18
categories: hexo
tags: hexo
---
## 前言

前两天看了篇文章《[我为什么坚持写博客？](https://mp.weixin.qq.com/s?__biz=MzA4NTQwNDcyMA==&mid=402564613&idx=1&sn=d2b7c75b11046a0dcf8df77e737d2b4c&scene=0&pass_ticket=C8WkBHPS440SZCkLCXZUWdrKgRD747hKE%2FYCSOsT8uou03NRavHIDfG1DiA6c%2Bxd)》，颇有感触，于是乎萌生了写博客的冲动。目前比较流行[Hexo](https://hexo.io/)+[Github](https://github.com/)来搭建博客，网上相关的资料也比较多。这篇文章主要是对自己搭建的过程进行一个总结。


## 配置环境

* 在[Node官网](https://nodejs.org/en/download/)下载安装文件并安装。
* 安装完成后，配置环境变量
* [参考文献](http://jingyan.baidu.com/article/a948d6515d4c850a2dcd2e18.html)

## 创建Github repository

* 下载并安装[Git](http://git-scm.com/download/)
* 注册一个Github账号（这里就不赘述了），登录成功以后，点击New repository。
* 填写基本信息，如下图所示。序号1填写repository name，其中xxx可替换成你的名称。序号2填写repository的描述。为方便起见，序号3可勾选，Github会自动产生一次commit和readMe文件。点击序号4创建repository。蓝色圈圈里的为你的repository的ownername

![createGithubRepository](/images/createGithubRepository.jpg)

* 创建成功后，默认的分支为master。创建一个新的分支hexo。master分支用于存放博客的静态页面，hexo分支用于存放hexo的网页生成代码。点击红色圈圈中的按钮，复制仓库地址。

![createGithubHexoBranch](/images/createGithubHexoBranch.jpg)

## 安装部署Hexo，并关联Github repository
* 本地创建文件夹（如E:\hexo，后面直接称呼文件夹），右键选择Git Bash
* 运行npm install -g hexo，等待安装完成即可。
* 执行hexo init，等待部署完成。
* 运行npm install hexo-deployer-git --save，等待安装完成即可。
* 执行git init。完成后，当前分支为master。
* 执行git remote add origin git@github.com:ownername/xxx.github.io.git，其中ownername和xxx都替换成之前创建repository时记录和修改的内容。
* 执行git pull，拉取远程仓库的分支。成功后执行git branch -a可以看到红色的内容是远程仓库，绿色的是本地仓库的当前分支。
* 修改文件夹中的_config.yml文件。在文件中找到deploy标签，并修改如下。
``` bash
deploy:
  type: git
  repository: https://github.com/ownername/xxx.github.io.git
  branch: master // 静态页面存在仓库的master分支
```
* 依次执行hexo clean，hexo generate，hexo deploy，然后提示输入用户名密码，用户名为ownername，密码为Github的登录密码。
* 等待一段时间后，浏览器中打开http://xxx.github.io/，就能看到博客的页面了。

## 维护博客
* 首先需要对hexo进行配置，参考[官方文档](https://hexo.io/docs/)
* 选择一款自己喜欢的主题，[主题下载](https://hexo.io/themes/)
* 每一款主题都有自己的文档，参照文档进行设置。我选择的是[NexT](http://theme-next.iissnan.com/)
* 设置完成后就可以写博客了，语法为Markdown。[参考网站](http://sspai.com/25137)，[在线编辑器](https://www.zybuluo.com/mdeditor)
* 有时候文章会出现乱码，需要保存为utf-8。（这篇文章是Sublime编辑的，Preference里的Setting Default设置）
* 执行hexo clean，hexo generate，hexo deploy将修改发布到远程仓库，等待一段时间后，就可以看到这篇文章了。

## 多地更新
由于远程仓库的hexo分支存储了hexo的网站生成代码，因此在不同的电脑上都可以将代码拉下来，进行编辑和维护，实现多地更新博客。
