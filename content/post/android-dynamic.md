---
title: "Android动态化"
date: 2019-03-14T15:18:46+08:00
draft: false
---

移动互联网已经发展十年了，随着业务成熟和功能的相对稳定，整体重心开始偏向运营，强烈的运营需求对客户端架构和发布模式都提出了更高的要求。如果每个修改都需要经历开发、上线、版本覆盖等漫长的过程，根本无法达到快速响应的要求。

## 常见的动态化方案

移动端动态化方案在最近几年一直是大家关注的重点，虽然它已经发展了很多年，但是每年都会有新的变化，这里我们先来看看各大公司有哪些已知的动态化方案。

![](/images/android-dynamic-1.jpg)

动态化方案分为下面四种类型：

![](/images/android-dynamic-2.png)

* Web 容器增强。基于 H5 实现，但是还有离线包等各种优化手段加持，代表方案有 [PWA](https://juejin.im/post/6844903556470816781)、腾讯的 [VasSonic](https://github.com/Tencent/VasSonic)、淘宝的 zCache 以及大部分的小程序方案。
* 虚拟运行环境。使用独立的虚拟机运行，但最终使用原生控件渲染，代表方案有 React Native、Weex、快应用等。
* 业务插件化。基于 Native 的组件化开发，这种方式在淘宝、支付宝、美团、滴滴、360 等航母应用上十分常见。代表方案有阿里的 [Atlas](https://github.com/alibaba/atlas/tree/master/atlas-docs)、360 的[RePlugin](https://github.com/Qihoo360/RePlugin)、滴滴的 [VirtualAPK](https://github.com/didi/VirtualAPK) 等。除此之外，我认为各个热修复框架应该也属于业务插件化的一种类型，例如微信的 [Tinker](https://github.com/Tencent/tinker)、美团的 [Robust](https://github.com/Meituan-Dianping/Robust)、阿里的 [AndFix](https://github.com/alibaba/AndFix)。
* 布局动态化。插件化或者热修复虽然可以做到页面布局和数据的动态修改，但是代价巨大，而且也不容易实现个性化运营。为了实现“千人千面”，淘宝和美团的首页结构都可以通过动态配置更新。代表的方案有阿里的 [Tangram](https://github.com/alibaba/Tangram-Android/blob/master/README-ch.md)、Facebook 的 [Yoga](https://yogalayout.com/)。


## 动态化方案的选择

四大动态化方案哪家强，我们又应该如何选择？在回答这个问题之前，我们先来看看它们的差别。

![](/images/android-dynamic-3.jpg)


## 参考
* [40 | 动态化实践，如何选择适合自己的方案？](https://time.geekbang.org/column/article/89555)
* [利用Poplayer在手淘中实现稳定业务和临时业务分离](https://developer.aliyun.com/article/59050)
* [移动开发的罗曼蒂克消亡史](https://mp.weixin.qq.com/s/2xBnlmESZjq7UTtcfzqhcA)
* [当插件化遇上 Android P](https://www.infoq.cn/article/Aq7Pi8WOL2oWgZ6G2JZY)
* [Qigsaw](https://github.com/iqiyi/Qigsaw/blob/master/README.zh-CN.md)
* [Litho在美团动态化方案MTFlexbox中的实践](https://tech.meituan.com/2019/09/19/litho-practice-in-dynamic-program-mtflexbox.html)