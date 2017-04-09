---
layout: post
title: Jekyll搭建博客并添加百度统计 
tags: [tools]
category: 工具
---

 自己的这个博客是在gitpage上用Jekyll搭建起来的，使用的是[这个模版](https://mmistakes.github.io/hpstr-jekyll-theme/theme-setup/)。搭建以后一直是在github上build的，也没有添加点击统计的工具。下面介绍一下自己完成这本地build和添加百度统计的过程和遇到的问题。

 ---

### 添加百度统计

现在有很多站长工具可以统计网站的点击量。添加方式大同小异。我添加的是[百度统计](http://tongji.baidu.com/web/welcome/login)。注册添加后，会得到如下一段javaScript：

``` javascript
<script> 
var _hmt = _hmt || [];
(function() {
  var hm = document.createElement("script");
  hm.src = "//hm.baidu.com/hm.js?xxxxxxxxxxxxxxxxxxxxxxxxxxx";
  var s = document.getElementsByTagName("script")[0]; 
  s.parentNode.insertBefore(hm, s);
})();
</script>
```

需要将这段代码添加到网站全部页面的head标签前。对于Jekyll的网站来说可以通过以下三个步骤来实现：

+ 修改_config.yml，加入

```
baidu-analysis： xxxxxxxxxxxxxxxxxxxxxxxxxxx
```

+ 在_include中新建文件baidu-anaylysis.html加入以下代码


``` javascript
<script>
  var _hmt = _hmt || [];
  (function() {
    var hm = document.createElement("script");
    hm.src = "//hm.baidu.com/hm.js?{{ site.baidu-analyisis }}";
    var s = document.getElementsByTagName("script")[0];
    s.parentNode.insertBefore(hm, s);
  })();
</script>
```

+ 在博客的入口网页中添加

![include](/assets/img/2016-10-25-include.jpg)

Jekyll的入口一般在 \_layouts/default.html中。我的模版因为将header统一抽取了出来，加在了\_includes/head.html中。

+ 在百度统计的网站中心tab中检查首页代码状态，显示代码安装正确，就成功了。


### 本地build Jekyll

``` bash
gem install bundler
bundle install
bundle exec jekyll serve
```

注意在config_yml中不能有tab，否则会提示‘found character that cannot start any token while scanning for the next token at line’的错误。

本地server启动后可以在http://localhost:4000/ 访问