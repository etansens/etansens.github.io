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

### 新建文章
hexo new "Kafka Producer Test"

### 生成blog并启动本地服务
hexo clean && hexo g && hexo server

### 部署到github
hexo d
<!--more-->
## 提交文章源代码
将整个hexo划分为四个部分来用git管理：
hexo代码：npm安装，独立仓库，一般不修改
主题代码：fork别人的作品，作为单独的仓库
文章源码：作为blog库的一个分支source
blog产出：blog库master分支

### 1、hexo主目录下.gitignore添加排除部署相关目录
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
themes/

### 2、提交文章源代码
git add . && git commit -m"add new blog"

### 3、push文章源代码到github
#### a、关联远程仓库
git remote add github https://github.com/etansens/etansens.github.io.git
#### b、push到分支
git push github master:source