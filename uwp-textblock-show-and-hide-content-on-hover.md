---
title: "为UWP的TextBlock实现鼠标悬浮时显示和隐藏内容"
source: https://www.boris1993.com/uwp-textblock-show-and-hide-content-on-hover.html
date: 2023-10-21
updated: 2026-10-10
tags: [UWP, TextBlock]
categories: [学知识]
---

# 为UWP的TextBlock实现鼠标悬浮时显示和隐藏内容

这两天给我的窗边岛项目实现了X岛揭示板网页中的`防剧透`功能。这个功能本质上就是，当鼠标悬浮在文字上时显示原本的内容，当鼠标移出文字时则用黑块代替。即

| X岛揭示板 | 窗边岛 |
| --- | --- |
| ![](https://blog-static.boris1993.com/uwp-textblock-show-and-hide-content-on-hover/nmbxd-anti-spoiler.gif) | ![](https://blog-static.boris1993.com/uwp-textblock-show-and-hide-content-on-hover/xow-anti-spoiler.gif) |

虽然说起来很简单，但是好像网上并没有针对这个需求有什么相关的内容，所以在这里记录下我的实现，权当抛砖引玉。

## 背景

在`X岛揭示板`中，防剧透是通过`[h][/h]`这个自定义标签实现的，所以上面GIF图中的文字其实是`正常文字--[h]防剧透文字[/h]--正常文字--[h]防剧透文字[/h]--`。那么这里要做的就有两件事：解析这个自定义标签，以及在`TextBlock`控件中实现黑块和正常文字的互相替换。

## 为TextBlock对象填充内容

`TextBlock`对象有两种填充内容的方式：

-   直接将内容放入`TextBlock.Text`属性中。这种方式适合不包含防剧透标签的内容。

```csharp
new TextBlock
{
    VerticalAlignment = VerticalAlignment.Top,
    HorizontalAlignment = HorizontalAlignment.Stretch,
    Text = content,
    TextWrapping = TextWrapping.Wrap,
    IsTextSelectionEnabled = textSelectionEnabled,
};
```

-   将内容分散到各个`Run`对象中，并将这些`Run`对象放在`TextBlock.Inlines`属性中。我就是搭配这种方式实现的防剧透功能。

```csharp
var run1 = new Run { Text = "Run 1" };
var run2 = new Run { Text = "Run 2" };

var textBlock = new TextBlock
{
    VerticalAlignment = VerticalAlignment.Top,
    HorizontalAlignment = HorizontalAlignment.Stretch,
    TextWrapping = TextWrapping.Wrap,
    IsTextSelectionEnabled = textSelectionEnabled,
};

textBlock.Inlines.Add(run1);
textBlock.Inlines.Add(run2);
```

## 保存黑块下的原本内容

因为防剧透本质就是，平时用黑块替换掉要遮挡的内容，仅在鼠标悬浮时再用真正的内容替换掉黑块，所以我们需要一个地方来保存原本的内容。本来我想直接在`Run`对象上下功夫，但是可惜`Run`不像`TextBlock`有一个`DataContext`属性可以放东西，所以最后我还是把目光放在了`TextBlock`上。

`TextBlock.DataContext`是一个`object`类型的属性，所以我们可以随意放任何我们想放的东西。

当然为了扩展性考虑，我们最好还是给它创建一个类。

```csharp
class TextBlockDataContext
{
    // key用来放应该被防剧透的Run在Inline里的下标
    // value是这个Run实际的内容
    public Dictionary<int, string> IndexAndOriginalTextOfHiddenContent = new Dictionary<int, string>();
}
```

然后我在给一个段落创建`TextBlock`时，就可以把这个`TextBlockDataContext`对象放在`DataContext`属性中备用。

```csharp
textBlock = new TextBlock
{
    VerticalAlignment = VerticalAlignment.Top,
    HorizontalAlignment = HorizontalAlignment.Stretch,
    TextWrapping = TextWrapping.Wrap,
    IsTextSelectionEnabled = textSelectionEnabled,
    DataContext = new TextBlockDataContext { },
};
```

## 解析标签并生成`Run`对象

这部分的思路就是，整行文字会被`[h]`和`[/h]`标签切割成各自的`Run`，因为`TextBlock.Inlines`是一个有序的列表，所以在切割和生成`Run`对象时，我可以在`TextBlockDataContext.IndexAndOriginalTextOfHiddenContent`中记录下要防剧透的`Run`的下标和它实际的内容。同时，针对要防剧透的`Run`，我先用黑块字符`█`填充它的`Text`属性。

```csharp
int indexOfRun = 0;
var totalLength = content.Length;
var enteredHiddenBlock = false;
Run run;
while (content.Length > 0)
{
    var indexOfBeginHideMark = content.IndexOf("[h]");
    if (!enteredHiddenBlock && indexOfBeginHideMark >= 0)
    {
        run = new Run
        {
            Text = content.Substring(0, indexOfBeginHideMark),
        };

        textBlock.Inlines.Add(run);
        content = content.Substring(indexOfBeginHideMark + 3);

        enteredHiddenBlock = true;
        indexOfRun++;
        continue;
    }

    var indexOfEndHideMark = content.IndexOf("[/h]");
    if (indexOfEndHideMark > 0)
    {
        var text = content.Substring(0, indexOfEndHideMark);
        run = new Run
        {
            Text = new string('█', text.Length),
        };

        textBlock.Inlines.Add(run);
        content = content.Substring(indexOfEndHideMark + 4);
        (textBlock.DataContext as TextBlockDataContext).IndexAndOriginalTextOfHiddenContent.Add(indexOfRun, text);

        enteredHiddenBlock = false;
        indexOfRun++;
        continue;
    }

    run = new Run
    {
        Text = content,
    };
    textBlock.Inlines.Add(run);
    indexOfRun++;
    break;
}
```

## 实现鼠标悬浮时显示真实内容

`TextBlock`提供了两个事件`PointerEntered`和`PointerExited`，分别对应鼠标指针进入和离开`TextBlock`范围。所以我们就可以给这两个事件分别绑定`UnhidingContent`方法和`HidingContent`方法来实现鼠标悬浮时显示真正内容。

```csharp
private static void HidingContent(object sender, PointerRoutedEventArgs pointerRoutedEventArgs)
{
    var textBlock = sender as TextBlock;
    if (textBlock.DataContext is TextBlockDataContext dataContext)
    {
        foreach (var indexAndOriginalText in dataContext.IndexAndOriginalTextOfHiddenContent)
        {
            var textLength = indexAndOriginalText.Value.Length;
            (textBlock.Inlines.ElementAt(indexAndOriginalText.Key) as Run).Text = new string('█', textLength);
        }
    }
}

private static void UnhidingContent(object sender, PointerRoutedEventArgs pointerRoutedEventArgs)
{
    var textBlock = sender as TextBlock;
    if (textBlock.DataContext is TextBlockDataContext dataContext)
    {
        foreach (var indexAndOriginalText in dataContext.IndexAndOriginalTextOfHiddenContent)
        {
            (textBlock.Inlines.ElementAt(indexAndOriginalText.Key) as Run).Text = indexAndOriginalText.Value;
        }
    }
}
```

至此，与X岛揭示板网页端类似的防剧透功能就完成实现了。完整的代码可以参考对应的GitHub commit。
