---
title: "使用Hugo搭建一套自己的网站"
date: 2023-09-12T20:03:02+08:00
draft: false
toc: true
author: "李冬瓜"
tags: ["工具"]
categories: ["工具"]
---
一个用Go语言编写的静态网页生成器，能够快速搭建一个静态网站。
<!--more-->
# 使用Hugo搭建网站

## 过去如何搭建网站
方案：使用前后端分离技术，Java SpringBoot框架提供数据，H5渲染页面。
缺点：作为服务端研发，上手前端知识成本高，自己搭建弄出来的UI很丑。
>自己大学搭过的商城-前后端分离项目 也是用网课提供的代码搞的。

## 新型搭建网站方式
方案：使用Hugo框架搭建静态博客页面。
Hugo是什么：一个用Go语言编写的静态网页生成器，能够快速搭建一个静态网站。被称为最受欢迎且最热门的静态网站生成器之一。  

优点：
1. 不需要对前端知识熟悉，就能够使用Hugo在几分钟内快速搭建一套美观、简洁的博客网站；预览功能便捷
2. 支持网站切换多种主题；
3. 支持Markdown语法展示；
4. 支持展示样式多、功能多，项目持续迭代维护；支持多个高级特性  

Hugo github链接：<https://github.com/gohugoio/hugo>

Hugo 官方文档：<https://www.gohugo.org/>  

缺点：
1. 不支持评论等功能，是一个静态网站

## 样例展示
我的博客网站：<https://lisanzhu.github.io/>  

使用主题：eureka  

用处：记录个人生活、技术笔记

## 详细过程
### 使用Eureka模板快速搭建
>Eureka 可以通过使用 Github 模板Hugo Eureka Starters:<https://www.wangchucheng.com/zh/docs/hugo-eureka/getting-started/>快速安装。  

使用此种方式时请点击页面上的Use this template按钮，并设置你的仓库信息，创建一个仓库。  

本地创建一个目录关联此仓库。

对应项目目录
![img.png](/pic/hugo/project_dir_lvl2.png)

在posts目录中自定义页面，创建页面指令如下，用Markdown格式编辑页面
```shell
hugo new posts/c.md
```

启动服务器指令(-D 是否展示草稿版本)

```shell
hugo server -D
```

### 手动安装submodule方式搭建框架
环境搭建
下载hugo框架：macos 使用homebrew一键安装hugo
```shell
brew install hugo
```
使用hugo指令创建项目目录，并切换到指定工作目录下
```shell
hugo new site quickstart  & cd quickstart
```
工作目录结构如下
![img.png](/pic/hugo/submodule_dir.png)
项目目录介绍
- archetypes: 储存.md的模板文件，类似于hexo的scaffolds，该文件夹的优先级高于主题下的/archetypes文件夹
- config.toml: 配置文件
- content: 储存网站的所有内容，类似于hexo的source
- data: 储存数据文件供模板调用，类似于hexo的source/_data
- layouts: 储存.html模板，该文件夹的优先级高于主题下的/layouts文件夹
- static: 储存图片,css,js等静态文件，该目录下的文件会直接拷贝到/public，该文件夹的优先级高于主题下的/static文件夹
- themes: 储存主题
- public: 执行hugo命令后，储存生成的静态文件

初始化git，从github拉取网站主题
>备注：此种方法属于新建项目以后，将想要的主题添加进本项目；   
> 另外一种方法是使用主题的quick_start当做模板直接创建项目，详情见https://www.wangchucheng.com/zh/docs/hugo-eureka/getting-started/快速安装部分
```shell
git init 
git submodule add https://github.com/wangchucheng/hugo-eureka.git themes/eureka
```
将下载的主题应用至本项目
```shell
echo "theme = 'eureka'" >> hugo.toml
```

### 将页面放在github上
在项目根目录下生成public目录
```shell
hugo --theme=wangchucheng.com/hugo-eureka --baseURL="https://lisanzhu.github.io" 
```

切换到public目录下，以此为根目录推到指定仓库中，仓库名需要为 xx(用户名).github.io 
```shell
$ cd public
$ git init
$ git remote add origin https://github.com/lisanzhu/lisanzhu.github.io.git   
$ git add -A
$ git commit -m "first commit"
$ git push -u origin master
```
在项目仓库-setting中选择page选项，指定分支生成页面
![img.png](/pic/hugo/deploy_page.png)  

浏览器里访问：<https://lisanzhu.github.io/index.html>

### 其他
eureka中使用 toc:true 生成目录

生成public目录时，记得确认baseURL和网站地址相同，协议使用https




### 涉及eureka主题
文件头使用 toc:true 创建目录  

手动安装submodule出现部分模块缺失问题，修复方法：<https://www.cnblogs.com/joyingwol/p/eureka-install.html；>

标签使用：可以为文章使用标签进行分类，将相同标签文章展示在某列表页面下面
![img.png](/pic/hugo/tax.png) 
