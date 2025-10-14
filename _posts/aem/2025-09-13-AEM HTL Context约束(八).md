---
layout: post
title: "AEM开发系列-HTL Context约束(八)"
categories: AEM开发系列
tags: AEM
author: fengsh998
typora-root-url: ..
---

[context显式约束](https://github.com/adobe/htl-spec/blob/master/SPECIFICATION.md#1-expression-language-syntax-and-semantics)的Display Context 上查找到

```html
<!--/* Scripts */-->
<a href="#whatever" onclick="${myFunctionName @ context='scriptToken'}()">Link</a>
<script>var ${myVarName @ context="scriptToken"}="Bar";</script>
<script>var bar='${someText @ context="scriptString"}';</script>
 
<!--/* Styles */-->
<a href="#whatever" style="color: ${colorName @ context='styleToken'};">Link</a>
<style>
    a.${className @ context="styleToken"} {
        font-family: '${fontFamily @ context="styleString"}', sans-serif;
        color: #${colorHashValue @ context="styleToken"};
        margin-${side @ context="styleToken"}: 1em; /* E.g. for bi-directional text */
    }
</style>
```

__示例分析说明：__

```html
<a href="#whatever" onclick="${myFunctionName @ context='scriptToken'}()">Link</a>
```

__说明：__

<span style="color:red;">myFunctionName</span> 是一个动态的 JavaScript 函数名，通过 <span style="color:red;">@ context='scriptToken' </span>确保输出是安全的函数名（如 <span style="color:red;">myFunction</span>。最终输出可能是 `<a href="#whatever" onclick="myFunction()">Link</a>`。

__作用：__

防止 myFunctionName 包含恶意代码，如 alert('hacked')。

```html
<script>var ${myVarName @ context="scriptToken"}="Bar";</script>
```
__说明：__

<span style="color:red;">myVarName</span> 是一个动态的 JavaScript 变量名，<span style="color:red;">@ context="scriptToken" </span>确保它是安全的标识符。输出可能是 。

__作用：__

确保变量名合法，避免注入不安全的字符

```html
<script>var bar='${someText @ context="scriptString"}';</script>
```

__说明：__

someText 是一个动态的 JavaScript 字符串值，@ context="scriptString" 确保它被正确转义并包裹在引号中。输出可能是 ，如果 someText 包含引号（如 "Hello" World"），会转义为 "Hello" World".

__作用：__

防止字符串内容破坏 JavaScript 语法或注入恶意代码。

```html
<a href="#whatever" style="color: ${colorName @ context='styleToken'};">Link</a>
```

__说明：__

colorName 是一个动态的 CSS 颜色值，@ context='styleToken' 确保它是安全的 CSS 值，如 red 或 blue。输出可能是 `<a href="#whatever" style="color: red;">Link</a>`。

__作用：__

确保颜色值合法，避免注入不安全的 CSS 代码

```html
<style>
    a.${className @ context="styleToken"} {
        font-family: '${fontFamily @ context="styleString"}', sans-serif;
        color: #${colorHashValue @ context="styleToken"};
        margin-${side @ context="styleToken"}: 1em; /* E.g. for bi-directional text */
    }
</style>
```
__说明：__

* <span style="color:red;">className @ context="styleToken"</span>：确保 className 是安全的 CSS 类名，如 highlight。
* <span style="color:red;">fontFamily @ context="styleString"</span>：确保字体名称被正确转义并包裹在引号中，如 'Arial'。
* <span style="color:red;">colorHashValue @ context="styleToken"</span>：确保颜色值（如 FF0000）是安全的十六进制值。
* <span style="color:red;">side @ context="styleToken"</span>：确保动态的 CSS 属性部分（如 left 或 right）是安全的，适用于双向文本（如 RTL 语言）

输出示例：
```html
<style>
    a.highlight {
        font-family: 'Arial', sans-serif;
        color: #FF0000;
        margin-left: 1em;
    }
</style>
```

## **针对Sling模型示例**
```java
package com.example;

import org.apache.sling.api.resource.Resource;
import org.apache.sling.models.annotations.Model;
import org.apache.sling.models.annotations.injectorspecific.ValueMapValue;

@Model(adaptables = Resource.class)
public class DynamicContentModel {
    @ValueMapValue
    private String myFunctionName = "showMessage";
    
    @ValueMapValue
    private String myVarName = "userId";
    
    @ValueMapValue
    private String someText = "Hello \"World\"!";
    
    @ValueMapValue
    private String colorName = "blue";
    
    @ValueMapValue
    private String className = "highlight";
    
    @ValueMapValue
    private String fontFamily = "Helvetica Neue";
    
    @ValueMapValue
    private String colorHashValue = "00F";
    
    @ValueMapValue
    private String side = "left";
    
    @ValueMapValue
    private String debugInfo = "Dynamic content model loaded";
    
    // Getters
    public String getMyFunctionName() { return myFunctionName; }
    public String getMyVarName() { return myVarName; }
    public String getSomeText() { return someText; }
    public String getColorName() { return colorName; }
    public String getClassName() { return className; }
    public String getFontFamily() { return fontFamily; }
    public String getColorHashValue() { return colorHashValue; }
    public String getSide() { return side; }
    public String getDebugInfo() { return debugInfo; }
}
```
htl template.html
```html
<!--/* Template for dynamic script and style rendering */-->
<div data-sly-use.model="com.example.DynamicContentModel">
  <!--/* Scripts */-->
  <a href="#whatever" onclick="${model.myFunctionName @ context='scriptToken'}()">Click Me</a>
  <script>
    var ${model.myVarName @ context="scriptToken"} = "Bar";
    var bar = '${model.someText @ context="scriptString"}';
    console.log(bar);
  </script>

  <!--/* Styles */-->
  <a href="#whatever" style="color: ${model.colorName @ context='styleToken'};">Styled Link</a>
  <style>
    a.${model.className @ context="styleToken"} {
        font-family: '${model.fontFamily @ context="styleString"}', sans-serif;
        color: #${model.colorHashValue @ context="styleToken"};
        margin-${model.side @ context="styleToken"}: 1em;
    }
  </style>

  <!--/* Debug Comment */-->
  <!--/* Model data: ${model.debugInfo} */-->
</div>
```
最终输出：
```html
<div>
  <!--/* Scripts */-->
  <a href="#whatever" onclick="showMessage()">Click Me</a>
  <script>
    var userId = "Bar";
    var bar = "Hello \"World\"!";
    console.log(bar);
  </script>

  <!--/* Styles */-->
  <a href="#whatever" style="color: blue;">Styled Link</a>
  <style>
    a.highlight {
        font-family: 'Helvetica Neue', sans-serif;
        color: #00F;
        margin-left: 1em;
    }
  </style>

  <!--/* Debug Comment */-->
  <!--/* Model data: Dynamic content model loaded */-->
</div>
```

## 块语句

**data-sly-use**

__说明：__

data-sly-use 用于初始化一个 Sling 模型或脚本（Java、JavaScript 等），将其绑定到模板中的变量，以便在模板中访问其属性或方法。

__用法：__

__语法：__

data-sly-use.<identifier>="resourcePath | className | scriptPath"
常用于绑定后端模型或逻辑到模板。

```html
<div data-sly-use.model="com.example.UserModel">
  <h1>${model.userName}</h1>
  <p>${model.getGreeting()}</p>
</div>

<!-- 说明：
model 是绑定的标识符，com.example.UserModel 是一个 Sling 模型类。
假设 UserModel 提供 userName 属性和 getGreeting() 方法，输出可能是：
-->
<div>
  <h1>John Doe</h1>
  <p>Hello, John!</p>
</div>
```

**data-sly-text**

__说明：__

data-sly-text 用于替换元素的文本内容，确保输出内容经过安全转义（默认 context='text'）。

__用法：__

__语法：__

data-sly-text="expression" 用于直接输出文本，覆盖元素内部的静态内容。

```html
<p data-sly-text="${properties.title || 'Default Title'}">Static Title</p>

<!--
properties.title 的值替换<p>内的静态文本 Static Title。
如果 properties.title 为 Welcome, 输出为
-->

<p>Welcome</p>
```

**data-sly-attribute**

__说明：__

data-sly-attribute 用于动态设置 HTML 元素的属性值，支持单个属性或多个属性。

__用法：__

__语法：__

data-sly-attribute.<attributeName>="expression" 或 data-sly-attribute="mapObject"

用于动态添加或修改 HTML 属性。
***1、单个属性***
```html
<a data-sly-attribute.href="${properties.link || '#'}">Click Me</a>

//假设 properties.link = '/page' 
//输出
<a href="/page">Click Me</a>
```
***2、多个属性***
```html
<div data-sly-attribute="${properties.attributes}"></div>
//如果properties.attributes = { class: 'highlight', title: 'Tooltip' }
//输出
<div class="highlight" title="Tooltip"></div>
```
实际应用：动态设置链接、类名、标题或其他 HTML 属性，适用于动态样式或交互。

**data-sly-element**

__说明：__

data-sly-element 用于动态替换 HTML 元素的标签名。

__用法：__

__语法：__

data-sly-element="expression"

表达式必须返回有效的 HTML 标签名（如 div, span, h1）。

Display context约束中规定的标签有：section, nav, article, aside, h1, h2, h3, h4, h5, h6, header, footer, address, main, p, pre, blockquote, ol, li, dl, dt, dd, figure, figcaption, div, a, em, strong, small, s, cite, q, dfn, abbr, data, time, code, var, samp, kbd, sub, sup, i, b, u, mark, ruby, rt, rp, bdi, bdo, span, br, wbr, ins, del, table, caption, colgroup, col, tbody, thead, tfoot, tr, td, th

```html
<div data-sly-element="${properties.tagName || 'span'}">Content</div>
//如果 properties.tagName = 'h1'，输出
<h1>Content</h1>
//如果 properties.tagName 为空，输出
<span>Content</span>
```
实际应用：根据条件动态选择 HTML 标签，如标题级别（h1, h2）或语义化标签。

**data-sly-test**

__说明：__

data-sly-test 用于条件渲染，只有当表达式为 true 或非空时，元素及其内容才会被渲染。

__用法：__

__语法：__

data-sly-test.<identifier>="expression"
可以保存条件结果到 identifier 供后续使用。
```html
<div data-sly-test.isActive="${properties.isActive}">User is active</div>
<div data-sly-test="${!isActive}">User is inactive</div>
//如果 properties.isActive = true，输出
<div>User is active</div>
//如果 properties.isActive = false，输出
<div>User is inactive</div>
```
实际应用：实现条件显示，如显示/隐藏内容、提示信息或权限控制。

**data-sly-list**

__说明：__

data-sly-list 用于遍历数组或集合，渲染每个元素的内容。

__用法：__

语法：data-sly-list.<item>="expression"

item 是循环变量，expression 返回可迭代对象。
```html
<ul data-sly-list.item="${properties.items}">
  <li>${item} (Index: ${itemList.index})</li>
</ul>
//itemList.index 是内置变量，表示当前索引
//假设 properties.items = ['Apple', 'Banana', 'Orange']，输出
<ul>
  <li>Apple (Index: 0)</li>
  <li>Banana (Index: 1)</li>
  <li>Orange (Index: 2)</li>
</ul>
```
实际应用：渲染列表，如菜单、产品列表或评论区。

**data-sly-repeat**

__说明：__

data-sly-repeat 类似于 data-sly-list，但会重复整个元素（包括标签），而不是仅内容。

__用法：__

语法：data-sly-repeat.<item>="expression"

用于重复渲染整个 HTML 元素。
```html
<li data-sly-repeat.item="${properties.items}">${item}</li>
//假设 properties.items = ['Apple', 'Banana', 'Orange']，输出：
<li>Apple</li>
<li>Banana</li>
<li>Orange</li>
```
实际应用：生成重复的 HTML 结构，如卡片、表格行或导航项。

**data-sly-include**

__说明：__
data-sly-include 用于包含另一个 HTL 文件或资源的内容。

__用法：__

语法：data-sly-include="path"

path 是要包含的文件路径或资源路径。
```html
<div data-sly-include="/content/templates/header.html"></div>
//包含 /content/templates/header.html 的内容，假设其内容为
//<header>Site Header</header>
输出为
<div><header>Site Header</header></div>
```
实际应用：复用公共组件，如页头、页脚或侧边栏。

**data-sly-resource**

__说明：__

data-sly-resource 用于渲染 AEM 资源（如页面或组件），通常用于嵌入子资源。

__用法：__

语法：data-sly-resource="${'path' @ resourceType='type'}"

可指定资源路径和资源类型。
```html
<div data-sly-resource="${'content' @ resourceType='myapp/components/content'}"></div>

//渲染路径为 content 的资源，使用 myapp/components/content 组件。
输出取决于目标组件的渲染结果，例如：
<div><p>Content from resource</p></div>
```
实际应用：嵌入子页面、组件或内容片段，如动态加载文章或广告。

**data-sly-template**

__说明：__

data-sly-template 定义一个可重用的模板，类似于函数，需通过 data-sly-call 调用。

__用法：__

语法：data-sly-template.<templateName>="${@ param1, param2}"
```html
<template data-sly-template.card="${@ title, description}">
  <div class="card">
    <h2>${title}</h2>
    <p>${description}</p>
  </div>
</template>
```
定义名为 card 的模板，接受 title 和 description 参数。

实际应用：定义可复用的 UI 组件，如卡片、按钮或模态框。

**data-sly-call**

__说明：__

data-sly-call 用于调用 data-sly-template 定义的模板，并传递参数。

__用法：__

语法：data-sly-call="${templateName @ param1=value1, param2=value2}"
```html
<template data-sly-template.card="${@ title, description}">
  <div class="card">
    <h2>${title}</h2>
    <p>${description}</p>
  </div>
</template>
<div data-sly-call="${card @ title=properties.title, description=properties.description}"></div>

//假设 properties.title = 'My Card'，properties.description = 'This is a card'，输出：

<div>
  <div class="card">
    <h2>My Card</h2>
    <p>This is a card</p>
  </div>
</div>
```
实际应用：调用模板渲染动态内容，适用于模块化组件设计。

**data-sly-unwrap**

__说明：__

data-sly-unwrap 移除元素的标签，仅保留其内容，通常与条件渲染结合使用。

__用法：__

语法：data-sly-unwrap.<identifier>="expression"

如果表达式为 true，移除标签；否则保留。
```html
<div data-sly-unwrap="${properties.hideWrapper}">
  <p>Content</p>
</div>
//如果 properties.hideWrapper = true，输出
<p>Content</p>

//如果 properties.hideWrapper = false，输出
<div><p>Content</p></div>
```
实际应用：根据条件移除包裹元素，简化 DOM 结构。

**data-sly-set**

__说明：__

data-sly-set 用于定义一个变量，供模板后续使用。

__用法：__

语法：data-sly-set.<variableName>="expression"

变量在当前作用域内有效。
```html
<div data-sly-set.myVar="${properties.title || 'Default'}">
  <p>Title: ${myVar}</p>
</div>

//定义变量 myVar 为 properties.title 或 'Default'，输出

<div>
  <p>Title: Default</p>
</div>
```
实际应用：存储中间结果或简化复杂表达式，提高模板可读性。
总结：
* 数据绑定：data-sly-use, data-sly-set 用于绑定模型和定义变量。
* 内容渲染：data-sly-text, data-sly-attribute, data-sly-element 控制文本、属性和标签。
* 条件与循环：data-sly-test, data-sly-list, data-sly-repeat 实现动态逻辑和列表渲染。
* 模块化与复用：data-sly-include, data-sly-resource, data-sly-template, data-sly-call 支持内容和组件复用。
* 结构优化：data-sly-unwrap 简化 DOM 结构。

同一语句上优先级：
1. data-sly-template
2. data-sly-set, data-sly-test, data-sly-use
3. data-sly-call
4. data-sly-text
5. data-sly-element, data-sly-include, data-sly-resource
6. data-sly-unwrap
7. data-sly-list, data-sly-repeat
8. data-sly-attribute

## 工具
安装HTL工具以方便学习：https://github.com/adobe/aem-htl-repl

1、下载构建好稳定版本包(根据连接找最新版本哈)：Download the built package: com.adobe.granite.sightly.repl-1.0.4.zip

2、在本地的AEM的启动实的包管理中上传：

![img](/assets/articles/aem/htl/htl-1.jpg)

![img](/assets/articles/aem/htl/htl-1.jpg)

![img](/assets/articles/aem/htl/htl-2.jpg)

![img](/assets/articles/aem/htl/htl-3.jpg)

![img](/assets/articles/aem/htl/htl-4.jpg)

![img](/assets/articles/aem/htl/htl-5.jpg)

![img](/assets/articles/aem/htl/htl-6.jpg)

打开：http://localhost:4502/content/repl.html
![img](/assets/articles/aem/htl/htl-7.jpg)











