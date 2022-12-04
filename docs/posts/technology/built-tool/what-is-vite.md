---
title: 初识Vite
description: 一个基于浏览器原生 ES Modules 的开发服务器。利用浏览器去解析模块，在服务器端按需编译返回，完全跳过了打包这个概念，服务器随起随用
isOriginal: true
icon: ''
date: 2021-01-07
category:
  - 技术文章
  - 构建工具
tag:
  - Vite
# 是否置顶
sticky: false
# 是否收藏在博客主题的文章列表中。当填入数字时，数字越大，排名越靠前
star: false
# 是否将该文章添加至文章列表中。
article: true
# 是否将该文章添加至时间线中。
timeline: true
---
<CountView></CountView>

## vite是什么？
![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f6009e76565b47e59826183525369d88~tplv-k3u1fbpfcp-zoom-1.image)
> 作者尤大: [Vite](https://github.com/vitejs/vite)，一个基于浏览器原生 ES Modules 的开发服务器。利用浏览器去解析模块，在服务器端按需编译返回，完全跳过了打包这个概念，服务器随起随用。同时不仅有 Vue 文件支持，还搞定了热更新，而且热更新的速度不会随着模块增多而变慢。

有以下特点：<br />1、闪电般冷服务启动（就是输入启动command，如npm run dev 启动会很快）<br />2、即时热更新HRM （如[webpack-dev-server](https://webpack.docschina.org/guides/hot-module-replacement/) 实现的模块热更新）<br />3、真正的按需编译<br />

## vite与webpack 有什么不同？
### webpack
Webpack会把所有资源（包括Javascript，图像，字体和CSS等）打包后置于依赖关系中，如：<br />在`main.js`中引入 `a.js`和`b.js`
```javascript
// a.js
const a = () => { return 10 }
export { a }

// b.js
const b = () => { return 20 }
export { b }
// c.js
import { a } from './a'
import { b } from './b'

const c = return a() + b()
export { c }
```
经过webpack处理后打包生成`bundle.js`
```javascript
// bundle.js
const a = 10;
const b = 20;
const c = return a + b;
export { c };

```
可以看到bundle.js里面的c模块依赖a和b模块的返回值来计算正确结果，因此如若a模块代码进行了更改，比如`a = 30` 那么webpack会对`bundle.js`所有依赖的模块重新打包生成新的`bundle.js`
```javascript
//更改a模块后 生成新的bundle.js
// bundle.js
const a = 30;
const b = 20;
const c = return a + b;
export { c };
```
**总结：**修改了 bundle 模块中的一个子模块， 整个 bundle 文件都会重新打包然后输出，随着项目的扩大，整个项目依赖的子模块会越来越多，则需要打包的资源也增多了，打包的时间也会变得越来越长。
### vite
**重点：不需要打包，省去打包时间**<br />vite利用浏览器支持的ES Moudle，遇到代码里的import xx  from "xxxxxxx",就直接去找对应的资源，找到直接引入就完事了。例如，代码里有
```javascript
// a.js
const a = () => { return 10 }
export { a }

// b.js
const b = () => { return 20 }
export { b }
// c.js
import { a } from './a'
import { b } from './b'

const c = return a() + b()
export { c }
```
使用：<br />![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa70a0adfe074352b94eb4eeb9a14e75~tplv-k3u1fbpfcp-zoom-1.image)<br />打开控制台可以看到，模块 a b c 三个模块都被请求并加载了！![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bda46f82c2e8411d85265b087bb523fb~tplv-k3u1fbpfcp-zoom-1.image)<br />接下来看看 更改代码的时候，热更新怎样的。<br />以下只更改a模块里面的代码<br />![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21f61ac17918427c846701855bf0a826~tplv-k3u1fbpfcp-zoom-1.image)<br />可以看到b模块不会更新，只更新了与a依赖的模块c以及vue文件（注意，若是c模块未在代码里使用，尽管更改了模块 a b的代码都是不会在控制台看到加载的，这就是**按需引入**）<br />

## vite热更新原理是什么？

<br />[ES Moudle原理请戳](https://segmentfault.com/a/1190000014318751)<br />ES Modules 是用于处理模块的 ECMAScript 标准。 用法：
```javascript
//script导入
<script type="module" src="index.js"></script>

// import导入
import Vue from 'vue'

// export导出
export default str => str.toUpperCase()
export {a, b, c}
```
如在vite项目中：<br />当服务启动时，vite便对当前import的文件进行了监听，热更新其实就是当文件有更改时，vite会判断更改的是什么文件，而采用对应的plugin对文件进行重新编译处理，而后返回给浏览器。并且会在文件加上时间戳，避免浏览器因为缓存而不更新文件。如更改helloworld.vue里面的代码，websocker会发送一个指令给vite，根据指令来执行不同的更新操作。<br />![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c392062f20d84a27bb297e467c7128c4~tplv-k3u1fbpfcp-zoom-1.image)
## vite更新了什么？
官方中文文档： [https://vitejs.dev/](https://vitejs.dev/)<br />从V1迁移到新版本，最明显的是vue被当成插件plugin抽离出来了，而vite就真真切切成为了一个编译打包工具，新建的时候可以选择需要支持的框架或者单文件
```javascript
// 新建一个project
npm init @vitejs/app
```
![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69686e46bfd64b5eaae0adbbb2a07259~tplv-k3u1fbpfcp-zoom-1.image)<br />选择一个vue项目，发现根目录下多了个`vite.config.js`文件，这也是此次更新的一大块内容，比如对于打包，sourceMap，server都变得易配置。[详情](https://vitejs.dev/config/)<br />![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9682879a16004338b60d785d062d2a24~tplv-k3u1fbpfcp-zoom-1.image)



