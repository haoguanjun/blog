---
title: Web 性能：当毫秒级分辨率仍不够精确
date: 2018-10-08
categories: performance
---
有些时候，使用毫秒级分辨率来度量时间仍不够精确。W3C Web 性能工作小组与行业和社区的领导者开展了通力合作，通过标准化高分辨率时间规范而解决了这一问题。
<!-- more -->
有些时候，使用毫秒级分辨率来度量时间仍不够精确。W3C Web 性能工作小组与行业和社区的领导者开展了通力合作，通过标准化高分辨率时间规范而解决了这一问题。截至本周，该规范已经被作为推荐标准 (PR) 发布，并被诸多现代浏览器所广泛采用。请观看 What Time is it? 测试演示，了解此 API 的工作原理。

那么，为什么说毫秒仍不够精确呢？长期以来，Web 平台一直使用 JavaScript 日期对象的某个形式来度量时间，要么是通过 Date.now() 方法，要么是通过 DOMTimeStamp 类型。日期对象将以毫秒为单位显示出自 1970 年 1 月 1 日 UTC 以来的时间值。对于大多数实际用途而言，时间的这一定义已经足以代表 1970 年 1 月 1 日之后 285,616 年以内的任何时刻。

例如，在撰写本篇博文时，IE10 开发人员工具控制台中的 `Date.now()` 时间值为 1350509874902。这十三位数字代表了从时间基准 1970 年 1 月 1 日以来毫秒数之和。该时间对应为 2012 年 10 月 17 日 21:37:54 UTC。

尽管这一定义仍将为确定当前日历时间提供切实有效的帮助，但在有些情形中，这一定义已不够精确。例如，确定动画效果在每秒 60 帧的速率下是否流畅运行（每 16.667 毫秒绘制一帧动画）对于开发人员而言十分有帮助。但通过度量最后一次帧绘制回调的发生时间而计算瞬时 FPS 的简单方法仅能让开发人员确定 FPS 为 58.8 FPS (1/17) 或 62.5 FPS (1/16)。

类似地，在精确度量已用时间（例如使用导航计时、资源计时和用户计时 API 来测量网络和脚本时间）或试图同步动画场景或动画与音频时，亚毫秒级的分辨率也将更为可取。

为了解决这一问题，高分辨率时间规范定义了一个新的时间基准，该时间基准至少采用微秒级的分辨率（一微秒等于一毫秒的千分之一）。为了缩短代表这一数字的位数，并提高可读性，这一名为 performance.timing.navigationStart 的新时间基准并没有从 1970 年 1 月 1 日 UTC 开始度量时间，而是从文档导航启动之时开始度量时间。

该规范将 `performance.now()` 定义为 `Date.now()` 的相似方法，从而以高分辨率确定当前时间。`DOMHighResTimeStamp` 是定义了高分辨率时间值的 DOMTimeStamp 的相似类型。

例如，使用 IE10 开发人员工具控制台内的 performance.now() 和 Date.now() 来显示撰写本篇博文时的当前时间，我可以看到以下两个值：

> performance.now():        196.304879519774
> Date.now():        1350509874902

尽管这些时间值都代表了相同的时间实例，但是它们却是从不同的起始点开始度量的。很明显，performance.now() 时间值的可读性更高。

由于高分辨率时间是从文档导航启动之时开始度量，因此子文档中的 performance.now() 将从子文档，而非根文档的导航启动之时开始度量。例如，假设一个文档拥有同源的 iframe A 和跨源 iframe B，其中 iframe A 中的导航在根文档导航开始 5 毫秒后启动，而 iframe B 在根文档导航开始 10 毫秒后启动。如果我们对根文档导航启动 15 毫秒后的时间精确时刻进行度量，那么我们将在不同上下文中获得以下 performance.now() 调用值：

> iframe B 中的 performance.now()：                               5.123 ms
> iframe A 中的 performance.now()：                              10.123 ms
> 根文档中的 performance.now()：                    15.123 ms
> 任何上下文中的 Date.now()：                         134639846051 ms

![](https://msdnshared.blob.core.windows.net/media/MSDNBlogsFS/prod.evol.blogs.msdn.com/CommunityServer.Blogs.Components.WeblogFiles/00/00/00/38/71/metablogapi/2744.wpwmrjie-image1.png)
图：Date.now() 衡量的是自 1970 年 1 月 1 日以来的时间，而 performance.now() 衡量的是自文档导航启动以来的时间

这一设计不仅能确保跨源 iframe 中父级创建的时间内不会发生数据泄露，同时还能让您度量相对于您的启动的时间。例如，如果您使用资源计时界面（该界面使用高分辨率时间）来确定服务器响应子文档中的资源请求所需的时长，那么您不需要进行任何调整即可将子文档添加至根文档的时间纳入计算范围之内。

如果您希望进行跨帧时间比较，您只需请求 `top.performance.now()`，即可获得相对于根文档导航启动时间的时间值，此过程将在所有同源 iframe 中返回相同的值。

相比 `Date.now()`，该 API 的另一巨大优势在于 `performance.now()` 是单调递增的，而且不会受到时钟倾斜或调整的影响。`performance.now()` 后续调用之间的差异永远都不会为负。而 `Date.now()` 则无法提供这个保证，在实际操作过程中，我们也曾听到开发人员报告在分析数据中看到负时间。

高分辨率时间是彰显新想法如何迅速转变为可互操作的标准的另一典型示例，开发人员能够在现代支持 HTML5 的浏览器中充分依赖这一标准。感谢 W3C Web 性能工作小组中的每一位成员为设计该 API 所提供的帮助，同时也感谢其他浏览器供应商迅速实施这一标准，并认真观察该标准的互操作性。
