---
title: "如何将项目部署至Tomcat的根下"
source: https://www.boris1993.com/tomcat-deploy-to-root.html
date: 2018-01-12
updated: 2026-10-10
tags: [Tomcat]
categories: [小技巧]
---

# 如何将项目部署至Tomcat的根下

有两种方法可以实现将项目部署到Tomcat的根下。

1.  删掉自带的ROOT/目录，然后将项目的war包重命名为ROOT.war
    
2.  将项目正常部署(假设war包名为your\_project.war)，然后修改`conf/server.xml`中的Context Root为如下内容：
    

```xml
<Context path="" docBase="your_project" debug="0" reloadable="true"></Context>
```

参考文献：https://stackoverflow.com/questions/5328518/deploying-my-application-at-the-root-in-tomcat
