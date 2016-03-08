---
title: 生成并管理电脑内的ssh key
date: 2016-03-08 14:25:36
categories: hexo
tags: hexo
---

## 前言
我们在Github上拉取和提交代码，有两种方式：ssh和http。http每次都需要输入用户名密码，相对比较繁琐。而ssh则不用，只需要在电脑和Github上配置好密钥对即可。这篇文章介绍一下如何创建和配置ssh key。

## 准备工作
下载并安装[Git](http://git-scm.com/download/)。

## 查看本机是否存在ssh key
打开系统盘:\Users\Username（如C:\Users\iamjunwei），若存在.ssh文件夹，则已经存在ssh key，打开该文件夹可以看到所有的ssh key。

## 创建新的ssh key
* 在任意文件夹右键选择Git Bash。
* 执行以下命令。请将命令替换成你的邮箱（如mike123456@163.com）。
``` bash
ssh-keygen -t rsa -C "你的邮箱"
```
* 接下来会出现以下提示。其中括号里的路径应该是你之前查看ssh key的文件夹路径（这里只是举例，用的是我自己的路径）。这一步是提示输入你需要创建的ssh key的文件名，默认为id_rsa。为了避免与其他ssh key产生覆盖，我们新建一组ssh key文件。输入**/c/Users/iamjunwei/.ssh/github_iamjunwei**后，按回车确认。再一直按回车，直至命令完成。请注意：**/c/Users/iamjunwei/.ssh**请替换成你的ssh key文件夹路径，**github_iamjunwei**请替换成你新建的ssh key文件名。
``` bash
Generating public/private rsa key pair.
Enter file in which to save the key (/c/Users/iamjunwei/.ssh/id_rsa):
```
* 重新回到ssh key的文件夹下，可以发现，此时已经产生了两个新文件github_iamjunwei和github_iamjunwei.pub（这里只是举例，实际产生的是你输入的文件名）。

## 管理ssh key
在ssh key的文件夹下，创建一个文件config（无后缀名）。编辑config文件，新增一段内容如下。其中User字段后面请替换成你对应github的username，IdentityFile请替换成你生成的ssh key文件的路径。
``` bash
Host github.com
HostName github.com
User iamjunwei
PreferredAuthentications publickey
IdentityFile /c/Users/iamjunwei/.ssh/github_iamjunwei
```
大功告成！