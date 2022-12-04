---
title: vue cli项目打包优化，我能做的就这些了
description: vue cli本地构建打包也大约5min,到线上腾讯云docker构建打包全过程需要16~20min。因此不仅极大影响开发效率，也大大延迟了给测试交付的时间。
isOriginal: true
icon: ''
date: 2021-05-02
category:
  - 技术文章
  - Vue
tag:
  - vue
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

## 背景

因为公司的前端项目在打包构建方面时间属实长得离谱，冷启动大约5min，本地构建打包也大约5min,到线上腾讯云docker构建打包全过程需要16~20min。因此不仅极大影响开发效率，也大大延迟了给测试交付的时间。

## 优化思路

- 了解项目当前webpack配置
- 构建相关优化
- 打包体积相关优化
- docker相关优化


## 分析工具

### speed-measure-webpack-plugin

#### 介绍：

[speed-measure-webpack-plugin npm](https://www.npmjs.com/package/speed-measure-webpack-plugin)

> The first step to optimising your webpack build speed, is to know where to focus your attention.
> This plugin measures your webpack build speed, giving an output like this:

通过smp输出的分析可以清楚的了解到webpack构建过程中，每一阶段的loader以及plugin的工作花费的时间。

#### 使用方式：

```javascript
# Yarn
yarn add -D speed-measure-webpack-plugin

const SpeedMeasurePlugin = require('speed-measure-webpack-plugin')
module.exports = {
  chainWebpack: config => {
    config
      .plugin('speed-measure-webpack-plugin')
      .use(SpeedMeasurePlugin)
      .end()
  }
}
```

在本项目中使用其他的使用都会报error，但是以上的用法似乎不会区分plugin与loader的使用，甚至没有其他plugin的使用情况信息，迷惑~。


### webpack-bundle-analyzer

#### 介绍：

[webpack-bundle-analyzer npm](https://www.npmjs.com/package/webpack-bundle-analyzer)
用来分析webapck构建打包后的文件，如分包情况，占用体积等参数的分析。

#### 使用方式：

```javascript
# Yarn
yarn add -D webpack-bundle-analyzer

const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;
module.exports = {
  plugins: [
    new BundleAnalyzerPlugin()
  ]
}
```

但是在vue-cli中有report命令可以直接调用，然后去dist打包目录打开report.html。

```javascript
vue-cli-service build --report
or
vue-cli-service build --report-json
```

## 查看vue-cli当前webpack配置

### 介绍

vue-cli脚手架会有webpack的很多默认行为，因此我们得知道基于vue-cli的项目，当前的webpack都配置了啥，然后才能做针对性的分析与优化。

> `vue-cli-service` 暴露了 `inspect` 命令用于审查解析好的 webpack 配置。那个全局的 `vue` 可执行程序同样提供了 `inspect` 命令，这个命令只是简单的把 `vue-cli-service inspect` 代理到了你的项目中。

### 使用方式：

```javascript
#根据mode，分别生成开发环境、生产环境的配置
vue inspect --mode production > output.js
#输入命令后，在根目录会生产一个output.js文件
```

如果`vue command not found`的错可以全局安装注册一下vue命令`npm install -g vue-cli`

## 优化尝试

### hard-source-webpack-plugin加缓存

#### 介绍

[hard-source-webpack-plugin npm](https://www.npmjs.com/package/hard-source-webpack-plugin)

> HardSourceWebpackPlugin is a plugin for webpack to provide an intermediate caching step for modules. In order to see results, you'll need to run webpack twice with this plugin: the first build will take the normal amount of time. The second build will be signficantly faster.

在启动项目时会针对项目生成缓存，若是项目无package或其他变化，下次就不用花费时间重新构建，直接复用缓存。

#### 使用方式：

```javascript
#yarn
yarn add -D hard-source-webpack-plugin

const HardSourceWebpackPlugin = require('hard-source-webpack-plugin')
module.exports = {
	configureWebpack: config => {
  	config.plugin.push(
    	// 为模块提供中间缓存，缓存路径是：node_modules/.cache/hard-source
      // solve Configuration changes are not being detected
      new HardSourceWebpackPlugin({
        root: process.cwd(),
        directories: [],
        environmentHash: {
          root: process.cwd(),
          directories: [],
          files: ['package.json', 'yarn.lock']
        }
      })
      // 配置了files的主要原因是解决配置更新，cache不生效了的问题，配置后有包的变化，plugin会重新构建一部分cache
    )
  }
}
```

#### 注意：

`Could not freeze : Cannot read property 'hash' of undefined` 删除node_modules/.cache后，重新启动项目，产生这个问题的原因可能是异步加载模块时编译产生的错误，或者加上这个可解决：
 ``` 
 new HardSourceWebpackPlugin.ExcludeModulePlugin([
  {
    // HardSource works with mini-css-extract-plugin but due to how
    // mini-css emits assets, assets are not emitted on repeated builds with
    // mini-css and hard-source together. Ignoring the mini-css loader
    // modules, but not the other css loader modules, excludes the modules
    // that mini-css needs rebuilt to output assets every time.
    test: /mini-css-extract-plugin[\\/]dist[\\/]loader/
  }
])
 
 ```

### 缩小文件检索解析范围

为避免无用的检索与递归遍历，可以使用alias指定引用时候的模块，noParse，对不依赖本地代码的第三方依赖不进行解析。

```javascript
// 定义getAliasPath方法，把相对路径转换成绝对路径
const getAliasPath = dir => join(__dirname, dir)
module.exports = {
	configureWebpack: config => {
    config.module.noParse = /^(vu|vue-router|vuex|vuex-router-sync|lodash|echarts|axios|element-ui)$/
  }
  chainWebpack: config => {
    // 添加别名
    config.resolve.alias
      .set('@', getAliasPath('src'))
      .set('assets', getAliasPath('src/assets'))
      .set('utils', getAliasPath('src/utils'))
      .set('views', getAliasPath('src/views'))
      .set('components', getAliasPath('src/components'))
	}
  // 生产环境禁用eslint
  lintOnSave: !process.env.NODE_ENV !== 'production',
}
```

### 减少打包体积

#### image-webpack-plugin 图片压缩

对图片像素要求没很极致的，这个压缩还是可以使用的，压缩率肉眼看起来感觉是没太大区别。
这里注意一下，我没有对svg进行压缩，原因是压缩的svg,再通过构建时被打包成base64时，生成的base64会有问题，无法访问。

```javascript
module.exports = {
  chainWebpack: config => {
   // 对图片进行压缩
    config.module
      .rule('images')
      .test(/\.(png|jpe?g|gif)(\?.*)?$/)
      .use('image-webpack-loader')
      .loader('image-webpack-loader')
      .options({ bypassOnDebug: true })
      .end()
	}
}
```

#### UglifyJsPlugin删除console 注释（不建议）

uglifyJsPlugin 用来对js文件进行压缩，减小js文件的大小。其会拖慢webpack的编译速度，建议开发环境时关闭，生产环境再将其打开。
更建议规范团队成员的代码上去解决。

```javascript
#yarn
yarn add -D uglifyjs-webpack-plugin

const UglifyJsPlugin = require('uglifyjs-webpack-plugin')

module.exports = {
	configureWebpack: config => {
  	config.plugin.push(
    	new UglifyJsPlugin({
        uglifyOptions: {
          // 删除注释
          output: {
            comments: false
          },
          // 删除console debugger 删除警告
          compress: {
            warnings: false,
            drop_console: true, //console
            drop_debugger: false,
            pure_funcs: ['console.log'] //移除console
          }
        },
        sourceMap: false,
        parallel: true //使用多进程并行运行来提高构建速度。默认并发运行数：os.cpus().length - 1。
      })
    )
  }
}
```
### terser 删除console
terser仍在维护，而UglifyJs无人维护了，terser功能上比后者强大很多
```javascript
chainWebpack: config => {
config.when(isProd, config => {
  config.optimization.runtimeChunk('single')
  // 配置删除 console.log
  config.optimization.minimizer('terser').tap(args => {
    // remove debugger
    args[0].terserOptions.compress.drop_debugger = true
    // 移除 console.log
    args[0].terserOptions.compress.pure_funcs = ['console.log']
    // 去掉注释 如果需要看chunk-vendors公共部分插件，可以注释掉就可以看到注释了
    args[0].terserOptions.output = {
      comments: false
    }
    return args
  })
})
}
```

### DLL动态链接库

这个插件是在一个额外的独立的 webpack 设置中创建一个只有 dll 的 bundle(dll-only-bundle)。 这个插件会生成一个名为 manifest.json 的文件，这个文件是用来让 DLLReferencePlugin 映射到相关的依赖上去的。

> 可以简单理解为把一些依赖从项目的bundle中拆分出去，通过映射关系用请求来加载。我认为拆分出去之后不会再在项目里被解析，因此对构建，体积都是有所帮助的。

配置DllPlugin,可以分为下面几个步骤：

1. 新建webpack.dll.config.js文件(其他命名都可以)，配置需要拆分的插件；
1. 在package.json文件中新建一条命令来专门打包，`"build:dll":"webpack --config webpack.dll.config.js"`; 运行该命令；
1. 在vue.config.js 文件中配置`DllReferencePlugin`,主要把dll引用到需要预编译的依赖；
1. 在index.html手动引入拆分的bundle包（放到cdn的话会更好）

安装：

```javascript
#yarn 
yarn add webpack-cli@^3.2.3 add-asset-html-webpack-plugin@^3.1.3 clean-webpack-plugin@^1.0.1 --dev
```

```javascript
// webpack.dll.config.js
/* eslint-disable @typescript-eslint/no-var-requires */
const path = require('path')
const webpack = require('webpack')
const CleanWebpackPlugin = require('clean-webpack-plugin')
// dll文件存放的目录
const dllPath = 'public/vendor'

module.exports = {
  entry: {
    // 需要提取的库文件
    vendor: ['vue', 'vue-router', 'vuex'],
    utils: ['axios', 'lodash']
  },
  output: {
    path: path.join(__dirname, dllPath),
    filename: '[name].dll.js',
    // vendor.dll.js中暴露出的全局变量名
    // 保持与 webpack.DllPlugin 中名称一致
    library: '[name]_[hash]'
  },
  plugins: [
    // 清除之前的dll文件
    new CleanWebpackPlugin(['*.*'], {
      root: path.join(__dirname, dllPath)
    }),
    // manifest.json 描述动态链接库包含了哪些内容
    new webpack.DllPlugin({
      path: path.join(__dirname, dllPath, '[name]-manifest.json'),
      // 保持与 output.library 中名称一致
      name: '[name]_[hash]',
      context: process.cwd()
    })
  ]
12

```

在`vue.config.js plugin`中使用

```javascript
config.plugin.push(
  new DllReferencePlugin({
    context: process.cwd(),
    manifest: require('./public/vendor/vendor-manifest.json')
  }),
    new DllReferencePlugin({
    context: process.cwd(),
    manifest: require('./public/vendor/utils-manifest.json')
  }),
    // 将 dll 注入到 生成的 html 模板中
    new AddAssetHtmlPlugin({
    // dll文件位置
    filepath: getPath('./public/vendor/*.js'),
    // dll 引用路径
    publicPath: './vendor',
    // dll最终输出的目录
    outputPath: './vendor'
  })
)
```

### splitChunks 分割代码

[split-chunks-plugin webpack](https://webpack.docschina.org/plugins/split-chunks-plugin/)

- chunks: 表示哪些代码需要优化，有三个可选值：initial(初始块)、async(按需加载块)、all(全部块)，默认为async
- minSize: 表示在压缩前的最小模块大小，默认为30000
- minChunks: 表示被引用次数，默认为1
- maxAsyncRequests: 按需加载时候最大的并行请求数，默认为5
- maxInitialRequests: 一个入口最大的并行请求数，默认为3
- automaticNameDelimiter: 命名连接符
- name: 拆分出来块的名字，默认由块名和hash值自动生成
- cacheGroups: 缓存组。缓存组的属性除上面所有属性外，还有test, priority, reuseExistingChunk
  - test: 用于控制哪些模块被这个缓存组匹配到
  - priority: 缓存组打包的先后优先级
  - reuseExistingChunk: 如果当前代码块包含的模块已经有了，就不在产生一个新的代码块

```javascript
config.optimization = {
  runtimeChunk: 'single',
  splitChunks: {
    chunks: 'all', // 表示哪些代码需要优化，有三个可选值：initial(初始块)、async(按需加载块)、all(全部块)，默认为async
    maxInitialRequests: Infinity, // 按需加载时候最大的并行请求数，默认为5
    minSize: 30000, // 依赖包超过300000bit将被单独打包
    // 缓存组
    // priority: 缓存组打包的先后优先级
    // minChunks: 表示被引用次数，默认为1
    cacheGroups: {
      //公共模块
      commons: {
        name: 'chunk-commons',
        test: resolve('src'), // can customize your rules
        minSize: 100, //大小超过100个字节
        minChunks: 3, //  minimum common number
        priority: 5,
        reuseExistingChunk: true
      },
      // 第三方库
      libs: {
        name: 'chunk-libs',
        test: /[\\/]node_modules[\\/]/,
        priority: 10,
        chunks: 'initial', // only package third parties that are initially dependent
        reuseExistingChunk: true,
        enforce: true
      },
      echarts: {
        name: 'chunk-echarts',
        test: /[\\/]node_modules[\\/]echarts[\\/]/,
        chunks: 'all',
        priority: 12,
        reuseExistingChunk: true,
        enforce: true
      }
    }
  }
}
```
### compression-webpack-plugin gzip打包

用法：<https://www.npmjs.com/package/compression-webpack-plugin>
```javascript
const CompressionWebpackPlugin = require("compression-webpack-plugin");

const IS_PROD = ["production", "prod"].includes(process.env.NODE_ENV);
const productionGzipExtensions = /.(js|css|json|txt|html|ico|svg)(?.*)?$/i;

module.exports = {
  configureWebpack: config => {
    const plugins = [];
    if (IS_PROD) {
      plugins.push(
        new CompressionWebpackPlugin({
          filename: "[path].gz[query]",
          algorithm: "gzip",
          test: productionGzipExtensions,
          threshold: 10240,
          minRatio: 0.8
        })
      );
    }
    config.plugins = [...config.plugins, ...plugins];
  }
};
```

### terser-webpack-plugin 多线程压缩js

用法：<https://www.npmjs.com/package/terser-webpack-plugin>

vue-cli3默认的webpack有此优化

### externals & cdn

externals 配置选项提供了「从输出的 bundle 中排除依赖」的方法。防止将某些 import 的包(package)打包到 bundle 中，而是在运行时(runtime)再去从外部获取这些扩展依赖(external dependencies)。

这个属性很好理解，而且使用起来也非常方便，非常的nice! 最简单的方法是配置名称，当然你也可以编写一些复杂的配置[官方文档](https://www.webpackjs.com/configuration/externals/)

```javascript
//vue.config.js
...
configureWebpack:{
	externals: {
      "vue": "Vue",
      "element-ui": "ELEMENT"
    },
}
// 然后在 index.html 手动cdn引入(或者用插件自动添加)
```

## 总结

以上是我在公司项目做的一些小优化，对其他项目不一定适用，甚至使用上也不是最优，但目前对本项目很大程度上还是有帮助的。更多webpack优化的思路，我附上个思维导图吧。(仅供学习，记录📝)
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dbedc66bbafc46ecbf24566346569527~tplv-k3u1fbpfcp-zoom-1.image)

## 其他

[Vue CLI](https://cli.vuejs.org/zh/)

[vue-cli中的configureWebpack设置](https://github.com/fncheng/blog/issues/26)

[如何使用 docker 部署前端项目](https://shanyue.tech/frontend-engineering/docker.html#)

