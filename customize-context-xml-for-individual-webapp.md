---
title: "为webapp设定单独的context.xml"
source: https://www.boris1993.com/customize-context-xml-for-individual-webapp.html
date: 2019-03-01
updated: 2026-10-10
tags: [Tomcat]
categories: [小技巧]
---

# 为webapp设定单独的context.xml

要给某个webapp设定单独的`context.xml`，只需要在`${WEBAPP_ROOT}/webapp`目录下新建一个`META-INF`目录，并将`context.xml`放进去，就可以了。
