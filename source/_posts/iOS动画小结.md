---
title: iOS动画小结
tags: 动画相关
categories: 技术分享
---
# iOS动画小结
## 前言
### UIView，CALayer的关系及区别
框架及继承
- CALayer 基于 QuartzCore 框架
     QuartzCore中常见的如：CAAnimation(核心动画架构)、CAEmitterLayer(粒子动画Emitter发射器)、      CAEmitterCell(粒子动画)、CATransform3D、CALayer等
- UIView 基于 UIKit 框架
- UIView是直接继承自UIResponder的
- CALayer是直接继承自NSObject
结构及功能
- view负责了用户的交互以及对layer的管理，layer则负责了所有能让用户看到的东西。
- 每个 UIView 内部都有一个 CALayer 在背后提供内容的绘制和显示，并且 UIView 的尺寸样式都由内部的 Layer 所提供。两者都有树状层级结构，layer 内部有SubLayers，View 内部有SubViews.但是 Layer 比 View 多了个anchorPoint。[position与anchorPoint](https://www.cnblogs.com/jgl-blog/p/5735307.html)
- 在 View显示的时候，UIView做为Layer的CALayerDelegate,View 的显示内容取决于内部的 CALayer 的 display。
- layer 内部维护着三个layer tree,分为 model layer tree(模型图层树) 、presentation layer tree（表示图层树） 、render layer tree（渲染图层树）,在做 iOS动画的时候，我们修改动画的属性，在动画的其实是 Layer 的 presentationLayer的属性值,而最终展示在界面上的其实是提供 View的modelLayer。
这三种图层树有什么作用呢？说到有啥作用，就不得不提Core Animation 核心动画了。因为这三个图层在核心动画中才能显示出它们的特点和用处。下面是官方文档的说明：
```
模型图层树 中的对象是应用程序与之交互的对象。此树中的对象是存储任何动画的目标值的模型对象。每当更改图层的属性时，都使用其中一个对象。
表示图层树 中的对象包含任何正在运行的动画的飞行中值。层树对象包含动画的目标值，而表示树中的对象反映屏幕上显示的当前值。您永远不应该修改此树中的对象。相反，您可以使用这些对象来读取当前动画值，也许是为了从这些值开始创建新动画。
渲染图层树 中的对象执行实际动画，并且是Core Animation的私有动画。
```
[![r8sob6.png](https://s3.ax1x.com/2020/12/17/r8sob6.png)](https://imgchr.com/i/r8sob6)
详细内容参考文章：[iOS 详解 CALayer 中的"模型层"和"展示层"](https://www.jianshu.com/p/d09e7929f269)

## 动画详细介绍
#### 一、简介
iOS动画主要是指Core Animation框架。官方使用文档地址为：Core Animation Guide。
Core Animation是iOS和macOS平台上负责图形渲染与动画的基础框架。Core Animation可以作用与动画视图或者其他可视元素，为你完成了动画所需的大部分绘帧工作。你只需要配置少量的动画参数（如开始点的位置和结束点的位置）即可使用Core Animation的动画效果。Core Animation将大部分实际的绘图任务交给了图形硬件来处理，图形硬件会加速图形渲染的速度。这种自动化的图形加速技术让动画拥有更高的帧率并且显示效果更加平滑，不会加重CPU的负担而影响程序的运行速度。
总的来说，从涉及类的形式来看，iOS动画有：基于UIView的仿射形变动画，基于CAAnimation及其子类的动画，基于CG的动画，暂时先总结前两种动画。
#### 二、动画类型介绍
##### 1、UIView动画
UIView动画实质上是对Core Animation的封装，提供简洁的动画接口。
UIView动画可以设置的动画属性有:
- 大小变化(frame)
- 拉伸变化(bounds)
- 中心位置(center)
- 旋转(transform)
- 透明度(alpha)
- 背景颜色(backgroundColor)
- 拉伸内容(contentStretch)

UIview 类方法动画 （iOS13.0后废弃，需要用block动画代替）
###### 1）动画的开始和结束方法
```
[UIView beginAnimations:(nullable NSString *) context:(nullable void *)];
```
第一个参数：动画标识
第二个参数：附加参数，在设置了代理的情况下，此参数将发送到setAnimationWillStartSelector和setAnimationDidStopSelector所指定的方法。大部分情况下，我们设置为nil即可。
1.2 结束动画标记
```
[UIView commitAnimations];
```
###### 2）动画参数的设置方法
```
    //动画持续时间
    [UIView setAnimationDuration:(NSTimeInterval)];
    //动画的代理对象
    [UIView setAnimationDelegate:(nullable id)];
    //设置动画将开始时代理对象执行的SEL
    [UIView setAnimationWillStartSelector:(nullable SEL)];
    //设置动画结束时代理对象执行的SEL
    [UIView setAnimationDidStopSelector:(nullable SEL)];
    //设置动画延迟执行的时间
    [UIView setAnimationDelay:(NSTimeInterval)];
    //设置动画的重复次数
    [UIView setAnimationRepeatCount:(float)];
    //设置动画的曲线
    [UIView setAnimationCurve:(UIViewAnimationCurve)];
    UIViewAnimationCurve的枚举值如下：
    UIViewAnimationCurveEaseInOut,         // 慢进慢出（默认值）
    UIViewAnimationCurveEaseIn,            // 慢进
    UIViewAnimationCurveEaseOut,           // 慢出
    UIViewAnimationCurveLinear             // 匀速
    //设置是否从当前状态开始播放动画
    [UIView setAnimationBeginsFromCurrentState:YES];
    假设上一个动画正在播放，且尚未播放完毕，我们将要进行一个新的动画：
    当为YES时：动画将从上一个动画所在的状态开始播放
    当为NO时：动画将从上一个动画所指定的最终状态开始播放（此时上一个动画马上结束）
    //设置动画是否继续执行相反的动画
    [UIView setAnimationRepeatAutoreverses:(BOOL)];
    //是否禁用动画效果（对象属性依然会被改变，只是没有动画效果）
    [UIView setAnimationsEnabled:(BOOL)];
    //设置视图的过渡效果
    [UIView setAnimationTransition:(UIViewAnimationTransition) forView:(nonnull      UIView *) cache:(BOOL)];
     第一个参数：UIViewAnimationTransition的枚举值如下
         UIViewAnimationTransitionNone,              //不使用动画
         UIViewAnimationTransitionFlipFromLeft,      //从左向右旋转翻页
         UIViewAnimationTransitionFlipFromRight,     //从右向左旋转翻页
         UIViewAnimationTransitionCurlUp,            //从下往上卷曲翻页
         UIViewAnimationTransitionCurlDown,          //从上往下卷曲翻页
     第二个参数：需要过渡效果的View
     第三个参数：是否使用视图缓存，YES：视图在开始和结束时渲染一次；NO：视图在每一帧都渲染
```
##### 2、UIview Block动画
iOS4.0以后，增加了Block动画块，提供更简洁的方式来实现动画。
Block动画方法
##### 1）Block动画方法
###### 1、最简洁的Block动画：包含时间和动
```
 [UIView animateWithDuration:(NSTimeInterval)  //动画持续时间
                  animations:^{
                  //执行的动画
 }];
```
 
###### 2、带有动画完成回调的Block动画
```
 [UIView animateWithDuration:(NSTimeInterval)  //动画持续时间
                  animations:^{
                //执行的动画
 }                completion:^(BOOL finished) {
                //动画执行完毕后的操作
 }];
```
 
###### 3、可设置延迟时间和过渡效果的Block动画
```
 [UIView animateWithDuration:(NSTimeInterval) //动画持续时间
                       delay:(NSTimeInterval) //动画延迟执行的时间
                     options:(UIViewAnimationOptions) //动画的过渡效果
                  animations:^{
                   //执行的动画
 }                completion:^(BOOL finished) {
                   //动画执行完毕后的操作
 }];
 ```

UIViewAnimationOptions的枚举值如下，可组合使用：
```  UIViewAnimationOptionLayoutSubviews            //进行动画时布局子控件
 UIViewAnimationOptionAllowUserInteraction      //进行动画时允许用户交互
 UIViewAnimationOptionBeginFromCurrentState     //从当前状态开始动画
 UIViewAnimationOptionRepeat                    //无限重复执行动画
 UIViewAnimationOptionAutoreverse               //执行动画回路
 UIViewAnimationOptionOverrideInheritedDuration //忽略嵌套动画的执行时间设置
 UIViewAnimationOptionOverrideInheritedCurve    //忽略嵌套动画的曲线设置
 UIViewAnimationOptionAllowAnimatedContent      //转场：进行动画时重绘视图
 UIViewAnimationOptionShowHideTransitionViews   //转场：移除（添加和移除图层的）动画效果
 UIViewAnimationOptionOverrideInheritedOptions  //不继承父动画设置
 
 UIViewAnimationOptionCurveEaseInOut            //时间曲线，慢进慢出（默认值）
 UIViewAnimationOptionCurveEaseIn               //时间曲线，慢进
 UIViewAnimationOptionCurveEaseOut              //时间曲线，慢出
 UIViewAnimationOptionCurveLinear               //时间曲线，匀速
 
 UIViewAnimationOptionTransitionNone            //转场，不使用动画
 UIViewAnimationOptionTransitionFlipFromLeft    //转场，从左向右旋转翻页
 UIViewAnimationOptionTransitionFlipFromRight   //转场，从右向左旋转翻页
 UIViewAnimationOptionTransitionCurlUp          //转场，下往上卷曲翻页
 UIViewAnimationOptionTransitionCurlDown        //转场，从上往下卷曲翻页
 UIViewAnimationOptionTransitionCrossDissolve   //转场，交叉消失和出现
 UIViewAnimationOptionTransitionFlipFromTop     //转场，从上向下旋转翻页
 UIViewAnimationOptionTransitionFlipFromBottom  //转场，从下向上旋转翻页
```
###### 4、Spring动画
iOS7.0后新增Spring动画（iOS系统动画大部分采用Spring Animation，适用于所有可被添加动画效果的属性）
```
 [UIView animateWithDuration:(NSTimeInterval)//动画持续时间
                       delay:(NSTimeInterval)//动画延迟执行的时间
      usingSpringWithDamping:(CGFloat)//震动效果，范围0~1，数值越小震动效果越明显
       initialSpringVelocity:(CGFloat)//初始速度，数值越大初始速度越快
                     options:(UIViewAnimationOptions)//动画的过渡效果
                  animations:^{
                     //执行的动画
 }
                  completion:^(BOOL finished) {
                     //动画执行完毕后的操作
 }];
```
 
5、Keyframes动画
```
 [UIView animateKeyframesWithDuration:(NSTimeInterval)//动画持续时间
                                delay:(NSTimeInterval)//动画延迟执行的时间
                              options:(UIViewKeyframeAnimationOptions)//动画的过渡效果
                           animations:^{
                         //执行的关键帧动画
 }
                           completion:^(BOOL finished) {
                         //动画执行完毕后的操作
 }];
```

UIViewKeyframeAnimationOptions的枚举值如下，可组合使用：
```
UIViewAnimationOptionLayoutSubviews           //进行动画时布局子控件
UIViewAnimationOptionAllowUserInteraction     //进行动画时允许用户交互
UIViewAnimationOptionBeginFromCurrentState    //从当前状态开始动画
UIViewAnimationOptionRepeat                   //无限重复执行动画
UIViewAnimationOptionAutoreverse              //执行动画回路
UIViewAnimationOptionOverrideInheritedDuration //忽略嵌套动画的执行时间设置
UIViewAnimationOptionOverrideInheritedOptions //不继承父动画设置

UIViewKeyframeAnimationOptionCalculationModeLinear     //运算模式 :连续
UIViewKeyframeAnimationOptionCalculationModeDiscrete   //运算模式 :离散
UIViewKeyframeAnimationOptionCalculationModePaced      //运算模式 :均匀执行
UIViewKeyframeAnimationOptionCalculationModeCubic      //运算模式 :平滑
UIViewKeyframeAnimationOptionCalculationModeCubicPaced //运算模式 :平滑均匀
```
各种运算模式的直观比较如下图：
[![r8gesA.png](https://s3.ax1x.com/2020/12/17/r8gesA.png)](https://imgchr.com/i/r8gesA)


增加关键帧的方法：
```
 [UIView addKeyframeWithRelativeStartTime:(double)//动画开始的时间（占总时间的比例）
                         relativeDuration:(double) //动画持续时间（占总时间的比例）
                               animations:^{
                             //执行的动画
 }];
```

#### 6、转场动画
##### 6.1 从旧视图转到新视图的动画效果
```
 [UIView transitionFromView:(nonnull UIView *)
                     toView:(nonnull UIView *)
                   duration:(NSTimeInterval)
                    options:(UIViewAnimationOptions)
                 completion:^(BOOL finished) {
                     //动画执行完毕后的操作
 }];
```
在该动画过程中，fromView 会从父视图中移除，并将 toView 添加到父视图中，注意转场动画的作用对象是父视图（过渡效果体现在父视图上）。
调用该方法相当于执行下面两句代码：
```
[fromView.superview addSubview:toView];
[fromView removeFromSuperview];
```
##### 6.2 单个视图的过渡效果
```
 [UIView transitionWithView:(nonnull UIView *)
                   duration:(NSTimeInterval)
                    options:(UIViewAnimationOptions)
                 animations:^{
                 //执行的动画
 }
                 completion:^(BOOL finished) {
                 //动画执行完毕后的操作
 }];
```
### 3、核心动画
#### 1、简介
Core Animation(核心动画)是一组功能强大、效果华丽的动画API，无论在iOS系统或者在开发App的过程中，都有大量应用。
核心动画所在的位置如下图所示：
[![r828k6.png](https://s3.ax1x.com/2020/12/17/r828k6.png)](https://imgchr.com/i/r828k6)
可以看到，核心动画位于UIKit的下一层，相比UIView动画，它可以实现更复杂的动画效果。
核心动画作用在CALayer（Core animation layer）上，CALayer从概念上类似UIView，我们可以将UIView看成是一种特殊的CALayer（可以响应事件）。
实际上，每一个view都有其对应的layer，这个layer是root layer：
```
    @property(nonatomic,readonly,strong)                CALayer  *layer;
```

给view加上动画，本质上是对其layer进行操作，layer包含了各种支持动画的属性，动画则包含了属性变化的值、变化的速度、变化的时间等等，两者结合产生动画的过程。
核心动画和UIView动画的对比：UIView动画可以看成是对核心动画的封装，和UIView动画不同的是，通过核心动画改变layer的状态（比如position），动画执行完毕后实际上是没有改变的（表面上看起来已改变）。
##### 2、核心动画优点：
1）性能强大，使用硬件加速，可以同时向多个图层添加不同的动画效果
2）接口易用，只需要少量的代码就可以实现复杂的动画效果。
3）运行在后台线程中，在动画过程中可以响应交互事件（UIView动画默认动画过程中不响应交互事件）。
##### 3、核心动画类的层次结构：
[![r82B7t.png](https://s3.ax1x.com/2020/12/17/r82B7t.png)](https://imgchr.com/i/r82B7t)

CAAnimation是所有动画对象的父类，实现CAMediaTiming协议，负责控制动画的时间、速度和时间曲线等等，是一个抽象类，不能直接使用。
CAPropertyAnimation ：是CAAnimation的子类，它支持动画地显示图层的keyPath，一般不直接使用。
iOS9.0之后新增CASpringAnimation类，它实现弹簧效果的动画，是CABasicAnimation的子类。
核心动画类中可以直接使用的类有：
CABasicAnimation
CAKeyframeAnimation
CATransition
CAAnimationGroup
CASpringAnimation
##### 4、详细介绍
###### 1）CAAnimation
CAAnimation是所有动画对象的父类，负责控制动画的持续时间和速度，是个抽象类，不能直接使用，应该使用它具体的子类
属性解读

|  属性   | 描述  |
|  ----  | ----  |
|duration  | 动画的持续时间 |
| repeatCount  | 重复次数，无限循环可以设置HUGE_VALF或者MAXFLOAT |
|repeatDuration  | 重复时间 |
|beginTime  | 可以用来设置动画延迟执行时间，若想延迟2s，就设置CACurrentMediaTime()+2，CACurrentMediaTime()为图层的 当前时间 |
|autoreverses  | 动画自动逆向执行，默认为No |
|speed  | 动画执行速度 |
|removedOnCompletion  | 动画的持续时间 |
|duration  | 默认为YES，代表动画执行完毕后就从图层上移除，图形会恢复到动画执行前的状态。如果想让图层保持显示动画执行后的状态，那就设置为NO，不过还要设置fillMode为kCAFillModeForwards |
|fillMode  | 决定当前对象在非active时间段的行为。比如动画开始之前或者动画结束之后 |
|timingFunction  | 速度控制函数，控制动画运行的节奏，例：anima.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseIn] |
|delegate  | 动画代理，监听动画的开始和结束 |


- fillMode属性值（要想fillMode有效，最好设置removedOnCompletion = NO）
kCAFillModeRemoved这个是默认值，也就是说当动画开始前和动画结束后，动画对layer都没有影响，动画结束后，layer会恢复到之前的状态
CAFillModeForwards 当动画结束后，layer会一直保持着动画最后的状态
kCAFillModeBackwards 在动画开始前，只需要将动画加入了一个layer，layer便立即进入动画的初始状态并等待动画开始。
kCAFillModeBoth这个其实就是上面两个的合成.动画加入后开始之前，layer便处于动画初始状态，动画结束后layer保持动画最后的状态

- 速度控制函数(CAMediaTimingFunction)
`kCAMediaTimingFunctionLinear（线性）`：匀速，给你一个相对静态的感觉
`kCAMediaTimingFunctionEaseIn（渐进）`：动画缓慢进入，然后加速离开
`kCAMediaTimingFunctionEaseOut（渐出）`：动画全速进入，然后减速的到达目的地
`kCAMediaTimingFunctionEaseInEaseOut`（渐进渐出）：动画缓慢的进入，中间加速，然后减速的到达目的地。这个是默认的动画行为。 
- delegate：动画代理
代理方法如下：
```
- (void)animationDidStart:(CAAnimation *)anim;  //动画开始
- (void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag; //动画结束
```
#### 核心动画类的核心方法：
##### 1.初始化CAAnimation对象一般使用animation方法生成实例

 ```
 + (instancetype)animation;
如果是CAPropertyAnimation的子类，还可以通过animationWithKeyPath生成实例
 + (instancetype)animationWithKeyPath:(nullable NSString *)path;
 ```
##### 2.设置动画的相关属性
设置动画的执行时间，执行曲线，keyPath的目标值，代理等等
##### 3.动画的添加和移除
调用CALayer的addAnimation:forKey:方法将动画添加到CALayer中，这样动画就开始执行了
```
 - (void)addAnimation:(CAAnimation *)anim forKey:(nullable NSString *)key;
调用CALayer的removeAnimation方法停止CALayer中的动画
 - (void)removeAnimationForKey:(NSString *)key;
 - (void)removeAllAnimations;
```
##### ４.核心动画类的常用属性
keyPath：可以指定keyPath为CALayer的属性值，并对它的值进行修改，以达到对应的动画效果，需要注意的是部分属性值是不支持动画效果的。
以下是具有动画效果的keyPath：
```
     //CATransform3D Key Paths : (example)transform.rotation.z
     //rotation.x
     //rotation.y
     //rotation.z
     //rotation 旋轉
     //scale.x
     //scale.y
     //scale.z
     //scale 缩放
     //translation.x
     //translation.y
     //translation.z
     //translation 平移
     
     //CGPoint Key Paths : (example)position.x
     //x
     //y
     
     //CGRect Key Paths : (example)bounds.size.width
     //origin.x
     //origin.y
     //origin
     //size.width
     //size.height
     //size
     
     //opacity
     //backgroundColor
     //cornerRadius 
     //borderWidth
     //contents 
     
     //Shadow Key Path:
     //shadowColor 
     //shadowOffset 
     //shadowOpacity 
     //shadowRadius 
```
##### 2)   CAPropertyAnimation
CAPropertyAnimation也是一个抽象类，自身并不能对layer进行动画操作，需要其子类
CABasicAnimation和CAKeyframeAnimation来实现动画操作。
属性解读：

|  属性   | 描述  |
|  ----  | ----  |
| keyPath  |指定接收层动画的动画类型|
| cumulative  | 下一次动画执行是否接着刚才的动画，默认为false |
| additive  |如何处理多个动画在同一时间段执行的结果，若为true，同一时间段的动画合成为一个动画，默认为false。（使用 CAKeyframeAnimation 时必须将该属性指定为 true ，否则不会出现期待的结果）|


##### 3）CABasicAnimation —— 基本动画
基本动画，是CAPropertyAnimation的子类

|  属性   | 描述  |
|  ----  | ----  |
| fromValue  | NSValue类型，keyPath相应属性的初始值 |
| toValue  | NSValue类型， keyPath相应属性的结束值 |

CABasicAnimation可以设定keyPath的起点，终点的值，动画会沿着设定点进行移动，CABasicAnimation可以看成是只有两个关键点的特殊的CAKeyFrameAnimation。
动画过程说明：
随着动画的进行，在长度为duration的持续时间内，keyPath相应属性的值从fromValue渐渐地变为toValue
keyPath内容是CALayer的可动画Animatable属性
如果fillMode==kCAFillModeForwards同时removedOnComletion=NO，那么在动画执行完毕后，图层会保持显示动画执行后的状态。但在实质上，图层的属性值还是动画执行前的初始值，并没有真正被改变。

##### 4）CAKeyframeAnimation —— 关键帧动画
关键帧动画，也是CAPropertyAnimation的子类，与CABasicAnimation的区别是：
`CABasicAnimation`:只能从一个数值（fromValue）变到另一个数值（toValue）
`CAKeyframeAnimation`:会使用一个NSArray保存这些数值。
可以设定keyPath起点、中间关键点（不止一个）、终点的值，每一帧所对应的时间，动画会沿着设定点进行移动。

|  属性   | 描述  |
|  ----  | ----  |
| values  | NSArray类型，里面的元素（NSValue）称为“关键帧”(keyframe)。动画对象会在指定的时间（duration）内，依次显示values数组中的每一个关键帧 |
| path  | 可以设置一个CGPathRef、CGMutablePathRef，让图层按照路径轨迹移动。path只对CALayer的anchorPoint和position起作用。如果设置了path，那么values将被忽略 |
| keyTimes  | NSArray类型，可以为对应的关键帧指定对应的时间点，其取值范围为0到1.0，keyTimes中的每一个时间值都对应values中的每一帧。如果没有设置keyTimes，各个关键帧的时间是平分的|


##### 5）CAAnimationGroup —— 动画组
动画组，是CAAnimation的子类，可以保存一组动画对象，将CAAnimationGroup对象加入layer层后，组中所有动画对象可以同时并发运行

|  属性   | 描述  |
|  ----  | ----  |
| animations  | 用来保存一组动画对象的NSArray。默认情况下，一组动画对象是同时运行的，也可以通过设置各个动画对象的beginTime属性来更改动画的开始时间|

##### 6）CASpringAnimation —— 弹性动画
CASpringAnimation是iOS9.0新加入动画类型，是CABasicAnimation的子类，用于实现弹簧动画。

|  属性   | 描述  |
|  ----  | ----  |
| mass  | 质量（影响弹簧的惯性，质量越大，弹簧惯性越大，运动的幅度越大）|
|  stiffness  | 弹性系数（弹性系数越大，弹簧的运动越快）  |
|  damping  | 阻尼系数（阻尼系数越大，弹簧的停止越快）  |
|  initialVelocity  | 初始速率（弹簧动画的初始速度大小，弹簧运动的初始方向与初始速率的正负一致，若初始速率为0，表示忽略该属性） |
|  settlingDuration  | 结算时间（根据动画参数估算弹簧开始运动到停止的时间，动画设置的时间最好根据此时间来设置） |


CASpringAnimation和UIView的SpringAnimation对比：
1. CASpringAnimation 可以设置更多影响弹簧动画效果的属性，可以实现更复杂的弹簧动画效果，且可以和其他动画组合。
2. UIView的SpringAnimation实际上就是通过CASpringAnimation来实现。

##### 7）CATransition

最后讲一下事务，在核心动画里面存在事务（CATransaction）这样一个概念，它负责协调多个动画原子更新显示操作。
简单来说事务是核心动画里面的一个基本的单元，动画的产生必然伴随着layer的Animatable属性的变化，而layer属性的变化必须属于某一个事务。
因此，核心动画依赖事务。

- 事务的作用：保证一个或多个layer的一个或多个属性变化同时进行
- 事务分为隐式和显式：
1.隐式：没有明显调用事务的方法，由系统自动生成事务。比如直接设置一个layer的position属性，则会在当前线程自动生成一个事务，并在下一个runLoop中自动commit事务。
2.显式：明显调用事务的方法（[CATransaction begin]和[CATransaction commit]）

CA事务的可设置属性（会覆盖隐式动画的设置）：
```
    animationDuration：动画时间
    animationTimingFunction：动画时间曲线
    disableActions：是否关闭动画
    completionBlock：动画执行完毕的回调
```

事务支持嵌套使用：当最外层的事务commit后动画才会开始。

注意：只有非root layer才有隐式动画，根layer（rootLayer）类似于UIView中的一个layer层，非RootLayer层就是自己创建的layer层。
可以设置隐式动画关闭。当在设置隐式动画的动画效果时候，需要在提交的commit方法前面实现，否则，没有动画效果。如上面例子中的，在设置layer的position属性时候，没有提交上去，因此，效果图中的layer有一个瞬移的效果。

注意：如果父视图中的两个子视图互相切换，转场动画应加给父视图！

|  属性   | 描述  |
|  ----  | ----  |
| type  | 动画过渡类型 |
| subtype  | 动画过渡方向 |
| startProgress  | 动画起点(在整体动画的百分比) |
| endProgress  | 动画终点(在整体动画的百分比) |

动画过渡类型:
type的enum值如下：
```

    kCATransitionFade 渐变
    kCATransitionMoveIn 覆盖
    kCATransitionPush 推出
    kCATransitionReveal 揭开
```

动画过渡方向
subtype的enum值如下：
```
    kCATransitionFromRight 从右边
    kCATransitionFromLeft 从左边
    kCATransitionFromTop 从顶部
    kCATransitionFromBottom 从底部
```



还有一些私有动画类型，效果很炫酷，不过不推荐使用。私有动画类型的值有："cube"、"suckEffect"、"oglFlip"、 "rippleEffect"、"pageCurl"、"pageUnCurl"等等。
#### Written By 多点-移动运营研发部-李涛
