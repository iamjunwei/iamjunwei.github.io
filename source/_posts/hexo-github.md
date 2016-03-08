---
title: hexo+github搭建自己的博客
date: 2016-03-04 13:42:18
categories: hexo
tags: hexo
---
## 1. 前言

前两天看了篇文章《[我为什么坚持写博客？](https://mp.weixin.qq.com/s?__biz=MzA4NTQwNDcyMA==&mid=402564613&idx=1&sn=d2b7c75b11046a0dcf8df77e737d2b4c&scene=0&pass_ticket=C8WkBHPS440SZCkLCXZUWdrKgRD747hKE%2FYCSOsT8uou03NRavHIDfG1DiA6c%2Bxd)》，颇有感触，于是乎萌生了写博客的冲动。目前比较流行[Hexo](https://hexo.io/)+[Github](https://github.com/)来搭建博客，网上相关的资料也比较多。这篇文章主要是对自己搭建的过程进行一个总结。

## 2. 准备工作
2.1. 在[Node官网](https://nodejs.org/en/download/)下载安装文件并安装。[参考文献](http://jingyan.baidu.com/article/a948d6515d4c850a2dcd2e18.html)
2.2. 下载并安装[Git](http://git-scm.com/download/)。
2.3. **请注意**，我博客的名字是**iamjunwei**，下面介绍的搭建过程中，均是以**iamjunwei**为例进行说明的。你可以用自己喜欢的名字，替换**iamjunwei**即可。

## 3. 创建Github repository

3.1. 注册一个Github账号（username填入**iamjunwei**，填写邮箱密码即可）。登录成功以后，点击New repository。（需要登录注册的邮箱进行验证）
3.2. 填写基本信息，如下图所示。Repository name填入iamjunwei.github.io，Description填入博客的描述，勾选Initialize this repository with a README，点击Create repository，完成创建过程。

![createGithubRepository](/images/createGithubRepository.jpg)

3.3. 创建成功后，默认的分支为master。创建一个新的分支hexoblog。master分支用于存放博客的静态页面，hexoblog分支用于存放hexo的网页生成代码。

![createGithubHexoBranch](/images/createGithubHexoBranch.jpg)

3.4. 在任意文件夹右键选择Git Bash，执行以下命令。其中“你的邮箱”可以是注册Github账号的邮箱。该命令会在电脑内生成Github的ssh的密钥对。具体的操作详见[生成并管理电脑内的ssh key](/2016/03/08/ssh-key/)。
```
ssh-keygen -t rsa -C "你的邮箱"
```
3.5. 找到密钥文件夹（系统盘下，如C:\Users\username\\.ssh文件夹下），打开上一步生成的对应pub文件，复制其内容。
3.6. 打开之前创建好的Github repository页面，选择右上角下拉按钮中的Settings选项，如下图所示。

![createHexoBlogGithubSetting](/images/createHexoBlogGithubSetting.jpg)

3.7. 依次选择左侧菜单的SSH Keys，右上角的New SSH Key，填写好Title，把复制好的pub文件内容粘贴到下方Key的文本框中，最后点击Add SSH Key完成SSH Key的添加。

![createHexoBlogGithubSSHKey](/images/createHexoBlogGithubSSHKey.jpg)

## 4. 安装部署Hexo，并关联Github repository
4.1. 本地创建文件夹（如E:\hexoblog，后面直接称呼blogfolder），右键选择Git Bash。
4.2. 依次运行以下命令。完成之后，本地分支为hexoblog，对应远程分支为hexoblog。
``` bash
npm install -g hexo
hexo init
npm install hexo-deployer-git --save
git init
git remote add origin git@github.com:iamjunwei/iamjunwei.github.io.git
git pull
git checkout -b hexoblog origin/hexoblog
```
4.3. 修改blogfolder文件夹中的_config.yml文件。在文件中找到deploy标签，并修改如下：
``` bash
deploy:
  type: git
  repository: git@github.com:iamjunwei/iamjunwei.github.io.git
  branch: master
```
4.4. 依次执行以下命令，等待发布完成。（第一次deploy可能会提示警告，输入yes并回车）
``` bash
hexo clean
hexo generate
hexo deploy
```
4.5. 等待一段时间后，浏览器中打开`http://iamjunwei.github.io`，就能看到博客的页面了（我等待了大概二十分钟）。

## 5. 维护博客
5.1. 首先需要对hexo进行配置，参考[官方文档](https://hexo.io/docs/)。
5.2. 选择一款自己喜欢的主题，[主题下载](https://hexo.io/themes/)。
5.3. 每一款主题都有自己的文档，参照文档进行设置。我选择的是[NexT](http://theme-next.iissnan.com/)。
5.4. 在blogfolder文件夹中右键打开Git Bash，执行以下命令。其中blogname是文章的文件名。新生成的文件位置在source/_post/blogname.md。
``` bash
hexo new blogname
```
5.5. 编辑该文件的内容即可完成博客的书写。书写博客的语法为Markdown。[参考网站](http://sspai.com/25137)，[在线编辑器](https://www.zybuluo.com/mdeditor)，可以参考学习。一篇简单的博文内容如下。
``` bash
---
title: 博文标题
date: 2016-03-06 17:32:43
---
这是我的第一篇博客！
```
5.6. 有时候文章会出现乱码，需要保存为utf-8。（这篇文章是Sublime编辑的，Preference里的Setting Default设置）
5.7. 在Git Bash中依次执行以下命令，将修改发布到远程仓库master分支，等待一段时间后，就可以看到这篇文章了。
``` bash
hexo clean
hexo generate
hexo deploy
```
5.8. 提交代码至远程hexoblog分支。
我们在新建和修改了博文以后，在blogfolder文件夹中右键选择Git Bash，执行如下代码，其中commit message为提交的日志记录提示（如第一篇文章）。执行完这些命令之后，我们就将网页生成代码提交到远程repository的hexoblog分支了。
``` bash
git pull
git add -A
git commit -m "commit message"
git pull --rebase
git push origin hexoblog:hexoblog
```

## 6. 如果换了地点或者电脑，如何维护博客呢？
我们在创建repository的时候，创建了一个新的hexoblog分支，这个分支就是用于存放生成博客网站所需要的代码的。而master分支是存放生成出来的静态网页。
若我们更换了地点或者电脑之后，只需要依次：
* 生成新的密钥对，并添加到Github repository的SSH Key。步骤与3.4至3.7相同。
* 任意文件夹右键选择Git Bash，执行以下命令。完成之后，当前文件夹的分支是hexoblog，对应远程hexoblog分支。这样我们就把网页生成代码拉取到本地了，接下来就可以尽情的撰写啦~
``` bash
git clone git@github.com:iamjunwei/iamjunwei.github.io.git
cd ./iamjunwei.github.io
git checkout -b hexoblog origin/hexoblog
```
