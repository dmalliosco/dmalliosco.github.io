---
title: iOS客户端布局浅谈
tags: 布局漫谈
categories: 技术分享
---
## iOS客户端布局浅谈
### 一、前言
日常开发中，UI搭建、调试会占用我们大部分的时间，以至于移动端开发经常会被调侃为搭界面的。提高UI布局技术可以提高开发效率，把更多的时间放在优化、逻辑方面，而不是被界面业务绑死。
### 二、UI布局约束概览
- nib：nib是NeXT interface builder的英文缩写，以二进制的形式存储界面信息，是IB3.0以前的文件格式。
- xib：xib是xml interface builder的英文缩写，是IB3.0之后苹果公司推出的新一代，以xml格式存储界面信息，在最终执行前，xib文件会被编译为nib文件。
- storyboard：是苹果最新推出的用于在界面开发中替代xib文件的一种新技术。本质上是一个xml文件的集中管理区，不但可以描述xib单个界面的结构，还可以描述界面之间的跳转及依赖关系。主要是靠手拖，感觉像积木玩具。
- frame：等效于代码版的storyboard，但更灵活。目前比较常用。多众多机型尺寸还是有点棘手。计算比较头疼。（据统计性能最好的布局方式！！！）
- AutoLayout：自动布局（AutoLayout）是iOS6发布的界面布局技术，该算法的主要思想是：将基于约束系统的布局规则（本质上是表示视图布局关系的线性方程组）转化为表示规则的视图几何参数。实际上AutoLayout算法本身并非有苹果发明，只是苹果用Objective-C去实现了该算法，方便iOS开发者使用。AutoLayout有多种使用方式，如①可视化工具：Xcode的Interface Builder，②纯代码：以[Masonry](https://github.com/SnapKit/Masonry)为代表。更多内容见自动布局 [Auto Layout (原理篇)](https://www.jianshu.com/p/3a872a0bfe11)。
- FlexBox：弹性布局（Flexible Box）。对，就是目前Web端最流行的布局方式（以前是盒子模型），现在APP上也能使用。此方案就扩展出很多技术，如[Yoga](https://github.com/facebook/yoga)（最牛逼的代表，Facebook出品，衍生出很多上层方案，如跨平台的[ReactNative](https://github.com/facebook/react-native)、Android的[Litho](https://tech.meituan.com/2019/03/14/litho-use-and-principle-analysis.html)、iOS的Yogakit）、[FlexboxLayout](https://github.com/google/flexbox-layout)（Android代表，Google出品）、[FlexLib](https://github.com/zhenglibao/FlexLib)、[FLEX](https://github.com/FLEXTool/FLEX)等。
- swiftUI：[官网](https://developer.apple.com/xcode/swiftui/)，苹果官方推荐。更多内容见苹果发布全新 [SwiftUI 框架：一次编码，五端通用]((https://www.infoq.cn/article/Puii*HdQWCDjPzvTNcKq))。
三、性能概览
[![iOS布局耗时测试.png](https://s3.ax1x.com/2020/12/16/rlDxNd.png)](https://imgchr.com/i/rlDxNd)
四、Masonry
[![rlrgPA.png](https://s3.ax1x.com/2020/12/16/rlrgPA.png)](https://imgchr.com/i/rlrgPA)
[![rlrxqU.png](https://s3.ax1x.com/2020/12/16/rlrxqU.png)](https://imgchr.com/i/rlrxqU)

1、使用前：AutoLayout关于更新的几个方法的区别
- setNeedsLayout：告知页面需要更新，但是不会立刻开始更新。执行后会立刻调用layoutSubviews。
- layoutIfNeeded：告知页面布局立刻更新。所以一般都会和setNeedsLayout一起使用。如果希望立刻生成新的frame需要调用此方法，利用这点一般布局动画可以在更新布局后直接使用这个方法让动画生效。
- layoutSubviews：系统重写布局。
- setNeedsUpdateConstraints：告知需要更新约束，但是不会立刻开始。
- updateConstraintsIfNeeded：告知立刻更新约束。
- updateConstraints：系统更新约束（推荐将更新约束写在这里）。
2、基本使用
- mas_makeConstraints:添加约束。
- mas_updateConstraints：更新约束、亦可添加新约束。
- mas_remakeConstraints：重置之前的约束。
3、常用宏定义
Masonry使用链式方式编程，有定义一些宏来方便开发，如下：

```
#define offset(...)                      mas_offset(__VA_ARGS__)#define equalTo(...)                     mas_equalTo(__VA_ARGS__)#define greaterThanOrEqualTo(...)        mas_greaterThanOrEqualTo(__VA_ARGS__)#define lessThanOrEqualTo(...)           mas_lessThanOrEqualTo(__VA_ARGS__)
```
如：
// make.left 以下两种写法等价

```
make.left.equalTo(superView.mas_left).offset(10);
make.left.mas_equalTo(10);
make.left.mas_equalTo(superView.mas_left).mas_offset(10);
```
4、约束属性的关系
约束属性有三种关系，分别是等于，大于，小于。
- 等于（equivalent to NSLayoutRelationEqual）
```
//.equalTo
// view.width = 60
make.width.mas_equalTo(60);
```

- 小于（equivalent to NSLayoutRelationLessThanOrEqual）
```
//.lessThanOrEqualTo
// view.width <= 120
make.width.lessThanOrEqualTo(120);
```

- 大于（equivalent to NSLayoutRelationGreaterThanOrEqual）
```
//.greaterThanOrEqualTo 
// view.width >= 60
make.width.lessThanOrEqualTo(60);
```
5、常见写法
对于自动布局我们只需要设置好view的position，包括X轴和Y轴上的位置，以及view的长宽即可。
Masonry除了设置单个约束以外，还提供了很多方便的复合约束设置方式。以下分别介绍单个约束属性设置方式和复合约束设置方式。
6、长度和宽度
```
// make width = 60
make.width.mas_equalTo(60);
// make height = 60
make.height.mas_equalTo(60);
// make height = 60, width = 60
make.size.equalTo(CGSizeMake(60, 60));
// make view1.height = view2.height view1.width = view2.width
make.size.mas_equalTo(view2);
// make width and height greater than or equal to view2
make.size.greaterThanOrEqualTo(view2);
// make width = superview.width + 100, height = superview.height - 50
make.size.equalTo(superview).sizeOffset(CGSizeMake(100, -50));
```
7、边距对齐（位置设置）
```
make.left.mas_equalTo(superView.mas_left).mas_offset(10);
make.left.mas_greaterThanOrEqualTo(10);
```
约束的链式写法中，不包含其他相对的view时，默认为其superview view.left 等于 label.lefet，以下三种写法等价：
```make.left.mas_equalTo(100); 
make.left.mas_equalTo(view.superview.mas_left).offset(10);
make.left.mas_equalTo(view.superview).offset(10);
```
view.left 大于等于 label.left，以下两种写法等同：
```
make.left.greaterThanOrEqualTo(label);
make.left.greaterThanOrEqualTo(label.mas_left);
```
上下左右四个边缘等于父视图的边缘，也就是top, left, bottom, right equal view2，以下三种写法等同（复合约束写法）：
```
make.edges.equalTo(superview);
make.edges.equalTo(superview).insets(UIEdgeInsetsMake(0, 0, 0, 0))
make.left.right.and.bottom.equalTo(superview);
```
PS：如果top, left, bottom, right的边距各自不同：
```
// make top = superview.top + 5, left = superview.left + 10,
// bottom = superview.bottom - 15, right = superview.right - 20
make.edges.equalTo(superview).insets(UIEdgeInsetsMake(5, 10, 15, 20))
```
8、中心对齐（位置设置）
X轴上中心和superview的X轴上中心对齐。
```
make.centerX.equalTo(superview.mas_centerX).offet(0)
make.centerX.equalTo(superview.mas_centerX) 
make.centerX.equalTo(superview)
```
X轴和Y轴中心对齐
```
make.center.equalTo(superview);
make.center.equalTo(superview).centerOffset(CGPointMake(0, 0));
```
PS：如果X轴和Y轴上的中心对齐各自不同：
```// make centerX = superview.centerX - 5, centerY = superview.centerY + 10
make.center.equalTo(superview).centerOffset(CGPointMake(-5, 10));
```
9、约束数字值
Masonry允许将宽度和高度设置为常量值。如果要将视图设置一个最小和最大宽度时，可以在block中同时设定：
//width >= 200 && width <= 400
```
make.width.greaterThanOrEqualTo(@200);
make.width.lessThanOrEqualTo(@400);
```
Masonry不允许把边距属性（left，top，right，bottom，centerX，centerY）的约束设置为常量值，如果设置了，会默认这些属性是相对父视图设置的。
```//creates view.left = view.superview.left + 10
make.left.equalTo(@10)
```
```
//creates view.left = view.superview.left + 10
make.left.lessThanOrEqualTo(@10)
```
// 或者使用mas_equalTo这种时：
```
make.top.mas_equalTo(42);
make.height.mas_equalTo(20);
make.size.mas_equalTo(CGSizeMake(50, 100));
make.edges.mas_equalTo(UIEdgeInsetsMake(10, 0, 10, 0));
make.left.mas_equalTo(view).mas_offset(UIEdgeInsetsMake(10, 0, 10, 0));
```
10、优先级
自定义优先级
```
// .priority allows you to specify an exact priority  
make.top.equalTo(label.mas_left).with.priority(600);
```
高优先级
```
// .priorityHigh equivalent to **UILayoutPriorityDefaultHigh**  
make.left.equalTo(label.mas_left).with.priorityHigh();
```
中等优先级
```
// .priorityMedium is half way between high and low  
make.left.equalTo(label.mas_left).with.priorityMedium();
```
低优先级
```
// .priorityLow equivalent to **UILayoutPriorityDefaultLow**
make.left.greaterThanOrEqualTo(label.mas_left).with.priorityLow();
```
11、equalTo 和 mas_equalTo的区别
```
#define mas_equalTo(...) equalTo(MASBoxValue((__VA_ARGS__)))
#define mas_greaterThanOrEqualTo(...) greaterThanOrEqualTo(MASBoxValue((__VA_ARGS__)))
#define mas_lessThanOrEqualTo(...) lessThanOrEqualTo(MASBoxValue((__VA_ARGS__)))
#define mas_offset(...) valueOffset(MASBoxValue((__VA_ARGS__)))
```
得出结论：mas_equalTo只是对其参数进行了一个BOX(装箱) 操作，目前支持的类型：数值类型（NSNumber）、 点（CGPoint）、大小（CGSize）、边距（UIEdgeInsets），而equalTo：这个方法不会对参数进行包装。
12、多个视图（大于2）有相同规则的布局方法
```
/** NSArray+MASAdditions.h

 *  多个控件固定间隔的等间隔排列，变化的是控件的长度或者宽度值
 *
 *  @param axisType        轴线方向
 *  @param fixedSpacing    间隔大小
 *  @param leadSpacing     头部间隔
 *  @param tailSpacing     尾部间隔
 */- (void)mas_distributeViewsAlongAxis:(MASAxisType)axisType 
                    withFixedSpacing:(CGFloat)fixedSpacing l
                          eadSpacing:(CGFloat)leadSpacing 
                         tailSpacing:(CGFloat)tailSpacing;
/**
 *  多个固定大小的控件的等间隔排列,变化的是间隔的空隙
 *
 *  @param axisType        轴线方向
 *  @param fixedItemLength 每个控件的固定长度或者宽度值
 *  @param leadSpacing     头部间隔
 *  @param tailSpacing     尾部间隔
 */- (void)mas_distributeViewsAlongAxis:(MASAxisType)axisType 
                 withFixedItemLength:(CGFloat)fixedItemLength 
                         leadSpacing:(CGFloat)leadSpacing 
                         tailSpacing:(CGFloat)tailSpacing;
```
eg1：水平方向排列、固定控件间隔、控件长度不定
```
- (void)test_masonry_horizontal_fixSpace {
    // 实现masonry水平固定间隔方法
    [self.masonryViewArray mas_distributeViewsAlongAxis:MASAxisTypeHorizontal withFixedSpacing:30 leadSpacing:10 tailSpacing:10];   
    // 设置array的垂直方向的约束
    [self.masonryViewArray mas_makeConstraints:^(MASConstraintMaker *make) {   
        make.top.equalTo(150);
        make.height.equalTo(80);
    }];
}
```
[![rlWKn1.png](https://s3.ax1x.com/2020/12/16/rlWKn1.png)](https://imgchr.com/i/rlWKn1)

eg2：水平方向排列、固定控件长度、控件间隔不定
```
- (void)test_masonry_horizontal_fixItemWidth {
    // 实现masonry水平固定控件宽度方法
    [self.masonryViewArray mas_distributeViewsAlongAxis:MASAxisTypeHorizontal withFixedItemLength:80 leadSpacing:10 tailSpacing:10];  
 // 设置array的垂直方向的约束
[self.maViewArray mas_makeConstraints:^(MASConstraintMaker *make) {      
        make.top.equalTo(150);
        make.height.equalTo(80);
    }];
}
```
[![rlWY1H.png](https://s3.ax1x.com/2020/12/16/rlWY1H.png)](https://imgchr.com/i/rlWY1H)
eg3：垂直方向排列、固定控件间隔、控件高度不定

```
- (void)test_masonry_vertical_fixSpace {
    // 实现masonry垂直固定控件高度方法
    [self.masonryViewArray mas_distributeViewsAlongAxis:MASAxisTypeVertical withFixedSpacing:30 leadSpacing:10 tailSpacing:10];
// 设置array的水平方向的约束
[self.masonryViewArray mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.equalTo(150);
        make.width.equalTo(80);
    }];
}
```
[![rlWanI.png](https://s3.ax1x.com/2020/12/16/rlWanI.png)](https://imgchr.com/i/rlWanI)
eg4：垂直方向排列、固定控件高度、控件间隔不定
```
- (void)test_masonry_vertical_fixItemWidth {
    // 实现masonry垂直方向固定控件高度方法
    [self.masonryViewArray mas_distributeViewsAlongAxis:MASAxisTypeVertical withFixedItemLength:80 leadSpacing:10 tailSpacing:10];
    // 设置array的水平方向的约束
    [self.masonryViewArray mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.equalTo(150);
        make.width.equalTo(80);
    }];
}
```
[![rlWBAf.png](https://s3.ax1x.com/2020/12/16/rlWBAf.png)](https://imgchr.com/i/rlWBAf)
13、Content Compression Resistance和Content Hugging
[![rlWccj.png](https://s3.ax1x.com/2020/12/16/rlWccj.png)](https://imgchr.com/i/rlWccj)
// label1: 位于左上角
```
[_label1 mas_makeConstraints:^(MASConstraintMaker *make) {
    make.top.equalTo(_contentView1.mas_top).with.offset(5);
    make.left.equalTo(_contentView1.mas_left).with.offset(2);
    make.height.equalTo(@40);
}];
```
// label2: 位于右上角
```
[_label2 mas_makeConstraints:^(MASConstraintMaker *make) {
    //左边贴着label1，间隔2
    make.left.equalTo(_label1.mas_right).with.offset(2);
    //上边贴着父view，间隔5
    make.top.equalTo(_contentView1.mas_top).with.offset(5);
    //右边的间隔保持大于等于2，注意是lessThanOrEqual
    //这里的“lessThanOrEqualTo”放在从左往右的X轴上考虑会更好理解。
    //即：label2的右边界的X坐标值“小于等于”containView的右边界的X坐标值。
    make.right.lessThanOrEqualTo(_contentView1.mas_right).with.offset(-2);
    make.height.equalTo(@40);
}];
```

//设置label1的content hugging 为1000
```
[_label1 setContentHuggingPriority:UILayoutPriorityRequired
                           forAxis:UILayoutConstraintAxisHorizontal];

//设置label1的content compression 为1000
[_label1 setContentCompressionResistancePriority:UILayoutPriorityRequired                                       forAxis:UILayoutConstraintAxisHorizontal];

//设置右边的label2的content hugging 为1000
[_label2 setContentHuggingPriority:UILayoutPriorityRequired
                           forAxis:UILayoutConstraintAxisHorizontal];

//设置右边的label2的content compression 为250
[_label2 setContentCompressionResistancePriority:UILayoutPriorityDefaultLow                                         forAxis:UILayoutConstraintAxisHorizontal];
```
五、UIStackVIew
```
typedef NS_ENUM(NSInteger, UILayoutConstraintAxis) {
UILayoutConstraintAxisHorizontal = 0,//水平
UILayoutConstraintAxisVertical = 1//垂直
};

typedef NS_ENUM(NSInteger, UIStackViewAlignment) {
UIStackViewAlignmentFill,//子视图填充StackView
UIStackViewAlignmentLeading,//子视图左对齐（axis为垂直方向而言）
UIStackViewAlignmentTop = UIStackViewAlignmentLeading,//子视图顶部对齐（axis为水平方向而言）
UIStackViewAlignmentFirstBaseline, // 按照第一个子视图的文字的第一行对齐，同时保证高度最大的子视图底部对齐（只在axis为水平方向有效）
UIStackViewAlignmentCenter,//子视图居中对齐
UIStackViewAlignmentTrailing,//子视图右对齐(axis为垂直方向而言）
UIStackViewAlignmentBottom = UIStackViewAlignmentTrailing,//子视图底部对齐（axis为水平方向而言）
UIStackViewAlignmentLastBaseline, // 按照最后一个子视图的文字的最后一行对齐，同时保证高度最大的子视图顶部对齐（只在axis为水平方向有效）
} NS_ENUM_AVAILABLE_IOS(9_0);

typedef NS_ENUM(NSInteger, UIStackViewDistribution) {
UIStackViewDistributionFill = 0, 
UIStackViewDistributionFillEqually, 
UIStackViewDistributionFillProportionally,
UIStackViewDistributionEqualSpacing, 
UIStackViewDistributionEqualCentering,} NS_ENUM_AVAILABLE_IOS(9_0);
```
#### Written By 多点-移动运营研发部-贺波
