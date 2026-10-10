---
title: "记我将博客的图片存储迁移到Backblaze"
source: https://www.boris1993.com/migrating-blog-images-to-backblaze.html
date: 2023-05-15
updated: 2026-10-10
tags: [对象存储, OSS, 博客图片, 博客]
categories: [瞎折腾]
---

# 记我将博客的图片存储迁移到Backblaze

之前我一直把博客的图片放在个人的OneDrive上，然后用嵌入功能得到外链放在博客里。但是考虑到国内访问OneDrive的延迟还是偏高，以及不折腾不舒服的心理作祟，于是乎在三月份的时候，我把博客的图片从OneDrive迁到了Backblaze对象存储。

## 前期准备

开始之前，我们需要准备好这些东西：

-   一个Backblaze免费账号
-   一个CloudFlare免费账号
-   一个域名
-   还有你的好心情 :-)

可能你会担心用对象存储是不是会产生高额的账单，或者会因为超出配额导致图片全部无法加载。说实话我之前不敢用对象存储就是因为有这方面的顾虑，但是在Backblaze这里我们完全不用担心。首先，我们用的是免费的账户，而且Backblaze甚至不要求你添加信用卡。此外，Backblaze和CloudFlare都是带宽联盟的成员，意味着Backblaze与CloudFlare之间的流量全部是免费的。

## 在Backblaze创建存储桶并上传图片

登录进Backblaze的B2 Cloud Storage之后，点`Create a Bucket`创建一个存储桶就行了。为了防止被人恶意刷流量，我建议创建一个私有的存储桶。加密和对象锁都不需要。

创建成功后，打开这个存储桶的`Bucket Settings`，在`Bucket Info`中添加`{"cache-control":"max-age=43200"}`来配置桶的缓存时间。虽然流量不要钱，但是能环保还是环保一点比较好不是？

因为我们创建的是私有存储桶，所以需要创建一个Application Key来允许第三方服务访问这个桶。虽然Backblaze默认提供了一个Master Application Key，但是这就像天天用root登录Linux主机一样，只有中午才能用，因为早晚会出事。在Application Keys页面，点`Add a New Application Key`，`Allow access to Bucket(s)`里面建议选我们这个桶而不是All，权限当然是Read and Write。创建成功之后，注意保存好`keyID`和`applicationKey`，因为`applicationKey`只会显示一次。

然后需要下载一个支持浏览对象存储的工具，比如我用的S3 Browser。然后在S3 Browser中新建一个连接，`REST Endpoint`填写存储桶的`Endpoint`，`Access Key ID`就是刚才记下来的`keyID`，`Secret Access Key`就是`applicationKey`。

![](https://blog-static.boris1993.com/migrating-blog-images-to-backblaze/s3-browser-add-account.png)

如果S3 Browser可以成功连接到刚才创建的存储桶，那就说明配置正确了。这时候就可以想好目录结构，以及上传图片了。比如我选择把图片按照对应的博文来分类，每个有图的博文都有一个对应的图片目录。

![](https://blog-static.boris1993.com/migrating-blog-images-to-backblaze/s3-browser-list-files.png)

## 在CloudFlare中配置域名

在到CloudFlare配置域名之前，我们先要知道指向一个文件的完整URL。进入Browse Files页面，然后进入这个存储桶，接着随便挑一个文件，点它最右边的详情图标，这里的`Friendly URL`就是我们要找的东西。记下URL里面的域名，我们接下来要用到。

![](https://blog-static.boris1993.com/migrating-blog-images-to-backblaze/backblaze-file-details.png)

接下来就可以到CloudFlare里面创建一条CNAME记录，并把刚才记下来的域名填到目标里面，并且启用CloudFlare的代理，这样我们才能享受到带宽联盟的优惠。此外，我们还会针对这个域名配置一些规则，这也需要打开CloudFlare的代理开关。

要注意这里只能是二级域名，如`blog-static.boris1993.com`，而不能是多级的（`blog.static.boris1993.com`），否则CloudFlare会无法申请证书，也就无法正常启用HTTPS。

![](https://blog-static.boris1993.com/migrating-blog-images-to-backblaze/cloudflare-cname-record.png)

这时候我们就可以用`https://sub-domain.your-domain.com/file/folder-name/image-name.png`访问这个图片了，但是目前我们只能得到一个401页面，因为我们必须要带上一个Access Token才能访问私有存储桶的文件。

## 为请求配置CloudFlare规则

前往CloudFlare的规则页面，选择`转换规则(Transform Rule)`，然后在`重写URL`这个tab中新增一个规则。

首先，我希望我可以直接用`https://blog-static.boris1993.com/folder-name/file-name.png`就能访问到图片（因为这样看起来更好看），所以我配置了一个路径重写，如果路径中不包含`/file/bucket-name`，那么就在路径中补上这一段。

选择`路径`的`重写到`，表达式类型选择`动态`，表达式填写`concat("/file/blog-pics", http.request.uri.path)`。这样CloudFlare就会自动补全完整的路径。

然后就是访问私有存储桶的Access Token。Backblaze支持把Access Token放在`Authorization`这个query parameter中，所以我们可以选择`查询`的`重写到`，表达式类型选择`静态(Static)`，值目前可以随便写，因为你就算现在拿到一个token，在24小时后也是会过期的，所以后面我会讲怎么用CloudFlare Workers来更新这个字段。

![](https://blog-static.boris1993.com/migrating-blog-images-to-backblaze/cloudflare-transform-rule.png)

接下来，根据Backblaze官方的建议，我们需要对响应头做一些修改。

切换到`修改响应头`，新增这样一条规则：

![](https://blog-static.boris1993.com/migrating-blog-images-to-backblaze/cloudflare-update-headers.png)

首先要正确配置`Access-Control-Allow-Origin`，来避免跨域问题，我偷懒了直接配了个`*`，不知道这么配会不会有盗链的问题，暂时先这样吧。

其次Backblaze建议修改`cache-control`这个header，来延长缓存的有效时间。

最后，需要从响应头中删掉一些Backblaze的header来增强安全性。

为了方便，我把要删掉的header放在这里：

-   x-bz-content-sha1
-   x-bz-file-id
-   x-bz-file-name
-   x-bz-info-s3b-last-modified
-   x-bz-info-sha256
-   x-bz-info-src\_last\_modified\_millis
-   x-bz-upload-timestamp

同时我为了能让浏览器缓存这个图片，我还让它添加了`ETag`这个header，但是我在浏览器里一直看不到这个header，如果有大佬知道为什么，还请不吝赐教。

## 自动更新访问存储桶的Token

因为后面要修改规则的内容，所以先得拿到规则集和规则的ID。规则ID好办，打开`重写URL`规则的编辑页面，我们就能在URL的最后一段得到这个规则的ID。但是规则集ID只能调CloudFlare API取得。

```http
GET https://api.cloudflare.com/client/v4/zones/YOUR_ZONE_ID/rulesets
Authorization: Bearer YOUR_CLOUDFLARE_API_TOKEN
```

把`YOUR_ZONE_ID`替换为你的域名的区域ID，以及把`YOUR_CLOUDFLARE_API_TOKEN`换成你的API令牌。我当时因为不知道这个API需要哪些权限，始终创建不出带有正确权限的API令牌，所以干脆用了`Global API Key`。

这个请求会返回一系列规则集，有CloudFlare内部的，也有我们自己的。理论上，名字是`default`并且`phase`是`http_request_transform`的那个就是我们要的。但是为了确认，可以再执行这个请求：

```http
GET https://api.cloudflare.com/client/v4/zones/YOUR_ZONE_ID/rulesets/RULE_SET_ID
Authorization: Bearer YOUR_CLOUDFLARE_API_TOKEN
```

跟上条请求一样，替换掉`YOUR_ZONE_ID`和`YOUR_CLOUDFLARE_API_TOKEN`，以及将`RULE_SET_ID`替换为上面找到的规则集的`id`。执行后会返回这个规则集下的规则。如果返回内容中有我们之前创建的那条`重写URL`的规则，那么这就是我们要找的规则集。

然后为了安全起见，我们要为这个CloudFlare Worker创建一个API令牌。进入`我的个人资料` –> `API令牌`，然后点击`创建令牌`，在接下来的页面中中选择`创建自定义令牌`，然后如图创建一个令牌。

![](https://blog-static.boris1993.com/migrating-blog-images-to-backblaze/cloudflare-token-for-worker.png)

添加成功后，妥善保存这个令牌。

接下来前往CloudFlare Workers，创建一个新的Worker。然后到`设置` –> `变量`，添加如下环境变量：

| 变量名 | 值 |
| --- | --- |
| B2KeyID | Backblaze的`keyID` |
| B2AppKey | Backblaze的`applicationKey` |
| B2BucketName | Backblaze的存储桶名 |
| CfAuthKey | 上面创建的CloudFlare API令牌 |
| CfHostname | 上面在CloudFlare创建的二级域名 |
| CfZoneID | 你的域名的区域ID |
| CfRulesetID | 上面拿到的规则集ID |
| CfRuleID | 上面拿到的规则ID |

然后进入`触发器`，将`路由`中的那条记录禁用，因为我们不会用HTTP请求来触发这个Worker。然后再`Cron触发器`中添加一个Cron触发器。Backblaze说一个token的有效期最大不超过24小时，我为了保险起见，选择每半小时就触发这个Worker来生成一个新的token，即`*/30 * * * *`。

![](https://blog-static.boris1993.com/migrating-blog-images-to-backblaze/cloudflare-worker-trigger.png)

至此前置任务完成，点击右上角的`快速编辑`，然后将如下脚本粘贴进去，然后点击`保存并部署`。

```javascript
addEventListener("fetch", (event) => {
  event.respondWith(handleRequest(event.request));
});
addEventListener("scheduled", (event) => {
  event.waitUntil(updateRule());
});

const getB2Token = async () => {
  const res = await fetch(
    "https://api.backblazeb2.com/b2api/v2/b2_authorize_account",
    {
      headers: {
        Authorization: "Basic " + btoa(B2KeyID + ":" + B2AppKey),
      },
    }
  );
  const data = await res.json();
  return data.authorizationToken;
};

const updateRule = async () => {
  const b2Token = await getB2Token();

  const res = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${CfZoneID}/rulesets/${CfRulesetID}/rules/${CfRuleID}`,
    {
      method: "PATCH",
      headers: {
        "Authorization": `Bearer ${CfAuthKey}`
      },
      body: 
      `{
        "description": "Replace path for static files for blog",
        "action": "rewrite",
        "expression": "(http.host eq \\\"${CfHostname}\\\" and not starts_with(http.request.uri.path, \\\"/file/${B2BucketName}\\\"))",
        "action_parameters": {
          "uri": {
              "path": {
                  "expression": "concat(\\\"/file/${B2BucketName}\\\", http.request.uri.path)"
              },
              "query": {
                  "value": "Authorization=${b2Token}"
              }
          }
        }
      }`,
    }
  );

  const data = await res.text();
  console.log(data);
  return data;
};

async function handleRequest(request) {
  const data = await updateRule();
  return new Response(data);
}
```

等Worker被触发之后，就可以在浏览器中访问上面配置的域名，来测试到存储桶的连接是否正常。如果测试没问题，就可以把博客中的图片链接换到新地址了。
