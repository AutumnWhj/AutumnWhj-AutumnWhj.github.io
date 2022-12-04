---
title: V8的垃圾回收机制♻️
description: 在内存管理上往往是分为新生代内存空间与老生代内存空间来进行高效的内存管理。
isOriginal: true
icon: ''
date: 2021-01-21
category:
  - 技术文章
  - Node
tag:
  - node
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

## V8内存管理

在Node中，通过JavaScript只能使用部分内存，64位系统下约为1.5G，32位系统下约为0.7G，因此但凡要做一个读取2G文件的操作，都会导致内存溢出，因此要深入了解V8的内存管理机制，才能避免问题并能更好的管理内存。

V8为何要限制内存大小？表层原因，最初V8为浏览器设计的，不太可能遇到大内存的场景，在此应用上绰绰有余；深层原因是V8的垃圾回收机制。

提一嘴， 在V8中，JavaScript对象是通过堆来存储的，堆的物理结构是数组，而逻辑结构是一颗树。

## V8垃圾回收机制

### 内存分代

在内存管理上往往是分为新生代内存空间与老生代内存空间来进行高效的内存管理。

### 新生代——Scavenge算法

主要采用Cheney算法，一种采用复制复制方式实现的垃圾回收算法，它将内存一分为二，每一部分空间叫semispace。处于使用状态的空间叫From，闲置状态的是TO，核心是通过将存活对象在这两个semispace空间进行复制。类比GC是一个过滤筛，每次GC把From中的存活对象筛出来，扔到TO这个盘子，然后反转，把TO这个盘子当做From，用筛子又筛一遍，看看还剩下哪些存活对象。详细步骤如下：

-   **划分内存空间**：将内存分为From（使用状态）、To（闲置状态）两块空间；

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f6e33498ea241b7b62d432de82c14ee~tplv-k3u1fbpfcp-zoom-1.image)

-   **垃圾回收进行时：** 垃圾回收开始时，会先检查From里面的对象是存活状态和还是非存活状态，存活对象会被复制到To空间，To空间保存着这些存活对象，非存活对象就被释放掉；
-   **反复过筛：** 当Scavenge回收完成后，Form和To会进行翻转(即上次的To变成了From，From变成了To)，下一次Scavenge回收仍然是对From空间里面的对象进行检查，重复2步骤，

<!---->

-   **晋升：** 若From中的存活对象多次检查仍然未被释放，会被认为是存活周期较长的对象，而晋升到老生代内存中去，或者To空间已被占用超过25%了，那也晋升到老生代中去。

### 老生代——Mark-sweep & Mark-compact算法

老生代即在存储中存活较久以及占用内存比较大的对象，因此`Scavenge`反复复制的算法就不适用了，一是大部分是存活的对象过筛的效率降低，而是浪费一半的内存空间。

因此老生代使用`Mark-sweep`标记清除算法对对象进行标记，而未标记📌的就会在GC过程中给清除掉，上面说了堆的物理结构是数组，经过`Mark-sweep`把对象清除后，空间就会变得不连续，再新的对象进来时不好做内存分配，因此Mark-compack算法就是用来整理堆空间的。详细步骤如下：

-   **标记清除：** `Mark-sweep`标记清除，标记阶段只标记存活的对象，其他未做标记的会在清除阶段给清除掉；

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/df3510f48cd44e42a5c4583ee1e0aed9~tplv-k3u1fbpfcp-zoom-1.image)

-   **空间整理：** 清除掉的对象会在内存形成零零散散分布的内存块，为了后续插入内存大的对象有足够空间，`Mark-sweep`后会进行`Mark-compact`把死亡对象的空间都整理在存活对象的右边👉，清理时直接把右边👉的空间都清空就行；
-   **增量清除：** 若碰到内存占用大时，一次垃圾回收会造成全停顿时间较长，因此在标记阶段使用增量标记Incremental Marking，可边标记边执行应用逻辑，尽可能少的阻塞js逻辑的执行。（虽然用增量标记使得垃圾回收的整个过程耗时增加，但是对于程序的执）

## 避免内存泄露

### 全局变量

在全局作用域中定义变量时，会自动挂载到window对象上，垃圾回收时，该对象被认为有被window对象引用，因此不会被标记清楚掉，一直存在内存中。

```
如在控制台输入
var a = 100
function b() { console.log('b') }

window.a = 100
```

**解决**：尽可能不用全局变量，实在使用在程序执行完前要把他设为null, 如`a=null`

### 监听器，计时器

```
监听某一个事件
window.addEventListener()

计时
const a = 0;
const b = function() {
    a++
};
window.setInterval(b, 1000);
```

**解决**：用`window.removeEventListener()`移除监听器，`window.clearInterval()`移除计时器，可在程序中手动移除，或在程序执行完前移除，如vue`beforeDestroy`组件销毁前时把监听，定时器移除。

### 闭包

内层函数中访问到其外层函数的作用域形成了闭包，少用闭包

```
function a() {
    let b = 'b';
    return function c() {
        console.log(b)
    }
}
c里面会一直对b有引用
```

### dom绑定

```
let btn = $('#btn')
btn.onclick = (event) => { console.log(event.target) }
btn的引用：
1、btn.onclick ——>  btn
2、event.target ——> btn
btn = null 只是清楚 btn.onclick 的引用，但是一直存在引用event.target === btn的dom。
```

**解决**：移除dom元素，`document.body.removeChild(btn)`

### 弱引用WeakMap

`弱引用`，它的键名所引用的对象均是弱引用，弱引用是指垃圾回收的过程中不会将键名对该对象的引用考虑进去，只要所引用的对象没有其他的引用了，垃圾回收机制就会释放该对象所占用的内存

```
// Map是强引用，key的值为null,map还是保存着对key的引用，会存在内存泄露问题
let map = new Map()
let key = new Array(1000000).fill(0)
map.set(key, 1)
key = null

// WeapMap是弱引用，key的值为null时，会自动delete掉weak中的key键值
let weak = new WeakMap()
let key = new Array(1000000).fill(0)
weak.set(key, 1)
key = null
```

**Map强引用无法清理内存测试：**\
以下使用Map发现heapUsed执行 `key=null`时，还是没办法清理掉内存的。

```
node --expose-gc
global.gc() // 手动执行gc垃圾回收
process.memoryUsage() // 查看当前内存使用
let map = new Map()
let key = new Array(1000000).fill(0)
map.set(key, 1)
global.gc()
process.memoryUsage()
// 清理内存 对比发现清不掉
key = null
global.gc()
process.memoryUsage()
```




## 参考

《深入浅出Nodejs》