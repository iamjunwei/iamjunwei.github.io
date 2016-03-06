---
title: hexo+github搭建自己的博客
date: 2016-03-04 13:42:18
categories: hexo
tags: hexo
---
## 前言

前两天看了篇文章《[我为什么坚持写博客？](https://mp.weixin.qq.com/s?__biz=MzA4NTQwNDcyMA==&mid=402564613&idx=1&sn=d2b7c75b11046a0dcf8df77e737d2b4c&scene=0&pass_ticket=C8WkBHPS440SZCkLCXZUWdrKgRD747hKE%2FYCSOsT8uou03NRavHIDfG1DiA6c%2Bxd)》，颇有感触，于是乎萌生了写博客的冲动。目前比较流行[Hexo](https://hexo.io/)+[Github](https://github.com/)来搭建博客，网上相关的资料也比较多。这篇文章主要是对自己搭建的过程进行一个总结。

# 准备工作
* 在[Node官网](https://nodejs.org/en/download/)下载安装文件并安装。[参考文献](http://jingyan.baidu.com/article/a948d6515d4c850a2dcd2e18.html)
* 下载并安装[Git](http://git-scm.com/download/)

## 创建Github repository

* 注册一个Github账号（这里就不赘述了），登录成功以后，点击New repository。（需要登录注册的邮箱进行验证）
* 填写基本信息，如下图所示。序号1填写repository name，其中xxx可替换成你的名称。序号2填写repository的描述。为方便起见，序号3可勾选，Github会自动产生一次commit和readMe文件。点击序号4创建repository。蓝色圈圈里的为你的repository的ownername

![createGithubRepository](/images/createGithubRepository.jpg)

* 创建成功后，默认的分支为master。创建一个新的分支hexo。master分支用于存放博客的静态页面，hexo分支用于存放hexo的网页生成代码。点击红色圈圈中的按钮，复制仓库地址。

![createGithubHexoBranch](/images/createGithubHexoBranch.jpg)

* 在任意文件夹右键选择Git Bash，执行命令ssh-keygen -t rsa -C "你的邮箱"。你的邮箱可以是注册Github账号的邮箱，一直回车完成命令。这一步是在电脑内生成Github的ssh的密钥对。
* 找到密钥文件夹（系统盘下，如C:\Users\username\.ssh文件夹下），打开id_rsa.pub文件，复制其内容。
* 打开之前创建好的Github repository页面，如下图所示。选择红色圈圈里的Settings选项。

![createHexoBlogGithubSetting](/images/createHexoBlogGithubSetting.jpg)

* 依次选择左侧菜单的SSH Key，右上角的New SSH Key，填写好Title，把复制好的id_rsa.pub中的内容粘贴到下方Key的文本框中，最后点击Add SSH Key完成SSH Key的添加。

![createHexoBlogGithubSSHKey](/images/createHexoBlogGithubSSHKey.jpg)

## 安装部署Hexo，并关联Github repository
* 本地创建文件夹（如E:\hexo，后面直接称呼文件夹），右键选择Git Bash
* 运行npm install -g hexo，等待安装完成即可。
* 执行hexo init，等待部署完成。
* 运行npm install hexo-deployer-git --save，等待安装完成即可。
* 执行git init。完成后，当前分支为master。
* 执行git remote add origin git@github.com:ownername/xxx.github.io.git。需要注意的是，这几步中，ownername和xxx都替换成之前创建repository时记录和修改的内容。
* 执行git pull，拉取远程仓库的分支。成功后执行git branch -a可以看到红色的内容是远程仓库，绿色的是本地仓库的当前分支。
* 执行git checkout -b hexo origin/hexo。完成之后，本地分支为hexo，对应远程分支为hexo。
* 修改文件夹中的_config.yml文件。在文件中找到deploy标签，并修改如下：
``` bash
deploy:
  type: git
  repository: git@github.com:ownername/xxx.github.io.git
  branch: master
```
* 依次执行hexo clean，hexo generate，hexo deploy，等待发布完成。（第一次deploy会弹出一个openSSH的对话框，输入yes并回车）
* 等待一段时间后，浏览器中打开http://xxx.github.io/，就能看到博客的页面了（我等待了大概二十分钟）。

## 维护博客
* 首先需要对hexo进行配置，参考[官方文档](https://hexo.io/docs/)
* 选择一款自己喜欢的主题，[主题下载](https://hexo.io/themes/)
* 每一款主题都有自己的文档，参照文档进行设置。我选择的是[NexT](http://theme-next.iissnan.com/)
* 在文件夹中右键打开Git Bash，执行hexo new filename。其中filename是文章的文件名。新生成的文件位置在source/_post/filename.md。
* 编辑该文件的内容即可完成博客的书写。书写博客的语法为Markdown。[参考网站](http://sspai.com/25137)，[在线编辑器](https://www.zybuluo.com/mdeditor)，可以参考学习。一篇简单的博文内容如下。
``` bash
---
title: 博文标题
date: 2016-03-06 17:32:43
---
这是我的第一篇博客！
```
* 有时候文章会出现乱码，需要保存为utf-8。（这篇文章是Sublime编辑的，Preference里的Setting Default设置）
* 执行hexo clean，hexo generate，hexo deploy将修改发布到远程仓库master分支，等待一段时间后，就可以看到这篇文章了。

## 如果换了地点或者电脑，如何维护博客呢？
我们在创建repository的时候，创建了一个新的hexo分支，这个分支就是用于存放生成博客网站所需要的代码的。而master分支是存放生成出来的静态网页。
我们在新建和修改了博文以后，在文件夹中右键选择Git Bash，执行如下代码，其中commit message为提交的日志记录提示（如第一次文章）
``` bash
git pull
git add .
git commit -m "commit message"
git pull --rebase
git push origin hexo:hexo
```
执行完这些命令之后，我们就将网页生成代码提交到远程repository的hexo分支了。
若我们更换了地点或者电脑之后，只需要依次：
* 生成新的密钥对，并添加到Github repository的SSH Key。
* 任意文件夹右键选择Git Bash，执行Git clone git@github.com:ownername/xxx.github.io.git。
* 执行cd ./xxx.github.io，进入仓库文件夹。
* 执行git checkout -b hexo origin/hexo，拉取并切换到hexo分支。完成之后，文件夹中就是生成网页的代码了，接下来就可以尽情的撰写啦~
