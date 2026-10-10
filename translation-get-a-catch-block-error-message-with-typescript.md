---
title: "在TypeScript的catch代码块中获取错误信息"
source: https://www.boris1993.com/translation-get-a-catch-block-error-message-with-typescript.html
date: 2024-06-21
updated: 2026-10-10
tags: [TypeScript]
categories: [翻译]
---

# 在TypeScript的catch代码块中获取错误信息

最近在试着用TypeScript写点东西，在用`catch (error) {}`代码块处理异常的时候，看到了一个很难理解的错误`TS18046: error is of type unknown`。网上一顿冲浪之后，看到了Kent C. Dodds的一篇博客Get a catch block error message with TypeScript。我跟着文章的内容成功解决了这个问题，并且解答了我的疑惑，所以想要翻译出来帮助到更多的人。

以下内容除特别注明外，皆翻译自原文。我亦不对内容做任何的担保，并不对任何可能产生的后果（包括但不限于文件丢失或功能异常）负责。

* * *

好吧，咱们看看这个代码：

```javascript
const reportError = ({ message }) => {
    // 把错误信息发给我们的日志服务...
}

try {
    throw new Error('Oh no!')
} catch (error) {
    // 我们会让代码继续执行，但先把错误报告出去
    reportError({ message: error.message })
}
```

这么写应该足够好了吧？嘛，毕竟这是JavaScript。要是换成TypeScript的话：

```typescript
const reportError = ({ message }: { message: string }) => {
    // 把错误信息发给我们的日志服务...
}

try {
    throw new Error('Oh no!')
} catch (error) {
    // 我们会让代码继续执行，但先把错误报告出去
    reportError({ message: error.message })
}
```

这时候`reportError`的`error.message`这部分就要报错了。因为（就在最近）TypeScript把`error`的类型定义成了`unknown`。这倒也是事实，因为它确实没法保证抛出的错误的类型。哦对，这也是你不能用promise的泛型(`Promise<ResolvedValue, NopeYouCantProvideARejectedValueType>`)，来给promise reject的`.catch(error => {})`指定类型的原因。而且，被抛出来的东西可能都不是一个error，它可以是任何东西：

```typescript
throw '啥玩意？！' // 译者注：原文为 'What the!?'，请尝试用东北口音理解
throw 7
throw { wut: 'is this' } // 译者注：wut可以用来表达“傻眼”语境下的what，类似“什么鬼”
throw null
throw new Promise(() => {})
throw undefined
```

说真的，你可以想throw啥就throw啥，啥东西都行。那，要解决上面提到的错误信息好像挺简单对吧，我们在`catch`中声明代码只会抛出error不就行了？

```typescript
try {
    throw new Error('Oh no!')
} catch (error: Error) {
    // 我们会让代码继续执行，但先把错误报告出去
    reportError({ message: error.message })
}
```

想的美！现在你会得到这么一条TypeScript的编译错误：

```plaintext
Catch clause variable type annotation must be 'any' or 'unknown' if specified. ts(1196)
```

报这个错的原因是，尽管看起来在我们的代码里不可能会抛出来其他的东西，但JavaScript就这么逗，一个第三方库完全有可能做点什么奇怪的事，比如给Error的构造函数来个猴子补丁（译者注：monkey-patching），让它抛出点不一样的东西：

```typescript
Error = function () {
    throw 'Flowers'
} as any
```

那咱们开发者该怎么办？我们只能尽力，比如这样：

```typescript
try {
    throw new Error('Oh no!')
} catch (error) {
    let message = 'Unknown Error'
    if (error instanceof Error) message = error.message
    // 我们会让代码继续执行，但先把错误报告出去
    reportError({ message })
}
```

妥了这不！现在TypeScript也不跟我们嚷嚷有问题了，而且万一这个error是什么奇怪的东西，我们也用了合适的办法来处理它了。而且这段代码我们还能继续优化成这样：

```typescript
try {
    throw new Error('Oh no!')
} catch (error) {
    let message
    if (error instanceof Error) message = error.message
    else message = String(error)
    // 我们会让代码继续执行，但先把错误报告出去
    reportError({ message })
}
```

那么现在，如果这个error不是一个`Error`对象，那么我们就直接把它变成一个字符串并祈祷这个错误信息能有点用。

然后，我们还能把这段代码抽出来做成一个工具方法来给所有的catch块用：

```typescript
function getErrorMessage(error: unknown) {
    if (error instanceof Error) return error.message
    return String(error)
}

const reportError = ({ message }: { message: string }) => {
    // 把错误信息发给我们的日志服务...
}

try {
    throw new Error('Oh no!')
} catch (error) {
    // 我们会让代码继续执行，但先把错误报告出去
    reportError({ message: getErrorMessage(error) })
}
```

这个写法在我的项目里面特别好用。希望也能帮助到你！

更新：Nicolas针对error对象并不真的是error的情况提了一个很好的建议。此外Jesse也提出了一个建议，在可能的情况下把error对象也转换成字符串。把这些结合起来，我们就得到了这样的一份代码：

```typescript
type ErrorWithMessage = {
    message: string
}

function isErrorWithMessage(error: unknown): error is ErrorWithMessage {
    return (
        typeof error === 'object' &&
        error !== null &&
        'message' in error &&
        typeof (error as Record<string, unknown>).message === 'string'
    )
}

function toErrorWithMessage(maybeError: unknown): ErrorWithMessage {
    if (isErrorWithMessage(maybeError)) return maybeError

    try {
        return new Error(JSON.stringify(maybeError))
    } catch {
        // fallback in case there's an error stringifying the maybeError
        // like with circular references for example.
        return new Error(String(maybeError))
    }
}

function getErrorMessage(error: unknown) {
    return toErrorWithMessage(error).message
}
```

简直太好使了！

**结论**

我觉得关键在于，尽管TypeScript有一些奇怪的地方，但也绝对不要因为你觉得这不可能就忽略TypeScript抛出的编译错误或警告。大多数情况下，意外是非常有可能发生的，而TypeScript很好的强制你去处理这些“不太可能发生”的情况……然后你也很有可能会发现，这些情况并没有你想的那么少见。

* * *

译者的碎碎念：作为一个TypeScript纯新手的我，最后这段代码给我看傻了。Java里面非常常见的`try {...} catch (Exception e) {...}`在TypeScript里面竟然能玩这么花……
