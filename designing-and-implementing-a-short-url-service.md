---
title: "设计并实现一个简易的短URL服务"
source: https://www.boris1993.com/designing-and-implementing-a-short-url-service.html
date: 2023-02-03
updated: 2026-10-10
tags: [短链接]
categories: [学知识]
---

# 设计并实现一个简易的短URL服务

突然就对短链接服务的原理来了兴趣，于是就查了些资料，自己实现了一个很简陋的演示性的短链接服务。

## 短链接服务是怎么工作的

短链接服务这玩意，说来其实非常简单，就是给用户传来的URL起个别名，然后把别名与原链接的映射关系记录在数据库里。

用户访问短链接时，请求首先会到短链接服务的服务器；短链接服务端收到请求，取出对应的原URL，最后通知用户端的浏览器做个跳转。

## 301跳转？还是302跳转？

尽管按照语义来讲，301跳转更合适，因为一个短URL必定只对应一个长URL，但是看起来生产上更多使用302跳转，因为这样的话请求会经过短网址提供商的服务器，短网址提供商就可以收集到用户的一些信息，然后把这些信息变现。

## 如何生成短链接

上面说到，短链接服务的核心就是要给长链接生成一个“别名”，那么这个别名应该怎么生成呢？

我相信不少人一上来就会想到哈希算法，比如给原URL做个MD5，虽然不是不行，就是哈希算法有碰撞这么个问题，虽然影响不大吧，但处理起来还是个麻烦。

上网一顿冲浪，我发现其实这个生成的算法非常简单，就是直接用发号器生成一个ID，把这个ID跟原链接绑定就行。足够简单，而且不会碰撞。

不过既然都提到这两种算法了，不如顺便介绍一下。

### 发号器方案

发号器方案本质上就是生成分布式ID，如果要简单处理，那么可以使用`Redis`的`incr`操作，或者取数据库的自增序列；复杂情况的话，可以让数据库集群中每个节点各负责生成某一范围的数字，或者使用雪花算法等UUID生成算法。

在得到发号器生成的数字之后，再将其转换为62进制数，就可以当成短URL的ID了。这么做的原因，一方面是可以一定程度上防止直接暴露序列的值产生的安全问题；另一方面，因为为了保证序列够用，发号器返回的数字会比较大，将低进制数转换为高进制数可以显著减少字符数量。

### 哈希算法方案

> 1.  将长网址 md5 生成 32 位签名串,分为 4 段, 每段 8 个字节
> 2.  对这四段循环处理, 取 8 个字节, 将他看成 16 进制串与 0x3fffffff(30位1) 与操作, 即超过 30 位的忽略处理
> 3.  这 30 位分成 6 段, 每 5 位的数字作为字母表的索引取得特定字符, 依次进行获得 6 位字符串
> 4.  总的 md5 串可以获得 4 个 6 位串,取里面的任意一个就可作为这个长 url 的短 url 地址
> 
> 摘自 短网址(short URL)系统的原理及其实现

## 技术选型

解决了理论问题，接下来就要面对现实问题：用什么实现，和跑在哪里。

因为这只是一个演示性的短链接服务，目前定位是就我一个人玩，所以我一方面不想花时间在部署和维护上，另一方面也想趁机玩点没玩过的东西。所以我决定把这玩意放在`CloudFlare Workers`上面，用`TypeScript`语言开发，数据存放在`CloudFlare Workers KV`数据库里。这样，我就只需要关心代码怎么写，其他的包括维护数据库、估算服务器压力这些事都不用担心。

数据库中我需要用两个表，一个表用来存放当前的序列值，和`短URL -> 原URL`的映射，这个表是服务的核心；另一个表用来存放`长URL -> 短URL`的映射，这么设计的原因是，针对相同的长URL，我不需要在生成新的短URL，既节省空间，也能稍微节省点能源不是。

而生成短链接的算法，我当然选择最简单的数据库序列。但因为`CloudFlare Workers KV`并不支持真正的序列，所以我在数据库里面用一个专门的key当作序列来用。这个选型有一个风险就是，在高并发状态下我无法保证序列的值不会重复，因为`取出序列 -- 生成ID -- 保存新的序列`这个操作不是原子性的，高并发状态下可能会有多个请求同时取到相同的序列，进而生成相同的ID，最后就会产生错误的结果。不过，还是那句话，就我一个人用的玩意，暂时先不考虑那么多。

## 流程

这个服务的流程分两大部分，生成新的短URL，和查询短URL并完成跳转。查询操作没什么梗，查到了就返回，查不到就404呗。

生成新的短URL的话，大致就是这么个流程：

```
graph TD;
    start[开始];
    finish[结束];
    request_received[收到生成的请求];
    check_existing_record{检查是否已经生成过};
    return_existing_record[返回已有的短URL];
    fetch_current_sequence[查询当前的序列];
    calculate_base62[计算序列的62进制数值];
    increase_sequence_number[序列增1];
    save_to_database[将短URL和新的序列存入数据库];
    return_new_generated_short_url[返回生成的短URL];
    
    start --> request_received;
    request_received --> check_existing_record;

    check_existing_record -->|Y| return_existing_record;
    return_existing_record --> finish;

    check_existing_record -->|N| fetch_current_sequence;
    fetch_current_sequence --> calculate_base62;
    calculate_base62 --> increase_sequence_number;
    increase_sequence_number --> save_to_database;
    save_to_database --> return_new_generated_short_url;
    return_new_generated_short_url --> finish;
```

## 代码实现

这里就只放具体实现相关的代码了，完整的代码库可以到参考文档第一条的GitHub仓库看到。

```typescript
import * as url from 'url';
import { RequestBody, ResponseBody, ShortUrl } from './model';

// 起始的序列值
const INITIAL_SEQUENCE_NUMBER = 100000;

export interface Env {
    [x: string]: any;
}

export default {
    async fetch(
        request: Request,
        env: Env,
        ctx: ExecutionContext
    ): Promise<Response> {
        switch (request.method) {
            case 'POST':
                return await handlePostRequest(request, env);
            case 'GET':
            default:
                return await handleGetRequest(request, env);

        }
    },
};

async function handleGetRequest(
    request: Request,
    env: Env
): Promise<Response> {
    // 取URL中的path部分
    let url_parts = url.parse(request.url);
    let path = url_parts.pathname;

    // 如果没有path部分，或者path有多层
    // 那么视为无效请求
    // 合法的短URL格式为：https://mydomain.com/RlB2PdD
    if (path == null || path.split(/\/(?=.)/).length !== 2) {
        console.info("No short URL key provided or invalid path. Returning 400");
        return new Response("No short URL key provided or the path is invalid.", {
            status: 400
        });
    }

    
    let pathParts = path?.split("/");

    // 专门处理下favicon.ico的请求
    // 可能是我的实现有问题，不一定必须
    if (pathParts[1] === "favicon.ico") {
        return new Response();
    }

    // 取出path，即短URL的key
    let key = pathParts[1];

    console.info(`Looking for the target URL with key ${key}`);

    // 对env.SHORT_URL操作，就是对SHORT_URL这个KV数据库做操作
    // 这里就是从数据库中查询这个key对应的长URL
    let shortUrlJson = await env.SHORT_URL.get(key);
    if (shortUrlJson === null) {
        console.info(`No target URL found for key ${key}`);
        return new Response("No target URL found", {
            status: 404
        });
    }

    // 构造返回的JSON，然后返回一个HTTP 302让浏览器跳转
    let shortUrlObject = JSON.parse(shortUrlJson) as ShortUrl;

    console.info(`Target URL for key ${key} is ${shortUrlObject.url}`);

    return Response.redirect(shortUrlObject.url, 302);
}

async function handlePostRequest(
    request: Request,
    env: Env
): Promise<Response> {
    let requestBody = await request.json() as RequestBody;
    let targetUrl = requestBody.url!;

    console.info(`Creating a short URL for target ${targetUrl}`);

    // 查询这个长URL是否已经有对应的短URL
    // SHORT_URL_MAPPING表记录的是长URL对应的短URL
    let existingShortUrl = await env.SHORT_URL_MAPPING.get(targetUrl) as string;
    if (existingShortUrl !== null) {
        // 查到了，就直接返回
        console.info(`Existing short URL key ${existingShortUrl} found for ${targetUrl}`);
        let responseBody = new ResponseBody(existingShortUrl);
        return new Response(
            JSON.stringify(responseBody),
            {
                status: 201,
                headers: {
                    'content-type': 'application/json'
                }
            });
    }

    // 取出当前的序列值，将其转换为62进制，作为短URL的key
    let curentSequence = await getCurrentSequence(env);
    let key = string10to62(curentSequence);

    let data = new ShortUrl(targetUrl);

    // 保存短URL，更新序列
    await env.SHORT_URL.put(key, JSON.stringify(data));
    await env.SHORT_URL_MAPPING.put(targetUrl, key);
    await env.SHORT_URL.put("sequence", `${++curentSequence}`);

    console.info(`Created a new short URL key ${key} for ${targetUrl}`);

    // 返回生成的结果
    let responseBody = new ResponseBody(key);
    return new Response(
        JSON.stringify(responseBody),
        {
            status: 201,
            headers: {
                'content-type': 'application/json'
            }
        });
}

/**
 * 取出当前的序列值，如果数据库中未初始化，
 * 那么就将初始序列写入数据库，然后返回初始序列。
 * 这个方法不涉及序列的更新
 */
async function getCurrentSequence(env: Env): Promise<number> {
    let currentSequence = await env.SHORT_URL.get("sequence");

    if (currentSequence === null) {
        await env.SHORT_URL.put("sequence", `${INITIAL_SEQUENCE_NUMBER}`);

        return INITIAL_SEQUENCE_NUMBER;
    }

    return currentSequence;
}

/**
 * 将10进制数转换为62进制
 */
function string10to62(number: number) {
    var chars = '0123456789abcdefghigklmnopqrstuvwxyzABCDEFGHIGKLMNOPQRSTUVWXYZ'.split('');
    var radix = chars.length;
    var qutient = +number;
    var arr = [];
    do {
        let mod = qutient % radix;
        qutient = (qutient - mod) / radix;
        arr.unshift(chars[mod]);
    }
    while (qutient);
    return arr.join('');
}
```

## 一些改进空间

因为针对相同的长URL并不需要每次都返回相同的短URL，所以`长URL -> 短URL`表中，我可以给每条记录都加一个TTL，在有效期内，每次针对相同的长URL的生成请求都会返回同一个短URL，同时刷新TTL；而超过有效期后，这条映射就会被删除，对应的长URL则会生成新的短URL。这样一定程度上既可以防止恶意刷接口炸数据库，同时也可以清除掉不太可能再被用到的数据。

而在如上改动的影响下，必然会出现多个短URL对应同一个长URL的情况，这多少也是浪费了一些空间。所以我感觉可以在`短URL -> 长URL`映射表中，增加一个最后访问时间字段，每有一个短URL的请求，就更新这个时间到请求的时间。再启动一个定时任务，定时扫描每个短链接的最后访问时间，并将在指定时间（如半年）内没有被访问过的短链接删除。（我觉得，应该没有人把短链接当成永久链接吧？就算不考虑被删，万一服务商跑路了呢？

此外，还可以给`短URL -> 长URL`映射表中再增加一个访问次数字段，以便结合其他收集到的数据来做分析。

## 参考文档

-   cf-worker-short-url - GitHub
-   短网址服务(TinyURL)生成算法
-   短 URL 系统是怎么设计的？ - iammutex的回答 - 知乎
-   短网址(short URL)系统的原理及其实现
