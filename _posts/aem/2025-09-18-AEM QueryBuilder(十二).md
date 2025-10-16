---
layout: post
title: "AEM开发系列【后端】- 查询生成器QueryBuilder(十二)"
categories: AEM开发系列
tags: AEM
author: fengsh998
typora-root-url: ..
---

# Query Builder 简介

Query Builder是AEM提供的一个对JCR（crxde）的资源数据进行查询的工具，同时也支持jar包，因此可以在AEM的项目中使用QueryBuilder的JAVA API进行查询操作。

本地的AEM SDK启动后可以通过 http://localhost:4502/libs/cq/search/content/querydebug.html进行打开QueryBuilder

![img](/assets/articles/aem/qb/qb-1.jpg)

extract facets: 勾选时则在查询出结果后提取出维度，如果不勾选则不提取

JSON QueryBuilder Link: 可在查询完成后，点击时打开新的窗口通过json的格式查看结果。

# 常见搜索谓词

## path

路径：当设定路径时，则搜索的范围就会被限定在对应的路径中进行扫索，这个和操作系统的文件夹目录是一个意思。

<span style="color:red;">不支持Facet(维度)提取</span>

<div style="background-color:#fafad2;padding:10px;border-radius:5px;">
  <p>如下面都可以作为路径(包括无子结点的完整路径):</p>
  <p>/</p>
  <p>/content</p>
  <p>/content/dam/wetrain/asset.jpg</p>
</div>

Query Builder Debugger中输入，执行search，可以看到结果
```shell
path=/content
```


## type

类型：该类型对应的值则为任何结点中主类型jcr:primaryType中对应的值,如:nt:unstructured,cq:Page

<span style="color:green;">支持Facet(维度)提取及统计</span>

```shell
path=/content
type=nt:unstructured
```

## nodename

结点名称：在CRXDE中，任何一个资源结点的名称。

<span style="color:green;">支持Facet(维度)提取及统计</span>

支持*,?的通配搜索,*支持任何一个或多个字符通配，？单个字符通配

```shell
path=/content
type=nt:unstructured
nodename=*cont*
```
或
```shell
path=/content
type=nt:unstructured
nodename=*c?nt*
```


## property

属性：即查询的属性名为任何结点Properties中name对应的称为属性。

<span style="color:green;">支持Facet(维度)提取及统计</span>

![img](/assets/articles/aem/qb/qb-2.jpg)

示例：

**单值查询**，如查询属性layout，对应的值为responsiveGrid的数据

关键词：***value***

```shell
path=/content/wetrain
property=layout
property.value=responsiveGrid
```

```shell
path=/content
1_property=sling:resourceType
1_property.value=wetrain/components/teaser
1_property.operation=like
```

**多值查询**, 如查询属性jcr:title，存在多个值时（eg: en, us）。

关键词：***N_value***

```shell
path=/content/wetrain
property=jcr:title
property.1_value=en
property.2_value=us
```

![img](/assets/articles/aem/qb/qb-3.jpg)

关键词：***and***

注意在进行多值搜索时，AEM SDK 5.3开始支持多值合并搜索，默认为OR操作，如果多值为AND操作时则为：
```shell
path=/content/wetrain
property=jcr:title
property.1_value=en
property.2_value=us
property.and=true
```

关键词：***operation***

属性值支持条件过滤操作,相应的值有:
* equals 表示完全匹配(默认)
* unequals 表示不相等比较
* like 模糊匹配，使用jcr:like xpath函数
* not 表示不匹配该属性，(value的值设置会被忽略)
* exists表示属性存在性检测。与not相反,(value的值设置会被忽略)

```shell
#查询/content/wetrain下面所有资源属性jcr:title值不等于en的数据
path=/content/wetrain
property=jcr:title
property.value=en
property.operation=unequals
```

```shell
#查询/content/wetrain下面所有资源属性jcr:title值包含E的数据
path=/content/wetrain
property=jcr:title
property.value=%E%
property.operation=like
```

```shell
#查询/content/wetrain下面所有资源节点没有包含属性为jcr:title的数据，即查出来的所有节点都不存在jcr:title这个属性
path=/content/wetrain
property=jcr:title
property.operation=not
```

```shell
#查询/content/wetrain下面所有资源节点包含属性为jcr:title的数据，即查出来的所有节点都带有jcr:title这个属性
path=/content/wetrain
property=jcr:title
property.operation=exists
```

关键词：***depth***

属性扫索深度

```shell
path=/content/wetrain
property=jcr:title
property.operation=exists
property.depth=2
```

## 根(root)

抽像的根属性，是指在Query builer搜索第一层的支持的参数统称为根参数。

关键词:***p.offset***

用于分页查询的偏移起始数据，和mysql的SQL语句中的offset一样效果。

```shell
#从第10条记录开始取一页数据
path=/content/wetrain
type=nt:unstructured
p.offset=10
```

关键词:***p.limit***

用于分页查询的取数据的记录数量，-1为所有和mysql的SQL语句中的limit一样效果。默认返回10条记录

```shell
#从第10条记录开始取一条数据
path=/content/wetrain
type=nt:unstructured
p.offset=10
p.limit=1
```

关键词:***p.guessTotal***

预估总数，为减小计算出查询的准备条数，在查询时预估一个粗略的总条数。

如果记录数大于13时，显示16+，如果设置过大时，则显示最大记录数。如果设置过小，则以p.offset+p.limit作为结果显示

```shell
path=/content/wetrain
type=nt:unstructured
p.offset=10
p.limit=6
p.guessTotal=13
```

关键词:***p.excerpt***

当设为true时，在搜索的结果中包含全文摘录

```shell
path=/content/wetrain
type=nt:unstructured
p.excerpt=true
```

关键词:***p.hits***

仅适用于JSON servlet）选择点击作为JSON写入的方式，并使用这些标准点击（可通过ResultHitWriter服务扩展），值可以为：simple，full，selective

```shell
path=/content/wetrain
type=cq:Page
p.hits=simple
```
simple:

![img](/assets/articles/aem/qb/qb-4.jpg)

full:

![img](/assets/articles/aem/qb/qb-5.jpg)

selective:

![img](/assets/articles/aem/qb/qb-6.jpg)


## group

组查询: 支持多路径，多组件查询。

关键词：***p.or*** 

组合的或条件

```shell
#查询出资源结点属性jcr:title值为Epic Journey 或 属性navTitle值为My Page的数据
path=/content
group.p.or=true
group.1_property=jcr:title
group.1_property.value=Epic Journey
group.2_property=navTitle
group.2_property.value=My Page
```

关键词：***p.not***

使组是否生效

关键词：***<谓词>***

即可以通过group.<谓词>的方式来拼接。

关键词：***<N_谓词>***

当多个谓词并存时，可以能过“序号_”进行区分，如group.1_property,group.2_property





[【查询生成器】](https://experienceleague.adobe.com/zh-hans/docs/experience-manager-65/content/implementing/developing/platform/query-builder/querybuilder-api)

[【谓词-中文】](https://experienceleague.adobe.com/zh-hans/docs/experience-manager-65-lts/content/implementing/developing/platform/query-builder/querybuilder-predicate-reference)

[【谓词-英文】](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/developing/platform/query-builder/querybuilder-predicate-reference)



