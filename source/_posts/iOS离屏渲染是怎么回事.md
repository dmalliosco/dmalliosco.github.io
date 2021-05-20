---
title: iOS离屏渲染是怎么回事?
tags: 性能优化
categories: 技术分享
---
# 什么是离屏渲染？
在几乎所有的面试过程中，我们都会被问到一个问题，什么是离屏渲染？为什么我们会被问到这个问题呢？看起来这个问题跟我们的开发并没有任何的关联。
## 1.了解CPU和GPU的区别
**CPU**（Central Processing Unit）:现代计算机整个系统的运算核心、控制核心。
**GPU**（Graphics Processing Unit）:可进行绘图运算工作的专用微型处理器，是连接计算机和显示器终端的纽带。
![CPU和GPU](https://s3.ax1x.com/2021/02/08/yUksQe.png)

CPU需要很强的通用性来处理各种不同的数据类型，同时又要逻辑判断又会引入大量的分支跳转和中断的处理。这些都使得CPU的内部结构异常复杂。
而GPU面对的则是类型高度统一的、相互无依赖的大规模数据和不需要被打断的纯净的计算环境。

**CPU/GPU的内部组成**
[![CPU和GPU的内部组成](https://s3.ax1x.com/2021/02/08/yUk7Lj.png)](https://imgchr.com/i/yUk7Lj)
图片来自nVidia CUDA文档。其中绿色的是计算单元，橙红色的是存储单元，橙黄色的是控制单元。

苹果在各种文档中常常提到的硬件加速器实际上就是GPU。
在苹果的系统中，GPU主要负责：
1.手机和mac的渲染问题；
2.因为其卓越的运算能力，通常还用来处理音视频的编解码/视频滤镜/特效处理/人工智能机器学习计算。
## 2.显示器成像的原理（几代显示器的变迁）

### 2.1 随机扫描显示
[![随机扫描显示](https://s3.ax1x.com/2021/02/08/yU8Azj.png)](https://imgchr.com/i/yU8Azj)

采用随机定位的方式控制电子束的移动，一般会带有图形数据输入设备（例如光笔、作图板、跟踪球等），人机交互性好，与计算机对接方便。
优点：电子束随机定位、线条质量好、缓存容量少、人机交互方便；
缺点：不能显示太复杂的画面，而且亮度等级和颜色种类较少。

### 2.2 光栅扫描显示系统组成

[![屏幕扫描](https://s3.ax1x.com/2021/02/08/yU8uwV.png)](https://imgchr.com/i/yU8uwV)
   
光栅扫描显示：电子束通过偏转不断的从左到右从上到下将图片逐行逐点的扫描到显示屏上，通过控制电子束的强弱来控制黑白灰度等级或彩色的图像。整个显示屏面扫描线为2^n行，每一行又分为2^m个点。这样整个显示屏面就分为了2^n * 2^m个小方块，每个方块就是我们说的一个像素。

[![yU8lYF.png](https://s3.ax1x.com/2021/02/08/yU8lYF.png)](https://imgchr.com/i/yU8lYF)
  帧缓存存储器(左)    显示器(右)
 
我们当前使用的显示器都是彩色的，以彩色的为例，在不经过特殊处理的情况下，我们要显示一个图形，系统会自动生成一个跟屏幕像素一致的帧缓冲区，一个像素点通过红、黄、蓝、透明度4个通道来判定当前的颜色，每个通道占8位，所以帧缓冲区的大小可以等于屏幕像素* 4*8。所以我们显示的任何一个图形，都会先绘制到帧缓冲区然后通过视频控制器绘制到屏幕上。

[![高级光栅扫描显示](https://s3.ax1x.com/2021/02/08/yU80YD.png)](https://imgchr.com/i/yU80YD)

CPU处理好图片生成纹理，通过系统总线将数据传输到GPU处理,处理完毕后放到帧缓冲区,然后通过显示器控制器绘制到显示器屏幕上。

## 3.屏幕成像和卡顿问题
### 3.1屏幕撕裂
[![屏幕撕裂1](https://s3.ax1x.com/2021/02/08/yU86OI.png)](https://imgchr.com/i/yU86OI)

[![屏幕撕裂2](https://s3.ax1x.com/2021/02/08/yU8gmt.png)](https://imgchr.com/i/yU8gmt)
在屏幕扫描的过程中是逐行扫描。最理想的结果是，每一次的扫描，在CPU和GPU协作完成之后，得到结果，然后通过显示控制器显示到显示器上。GPU在显示一个比较复杂的图层的时候，需要消耗时间，例如扫描展示上图的时候，在逐行扫描绘制的到一半的时候，帧缓冲区的内容产生了更新，这时就会出现上半部分是上一帧的数据，下半部分是下一帧的数据，出现屏幕撕裂的现象。
[![缓存区展示](https://s3.ax1x.com/2021/02/08/yU8RTf.png)](https://imgchr.com/i/yU8RTf)
### 3.2 垂直同步+双缓存区
#### 双缓存区
用两个帧缓冲区来存储数据，来互相交叠的显示图像
[![双缓存区](https://s3.ax1x.com/2021/02/08/yU87Xn.png)](https://imgchr.com/i/yU87Xn)
#### 垂直同步
当一个帧缓冲区的数据完成一次完整的屏幕绘制后，会发送一个信号告知此次绘制完成，让GPU切换缓冲区，开始进行下一次绘制，这个信号就是我们说的垂直同步信号。就是上面屏幕扫描示意图上的那根黄色的线。

### 3.3 掉帧Jank
[![掉帧Jank](https://s3.ax1x.com/2021/02/08/yU8L7V.png)](https://imgchr.com/i/yU8L7V)
当前屏幕在显示A帧的时候，显示完毕后刷新时发现B帧没有准备好，无法显示B帧，只能继续显示A帧，这个过程叫做掉帧。

### 3.4 三缓存区解决掉帧
为了减少掉帧，可以使用A,B,C 3个缓冲区来处理掉帧。
苹果是使用双缓冲区、

### 3.5 屏幕卡顿的本质（掉帧的本质）
- CPU和GPU的计算时间过长，导致时间超过了一帧显示时间，在发送垂直同步刷新信号时，没有准备好下一帧的数据，导致掉帧了；（不要把图层数设置的过于复杂）
- 双缓存区只是减少屏幕撕裂问题，并不解决掉帧问题。
## 4.渲染流程（屏幕的渲染本质）

[![渲染本质](https://s3.ax1x.com/2021/02/08/yUG9XR.png)](https://imgchr.com/i/yUG9XR)
一个图形的显示是有CPU和GPU协同完成的，CPU负责Application，图形处理、像素化、光栅化都是由GPU来处理。
### Application
得到图源和纹理，例如图片的编解码等，由CPU来进行完成。
### Geometry Processing（图形处理（OpenGL/Metal））
**顶点着色器**：在这里确定图形具体在硬件显示的位置，将坐标转换成屏幕上的坐标
**片元着色器**：计算每一个像素点的颜色值
### Rasterization（光栅化）
将图形的每个像素的颜色计算出来以后，放入帧缓冲区中。
Piexl Processing (像素处理)
将帧缓冲区中的数据，绘制到屏幕上
## 5.CoreAnimation渲染流水线
[![渲染流水线](https://s3.ax1x.com/2021/02/08/yUGF76.png)](https://imgchr.com/i/yUGF76)
在Application中处理事件，处理完毕后提交刚刚的动作，然后在CoreAnimation中解码，解码完毕后将结果放到RunLoop中，在下一次 RunLoop执行到之后，GPU开始进行绘制渲染，渲染完毕后，开始显示。
# 离屏渲染是怎么触发的?
## 1.怎么检测离屏渲染
通过系统的模拟器，可以直接检测离屏渲染
[![检测离屏渲染](https://s3.ax1x.com/2021/02/08/yUG7CD.png)](https://imgchr.com/i/yUG7CD)
iOS 9.0以后，苹果对做了优化，对单个图层设置圆角，不会触发离屏渲染
## 2.触发离屏渲染的两个原因
### 2.1 苹果在显示一些复杂的需求的时候，没有办法一次性完成图层的动作，必须要分步骤完成，因此会产生离屏渲染。

[![遮罩产生离屏渲染的过程](https://s3.ax1x.com/2021/02/08/yUJArn.png)](https://imgchr.com/i/yUJArn)

离屏渲染需要开辟额外的存储空间，同时离屏渲染的合成
### 2.2 主动打开了光栅化（shouldRaserize）
开启光栅化后，会触发离屏渲染，GPU 会强制把图层的渲染结果保存下来，方便下次复用。
当我们确定必须是使用离屏渲染的时候，可以主动开启光栅化，让离屏渲染缓冲区做复用，来减少性能消耗，例如毛玻璃效果。
shouldRasterize 光栅化使用建议：
- 如果layer不能被复用，则没有必要打开光栅化；
- 如果layer不是静态的，需要被频繁修改，比如处于动画之中，那么开启离屏渲染反而影响效率；
- 离屏渲染缓存内容有时间限制，缓存内容100ms内没有被使用，则会被丢弃；
- 离屏渲染空间有限，超过2.5倍屏幕像素大小的话，也会失效，且无法复用。 

### 3.关于圆角在离屏渲染上的原理探索
[![原理探索](https://s3.ax1x.com/2021/02/08/yUJZV0.png)](https://imgchr.com/i/yUJZV0)
设置layer.cornerRadius只会设置backgroundColor和border的圆角，不会设置content的圆角；除非同时设置了layer.masksToBounds为True（对应view中的clipsToBounds属性）。

#### 普通渲染的逻辑
[![普通渲染](https://s3.ax1x.com/2021/02/08/yUJJVx.png)](https://imgchr.com/i/yUJJVx)
当subLayer绘制到屏幕上以后，就会将subLayer从帧缓存区中移除，从而节省空间

#### 离屏渲染的逻辑（圆角）
[![圆角绘制](https://s3.ax1x.com/2021/02/08/yUJYa6.png)](https://imgchr.com/i/yUJYa6)
[![绘制完成](https://s3.ax1x.com/2021/02/08/yUJaGD.png)](https://imgchr.com/i/yUJaGD)
### 4.常见触发离屏渲染的几种情况

- 使用了mask的layer（layer.mask）
- 需要进行裁剪的layer(layer.masksToBounds/view.clipsToBounds)
- 设置了组透明度为YES，并且透明度不为1的layer(layer.allowsGroupOpacity/layer.opacity)
- 添加了阴影的layer(layer.shadow*)
- 采用了光栅化的layer(layer.shouldRasterize)
- 绘制了文字的layer(UILabel,CATextLayer,Core Text 等)

# 开发过程中怎么避免离屏渲染?
## 1.避免圆角离屏渲染的手段
- 不要使用不必要的 mask，可以预处理图片为圆形，通过 Core Graphics手动绘制圆角。
- 使用中间为圆形透明的白色背景视图覆盖。但这种方式不能解决背景色为图片或渐变色的情况。
- 用 UIBezierPath 贝塞尔曲线绘制闭合带圆角的矩形，再将不带圆角的 layer 渲染成图片，添加到贝塞尔矩形中。这种方法效率更高，但是 layer 的布局一旦改变，贝塞尔曲线都需要手动地重新绘制，稍微麻烦。
## 2.项目中使用圆角避免离屏渲染的方案
- 对于单一图层，例如UIImageView只有图片没有背景色、单纯的一个UIView等。直接使用系统提供的layer.cornorRadius；
- 能够使用中间为圆形透明的白色背景视图覆盖方式进行的，优先使用这种方法。
- 使用Core Graphics +贝塞尔曲线绘制圆角的方式，牺牲CPU计算

# 常用控件对于集中设置圆角方式的比较

UIButton （设置背景图）

| 圆角的形式                                        | 产生效果    | 占用内存      |
|----------------------------------------------|---------|-----------|
| layer.cornerRadius + clipsToBounds           | 产生离屏渲染  | 内存占用10M   |
| imageView.layer.cornerRadius + clipsToBounds | 不发生离屏渲染 | 占用内存10.2M |
| Core Graphics绘制图片圆角                          | 不发生离屏渲染 | 占用内存9.9M  |


## 3.项目中对离屏渲染的优化
- 使用AsyncDisplayKit
- 预处理:CoreGraphics进行圆角剪切
- 视频圆角：添加一个图层盖住 （addsubview）
- 图片没有背景颜色 ，放心大胆的用layer.cornerRadius
- 阴影使用shawdowPath绘制
- 需要使用mask遮罩时，将shouldRasterize打开
- 用模糊效果，别用系统的，用开源的手动去渲染

#### Written By 多点-移动运营研发部-童庆