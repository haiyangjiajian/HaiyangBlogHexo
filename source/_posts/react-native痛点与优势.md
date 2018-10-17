---
layout: post
title: react native 痛点与优势
tags: [react_native]
category: coder
---

随着自己对react native开发的深入，对于其一些痛点和优势也有着更深的体会

---

### react native的痛点

1. 由于还不是稳定版本，版本更新太快，大概两周会有一个新的版本。更新新版可能会出现不兼容的问题，有时候需要手动解决。

<!-- more -->

2. 支持的组件不全面。大部分厂商并不支持react native。一些支持的现在一般也处在不稳定版本。比如截止到rn版本0.35，js版的本地数据库组件只有realm支持，现在realm版本为0.15，很多功能不全。
3. 程序的性能。现在普遍都说比原生的性能要差，但是差多少没有一个具体的衡量。直观的感觉是复杂的页面在一些配置较低的手机上会有肉眼可见卡顿的感觉。
4. 虽然大多界面可以同时生成ios和android的，但一些涉及到底层的东西需要在ios和android单独开发，然后在js层进行调用。
5. 学习成本高。要学习[javascript系列东西](https://zhuanlan.zhihu.com/p/22782487)，还需要涉及到ios，android开发相关知识。相比较而言，一个用过react的前端开发，写rn应该上手更快。
6. 关于react native的开发现在并没有一些best practice，也没有真正很有经验的人，很多只能摸索。对于小团队来说，试错成本有点高，一旦卡在一些问题上，网上解决方案很少，容易耽误了整体的进度。招聘有相关经验的人也较难。

### react native优势

1. 组件化开发，复用率高，组件丰富以后，ui开发较快，前端式开发。
2. 同时支持android和ios的ui界面。
3. 可以方便的进行代码热更新。
4. Learn once,write anywhere，未来js可能会有更大的通用性，比如现在微信小程序的开发技术和react native十分相似。现在还有用react native开发[mac桌面应用](https://github.com/ptmt/react-native-macos)，开发[web网页](https://github.com/necolas/react-native-web)
5. 可以和原生页面互相调用，作为一部分嵌入到一个已有的原生app中。
6. 它是一种介于在webview和原生开发之间的解决方案，它想要实现像web一样灵活，像原生一样的性能，虽然现在还都没有达到，但是它是一种有可能接近这个目标的解决方案。

待续。。。


