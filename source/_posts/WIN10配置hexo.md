---
title: WIN10配置hexo
tags: hexo
abbrlink: c7035f77
date: 2021-03-21 00:18:26
---
具体可以参考官方中文文档https://hexo.io/zh-cn/docs/index.html

<!-- more -->


# 安装Node.js
**方法一**：官方的安装地址：https://nodejs.org/en/download/
**方法二**：通过 nvs（推荐）或者nvm 安装。nvs安装地址：https://github.com/jasongin/nvs/
# 安装git（略）
# 安装hexo
```
$ npm install -g hexo-cli
$ npm install hexo-deployer-git --save
```
# 命令
```
$ hexo g #完整命令为hexo generate，用于生成静态文件
$ hexo s #完整命令为hexo server，用于启动服务器，主要用来本地预览
$ hexo d #完整命令为hexo deploy，用于将本地文件发布到github上
$ hexo n #完整命令为hexo new，用于新建一篇文章
```
# 建站
**1、**安装 Hexo 完成后，请执行下列命令，Hexo 将会在指定文件夹中新建所需要的文件。
```
$ hexo init <folder>
$ cd <folder>
$ npm install
```
接下来说明部分文件的作用。
**_config.yml**
网站的 *配置*  信息，您可以在此配置大部分的参数。

**package.json**
*应用程序* 的信息。EJS, Stylus 和 Markdown renderer 已默认安装，您可以自由移除。

**scaffolds**
*模版* 文件夹。当您新建文章时，Hexo 会根据 scaffold 来建立文件。

Hexo的模板是指在新建的文章文件中默认填充的内容。例如，如果您修改scaffold/post.md中的Front-matter内容，那么每次新建一篇文章时都会包含这个修改。

**source**
*资源* 文件夹是存放用户资源的地方。除 _posts 文件夹之外，开头命名为 _ (下划线)的文件 / 文件夹和隐藏的文件将会被忽略。Markdown 和 HTML 文件会被解析并放到 public 文件夹，而其他文件会被拷贝过去。

**themes**
*主题* 文件夹。Hexo 会根据主题来生成静态页面。

**2、**将hexo和github进行关联
首先在github上创建一个仓库，仓库名为\*\*\*.github.io，接着编辑**_config.yml**文件，在_config.yml最下方，添加如下配置：

```
deploy:
  type: git
  repository: git@github.com:***/***.github.io.git
  branch: main
```
**3、**将本地文件同步到github
```
$ hexo g
$ hexo d
```
此时，我们的博客已经搭建起来，并发布到Github上了，这时可以登陆自己的Github查看代码是否已经推送到对应Repository。最后到github的settings的GitHub Pages中查看，可以在那里看到一个网址，点击即可访问查看。