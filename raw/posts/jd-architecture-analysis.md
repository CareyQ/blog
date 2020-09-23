---
title: 京东首页前端技术剖析与对比
description: 逛京东的时候开着 chrome 控制台，无意间看到了下面这串似曾相识的代码，仔细扒拉看了之后，做了一个简单的分析，同时也提出了一些看法。
warning: true
mark: hot
categories:
  - 前端杂烩
tags:
  - 京东
  - 技术架构
date: 2015-09-09 10:35:05
---


逛京东的时候开着 chrome 控制台，无意间看到了下面这串似曾相识的代码，

![京东网页部分源码](http://www.barretlee.com/blogimgs/2015/09/09/20150903_c32208d1.jpg)

再看了下 localstorage，

![京东网页 localstorage](http://www.barretlee.com/blogimgs/2015/09/09/20150903_2d854b82.jpg)

看到这些内容，其实京东首页的前端架构雏形就出来了。

<!--more-->

JD 使用 seajs 作为模块加载器，使用 jd-jquery 为基本库，看到它的 jq 版本是 1.6.4，

![jd-jQuery](http://www.barretlee.com/blogimgs/2015/09/09/201593104.jpg)

对比了下"正版" [jquery-1.6.4](http://code.jquery.com/jquery-1.6.4.min.js) 的源码，很显然，JD 使用的是自己造的轮子，这说明京东的前端生态应该是十分完善的，有轮子就会有很多组件、插件，这对一个公司批量造网页很有裨益。

### 前后端情况

电商的主页都是以呈现为主，展示各个横向市场和纵向市场的入口，也可能会有一些个性化（所谓个性化，就是给不同的用户推荐不同的内容）的推荐。它不像交易、详情等页面，页面逻辑后端重而前端轻，后端需要做各种内容推荐、数据校验、类型判断等工作，而前端更多的是将后端的信息有序地呈现出来。

首页则不同，首页承载了大量的二级页面入口和一些频道的推荐内容，而这些数据更多的是由运营去维护，可以认为数据是死的，就算存在一些个性化的数据，也会使用 jsonp 的形式去加载，前端需要快速高效地处理这些数据，可见前端任务相当艰巨。加上作为一个网站的门面，它的安全稳定性也是极为重要的。

如果没有猜错，京东后端也有一个中间层，中间层负责组装数据，以模块为单位，根据前端的请求响应对应模块的内容，而数据是在另外一个运营平台上维护，运营填好的数据会即时的推送到 CDN 或者应用，中间层拼合数据。看不到的东西就不猜测了，我们来看看京东首页的整体结构。

### 前端技术简要分析

如果希望一个网站跑起来飞快，你觉得怎么做最靠谱？

我们都玩过微博，都上过手机淘宝，进入这些 app 应用，会发现很多页面几乎看不到加载的痕迹，因为他们是本地应用，很多图片、脚本、样式都已经打包在本地了，所以加载起来速度是很快的。如果希望一个网页也能飞奔起来，同样的道理，让请求的个数少一点，让请求的内容少一点。还有一个至关重要的，让那些次要的内容慢一点加载（我们称之为懒加载）。

#### 前端缓存和异步加载

京东在按需加载和数据缓存上的工作做的十分到位。

![JD-page-loader](http://www.barretlee.com/blogimgs/2015/09/09/201593105.jpg)

每个具有 lazyload 异步标识的模块，都包含两个属性，一个是渲染该模块需要的内容（数据+JS），一个是这个内容过期的时间，只要内容不变就不会过期，所以这里使用的是文件 hash 来标注。

![JD-data-via-localstorage](http://www.barretlee.com/blogimgs/2015/09/09/20150903_9f0924a6.jpg)

把需要请求的路径写在 dom 上，用户滚动时，一旦该模块进入了视窗，则请求 dom 上对应的 `data-path` 地址，拿到渲染这个模块所需要的脚本和数据，不过这中间还有一层本地缓存 localstorage，如果在本地缓存中匹配到了对应的 hash string 内容，则直接渲染，否则请求到数据之后更新本地缓存。dom 上的 `data-time` 会在页面加载时候，后端计算文件 hash，hash 不变则输出内容也不变。

这里其实存在两个请求，一个请求是加载数据和脚本，而这里的内容是：

```html
<div>{html}</div>
<script>
var data = {JSONSting};
seajs.use('path/to/$version$/script.js', function(Script){
  Script.init(data);
});
</script>
```

为啥不在返回的内容中直接把脚本也输出出来？为了让数据充分缓存下了不少功夫。数据的变化频率比较高，如果数据和初始化脚本包装在一起，虽然节约了一个请求，但一旦数据变化，整个脚本都得重新加载，而将数据和脚本分离，脚本可以长期缓存在本地，单独请求数据，这个量会小很多。直接改变上面的 `version` 版本号便可以让浏览器重新请求最新脚本。

从上面可以看出，任何一个模块的改动，在前端只会引起一个较小的加载变化，加上 http 的缓存策略，服务器的压力也是很小的。

#### 工程结构

比较常见的工程结构，如下：

```
.
├── build/
└── src/
    ├── widgets/
    ├── mods/
    |   ├── moduleA/        
    |   |   ├── index.js  
    |   |   ├── index.tpl
    |   |   └── index.less
    |   ├── moduleB/ 
    |   └── moduleC/  
    ├── index.js  
    ├── index.tpl
    └── index.less
```

所有 mods 中的 `tpl` 文件通过一些标签，引入到 `src/index.tpl` 中，需要同步渲染的模块信息直接引入，而异步渲染的模块内容，比如 `moduleA/index.tpl`，其内容就十分简单：

```
<div 
  lazyload 
  data-path="<% moduleA %>" 
  data-time="<% md5(moduleA + getData()) %>"></div>
```

只引入一个模块钩子（hook），然后按需加载/懒加载这个模块钩子内容。相比 JD 采用的也是类似的模型。

### 横向技术对比

看看上面列出的目录结构，一般情况下，为了减少网页的请求数，我们会把所有 mods 和 wedgets 中的 js 和 css 分别打包成一个文件，然后前端 combo 请求，提前加载但是懒执行，这是 CMD 的思维方式。而京东使用了更懒的方式：懒加载并且懒执行。

这种方式带来的好处就是，单个模块的更改，前端只更新一小部分缓存；而提前加载所有模块的方式，任何一个模块有改动，整体都得重新下载。脚本懒加载的缺点是，需要发起请求，如果需要加载多个模块，则需要发起多个脚本请求，可以看到，快速拖动 JD 首页，模块的加载速度不容乐观。当然，脚本是可以被浏览器缓存的，这个问题也就是首次访问或者清空了缓存才会出现。

对请求控制如此严格，怎么就没考虑下优化源码当中的[两大段 css 和 js 代码](view-source:http://www.jd.com/)呢？是不是也可以把 css 和 js 放到 localstorage 中，减少请求数。

源码中通过函数去加载资源：

```
var loadCss = function(){
  var style = loadFromLocalstorage();
  inserCss(style);
};
```

如果 localstorage 中不存在，也不需要重新发请求，后端脚本通过 cookie 判断是否需要同步输出代码：

```
// 伪代码
if(cookies('cssV') || cookie('cssV') !== 'setsV'){
  echo CSSCode;
}
```

如果发现 cookie 中的版本号与设定的版本号不一样，或者没有 cssV cookie，则同步内联输出 css 和 js。

### 小结

本文只是对京东首页用到的部分技术做一个简要的分析，页面加载速度确实十分可观，赞！

随着需求的多元化和终端设备的多元化，前端技术在 web 舞台上一直展现着优美的身姿，她在进化、在演变，几乎每隔一两个月就能听到新的前端技术出来，所以学是学不过来的，前端的学习就两个字："理解为什么"。