---
title: "hugo+gitpage搭建个人博客"
date: 2020-02-11T10:01:25+08:00
tags: ["hugo", "gitpage"]  
categories: ["hugo"]
keywords: ["hugo"]  
description: "使用hugo+gitpage搭建你的技术博客" 
summaryLength: 100
draft: false
---
### 安装hugo

```
brew install hugo  
```

```
hugo version
```

### 生成blog根目录

```
hugo new site blog
```

 config.toml 是配置文件，在里面可以定义博客地址、构建配置、标题、导航栏等等。
 
 themes 是主题目录，可以去 themes.gohugo.io 下载喜欢的主题。
 
content 是博客文章的目录。

### 安装主题

https://themes.gohugo.io/whiteplain/

### 使用hugo写文章

```
hugo new post/my-first-post.md
```

设置文章相关属性

```
title: "My First Post" 
date: 2017-12-14T11:18:15+08:00  
weight: 70  
markup: mmark  
draft: false  
keywords: ["hugo"]  
description: "第一篇文章"  
tags: ["hugo", "pages"]  
categories: ["pages"]  
author: ""  
```

### 预览

```
hugo server -D
```

确保本地网站正常，hugo server运行后在本地打开localhost:1313检查网站效果和内容，注意hugo server这个命令不会构建草稿，所以如果有草稿需要发布，将文章中的draft设置为false。

### 将hugo部署在gitpage

在Github创建一个仓库，例如名字叫blog，可以是私有的，这个仓库用来存放网站内容和源文件。

再创建一个名称为<username>.github.io的仓库，username为GitHub用户名，这个仓库用于存放最终发布的网站内容。

进入本地网站目录，将本地网站全部内容推送到远程blog仓库。

```
cd blog
git init
git remote add origin git@github.com:wukaiying/blog.git
git add -A
git commit -m "first commit"
git push -u origin master
```

删除本地网站目录下的public文件夹

创建public子模块
```
git submodule add -b master git@github.com:wukaiying/wukaiying.github.io.git public
```

然后就可以执行hugo命令，此命令会自动将网站静态内容生成到public文件夹，然后提交到远程blog仓库

```
cd public
git status
git add .
git commit -m "first commit"
git push -u orgin master
```


过一会就可以打开<username>.github.io查看网站了

注意：本地网站是关联的blog仓库，本地网站下的public文件夹是以子模块的形式关联的github.io仓库，他们是相对独立的。

### 自动部署脚本

完成博客编写后，使用该脚本可以更新public中内容，并将代码push到你的gitpage仓库中

```
#!/bin/bash

echo -e "\033[0;32mDeploying updates to GitHub...\033[0m"

# Build the project.
hugo # if using a theme, replace with `hugo -t <YOURTHEME>`

# Go To Public folder
cd public
# Add changes to git.
git add .

# Commit changes.
msg="rebuilding site `date`"
if [ $# -eq 1 ]
  then msg="$1"
fi
git commit -m "$msg"

# Push source and build repos.
git push origin master

# Come Back up to the Project Root
cd ..

```

### 最后

你需要定时将你的blog的内容定时push到你的blog仓库里面，防止本地数据丢失

```
cd blog
git push -u origin master
```
