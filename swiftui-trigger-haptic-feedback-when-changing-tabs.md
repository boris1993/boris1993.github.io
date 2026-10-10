---
title: "SwiftUI学习记录 - 切换TabView的tab时触发震动反馈"
source: https://www.boris1993.com/swiftui-trigger-haptic-feedback-when-changing-tabs.html
date: 2023-02-05
updated: 2026-10-10
tags: [iOS开发, SwiftUI, TabView, Haptic Feedback, 震动反馈]
categories: [学知识]
---

# SwiftUI学习记录 - 切换TabView的tab时触发震动反馈

最近在折腾iOS开发的时候，想实现在切换`TabView`的tab时触发震动反馈，因为一边写一边学嘛，基础不牢，只能在网上瞎搜，试了各种方法都不奏效。直到前两天看到Ice Cubes: for Mastodon的这段代码，豁然开朗。

之前写的时候，只知道给`selection`传一个绑定的对象，但是看完`IceCubes`的代码，再一看`selection`参数的文档，恍然大悟，这个参数要的其实是一个`SelectionValue`的绑定对象，所以我其实可以在这个对象的`set`方法上面做文章。

```swift
struct ContentView: View {
    @State
    private var selectedTab = Tab.Timeline

    var body: some View {
        TabView(selection: .init(
            get: {
                selectedTab
            },
            set: { newTab in 
                selectedTab = newTab

                // 在TabView更新selection的时候触发震动反馈就行了
                if (globalState.isHapticFeedbackEnabled) {
                    HapticsHelper.playHapticFeedback()
                }
            }
        )) {
            // Views
        }
    }
}
```

顺便提一嘴我之前走过的一个弯路，就是试图在`TabView`上用`.onTapGesture()`方法，结果虽然能震了，但是tab不会切换，当时还花了好长时间找怎么调用原本的`.onTapGesture()`来切tab，当然最后一无所获，最后认输，把震动反馈这个功能忍痛砍了……
