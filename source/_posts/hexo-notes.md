---
title: Hexo notes
date: 2020-01-18 21:46:40
tags: Hexo
categories: 技术
---
Hexo常用命令
```
hexo init [folder]  //新建一个网站。如果没有设置 folder ，Hexo 默认在目前的文件夹建立网站。
hexo server         //启动服务器。默认情况下，访问网址为： http://localhost:4000/。
hexo g              //生成html文件
hexo clean          //清空生成文件
hexo new [layout] <title>  //新建一篇文章。如果没有设置 layout 的话，默认使用 _config.yml 中的 default_layout 参数代替。
                           //如果标题包含空格的话，请使用引号括起来。
hexo new "post title with whitespace"
hexo d            //部署之前预先生成静态文件
```
