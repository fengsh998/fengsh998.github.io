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
date: 2023-10-27 14:30:00 +0800 //用于排序
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

如果没有安装giscus则先进行安装https://github.com/apps/giscus，然后找到自己对应的仓库在settings下开启Discussions 即可。然后到[giscus](https://giscus.app/zh-CN)官网上
输入你的仓库名检测一下是否OK。如：fengsh998/fengsh998.github.io，然后进行对应的Discussions的配置即可，不懂配的参考：https://www.lixueduan.com/posts/blog/02-add-giscus-comment/

高亮的用法，原来markdown 的写法是使用```语种的方式如：
```java
class J {
}
```
然而github pages _config.yml中配置使用的是Rouge高亮，查看[支持语种](https://github.com/rouge-ruby/rouge/wiki/List-of-supported-languages-and-lexers)
在此模板中高亮的写法
```
{% highlight java %}
...
{% endhighlight %}
```

单行高亮或高亮块中自定义自己部分字段使用div的方式来实现如：

```
<div style="background-color:#fafad2;padding:10px;border-radius:5px;">
  这是高亮块中的内容，<span style="color:blue;">这里蓝色</span>，继续其他内容。
</div>
```

<div style="background-color:#fafad2;padding:10px;border-radius:5px;">
  这是高亮块中的内容，<span style="color:#FF5733;">这里蓝色</span>，继续其他内容。
</div>


markdown语法参考

[速查](https://markdown.com.cn/cheat-sheet.html)

[扩展](https://theme-plume.vuejs.press/article/mcgayb5w/)

[github官方](https://docs.github.com/zh/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax)
  
关于页的修改，只需要把about.md中的进行修改内容即可。
> Add Markdown syntax content to file `_tabs/about.md`{: .filepath } and it will show up on this page.
{: .prompt-tip }

```
> 普通信息。
>     #//使用空的>来进行换行
> 换行号输出
{: .prompt-info }

> 这是一个有用的提示。
{: .prompt-tip }

> 警告：这是一个危险操作。
{: .prompt-warning }

> 危险：此操作不可逆！
{: .prompt-danger }
```

关于_posts中的的文章排序：
1、按文件名的日期排序，如果日期相同的情况下，安title的中的首字A-Z排，或ASCK码排。
2、在对应的文章中添加date字段进行。

markdown 中使用Emoji符号：

[Emoji Cheat Sheet](https://www.webfx.com/tools/emoji-cheat-sheet/)

[emojipedia](https://emojipedia.org)

markdown 中使用特殊符号HTML对照表：

[特殊符号](https://tool.chinaz.com/tools/htmlchar.aspx)

[特殊符号2](https://www.json.cn/document/html-entity/)

markdown 数学物理公式速查
当前模块如果要正确解释公式，需要在文章的标题加math: true,才能正解如：

```
---
title: xxx
..
math: true
---
```

[公式](https://www.cnblogs.com/zbzcDZF/p/16540630.html)

[LaTex公式](https://mp.weixin.qq.com/s/rAut9IAmBFtSW8iv38OOWA)


