在线文档
# jekyll 模板

使用模板 https://github.com/cotes2020/chirpy-starter
直接fork过来即可，然后就是修改_config.yml了

参考：https://github.com/august295/august295.github.io

jekyll的标准结构说明
```
.-----
 |
 |-- _data            //locates/zh-CN.yml 多语言配置
 |-- _posts           //文章放至处 格式必须是以yyyy-MM-DD-filename.md格式放致即可，不需要额外的文件夹区分
 |-- _tabs            //侧边栏显示的配置项
 |     |
 |     |---categories.md  //分类（不需要编辑）
 |     |---tags.md        //标签（不需要编辑）
 |     |---archives.md    //归档（不需要编辑）
 |     |---about.md       //关于（需要自己编辑）
 |-- assets           //数字资产(主要存放_posts中的文档编写时用的比图片，视频，及xmind,drawio等流程资源图)
 |     |
 |     |---media/images   //根据自已的需要区可建文件夹进行分类存放资源。如何引用如：在MD中 ![](/assets/media/images/xxx/xx.png)
 |     |---img/favicons/  //这里存放的是站点的图标的如:favicon.ico, favicon-16x16.png或favicon-32x32.png
```
对于_post中的.md文件必须开头以

```
---
layout: post
title: "用于显示在网页时的文章标题"
categories: 分类名(自定义，比如Android, iOS),当有多个.md的分类设置相同时，那么在站点的分类里就会自动归类显示了。
tags: 档签，用于检索用，比如C++，JAVA，等。
author: 作者
typora-root-url: ..
---
```
开启评论功能：

把_config.yml 打开，开启comments的provider设置，
默认提供三个评论系统[disqus | utterances | giscus] 根据自己选择对应配置即可开启

如
comments:
  provider: giscus
  ...
  giscus:
    repo: 仓库名eg:fengsh998/fengsh998.github.io
    repo_id: 仓库ID
    category: 分类
    category_id: 分类ID
  
关于页的修改，只需要把about.md中的进行修改内容即可。
> Add Markdown syntax content to file `_tabs/about.md`{: .filepath } and it will show up on this page.
{: .prompt-tip }
