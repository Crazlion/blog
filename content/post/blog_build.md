---
title: "hugo + github page 搭建自己的博客"
date: 2020-04-22T17:43:24+08:00
draft: true
categories: ["hugo"]
---
为了搭建自己的博客完善自己的知识体系
## windows本地搭建 ##
#### 1、下载hugo ####

直接访问官网：https://gohugo.io/  
本来打算在ubuntu的虚拟机搭的，但是直接下载的版本有点低，和后面的主题有点不兼容，可能我用的是阿里云的apt-get源  
所以直接下载[windows的0.61版本](https://github.com/gohugoio/hugo/releases/download/v0.61.0/hugo_0.61.0_Windows-64bit.zip)  
把hugo.exe拷贝到一个PATH目录下，感觉这样省事，之前设置过
#### 2、验证安装 ####
``` shell
hugo version
Hugo Static Site Generator v0.57.2-A849CB2D windows/amd64 BuildDate: 2019-08-17T17:54:13Z
```

#### 3、创建站点 ####
找一个目录，打开命令行
``` shell
hugo new site blog
```  

#### 4、下载主题 ####
因为现在还是一个空的站点，没有index.html，即使现在起也是一个空的
使用的一个简单主题[maupassant](https://github.com/flysnow-org/maupassant-hugo)  
```
cd themes
git clone https://github.com/flysnow-org/maupassant-hugo.git maupassant
```
修改config.toml
```
languageCode = "zh-cn"
title = "Crazy lion's blog"
theme = "maupassant"
```

#### 5、创建一个blog ####
这个主题有一个规定文档要放在post目录下
```
hugo new post/my-first-post.md
```

#### 6、启动服务 ####
```
hugo server -D
```
打开浏览器查看http://localhost:1313/

## github page搭建 ##
#### 7、编译hugo项目 ####
```
hugo -D
```
和6的参数是一样的，之前没有-D，放上去没有内容  
会生产一个public的文件夹

#### 8、创建仓库 ####
新建一个仓库，命名规则: (__**必须是自己github用户名**__).github.io  
本地克隆 git clone github.git(自己库的地址)

#### 9、上传 ####
* 简单粗暴直接把public下的文件直接拷贝过来
* 将git本地库关联至远程仓库  

提交4部曲
```
git status  # 查看当前修改状态。
git add .  # 添加所有修改过的文件。你也可以只添加某个文件。
git commit -m "Add a new post"  # "Add a new post" 是 commit message.
git push -u origin master #推送至远程仓库
```


## [github 地址](https://github.com/Crazlion/Crazlion.github.io)
## [博客地址](https://crazlion.github.io/)