# tieba首页-欧冠16强出炉-轮播图调试

## 背景

看到贴吧首页欧冠16强出炉的轮播图总感觉哪里怪怪的，在页面上随便瞎逛的时候，利用devtool中的Rendering面板中的Layout Shift Regions功能查看在页面LCP（Large Content Paint）后页面布局移动的地方，发现这个地方不同于其他两处轮播图，不是大家认为的轮播图的实现方式，这个发现不是看源码、是利用调试工具和对轮播图实现方式的理解。用你的鼠标hover到轮播图，当轮播图滚动的时候会看到你hover图片的移动轨迹，欧冠16强出炉的轮播图并没有发现这种轨迹，说明它一定有猫腻，那么进一步往下调试。

## Sources-Snippets

因为贴吧首页东西很多，我只想要专注于这个轮播图的调试，隐藏所有页面上其他东西的显示。devtool Element中只能选择隐藏自己选中的元素，我只能选择自己写个命令来执行这个操作。Snippets可以让我们写一些js脚本，来执行自己想做的事情。

<img src="/Users/chenqiong/Library/Application Support/typora-user-images/image-20210906040252905.png" alt="image-20210906040252905" style="zoom:50%;" />

两句js来满足我想要的需求。操作方式是你需要用选择工具选中你想要页面区域，然后运行这个脚本，运行这个脚本的方式有两种，第一种就是command + shift + P输入! + 文件名、或者直接找到面板中的这个文件、点击右下角的运行就可以了。

## 运行方式

首先想到的是打断点、因为轮播图包裹的元素发生了变化，所以我在ELement找到这个包裹元素，接着右键选择子树被修改时出发中断

![image-20210906041137768](/Users/chenqiong/Library/Application Support/typora-user-images/image-20210906041137768.png)

接着就进入到了Resources面板的js资源中进行调试，发现实现方式是通过XXX、就不打字了，我修改了一处js源代码、录个屏给大家看一下大家就明白了。录屏之前先说两个调试技巧。

### 调试技巧

当我们想要更改js源码、并且看到运行效果的时候、一般的常识是更改了之后并不会生效。一方面是因为js已经在页面加载的时候执行过了，另一方面则是因为重载会加载网络上的资源、覆盖掉自己更改的代码。那么有没有可能devtool在页面重载时只使用我们更改后的资源、不请求网络上的该资源呢？是可以的，通过Resources-Overrides可以做到。

![image-20210906043034688](/Users/chenqiong/Library/Application Support/typora-user-images/image-20210906043034688.png)

还有一个小技巧是当修改完代码后、可以打开Changes里面会记录修改前后的对比。如上图所示。

还有一个就是大家用的都很熟练的在针对Element中选中的元素、修改它的样式。建议大家以后在修改css时总是打开圆圈处的面板，可以直观的看到各种规则叠加在一起形成的最终的样式规则，点击你感兴趣的规则、就可以直接导航到实际编写的地方，如下划线的地方。

![image-20210906043539185](/Users/chenqiong/Library/Application Support/typora-user-images/image-20210906043539185.png)

接下来就是展示修改之后的效果：

<video src="/Users/chenqiong/Desktop/屏幕录制2021-09-06 上午4.42.55.mov"></video>

## 原理

上面我一直通过的是调试工具直观的感受该轮播图的实现方式、但是有时候你不得不去面对源代码、那些经过压缩的、对于一个使用devtool调试源代码极度不友好的网站，当然现代网站都属于这种类型。所以也就荒废了devtool一个很大的能力，能看懂经过压缩处理的代码并且正确的修改它们，比想象中要困难的多。

幸运的是这部分轮播图的源代码并没有经过怎么压缩。

![image-20210906045814672](/Users/chenqiong/Library/Application Support/typora-user-images/image-20210906045814672.png)

![image-20210906045718164](/Users/chenqiong/Library/Application Support/typora-user-images/image-20210906045718164.png)

![image-20210906050015304](/Users/chenqiong/Library/Application Support/typora-user-images/image-20210906050015304.png)

![image-20210906050109958](/Users/chenqiong/Library/Application Support/typora-user-images/image-20210906050109958.png)

这是四个实现轮播图重要的函数。具体的原理就是，设置定时器、每经过10ms就把第一个方块的宽度减小20px，每一张方块的宽度为180px也就是说需要9次，后面的方块才能通过正常文档流的方式流动到上一个方块的位置。设置和清除了9个定时器，多么奢侈的行为，可以一行代码就搞定了，并且效果都是一样的。

## 吐槽

第一点：原程序员想要通过9个定时器实现的效果用transition: width 90ms; 就可以实现一模一样的效果。可以自己通过在Element 面板上设置css样式进行实验，我没看出来什么不同、也不录屏了。

第二点：这样造成的CLS比该页面其他两个地方都要大，因为每一次都需要重新计算12个方块的位置，以及每次定时器执行的时候都要计算第一个方块的宽度变化。至于另外两个轮播图，建议固定容器大小使用background-image，只更改每次容器的背景图片就行，这样就不用重新计算样式。贴张Performance面板捕捉的experience, 这肯定是不开心的体验。

![image-20210906051808833](/Users/chenqiong/Library/Application Support/typora-user-images/image-20210906051808833.png)

吐槽了上面两点、再说点其他的可能有用的题外话、贴吧首页头部会在载入页面的过程中发生CLS，利用Network，把网络情况调成slow 3G就会看的比较明显、对于我这样的用户来说总是看着不舒服。

![image-20210906053012661](/Users/chenqiong/Library/Application Support/typora-user-images/image-20210906053012661.png)

还要吐槽一个点：该轮播图定时器采用的时间间隔是10ms、但是浏览器的渲染频率帧是60fps，也就是大约16ms才绘制一次屏幕。整体时长是90ms，但是人的视觉暂留时间在100ms左右，在上一个方块还在视觉印象中，马不停蹄的移动了9次，共90ms, 移动了180px，只能说有一点点过渡效果，还是全靠定时器的误差，而且效果并不理想。
