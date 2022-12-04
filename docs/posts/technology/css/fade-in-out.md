---
title: CSS动画-实现元素依次淡出
description: 公司网站动画要求不高，因此不必要上Lottie，用纯css就好，选啥，大伙得视自己公司项目而定
isOriginal: true
icon: ''
date: 2021-06-20
category:
  - 技术文章
  - CSS
tag:
  - css
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

在查看webpack打包出来的bundle时，发现一个叫Lottie的js占用的体积还挺大,并且有好几个诡异的json文件，于是想着能不能优化，然后去项目里搜，发现好几个弹窗里的动画实现用到了这玩意，而且lottie制作的动画一个json文件就很大了，什么炫酷的动画竟然要引入Lottie来解决？ 
![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2c9470ff925a41da8a596ecab1138f15~tplv-k3u1fbpfcp-zoom-1.image)
咋一看，这不挺简单的吗？ 就淡入加延时的效果，再加上气泡的实现。下面就来实现一波吧。

>  公司网站动画要求不高，因此不必要上Lottie，用纯css就好，选啥，大伙得视自己公司项目而定。

demo：[https://codepen.io/jlin2/pen/mdWGrvP](https://codepen.io/jlin2/pen/mdWGrvP)

## css动画animation

MDN：[https://developer.mozilla.org/en-US/docs/Web/CSS/animation](https://developer.mozilla.org/en-US/docs/Web/CSS/animation)

```css
animation: 3s ease-in 1s infinite reverse both running slidein;
animation是以下动画属性的简写：
```

| 属性               | 说明               | 可选值                  |
| ------------------ | ------------------ | ----------------------- |
| animation-name     | 指定动画的名称     | 跟@keyframes [name]相同 |
| animation-duration | 整个动画完成的时间 | 1s 2s 3s ........       |
| animation-timing-function | 动画以怎样的速度从开始到完成 | linear：匀速，ease-in：加速，ease-out：减速 cubic-bezier函数：自定义速度模式 |
| animation-delay | 动画开始的延时 | 1s 2s 3s ........ |
| animation-iteration-count | 动画重复的次数 | infinite：无限播放，1，2，3.... |
| animation-direction | 指定动画播放的方向 | normal、alternate、reverse、alternate-reverse |
| animation-fill-mode | 指定动画结束播放时的状态 | forwards、backwards、both...... |
| animation-play-state | 指定动画在播放过程中的状态，比如暂停、播放 | paused：暂停，running：运行 |

## 淡入实现fadeInUp

定义一个动画帧`@keyframes fadeInUp`，运用到相关页面元素上就可。

```css
.fadeInUp {
  animation-name: fadeInUp;
  animation-duration: 1s;
  animation-iteration-count:1;
}
@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translate3d(0, 100%, 0);
  }
  to {
    opacity: 1;
    transform: translate3d(0, 0, 0);
  }
}
```

## 延时delay

要求页面不同元素执行动画的时序不同，因此用`animation-delay`进行控制，可具体到毫秒 ,`animation-fill-mode`设置为`backwards`让动画回到第一帧状态。

```css
.delay1000 {
  animation-delay: 1s;
  animation-fill-mode: backwards;
}
.delay2000 {
  animation-delay: 2s;
  animation-fill-mode: backwards;
}
.delay3000 {
  animation-delay: 3s;
  animation-fill-mode: backwards;
}
.delay4000 {
  animation-delay: 4s;
  animation-fill-mode: backwards;
}
```

## 聊天气泡的实现

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/101d698d29a84d698dbf642ba7859f0e~tplv-k3u1fbpfcp-zoom-1.image)

### 实现三角小箭头

众所周知，实现三角形可以用border的transparent来实现。这里因为要显示边框的颜色，因此得实现两个小三角形，一个三角形用边框颜色填充，另外一个三角形跟矩形气泡同色，再偏移与边框宽度相等的px。

```css
.bubble{
  position: relative;
  padding: 10px 20px;
  border: 1px solid #c83c23;
  border-radius: 12px;
  margin-bottom: 24px;
  background: #fff;
}
.bubble::before{
  content: '';
  position: absolute;
  top: 20%;
}
.bubble::after{
  content: '';
  position: absolute;
  top: 20%;
}
.bubble-left::before{
  left: -12px;
  border-top: 8px solid transparent;
  border-bottom: 8px solid transparent;
  border-right:12px solid #c83c23;
}
.bubble-left::after{
  left: -10px;
  border-top: 8px solid transparent;
  border-bottom: 8px solid transparent;
  border-right:12px solid #fff;
}
.bubble-right::before{
  right: -12px;
  border-top: 8px solid transparent;
  border-bottom: 8px solid transparent;
  border-left:12px solid #c83c23;
}
.bubble-right::after{
  right: -10px;
  border-top: 8px solid transparent;
  border-bottom: 8px solid transparent;
  border-left:12px solid #fff;
}
```

### X偏移的弹出

更改keyframes中 to的X偏移量

```css
@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translate3d(0, 100%, 0);
  }
  to {
    opacity: 1;
    transform: translate3d(150, 0, 0);
  }
}
.offet-left {
  transform: translateX(-150px)
}
.offet-right {
  transform: translateX(150px)
}
```

## 背景图片也得淡入？

这时要用load事件监听图片加载完之后，再显示气泡的动画就行。

## 最终效果：


![2021-06-10 16.51.13.gif](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5a4985839c4c4964b5642a000703f539~tplv-k3u1fbpfcp-watermark.image)
## 总结：

demo：[https://codepen.io/jlin2/pen/mdWGrvP](https://codepen.io/jlin2/pen/mdWGrvP)

开发中基本很少要写动画效果，自己趁此机会熟悉了animation属性，其实有些同事一看到动画效果的交互都喜欢直接叫UI设计师用Lottie制作好，前端拿来直接用，省事；但是觉得还是要考虑具体的项目，像本司都是管理后台，基本都没动画，更别说酷炫的动画了。

用css写，把Lottie的json文件删除了足足6MB，感觉轻快了许多。

另外Lottie真的挺好用的，在UI大佬要非常炫酷动画时候，简直前端是不加班利器，附上地址：[https://airbnb.io/lottie/#/](https://airbnb.io/lottie/#/)

好用的动画实现库：https://animate.style/



