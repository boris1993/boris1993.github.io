---
title: "使用FlatMap“展平”一系列数据"
source: https://www.boris1993.com/flatten-a-group-of-data-by-flatmap.html
date: 2019-09-26
updated: 2026-10-10
tags: [Java, 代码技巧, FlatMap]
categories: [小技巧]
---

# 使用FlatMap“展平”一系列数据

在处理业务数据时，偶尔会需要从一系列逗号分隔的数据中取出各个元素并去重。本文将介绍如何通过Java 8中的`FlatMap`简化这个操作。

## 介绍需求

假设现在有这样子的一个排班信息：

| 日期 | 班组 |
| --- | --- |
| 2019-01-02 | Unit-2, Unit-3 |
| 2019-01-03 | Unit-2 |
| 2019-01-04 | Unit-1, Unit-3 |
| 2019-01-01 | Unit-2, Unit-3 |
| 2019-01-05 | Unit-3 |
| 2019-01-06 | Unit-1, Unit-2 |
| 2019-01-07 | Unit-1 |
| 2019-01-08 | Unit-2, Unit-3 |

而目标是，从中取出这个表中的所有班组，并去重和排序，即最终的结果是`Unit-1, Unit-2, Unit-3`。

## 准备示例数据

在代码中，我们这样来构造这些数据：

```java
// 准备假数据
WorkSchedule workSchedule1 = new WorkSchedule("2019-01-01", "Unit-2, Unit-3");
WorkSchedule workSchedule2 = new WorkSchedule("2019-01-02", "Unit-2");
WorkSchedule workSchedule3 = new WorkSchedule("2019-01-03", "Unit-1, Unit-3");
WorkSchedule workSchedule4 = new WorkSchedule("2019-01-04", "Unit-2, Unit-3");
WorkSchedule workSchedule5 = new WorkSchedule("2019-01-05", "Unit-3");
WorkSchedule workSchedule6 = new WorkSchedule("2019-01-06", "Unit-1, Unit-2");
WorkSchedule workSchedule7 = new WorkSchedule("2019-01-07", "Unit-1");
WorkSchedule workSchedule8 = new WorkSchedule("2019-01-08", "Unit-2, Unit-3");

List<WorkSchedule> workSchedules = new ArrayList<>(8);
workSchedules.add(workSchedule1);
workSchedules.add(workSchedule2);
workSchedules.add(workSchedule3);
workSchedules.add(workSchedule4);
workSchedules.add(workSchedule5);
workSchedules.add(workSchedule6);
workSchedules.add(workSchedule7);
workSchedules.add(workSchedule8);
```

## 不使用流的做法

如果不使用流，那么我们可以通过`for`循环取数据，和使用`TreeSet`实现去重和排序。

```java
TreeSet<String> unitNames = new TreeSet<>();

for (WorkSchedule workSchedule : workSchedules) {
    String[] arrUnitName = workSchedule.getUnits().split(",");

    for (String unitName : arrUnitName) {
        unitNames.add(unitName.trim());
    }
}
```

## 使用流的做法

如果使用流，则可以这样做：

```java
List<String> units = workSchedules.stream()
        // 根据逗号，将班组名拆分成 String[]
        // 这一步将生成8个新的 String[] 对象
        .map(i -> i.getUnits().split(","))
        // 从上面的 String[] 生成一个新的
        .flatMap(Arrays::stream)
        // 切掉每个元素两端的空格
        .map(String::trim)
        // 去重
        .distinct()
        // 排序
        .sorted()
        // 收集到一个List中
        .collect(Collectors.toList());
```

如果上面代码的注释还是无法让您理解，那么，我还准备了每一步流操作的示意图。(感谢`Intellij IDEA`提供了这个强大的功能)

首先，在`map`方法中，`List`的每个元素都会根据逗号分隔，并生成一个`String[]`对象。

![Split](https://blog-static.boris1993.com/flatten-a-group-of-data-by-flatmap/0-split.png)

`flatmap`方法会将上面`map`方法返回的各个流“拼接”成为一个流，即“展平”。

![FlatMap](https://blog-static.boris1993.com/flatten-a-group-of-data-by-flatmap/1-flatmap.png)

接下来继续使用`map`方法，对流中的每个元素进行`trim`操作。

![Trim](https://blog-static.boris1993.com/flatten-a-group-of-data-by-flatmap/2-trim.png)

然后调用`distinct`和`sorted`方法实现去重和排序。

![Distinct](https://blog-static.boris1993.com/flatten-a-group-of-data-by-flatmap/3-distinct.png)  
![Sort](https://blog-static.boris1993.com/flatten-a-group-of-data-by-flatmap/4-sort.png)

最后，使用`collect`方法，将流中的数据“收集”到一个`List`中。

![Collect](https://blog-static.boris1993.com/flatten-a-group-of-data-by-flatmap/5-collect.png)
