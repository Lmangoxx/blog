title: Workbox 入门
banner: gallery/taiwan/DSC04075.JPG
date: 2018-02-27 17:38:34
tags:
- Workbox
- ServiceWorker
- PWA
---

JavaScript Libraries for adding offline support to web apps.

一个为网页应用添加离线支持的 JavaScript 库。

> 本文内容基于 Workbox@3.0.0-beta.1

<!-- more -->

## 快速开始

Workbox 作为 SW 模块使用，提供了两个最主要的接口：

- [🔗](https://developers.google.com/web/tools/workbox/reference-docs/latest/module-workbox-sw.Router#registerRoute) `workbox.routing.registerRoute`，接受两个参数，第一个参数 capture 是正则表达式或 Express 风格的路由字符串，声明需要匹配那些请求，第二个参数用于告诉 Workbox 对前面拦截到的请求做何处理。
- [🔗](https://developers.google.com/web/tools/workbox/reference-docs/latest/module-workbox-sw.Strategies) `workbox.strategies.xxx`，用在 registerRoute 的第二个参数，表明使用何种缓存策略。

最简单的例子：

```js
importScripts('https://storage.googleapis.com/workbox-cdn/releases/3.0.0-beta.0/workbox-sw.js');

// JS 请求: 网络优先
workbox.routing.registerRoute(
  new RegExp('.*\.js'),
  workbox.strategies.networkFirst({
    cacheName: 'workbox:js',
  })
);
```

同样的，我们可以继续添加其他的一些缓存策略来应对 css、图片文件：

```js
// CSS 请求: 缓存优先，同时后台更新后下次打开页面才会被页面使用
workbox.routing.registerRoute(
  // Cache CSS files
  /.*\.css/,
  // Use cache but update in the background ASAP
  workbox.strategies.staleWhileRevalidate({
    // Use a custom cache name
    cacheName: 'workbox:css',
  })
);

// 图片请求: 缓存优先
workbox.routing.registerRoute(
  // Cache image files
  /.*\.(?:png|jpg|jpeg|svg|gif)/,
  // Use the cache if it's available
  workbox.strategies.cacheFirst({
    // Use a custom cache name
    cacheName: 'workbox:image',
    plugins: [
      new workbox.expiration.Plugin({
        // Cache only 20 images
        maxEntries: 20,
        // Cache for a maximum of a week
        maxAgeSeconds: 7 * 24 * 60 * 60,
      })
    ],
  })
);
```

对于这样一个简单的页面：

```js
<html>
<head>
  <link rel="stylesheet" href="./css/style.css">
</head>
<body>
  <h1>Workbox Get Started</h1>
  <img src="./images/google.local.png" alt="同域的文件">
  <script src="./js/index.js"></script>
</body>
</html>
```

### #1
我们可以看看第一次访问时的效果：
{% asset_img gs1.png Web 第一次访问时的效果 %}

和平常的访问没有任何区别，只是多了 sw.js 及其依赖（workbox 等）的请求，并在 `Application > Service Workers` 中可以看到 sw.js 已经被注册（但其中的 fetch 事件无法在这次访问就被捕获）。

### #2

于是我们刷新页面看看效果：
{% asset_img gs1.png Web 第二次访问时的效果 %}

- 全部的 css、png、js 文件均被 ServiceWorker 拦截（图中 from ServiceWorker 可以看出）
- workbox-core 在拦截后重新发起了 fetch 请求并返回页面，fetch 后服务端返回 304 依然可以使用浏览器本地缓存策略
- 上述命中规则的请求都被缓存到 Cache Storage 中

> 为了方便看到效果，我设置服务端 Cache Control 失效（`max-age=-1`），使得每次请求都能到达服务端

{% asset_img get-started-3.png Web Cache Storage %}

### #3

我们更新一下 css、 js 和 png 的内容，然后重新访问页面：

{% asset_img gs4.png Web 第三次访问时的效果 %}

- 由于 png 是 Cache First，所以直接从 ServiceWorker 的 Cache 返回，没有真正的网络请求发出
- 由于 js 是 Network First，所以会产生 fetch，且运行成功（底部 Console 有输出内容）
- css 虽然同样 fetch 了新的内容，但页面并没有生效，用的还是上一次的 Cache（但新的文件内容已经放到 Cache Storage 中）

### #4

不再做任何修改，刷新页面：

{% asset_img gs5.png Web 第四次访问时的效果 %}

- 新的 css 生效（`staleWhileRevalidate`）
- css、js 请求返回为 304，使用浏览器缓存

## 离线应用

要做到能够完全离线，我们还必须让主文档也能被缓存下来，例如我们还是使用 `networkFirst` 来处理主文档：

```js
// 主文档: 网络优先
workbox.routing.registerRoute(
  /index\.html/,
  workbox.strategies.networkFirst({
    cacheName: 'workbox:html',
  })
);
```

缓存成功后，即便断网，页面依旧可以访问及使用：

{% asset_img ol1.png Web 完全离线 %}

具体 [Demo](https://csbun.github.io/workbox-examples/workbox-get-started/index.html) 和 [原码](https://github.com/csbun/workbox-examples/tree/master/workbox-get-started)。

## 跨域请求

在大多数的应用场景中，我们通常会将静态资源放到 CDN 中，这就涉及到跨域问题。

```html
<div>
  <p>不同域的文件</p>
  <p><img src="https://developers.google.com/web/tools/workbox/thumb.png" alt="不同域的文件"></p>

  <p>不同域的文件 且 <code>access-control-allow-origin: *</code></p>
  <img src="https://unpkg.com/resize-image@0.0.4/example/google.png" alt="不同域的文件 且 allow cross origin">
</div>
<!-- 不同域的js 且 access-control-allow-origin: * -->
<script src="https://unpkg.com/jquery@3.3.1/dist/jquery.js"></script>
```

经测试，Workbox 可以用 `networkFirst` 和 `staleWhileRevalidate` 两种策略 Cache 跨域资源，而 `cacheFirst` 则完全不行。这与 [API 文档](https://developers.google.com/web/tools/workbox/reference-docs/latest/module-workbox-sw.Router#registerRoute) 中的对 `capture` 参数说法并不相同。

> 1. An Express-style route, like '/path/to/:anything' for same-origin or 'https://cross-origin.com/path/to/:anything' for cross-origin routes.
> 2. A regular expression that will be tested against request URLs. For cross-origin routes, you must use a RegExp that matches the start of the full URL, like new RegExp('https://cross-origin\.com/').

具体的 [Demo](https://csbun.github.io/workbox-examples/workbox-cross-origin/index.html) 和 [源码](https://github.com/csbun/workbox-examples/tree/master/workbox-cross-origin)


## CLI 工具

细心的小朋友一定发现了，上面的 `routing` 需要第三次访问才能真正从 Cache 中将缓存返回（或者支持离线），有没有办法将这个时间提前到第二次呢？这里，我们直接用 CLI 工具来解决这个问题。

当作为命令行工具时，Workbox 有 3 个主要的命令：

- `workbox wizard` 生成配置文件 _workbox-config.js_；
- `workbox generateSW` 生成 prefetch 的 ServiceWorker JS 文件（依赖 _workbox-config.js_）；
- `workbox injectManifest` 将 prefetch 代码注入到指定的 JS 文件（依赖 _workbox-config.js_）；

### generateSW

首先修改配置文件 _workbox-config.js_ 如下：

```javascript
module.exports = {
  "globDirectory": "./",        // 匹配根目录
  "globPatterns": [             // 匹配的文件
    "**/*.{css,png,html,js}"
  ],
  "globIgnores": [              // 忽略的文件
    "build/**",
    "workbox-config.js"
  ],
  "swDest": "build.sw.js"       // 目标输出
};
```

运行 `workbox generateSW` 我们即可在 _build/sw.js_ 中看到类似的内容：

```javascript
importScripts("https://storage.googleapis.com/workbox-cdn/releases/3.0.0-beta.1/workbox-sw.js");

self.__precacheManifest = [
  {
    "url": "css/style.css",
    "revision": "835ba5c376a3f48dba17d3a9dc152fc3"
  },
  // ...
  {
    "url": "js/index.js",
    "revision": "589daa65882023b238e57abbb6caa643"
  }
].concat(self.__precacheManifest || []);
workbox.precaching.suppressWarnings();
workbox.precaching.precacheAndRoute(self.__precacheManifest, {});
```

从方法名我们可以猜测出，将 `__precacheManifest` 列表中的文件（即配置中 `globPatterns` 匹配的文件）预加载下来，下次拦截到即从 Cache 中返回。详情可以查看 [precaching 接口文档](https://developers.google.com/web/tools/workbox/reference-docs/v3.0.0-alpha.5/workbox.precaching)。

### injectManifest

但是，很多时候，我们已经有一段 ServiceWorker 的逻辑，希望添加相关的 precaching 代码但不希望增加更多的 sw 文件。于是我需要在 **原本的 sw 文件(_sw.tpl.js_)** 中添加如下代码：

```javascript
// Workbox injectManifest
importScripts("https://storage.googleapis.com/workbox-cdn/releases/3.0.0-beta.1/workbox-sw.js");
workbox.precaching.precacheAndRoute([]);
// Workbox injectManifest End

// 其他自定义 sw 内容
// ...
```

并修改 _workbox-config.js_ (或通过 `workbox wizard --injectManifest` 生成)：

```javascript
module.exports = {
  "globDirectory": "./",
  "globPatterns": [
    "**/*.{css,png,html,js}"
  ],
  "globIgnores": [
    "build/**",
    "workbox-config.js",
    "js/sw.tpl.js"              // “原本的 sw 文件(sw.tpl.js)”
  ],
  "swDest": "build.sw.js",
  // 添加下面这行，指向 “原本的 sw 文件(sw.tpl.js)”
  "swSrc": "./js/sw.tpl.js"     // 输入源
};
```

最后通过运行 `workbox injectManifest` 我们即可在 _build/sw.js_ 中看到类似的内容：

```javascript
// Workbox injectManifest
importScripts("https://storage.googleapis.com/workbox-cdn/releases/3.0.0-beta.1/workbox-sw.js");
workbox.precaching.precacheAndRoute([
  {
    "url": "css/style.css",
    "revision": "835ba5c376a3f48dba17d3a9dc152fc3"
  },
  // ...
  {
    "url": "js/sw.tpl.js",
    "revision": "acc39fc40b04d67b8403e7347d32f43b"
  }
]);
// Workbox injectManifest End

// 其他自定义 sw 内容
// ...
```

再次进入页面看看效果，第一次访问，已经将全部资源 cache 下来了：

{% asset_img cli1.png 第一次访问全部进行 Cache %}

第二次访问，资源已经全部从 Service Worker 中返回，做到完全离线：

{% asset_img cli2.png 第二次访问 Web 完全离线 %}

具体 [Demo](https://csbun.github.io/workbox-examples/workbox-using-cli/index.html) 和 [原码](https://github.com/csbun/workbox-examples/tree/master/workbox-using-cli)。
