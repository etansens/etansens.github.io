---
title: Hello World
categories:
- Blog
tags:
- hexo
- mellow
---
<div align="center">
开博立贴
</div>

# 鸣谢
[教程](https://blog.csdn.net/sinat_37781304/article/details/82729029)
[官方](https://github.com/hexojs/hexo)
[主题](https://github.com/codefine/hexo-theme-mellow)
<!--more-->
### 新建文章
hexo new "Kafka Producer Test"
### 生成blog并启动本地服务
hexo clean && hexo g && hexo server
### 部署到github
hexo d
### 提交文章源代码
#### hexo主目录下.gitignore添加部署相关目录
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
themes/
#### 提交代码
git add . && git commit -m"add new blog"
### push文章源代码到github
#### 关联远程仓库
git remote add github https://github.com/etansens/etansens.github.io.git
#### push到分支
git push github master:source