---
title: "Shell脚本中 set -ex 命令的作用"
source: https://www.boris1993.com/shell-set-ex.html
date: 2019-02-15
updated: 2026-10-10
tags: [Shell]
categories: [学知识]
---

# Shell脚本中 set -ex 命令的作用

刚刚学会了一个很实用的shell命令`set -ex`，在这里分享一下。

稍有常识的人都能看出，这是`set`命令加上了`-e`和`-x`两个参数(废话么这不是)。那么，我就把这两个参数拆开，分别说一下它在脚本中的用处。

# `set -e`

先说说`set -e`，这个参数的含义是，当命令发生错误的时候，停止脚本的执行。

通常来说，我们会习惯于使用`&&`来实现这样的功能，比如：

```bash
#!/bin/bash

echo 1 && rm non-existent-file && echo 2
```

但是，写成一行呢，可读性有点差，分成多行的话，也得注意换行符`\`和`&&`号，我就有过好几次忘了加这俩东西，还是挺麻烦的是吧。

更麻烦的是，`&&`连接的命令之间不能写注释，也就是说，下面这个示例是不能用的：

```bash
#!/bin/bash

echo 1 \
  && rm non-existent-file \ # which will fail
  && echo 2
```

运行之后会是这个德行：

```text
1
rm: non-existent-file: No such file or directory
rm:  #: No such file or directory
rm: which: No such file or directory
rm: will: No such file or directory
rm: fail: No such file or directory
./test1.sh: line 5: syntax error near unexpected token `&&'
./test1.sh: line 5: `  && echo 2'
```

现在，就可以写成下面这样了：

```bash
#!/bin/bash

set -e
echo 1
rm non-existent-file # which will fail
echo 2
```

猜猜最后输出里面会不会把`2`打印出来？

# `set -x`

说完了`-e`，继续说说`-x`。`-x`参数的作用，是把将要运行的命令用一个`+`标记之后显示出来。

还是拿上面这个脚本举个例子，这次加上`-x`：

```bash
#!/bin/bash

set -ex
echo 1
rm non-existent-file # which will fail
echo 2
```

然后它的输出就变成了：

```text
+ echo 1
1
+ rm non-existent-file
rm: non-existent-file: No such file or directory
```

注意第一行和第三行前面那个`+`，这就是`-x`参数的作用。

# One more thing……

需要注意，这条命令需要放到整个shell脚本的开头，才会起作用。毕竟用脑子想想就知道，这是俩开关，不论放在中间还是结尾，都不会起到预期的作用。

# Credit

感谢这篇文章set -ex - The most useful bash trick of the year为我提供了这条命令的解释，和写作的思路。
