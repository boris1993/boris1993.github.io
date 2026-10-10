---
title: "让MyBatis把查询结果以Map形式返回"
source: https://www.boris1993.com/mybatis-return-query-results-in-a-map.html
date: 2019-09-12
updated: 2026-10-10
tags: [Java, MyBatis]
categories: [学知识]
---

# 让MyBatis把查询结果以Map形式返回

在使用MyBatis查询数据库时，偶尔会出现短时间高频查询的情况。或者，会被短时间高频率查询的数据需要通过另一个微服务的接口来获取。显而易见，高频率的查询数据库，或者高频率的调用微服务的接口，是会对性能产生一定影响的。

在这种情况下，或许可以考虑将相关的数据一次性全部查询出来，然后根据某一列作为key，将其存放在一个Map中，把这个Map当成一个临时的缓存。

## 示例数据

假设有这样的一组数据，其中保存了某公司的一系列基地的信息，包括基地名称、基地代号，和基地所在的区。

| id | base\_name | base\_code | location |
| --- | --- | --- | --- |
| 1 | 梅陇 | ML | 徐汇区 |
| 2 | 富锦路 | FJL | 宝山区 |
| 3 | 龙阳路 | LYL | 浦东新区 |
| 4 | 北翟路 | BDL | 普陀区 |
| 5 | 川沙 | CS | 浦东新区 |
| 6 | 石龙路 | SLL | 闵行区 |

## 代码

MyBatis查询的代码还是同往常一样，一个SELECT查询，返回类型就是这个表对应的实体类。

```xml
<select id="queryBaseByNameInMap" resultType="com.boris1993.demoapp.entity.BaseDTO">
    SELECT
        `id`,
        `base_name`,
        `base_code`,
        `location`
        FROM `base`
        WHERE del_flag = '0'
        <if test="baseName != null and baseName != ''">
            AND `base_name` = #{baseName}
        </if>
</select>
```

但是在Mapper接口中，需要在对应的方法上面加上`@MapKey()`注解。这个注解向MyBatis指明了使用**实体的哪一个属性**(而不是列名)作为Map的key。

```java
@MapKey("baseName")
Map<String, BaseDTO> queryBaseByNameInMap(@Param("baseName") String baseName);
```

## 执行结果

编写service、controller等部分不是本文重点，按下不表。在不传递`baseName`时，就可以查出这样子的数据了：

```json
{
  "北翟路": {
    "id": "4",
    "baseName": "北翟路",
    "baseCode": "BDL",
    "location": "普陀区"
  },
  "龙阳路": {
    "id": "3",
    "baseName": "龙阳路",
    "baseCode": "LYL",
    "location": "浦东新区"
  },
  "梅陇": {
    "id": "1",
    "baseName": "梅陇",
    "baseCode": "ML",
    "location": "徐汇区"
  },
  "富锦路": {
    "id": "2",
    "baseName": "富锦路",
    "baseCode": "FJL",
    "location": "宝山区"
  },
  "石龙路": {
    "id": "6",
    "baseName": "石龙路",
    "baseCode": "SLL",
    "location": "闵行区"
  },
  "川沙": {
    "id": "5",
    "baseName": "川沙",
    "baseCode": "CS",
    "location": "浦东新区"
  }
}
```

## 有什么用呢

试想一下，在某个表中，只保存了相关基地的名字，但是在业务中又需要返回这个基地的代号等信息，那么就可以这么做了：

```java
// 首先，调用另一个微服务的接口，把这些基地的信息取出来
Map<String, BaseDTO> bases = baseDataFeignClient.queryBaseByNameInMap();

// 接下来，按照业务需要，查询出了一个List
// 在这个List中，就包含了一系列的基地的名称
List<SomethingOutputVO> aListOfSomething = fetchAListOfSomething();

// 接下来，遍历这个List，逐个从bases中取得基地的代号
for (SomethingOutputVO item : aListOfSomething) {
    item.setBaseCode(bases.get(item.getBaseName()).get("baseCode"));
}
```

此时，这个Map就变成了一个小的缓存，避免了在循环中反复查询数据库或者反复调用微服务接口的问题。
