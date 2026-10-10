---
title: "使用Coze工作流将小红书笔记保存至Notion - 工作流配置"
source: https://www.boris1993.com/save_rednote_to_notion_with_coze_workflow.html
date: 2025-06-08
updated: 2026-10-10
tags: [小红书, 工作流, Coze, 智能体]
categories: [瞎折腾]
---

# 使用Coze工作流将小红书笔记保存至Notion - 工作流配置

半个月前我的小红书号不明不白的被封了。而在被封之后我才发现，“我”这个页面也彻底被屏蔽，我再也看不到我收藏的帖子。痛定思痛，既然保存在小红书上的收藏有再也看不到的可能，那么，我把帖子保存到我自己的Notion不就好了？

说干就干，在一顿网上冲浪之后，我看到有人已经实现了利用字节跳动火山引擎的“扣子（Coze）”实现了这个功能，但作者并没有放出来具体的实现。我一方面很不喜欢需要私信进群这种操作，另一方面还是自己做一个才最符合自己的需求（拜托，我可是程序员欸），所以自己动手实现了一个工作流，可以完整转存帖子的标题、内容、图片到自己的Notion。

## 大致的功能设计

关于这个工作流，我希望它能做到这些：

-   自动解析小红书分享链接（类似`94 某某某发布了一篇小红书笔记，快来看吧！ 😆 zop6CK2fPRDGwIT 😆 http://xhslink.com/a/5ZOdZhppuYteb，复制本条信息，打开【小红书】App查看精彩内容！`）的这种。我需要它能从这个分享链接中获取到对应笔记的真实地址。哦对，这条链接就不用试了，因为这是一条我自己编的。
-   自动解析小红书笔记，将里面的标题、内容、图片全部提取出来。
-   在成功提取出笔记内容之后，它可以将图片上传到Notion，并将其嵌入到Notion文档中。
-   可以通过手机直接分享到工作流，而不是拷出来分享链接，再打开Coze手动运行它。

我不需要它做的是：

-   转存视频。因为视频我可以直接保存到相册，或者手机录屏。而且对我来说，视频的信息密度比文字低，就算保存下来，我也要再把内容提炼成文字版保存。
-   转存评论。评论区的内容通常会很杂乱，没有全部转存的必要。有价值的内容我再手动拷到自己的帖子里面就是了。
-   监控小红书收藏夹。我觉得它暂时没这个本事，要做到这个，我可能需要单独开发一个程序。

## 简单介绍一下它具体是怎么工作的

这个工作流看起来是这样的

![](https://blog-static.boris1993.com/save_rednote_to_notion_with_coze_workflow/coze_workflow_save_rednote_to_notion_overview.png)

具体工作起来，会进行下面这几步操作：

-   首先使用配置好的token尝试连接Notion，如果连不上，那也就别费劲干后面的事了，赶紧报错通知用户检查
-   接下来先解析分享链接，把实际的短链接拿出来，然后再用短链接换到对应的长链接
-   得到长链接后，就可以调用插件得到这个笔记的标题、内容、图片地址，并能以JSON格式输出
-   得到了笔记的内容之后，我就可以把图片逐个下载下来，再上传到Notion，然后使用这些内容创建一篇帖子
-   最后，如果帖子创建成功，那么返回帖子的链接

## 在Notion中要做的准备

我的这个工作流是基于Notion的数据库实现的，所以首先需要在Notion中创建一个新的数据库。这个数据库将会包含三个列：

-   Name - 数据库默认存在并不可删除的列，作为帖子的标题
-   Images - 一个`Files & media`类型的列，图片将在这里保存及展示
-   Tags - 一个`Multi-select`类型的列，用于保存帖子的标签

最终你应该会得到一个这样的数据库：

![](https://blog-static.boris1993.com/save_rednote_to_notion_with_coze_workflow/notion_database.png)

接下来，因为我们需要调用Notion的API来上传图片和创建帖子，所以我们需要前往`Notion Integrations`页面为其创建一个API key，配置它的权限，并将刚才创建好的数据库与它绑定。

首先点`New Integration`创建一个API key。名字无所谓，类型选`Internal`。

创建成功后会自动跳转到`Configuration`页面。在`Capabilities`的`Content Capabilities`中勾选`Read`、`Update`和`Insert`权限，然后在`User Capabilities`中选择`No user information`。

![](https://blog-static.boris1993.com/save_rednote_to_notion_with_coze_workflow/notion_integration_capabilities.png)

接下来到`Access`页面，点`Edit access`并勾上刚刚创建的数据库。

![](https://blog-static.boris1993.com/save_rednote_to_notion_with_coze_workflow/notion_integration_access.png)

至此Notion部分的准备工作就结束了。

## 创建工作流

接下来就是到扣子平台创建工作流了。登陆后点击`创建`，然后选择`创建智能体`，起个名字点击创建，就会进入智能体的编辑界面。

### 配置工作流的变量

首先点击`变量`部分的加号，并创建三个变量：

-   `rednote_cookie` - 小红书的cookie，解析笔记内容要用到，并且这个cookie似乎不能来自被封禁的账号
-   `notion_page_root` - Notion数据库的ID，在Notion中点进去上面创建的数据库，将浏览器地址栏中的URL去掉`https://www.notion.so/`和问号后面的东西，剩下的这部分就是数据库ID
-   `notion_secret` - Notion的API key，回到创建API key的页面，点击`Internal Integration Secret`的`Show`并把内容拷过来就行

![](https://blog-static.boris1993.com/save_rednote_to_notion_with_coze_workflow/coze_workflow_variables.png)

然后，就可以开始开发工作流本体了。

### 配置工作流组件

这里我会逐个介绍每个节点的配置，至于节点之间的连接关系，如果你看文字看不明白的话，就参考上面的截图。

-   在**开始**节点添加一个`string`类型的输入，名为`rednote_share`，用来传入小红书的分享链接
-   添加[**notion数据库助手服务的get\_database\_info节点**](https://www.coze.cn/store/plugin/7489686963055018025)，添加两个输入变量
    -   `secret`引用用户变量`notion_secret`
    -   `url`填写数据库的URL，就是从浏览器地址栏复制出来的那个，记得去掉问号和后面的东西
-   添加一个**选择器**节点，检查**get\_database\_info**节点的`code`输出字段是否不等于0。如果满足条件，即说明数据库连接不成功，应当报错并退出
-   为上一步的**选择器**节点的**如果**添加一个**输出**节点
    -   添加两个输出变量
        -   `database_id`引用**get\_database\_info**的输出字段`id`
        -   `message`引用**get\_database\_info**的输出字段`msg`
    -   配置输出内容为`连接Notion数据库{{database_id}}失败，错误信息：{{message}}`
-   再为上一步的**选择器**节点的**否则**添加一个[**小红书的get\_url节点**](https://www.coze.cn/store/plugin/7454184307662405659)，其输入变量`share_content`引用**开始**节点的`rednote_share`
-   继续为**get\_url**添加[**小红书连接处理的shoturl节点**](https://www.coze.cn/store/plugin/7425905322386096140)，其输入变量`short_url`引用上一步**get\_url**节点的`url`输出字段
-   然后添加一个[**小红书笔记详情的GetNoteData节点**](https://www.coze.cn/store/plugin/7492043573437792293)，其输入变量`url`引用上一步**shoturl**节点的`long_url`输出字段
-   接下来创建一个**代码**节点，这是整个工作流的核心
    -   添加如下输入变量：
        -   `title`引用**GetNoteData**的`title`输出字段
        -   `content`引用**GetNoteData**的`content`输出字段
        -   `images`引用**GetNoteData**的`images`输出字段
        -   `tags`引用**GetNoteData**的`tags`输出字段
        -   `parent_id`引用用户变量`notion_page_root`
        -   `secret`引用用户变量`notion_secret`
    -   添加一个类型为`Object`的输出变量`response`，并添加如下输出字段
        -   `status`，类型为`Integer`
        -   `message`，类型为`String`
        -   `url`，类型为`String`
    -   点击代码节点的`在IDE中编辑`，清空内容并粘贴如下代码片段（我知道，这代码很不Pythonic）

```python
import os
import tempfile
import requests_async as requests


async def main(args: Args) -> Output:
    params = args.params

    parent_id = params['parent_id']
    secret = params['secret']
    title = params['title']
    content = params['content']
    images = params['images']
    tags = params['tags']

    file_upload_id_list = await transfer_images(secret, images)
    post = build_post(parent_id, title, content, file_upload_id_list, tags)
    create_post_response = await create_post(secret, post)

    # 构建输出对象
    ret: Output = {
        "response": create_post_response
    }
    return ret


async def transfer_images(secret: str, images: list[str]) -> list[str]:
    file_upload_id_list = []
    for image in images:
        tmp_file_path = await download_image_file(image)
        file_upload_id = await upload_file_to_notion(secret, tmp_file_path)
        file_upload_id_list.append(file_upload_id)

    return file_upload_id_list


async def download_image_file(url: str) -> str:
    tmp_file = tempfile.NamedTemporaryFile(delete=False, suffix='.jpg')
    tmp_file_path = tmp_file.name
    tmp_file.close()

    response = await requests.get(url)
    with open(tmp_file_path, 'wb') as file:
        file.write(response.content)
    
    return tmp_file_path


async def upload_file_to_notion(secret: str, file_path: str) -> str:
    # Initiate file upload
    url = 'https://api.notion.com/v1/file_uploads'
    response = await requests.post(url, headers={
        'Authorization': 'Bearer ' + secret,
        'Notion-Version': '2022-06-28',
    }, data={
        'mode': 'single_part',
    })
    file_upload_id = response.json()['id']

    # Upload the file
    with open(file_path, 'rb') as file:
        files =  {
            'file': (os.path.basename(file_path), file, 'image/jpeg')
        }
        url = f'https://api.notion.com/v1/file_uploads/{file_upload_id}/send'
        await requests.post(url, headers={
            'Authorization': 'Bearer ' + secret,
            'Notion-Version': '2022-06-28',
        }, files=files)
    
    return file_upload_id


def build_post(parent_id: str, title: str, content: str, file_ids: list[str], tags: list[str]) -> object:
    content_blocks = build_content_blocks(content)
    
    files = []
    for file_id in file_ids:
        files.append({
            'type': 'file_upload',
            'file_upload': {
                'id': file_id
            }
        })


    return {
        'parent': {
            'database_id': parent_id
        },
        'properties': {
            'Name': {
                'title': [
                    {
                        'text': {
                            'content': title
                        }
                    }
                ]
            },
            'Images': {
                'files': files
            },
            'Tags': {
                "multi_select": build_tag_properties(tags)
            }
        },
        'children': content_blocks
    }


def build_content_blocks(content: str) -> list[object]:
    blocks = [];
    paragraphs = content.replace('\t', '').split('\n')
    for paragraph in paragraphs:
        blocks.append({
            'object': 'block',
            'type': 'paragraph',
            'paragraph': {
                'rich_text': [
                    {
                        'type': 'text',
                        'text': {
                            'content': paragraph
                        }
                    }
                ]
            }
        })

    return blocks;


def build_tag_properties(tags: list[str]) -> list[object]:
    items = []
    for tag in tags:
        items.append({
            "name": tag
        })
    return items


async def create_post(secret: str, post: object) -> str:
    url = 'https://api.notion.com/v1/pages'
    response = await requests.post(url, headers={
        'Authorization': 'Bearer ' + secret,
        'Notion-Version': '2022-06-28',
    }, json=post)
    return response.json()
```

-   为**代码**节点添加一个**选择器**节点，判断**代码**节点的输出字段`status`是否为`200`，并分别为其添加如下后续节点
    -   **如果**部分添加输出节点，添加一个输入变量`output`，引用**代码**节点的`url`输出字段，输出内容为`{{output}}`，负责在帖子创建成功后输出帖子URL
    -   **否则**部分添加输出节点，添加一个输入变量`output`，引用**代码**节点的`message`输出字段，输出内容为`{{output}}`，负责在帖子创建失败后输出错误信息
-   添加一个**变量聚合**节点并关联**连接Notion失败的输出节点**、**创建帖子成功的URL输出节点**、**创建帖子失败的错误信息输出节点**，其`Group1`的配置参考下图  
    ![](https://blog-static.boris1993.com/save_rednote_to_notion_with_coze_workflow/coze_workflow_variable_aggregate.png)
-   最后为**变量聚合节点**添加一个**结束**节点，添加一个输出变量`output`引用**变量聚合节点**的`Group1`输出字段

如果最后你的工作流与我上面的截图一致，那么就大致可以认为你的工作流配置是正确的了。接下来就可以点击`试运行`按钮，贴一个小红书分享链接，来测试这个工作流。一切正常的话，工作流会输出一个Notion的URL，同时你在Notion的这个数据库中就能看见刚刚转存过来的帖子。

![](https://blog-static.boris1993.com/save_rednote_to_notion_with_coze_workflow/notion_saved_post_example.png)

## 接下来的计划

目前这个工作流虽然实现了转存小红书笔记的功能，但仍有下面几个问题：

-   小红书的cookie不会自动刷新，那么在cookie过期后，整个工作流就不能正常运行，需要用户手动去浏览器拷贝小红书的cookie并进入智能体配置页面更新
-   在手机上不能很方便的调用这个工作流，目前只能进入到工作流编辑页面，通过试运行的方式调用
-   Notion中的帖子不会去重，意味着如果我反复转存同一篇笔记，那么Notion中就会出现多个一模一样的帖子
-   小红书解析相关功能依赖别人提供的插件，而这些插件有被下架的风险

所以接下来我计划做两件事：

-   一个Chrome浏览器插件，它可以定时刷新小红书页面并截获cookie，然后调用扣子的API更新对应的变量值
-   一个基于iOS的Scriptable应用的脚本，使其支持系统的分享功能，它将可以调用扣子的API来调用这个工作流，并将小红书分享链接作为参数传入

至于帖子重复的问题我不打算用代码的方式解决，因为收益实在太低了，就算重复了，也无非手动去Notion里删一下的事。插件被下架的风险暂时也不打算考虑，现有的能用就先用着，哪天不能用了再说。
