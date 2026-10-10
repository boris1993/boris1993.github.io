---
title: "SwiftUI学习记录 - 使用Alamofire加载JSON并初始化列表"
source: https://www.boris1993.com/swiftui-initialize-list-with-remote-data.html
date: 2023-10-28
updated: 2026-10-10
tags: [iOS开发, SwiftUI, List, Alamofire]
categories: [学知识]
---

# SwiftUI学习记录 - 使用Alamofire加载JSON并初始化列表

这两天又捡起来了之前开的一个老坑，继续完成X岛揭示板的iOS客户端，而且刚刚完成了从JSON初始化版面列表的功能。  
这部分感觉最难的还是上手`Alamofire`，因为它返回结果不像我平时做Web开发那样通过方法返回（也有可能是我没学到位），而是要把反序列化得到的对象传给一个回调方法。而这个思路的差异也导致我刚开始学的时候非常的痛苦，因为怎么也找不到我想要的那种返回方式。  
我相信应该不止我一个人会遇到这种情况，所以打算在这里把完整的实现过程记录在这里，并希望后面有类似情况的同志能因为这篇文章而少掉几根头发。

## 定义JSON对应的结构体

X岛揭示板的`版面列表`API会返回一个类似这样的JSON：

```json
[
  {
    "id": "4",
    "sort": "1",
    "name": "综合",
    "status": "n",
    "forums": [
      {
        "id": "-1",
        "name": "时间线",
        "msg": "这里是匿名版最新的串"
      },
      {
        "id": "23",
        "fgroup": "3",
        "sort": "12",
        "name": "暴雪游戏",
        "showName": "暴雪游戏",
        "msg": "•本版发文间隔为15秒。",
        "interval": "15",
        "safe_mode": "0",
        "auto_delete": "0",
        "thread_count": "72",
        "permission_level": "0",
        "forum_fuse_id": "0",
        "createdAt": "2012-05-25 21:21:21",
        "updateAt": "2015-04-21 12:30:39",
        "status": "n"
      }
    ]
  }
]
```

所以，我们可以创建一个这样的结构体来用来反序列化它：

```swift
struct ForumGroup: Codable, Identifiable {
    var id: String
    var sort: String
    var name: String
    var status: String
    var forums: [Forum]

    private enum CodingKeys: String, CodingKey {
        case id
        case sort
        case name
        case status
        case forums
    }
}

struct Forum: Codable, Identifiable {
    var id: String = ""
    var fGroup: String? = ""
    var sort: String? = ""
    var name: String = ""
    var showName: String? = ""
    var msg: String = ""
    var interval: String? = ""
    var threadCount: String? = ""
    var permissionLevel: String? = ""
    var forumFuseId: String? = ""
    var createdAt: String? = ""
    var updateAt: String? = ""
    var status: String? = ""

    private enum CodingKeys: String, CodingKey {
        case id
        case fGroup
        case sort
        case name
        case showName
        case msg
        case interval
        // 因为X岛揭示板的API存在CamelCase和snake_case混用的情况
        // 所以需要CodingKeys来配置正确的映射
        case threadCount = "thread_count"
        case permissionLevel = "permission_level"
        case forumFuseId = "forum_fuse_id"
        case createdAt
        case updateAt
        case status
    }
}
```

## 编写网络请求

创建一个新的Swift文件`AnoBbsApiClient`，编写如下代码：

```swift
final class AnoBbsApiClient {
    private static let logger = LoggerHelper.getLoggerForNetworkRequest(name: "AnoBbsApiClient")

    public static func loadForumGroups(
        completion:@escaping ([ForumGroup]) -> Void,
        failure:@escaping (String) -> Void
    ) {
        let url = URL(string: XdnmbAPI.GET_FORUM_LIST)!
        AF.request(url, method: .get, interceptor: .retryPolicy) { $0.timeoutInterval = 10 }
            .cacheResponse(using: .cache)
            .validate()
            .responseDecodable(of: [ForumGroup].self) { response in
                switch response.result {
                case .success(let data):
                    completion(data)
                case .failure(let error):
                    failure(error.localizedDescription)
                }
            }
    }
}
```

是的，就这几行代码，花了我大概一整天时间来学明白，定稿之前不知道来来回回试了多少遍。前面都很好懂，重点就是`responseDecodable`这个方法调用，`of`参数指明我希望把返回的JSON反序列化成一个`ForumGroup`列表，后面的方法块中根据成功反序列化和发生任何错误的情况，分别调用`completion`和`failure`这两个回调方法。

## 渲染列表

现在回到展示版面的`ForumsView`，在`body`里面做如下实现：

```swift
var body: some View {
    NavigationStack {
        List {
            ForEach($forumGroups) { $forumGroup in
                Section {
                    ForEach(forumGroup.forums) { forum in
                        NavigationLink(destination: CookieListView(globalState: globalState)) {
                            if (forum.showName == nil || forum.showName!.isEmpty) {
                                Text(forum.name)
                            } else {
                                Text(forum.showName!)
                            }
                        }
                    }
                } header: {
                    Text(forumGroup.name)
                }
            }
        }
    }
    .onAppear {
        if (!isContentLoaded) {
            globalState.loadingStatus = String(localized: "msgLoadingForumList");
            shouldDisplayProgressView = true;

            AnoBbsApiClient.loadForumGroups { forumGroups in
                self.forumGroups = forumGroups
                isContentLoaded = true
                shouldDisplayProgressView = false;
            } failure: { error in
                showErrorToast(message: error)
                shouldDisplayProgressView = false;
            }
        }
    }
    .toast(isPresenting: $isErrorToastShowing) {
        AlertToast(type: .regular, title: errorMessage)
    }
}
```

在这个View展示时，如果版面列表没有被加载，那么就调用刚刚写的`loadForumGroups`方法获取版面列表，后面的第一个代码块就是`completion`这个回调的实现，负责把`loadForumGroups`方法得到的结果传给一个`@State`变量`forumGroups`，以及标记内容已经成功载入，并隐藏载入提示的风火轮；第二个代码块是`failure`这个回调的实现，负责显示一个带有错误信息的`Toast`并隐藏风火轮。

在这个`View`的`NavigationStack`里面，就可以监听`forumGroups`这个`@State`变量，并用变量里面的内容来渲染整个列表了。最后，我们就可以得到这样一个结果：

![](https://blog-static.boris1993.com/swiftui-initialize-list-with-remote-data/forum-list-view.jpeg)
