事情起因于 在mo gallery中的文章编辑器是基于tiptap开源项目开发，但是markdown上很多逻辑想对齐typora的按键交互，以tiptap自身特性很多场景是一致的，如：

```
- xxx
(tab)制表错位，shift+tab退制表也错位
- xxx

1. xxxx
（删除该行后需要向上合并列表，但是未能合并）
1. axxx
```

然后再通过codex对问题描述进行复现，但由于其规则过多导致上下文很容易出现问题，最后导致有些问题循环出现，甚至出现新的bug。



因此基于desktop端开发了一个调用自身的mcp服务器，用于通过codex等agent直接调用该mcp，实现类似于用户手动操作的功能，最后再根据需求进行扩展，实现对各场景的用例覆盖。

![image](https://cdn3.ldstatic.com/optimized/4X/5/6/d/56dabd94f06f5420deae15b4504aa55c643f17b6_2_690x376.png)

最后发现效果还是挺显著的，一方面对于编辑器的键盘交互逻辑，以自然语言很难去描述，让AI来复现，通过mcp方式可以很快定位到问题。



![image](https://cdn3.ldstatic.com/optimized/4X/1/c/e/1ce52ee2ef6a378ebd3c7ed51cda7476ff87fd55_2_661x500.png)

