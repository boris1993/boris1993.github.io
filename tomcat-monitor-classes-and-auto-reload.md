---
title: "配置Tomcat监控class和lib变更并自动重新加载"
source: https://www.boris1993.com/tomcat-monitor-classes-and-auto-reload.html
date: 2019-03-01
updated: 2026-10-10
tags: [Tomcat]
categories: [小技巧]
---

# 配置Tomcat监控class和lib变更并自动重新加载

在`context.xml`的`Context`标签中，设定`reloadable="true"`即可。

```xml
<Context reloadable="true">
    <!-- Other configurations -->
</Context>
```

配置完毕后重启Tomcat使配置生效，然后Tomcat在监控到项目的class或lib有变化后，就会自动重新加载这个webapp。

但是这个功能会显著增加Tomcat的性能消耗，故不建议在生产环境中使用。
