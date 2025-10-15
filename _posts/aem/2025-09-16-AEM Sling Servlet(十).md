---
layout: post
title: "AEM开发系列【后端】- Sling Servlet接口(十)"
categories: AEM开发系列
tags: AEM
author: fengsh998
typora-root-url: ..
---

**说明：**

Servlet 用于扩展服务器功能的类主要接口能力，详见的HTTP请求如(POST,GET,PUT,DELETE,OPTION,HEAD等)

**Sling 中有两种类型的servlet:**

**SlingSafeMethodsServlet**

这种是安全访问的，即常见的只读型请求如（GET,HEAD,OPTIONS）

**SlingAllMethodsServlet**

这种就是常见的增删改查的方式如：(POST,PUT,DELETE,GET)

# AEM Sling Servlet 有两种方式注册接口以供访问：

## 1、通过资源类型
使用resourceTypes（资源类型）- 使用这种方式，我们需要使用到节点的 sling:resourceType 属性。因此，我们可以在浏览器中找到sling:resourceType属性中指定的路径。
![img](/assets/articles/aem/sling/url.png)
上图是通过Sling:resourceType的方式进行解释URL中命中规则。

![img](/assets/articles/aem/sling/sling-cmp.jpeg)
此为通过Sling:resourceType指定的方式一步步找到最终的资源。

### FAQ访问示例：
```java
package com.demosite.core.servlets;

import com.drew.lang.annotations.NotNull;
import org.apache.sling.api.SlingHttpServletRequest;
import org.apache.sling.api.SlingHttpServletResponse;
import org.apache.sling.api.servlets.HttpConstants;
import org.apache.sling.api.servlets.SlingAllMethodsServlet;
import org.apache.sling.servlets.annotations.SlingServletResourceTypes;
import org.osgi.service.component.annotations.Component;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.servlet.Servlet;
import javax.servlet.ServletException;
import java.io.IOException;

@Component(service = Servlet.class)
@SlingServletResourceTypes(
        resourceTypes="demosite/components/page",
        selectors = "faq",
        extensions="json",
        methods= HttpConstants.METHOD_POST
        )
public class FAQServlet extends SlingAllMethodsServlet {
    private static final Logger log = LoggerFactory.getLogger(FAQServlet.class);
    @Override
    protected void doPost(@NotNull SlingHttpServletRequest request,
                          @NotNull SlingHttpServletResponse response) throws ServletException, IOException {
        log.debug(">>>doPost: req-> {}, resp: {}", request, response);
    }
}
```
__说明：__

@SlingServletResourceTypes(
        resourceTypes="<span style="color:red;">demosite/components/page</span>",  //资源类型这个是命中的关键。
        selectors = "faq",    //选择器
        extensions="json", //扩展命
        methods= HttpConstants.METHOD_POST  //这个是请求方式
        )
这里的resourceTypes可以多个，多个的写法：
<span style="color:red;">resourceTypes = {"dam:Asset", "sling:Folder", "dam:Folder"}</span>,
Sling框加中要求所以Servlet使有资源类型访问时，要求只能通过CRX设置的结点上有sling:resourceType这个properties才能进行访问。
如示例中配置了接口命中的资源为<span style="color:red;">demosite/components/page</span>

观察/content/demosite/<span style="color:red;">us/en</span>/jcr:content中的sling:resourceType属性值（demosite/components/page）

![img](/assets/articles/aem/sling/sling-1.jpg)

观察/content/demosite/us/jcr:content中的sling:resourceType属性值（demosite/components/page）

![img](/assets/articles/aem/sling/sling-2.jpg)

观察/content/demosite/jcr:content中的sling:resourceType属性值（demosite/components/page）

![img](/assets/articles/aem/sling/sling-3.jpg)

由上图可以看到，有三个结点都能匹配上"<span style="color:red;">demosite/components/page</span>"
因此，下面的任何一种路径都能触发代码中的doPost请求。
http://localhost:4502/<span style="color:red;">content/demosite</span>/jcr:content.faq.json 
http://localhost:4502/<span style="color:red;">content/demosite/us</span>/jcr:content.faq.json 
http://localhost:4502/<span style="color:red;">content/demosite/us/en</span>/jcr:content.faq.json 
这就是通过资源来访问的好处，可以直接到组件。


同理在crxde中随便找一个带sling:resourceType的属性结点，如下：
/content/demosite/us/en/jcr:content/cq:featuredimage

![img](/assets/articles/aem/sling/sling-4.jpg)
此时要想触发servlet接口时，则需要修改Java文件的注解@SlingServletResourceTypes中的属性改为：resourceTypes="core/wcm/components/image/v3/image"，修改完成后重新编译安装后，就可以通过http://localhost:4502/content/demosite/us/en/jcr:content/cq:featuredimage.faq.json 的URL来访问了，
可以通过代码打印的日志来验证是否接口被正确调用了。查看日志：全部命中

![img](/assets/articles/aem/sling/sling-5.jpg)

如果当接口不指定到具体的资源路径时，则只要是站点的任何结点的URL都能命中触发接口调用。
如修改为：resourceTypes="sling/servlet/default"
```java
...
@SlingServletResourceTypes(
        resourceTypes="sling/servlet/default",
        selectors = "faq",
        extensions="json",
        methods= HttpConstants.METHOD_POST
        )
...
```
当设置为sling/servlet/default: 

则可以任何方式访问都能命中如：

http://localhost:4502/content/demosite/us/en.faq.json
 
http://localhost:4502/content/demosite/us.faq.json

http://localhost:4502/content/demosite.faq.json

http://localhost:4502/content.faq.json

![img](/assets/articles/aem/sling/sling-6.jpg)

## 2、通过路径访问
使用paths(路径)- 通过这种方式，我们可以直接使用请求中指定的路径，然后将调用我们的servlet的接口。
```java
package com.demosite.core.servlets;

import com.drew.lang.annotations.NotNull;
import com.google.gson.Gson;
import org.apache.sling.api.SlingHttpServletRequest;
import org.apache.sling.api.SlingHttpServletResponse;
import org.apache.sling.api.servlets.HttpConstants;
import org.apache.sling.api.servlets.SlingAllMethodsServlet;
import org.osgi.framework.Constants;
import org.osgi.service.component.annotations.Component;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.servlet.Servlet;
import javax.servlet.ServletException;
import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

@Component(service = Servlet.class, property = {
        Constants.SERVICE_DESCRIPTION + "=API Path Servlet",
        "sling.servlet.methods=" + HttpConstants.METHOD_GET,
        "sling.servlet.paths=" + "/bin/api/test"
})
public class APIPathServlet extends SlingAllMethodsServlet {
    private static final Logger log = LoggerFactory.getLogger(APIPathServlet.class);
    @Override
    protected void doGet(@NotNull SlingHttpServletRequest request,
                         @NotNull SlingHttpServletResponse response) throws ServletException, IOException {
        log.debug("API Path Servlet Called");
        Map<String,Object> result = new HashMap<>();
        result.put("status", "success");
        result.put("message", "API Path Servlet is working!");
        response.setContentType("application/json");
        String json_str = new Gson().toJson(result);
        response.getWriter().write(json_str);
        log.debug("Api Path Servlet Response Sent: {}",json_str);
    }
}

```
__注解说明：property__

<span style="color:red;">Constants.SERVICE_DESCRIPTION + "=API Path Servlet"</span>： 用于描述接口说明。
<span style="color:red;">"sling.servlet.methods=" + HttpConstants.METHOD_GET </span>： 用于指定该接口访问使用的方式。
<span style="color:red;">"sling.servlet.paths=" + "/bin/api/test"</span>：指定访问路径。
则可以在浏览器中直接使用：http://localhost:4502/bin/api/test 进行访问，则返回。

![img](/assets/articles/aem/sling/sling-7.jpg)


参考：
[【Sling】](https://experienceleague.adobe.com/zh-hans/docs/experience-manager-65-lts/content/implementing/developing/platform/sling-cheatsheet)








