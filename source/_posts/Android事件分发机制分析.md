---
title: Android事件分发机制分析
tags: 底层原理
categories: 技术分享
---

# 事件分发机制分析

Android事件分发是一个老生常谈的知识点。日常开发和求职过程中，都会碰到Android事件分发的问题。

Android的控件分为两类，ViewGoup和View。ViewGroup是控件的容器，可以包含多个子控件。View是控件的最小单位，它不能包裹其它的View。Android的ViewGroup对应的数据结构是树。

网上的事件分发的文章大多数是用线性的思维去分析控件树的事件遍历，我深以为不妥，经常让读者云里雾里，只见树木不见森林。

本文我将从树的深度遍历来讲解DOWN事件的分发流程，从单链表的线性遍历讲解MOVE、UP事件的分发流程。  
  
  
**本文大纲**

**1. Android控件对应的多叉树** 

**2. 手势事件类型**  

**3. 事件分发涉及到的方法**

**4. DOWN事件的分发流程**

**5. MOVE、UP事件的分发流程**

**6. CANCEL事件触发时机以及分发流程**

**7. 通过实例讲解事件分发流程**

## 1. Android控件对应的多叉树

假设我们有一个这样的场景

屏幕上有一个FrameLayout名叫vp1。vp1有三个子控件vp2、vp3、vp4，它们的类型是FrameLayout。vp2有三个子控件v1、v2、v3。vp3有三个子控件v4、v5、v6。vp4有三个子控件v7、v8、v9。子控件的类型都是View。为了方便起见我们假设这些控件都是铺满屏幕的。我们将vp1控件树翻译成多叉树。

![控件多叉树](https://cdn.jsdelivr.net/gh/lizijin/bytestation@master/事件分发/byte_station_ViewGroup树形图.png)

**解释一下树节点ft的含义**

![ViewGroup节点](https://cdn.jsdelivr.net/gh/lizijin/bytestation@master/事件分发/byte_station_ViewGroup节点.png)


ft的是ViewGroup类中的mFirtstTouchTarget成员变量的简称。它对应的数据结构是**单链表**。当DOWN事件在onTouchEvent方法中返回true，会**回溯**设置父View的ft指针。

**举个例子**

当DOWN事件从vp1开始分发。
假设DOWN事件在v7的onTouchEvent方法中返回true。
那么v7的父控件vp4的ft会指向v7\(可以这样简单理解，实际上指向的是v7封装的一个TouchTarget对象\)。同理vp1的ft会指向vp4。所以就生成了一条vp1->vp4->v7的分发路径。这很重要。  


DOWN事件的分发是从vp1开始，假设所有的ViewGroup都不拦截事件，所有的View都不处理事件。事件会沿着最后一个子控件做深度遍历分发。以下用intercept代替onInterceptTouchEvent，touch代替onTouchEvent。
>调用流程如下
>
>vp1\(intercept\) -> vp4\(intercept\) \-> v9\(touch\) -> v8\(touch\) \->v7 \(touch\) \-> vp4\(touch\)\-> vp3 -> v6\(touch\) \-> v5\(touch\) \-> v4\(touch\) \-> vp3\(touch\)\-> vp2 \-> v3\(touch\) \-> v2\(touch\) \-> v1\(touch\) \-> vp2\(touch\)\-> vp1\(touch\)

**这只是DOWN事件分发的一个Case。根据是否拦截，View是否处理事件。它的遍历路径也会不一样。**  
  
MOVE、UP事件分发也是从vp1开始。但是他们不同于DOWN事件的分发。它不会进行深度遍历。深度遍历是比较耗时的。如果vp1有后代View分发了事件。那么必然会通过ft生成一条分发路径。MOVE事件只需沿着分发路径线性分发就可以了。还是用上面的例子。如果v7分发了DOWN事件。那么MOVE、UP事件的分发即vp1\(intercept\) \-> vp4\(intercept\) \-> v7\(touch\)  
  

## 2. 手势事件类型  

本文主要讲解四种事件类型，DOWN、MOVE、UP、CANCEL。用户划动手机屏幕然后离开。Android系统首先会触发DOWN事件，紧接着一连串的MOVE事件，以UP事件收场。**注意：触摸屏幕，事件之间没有任何依赖。有可能只有其中一种事件被分发。也有可能有多种事件类型被分发**  

| 事件类型	       | 解释          |    
| :-------------- | :----------- |
| ACTION_DOWN     |  手指按下屏幕  |   
| ACTION_MOVE     |  手指划动屏幕  |
| ACTION_CANCEL   |  事件被取消    | 
| ACTION_UP       |  手指离开屏幕  | 

## 3. 事件分发涉及到的方法  
| 方法名	       | 解释          |    
| :---------------- | :----------- |
| dispatchTouchEvent     |  事件分发逻辑  |   
| onInterceptTouchEvent  |  是否拦截事件 **(ViewGroup专属)**  |
| onTouchEvent   |  是否处理事件    | 



1.  dispatchTouchEvent方法是Android系统内部实现的事件分发逻辑。返回值为boolean类型。true表示该View或ViewGroup处理了事件，反之返回false。返回值含义同onTouchEvent的返回值。dispatchTouchEvent与onTouchEvent的区别在于，默认情况下前者的返回值依赖于后者的返回值，而且前者的侧重点在于制定事件分发的流程，后者的侧重点在于View或者ViewGroup是否处理该事件。

2.  onInterceptTouchEvent方法是ViewGroup专属的方法。当返回值为true表示ViewGroup\(假设vp1\)需要拦截掉该事件。这里有两种情况

    2.1   如果vp1的后代view没有处理DOWN事件。那么事件会直接交给vp1的onTouchEvent处理

    2.2  如果vp1的后代v7处理了DOWN事件，此时vp1拦截了MOVE事件。首先会生成CANCEL事件交由vp4分发。先后置空vp4，vp1的ft对象，接下来的MOVE事件同a步骤

3.  onTouchEvent方法返回值同dispatchTouchEvent方法。我们首先得明白一点。onTouchEvent返回true会且仅会落到某一个具体的控件上。不会有更多的控件的onTouchEvent返回true，换句话说有且仅有一个控件能够处理事件。**只有DOWN事件的返回值才有意义**。其它类型事件的返回值并不会影响事件分发的流程。我们以v8的onTouchEvent的DOWN事件返回值为例。

    3.1.  v8 DOWN事件返回true。表示v8处理该事件。在v8分发事件之前应该是 vp1\(onInterceptTouchEvent\) -> vp4\(onInterceptTouchEvent\) -> v9\(onTouchEvent\)-> v8\(onTouchEvent\)。此时v8返回true。系统会中断vp4的child遍历\(不再将事件交由v7分发\)。向上回溯设置vp4的ft指向v8，vp1的ft指向vp4。

    3.2.  v8 DOWN事件返回false。事件继续交由v8的亲兄弟v7分发。

## 4.DOWN事件的分发流程  

DOWN事件分发到vp1，会调用vp1的onInterceptTouchEvent。这里有拦截和不拦截两种情况。  

1.  如果返回true，vp1拦截DOWN事件，DOWN事件直接交由vp1的onTouchEvent处理。

2.  如果返回false，vp1不拦截DOWN事件，DOWN事件将会交由vp1的最后一个子View分发。即交由vp4分发。

拦截方法以此类推，如果vp4不拦截DOWN事件，将交由v9分发事件。因为v9是View类型。没有拦截方法，所以会直接调用v9的onTouchEvent方法。该方法有处理和不处理两种情况。  

1.  如果返回false，v9不处理事件。那么事件继续向前分发交由v8分发，同理调用v8的onTouchEvent方法，v8不处理事件，继续交由v7处理。v7也不处理，vp4的子View到此遍历完成。此时vp4的ft为空，直接调用vp4的onTouchEvent方法。  

2.  如果返回true，v9处理事件。系统会中断vp4的子View遍历，DOWN事件分发结束。同时往上递归回溯设置父View的ft对象。vp4的ft指向v9，vp1的ft指向vp4。  

 **总结：DOWN事件是通过深度遍历分发事件的。**  

## 5. MOVE、UP事件的分发流程  
  
MOVE事件分发到VP1。它与DOWN事件的区别是，它并不一定会调用onInterceptTouchEvent。它只有当vp1的ft不为空时才会调用onInterceptTouchEvent方法，否则会直接拦截掉事件。

1.   当vp1的ft为空。直接拦截掉MOVE事件。调用VP1的onTouchEvent方法，注意这里onTouchEvent方法的返回值不会影响事件分发的流程。

2.  当vp1的ft不为空\(当有后代View的onTouchEvent方法返回了true\)。调用onInterceptTouchEvent方法，如果返回true，同上，直接调换用VP1的onTouchEvent方法。如果返回false，会通过ft的单链表线性分发事件。  

UP事件同MOVE事件。这里就不分析了。  
  
**总结：MOVE、UP事件是通过线性遍历分发的。**  
  
  

## 6. CANCEL事件触发时机以及分发流程  
前面我们讲到的DOWN、MOVE、UP事件都是由手触摸屏幕产生的。并没有讲到CANCEL是如何产生的。CANCEL不是由手触摸屏幕产生的。它是由系统生成。并且分发给View的。有一种场景会触发系统产生CANCEL事件。  
还是上面的控件树。假设手机在屏幕的上半部分，所有的ViewGroup都不拦截事件，V9处理DOWN事件，当划动到屏幕的下半部分时，VP1拦截MOVE事件。当手指从上划动到下面时。系统将在vp1处，产生一个CANCEL事件，交由vp4分发。CANCEL事件的分发也是通过ft线性分发。当ViewGroup分发CANCEL事件后，会将ViewGroup的ft置为空。即将VP1，VP4的ft置为空。  
  

## 7. 通过实例讲解事件分发流程  

本段将通过三个场景实战讲解事件分发流程  

**1.  所有的View都不分发事件，所有的ViewGroup都不拦截事件** 

**2.  只有v7 onTouchEvent DOWN事件返回true**

**3.  只有v7处理事件，但是vp4在屏幕下半部分拦截MOVE事件**

  
自定义MyView  

```kotlin
class MyView @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
    var name: String = ""
    var mOnTouchValue = false//是否分发事件

    @RequiresApi(Build.VERSION_CODES.KITKAT)
    override fun onTouchEvent(event: MotionEvent): Boolean {
    

        Log.d("MyView", "$name onTouchEvent ${MotionEvent.actionToString(event.action)}")

        if (mOnTouchValue) {
            return true
        }
        return super.onTouchEvent(event)
    }

}
```

  
自定义MyFrameLayout

```kotlin
class MyFrameLayout @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0
) : FrameLayout(context, attrs, defStyleAttr) {
    var name: String = ""
    var mOnTouchValue = false//是否分发事件
    var mDispatchValueSuper = true //是否调用super.dispatchTouchEvent,如果false 直接return true
    var mRegionInterceptor = false//在某个特定区域 拦截事件

    override fun dispatchTouchEvent(ev: MotionEvent?): Boolean {

        return if (mDispatchValueSuper) {
            super.dispatchTouchEvent(ev)
        } else {
            true
        }
    }

    @RequiresApi(Build.VERSION_CODES.KITKAT)
    override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        Log.d("MyFrameLayout", "$name onInterceptTouchEvent ${MotionEvent.actionToString(ev.action)}")

        if (mRegionInterceptor) {
            val touchY = ev?.y
            if (touchY != null) {
                if (touchY > measuredHeight / 2) {
                    return true
                }

            }
        }
        return super.onInterceptTouchEvent(ev)
    }

    @RequiresApi(Build.VERSION_CODES.KITKAT)
    override fun onTouchEvent(event: MotionEvent): Boolean {
        Log.d("MyFrameLayout", "$name onTouchEvent ${MotionEvent.actionToString(event.action)}")
        if (mOnTouchValue) {
            return true
        }
        return super.onTouchEvent(event)
    }

}
```

  
  
**场景一 所有的View都不分发事件，所有的ViewGroup都不拦截事件**  

```kotlin
class TouchOneActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
//        setContentView(R.layout.activity_touch_one)
        //所有的view都不分发事件
        val view1 = MyView(this)
        val view2 = MyView(this)
        val view3 = MyView(this)
        val vp2 = MyFrameLayout(this)
        vp2.addView(view1)
        vp2.addView(view2)
        vp2.addView(view3)


        val view4 = MyView(this)
        val view5 = MyView(this)
        val view6 = MyView(this)

        val vp3 = MyFrameLayout(this)
        vp3.addView(view4)
        vp3.addView(view5)
        vp3.addView(view6)

        val view7 = MyView(this)
        val view8 = MyView(this)
        val view9 = MyView(this)

        val vp4 = MyFrameLayout(this)
        vp4.addView(view7)
        vp4.addView(view8)
        vp4.addView(view9)

        val vp1 = MyFrameLayout(this)
        vp1.addView(vp2)
        vp1.addView(vp3)
        vp1.addView(vp4)

        setContentView(vp1)
        view1.name = "view1"
        view2.name = "view2"
        view3.name = "view3"
        view4.name = "view4"
        view5.name = "view5"
        view6.name = "view6"
        view7.name = "view7"
        view8.name = "view8"
        view9.name = "view9"
        vp1.name ="vp1"
        vp2.name ="vp2"
        vp3.name ="vp3"
        vp4.name ="vp4"

    }
}
```

  
日志打印如下  
>>Vp1 onInterceptTouchEvent  Down
>>
>>Vp4 onInterceptTouchEvent  Down
>>
>>V9 onTouchEvent  Down
>>
>>V8 onTouchEvent  Down
>>
>>V7 onTouchEvent  Down
>>
>>Vp4 onTouchEvent  Down
>>
>>Vp3 onInterceptTouchEvent  Down
>>
>>V6 onTouchEvent  Down
>>
>>V5 onTouchEvent  Down
>>
>>V4 onTouchEvent  Down
>>
>>Vp3 onTouchEvent  Down
>>
>>Vp2onInterceptTouchEvent  Down
>>
>>V3 onTouchEvent  Down
>>
>>V2 onTouchEvent  Down
>>
>>V1 onTouchEvent  Down
>>
>>Vp2 onTouchEvent  Down
>>
>>Vp1 onTouchEvent  Down 

分发图如下  

![DOWN事件分发图](https://cdn.jsdelivr.net/gh/lizijin/bytestation@master/事件分发/byte_station_DOWN事件分发1.jpeg)

![MOVE事件分发图](https://cdn.jsdelivr.net/gh/lizijin/bytestation@master/事件分发/byte_station_MOVE事件分发1.jpeg)

![UP事件分发图](https://cdn.jsdelivr.net/gh/lizijin/bytestation@master/事件分发/byte_station_UP事件分发1.jpeg) 

 
 
**场景二 只有v7 onTouchEvent DOWN事件返回true**

```kotlin
class TouchTwoActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_touch_two)

        //View 7 分发事件
        val view1 = MyView(this)
        val view2 = MyView(this)
        val view3 = MyView(this)
        val vp2 = MyFrameLayout(this)
        vp2.addView(view1)
        vp2.addView(view2)
        vp2.addView(view3)


        val view4 = MyView(this)
        val view5 = MyView(this)
        val view6 = MyView(this)

        val vp3 = MyFrameLayout(this)
        vp3.addView(view4)
        vp3.addView(view5)
        vp3.addView(view6)

        val view7 = MyView(this)
        view7.mOnTouchValue = true
        val view8 = MyView(this)
        val view9 = MyView(this)

        val vp4 = MyFrameLayout(this)
        vp4.addView(view7)
        vp4.addView(view8)
        vp4.addView(view9)

        val vp1 = MyFrameLayout(this)
        vp1.addView(vp2)
        vp1.addView(vp3)
        vp1.addView(vp4)

        setContentView(vp1)
        view1.name = "view1"
        view2.name = "view2"
        view3.name = "view3"
        view4.name = "view4"
        view5.name = "view5"
        view6.name = "view6"
        view7.name = "view7"
        view8.name = "view8"
        view9.name = "view9"
        vp1.name ="vp1"
        vp2.name ="vp2"
        vp3.name ="vp3"
        vp4.name ="vp4"
    }
}
```

  
日志打印如下  
>>Vp1 onInterceptTouchEvent  Down
>>
>>Vp4 onInterceptTouchEvent  Down
>>
>>v9 onTouchEvent  Down
>>
>>v8 onTouchEvent  Down
>>
>>v7 onTouchEvent  Down  # v7消耗掉了事件，中断本层遍历，并回溯设置ft
>>
>>Vp1 onInterceptTouchEvent  Move
>>
>>Vp4 onInterceptTouchEvent  Move
>>
>>v7 onTouchEvent  Move
>>
>>Vp1 onInterceptTouchEvent  UP
>>
>>Vp4 onInterceptTouchEvent  UP
>>
>>v7 onTouchEvent  UP  

分发图如下  
  
DOWN事件分发图如下。步骤6、7是逻辑。并非实际打印

![DOWN事件分发图](https://cdn.jsdelivr.net/gh/lizijin/bytestation@master/事件分发/byte_station_Down事件分发2.jpeg)

![MOVE事件分发图](https://cdn.jsdelivr.net/gh/lizijin/bytestation@master/事件分发/byte_station_move事件分发2.jpeg
)  
  
**场景三 只有v7处理事件，但是vp1在屏幕下半部分拦截MOVE事件**  
场景三事件分发比较复杂。因为在ft不为空的ViewGroup上拦截事件会分发CANCEL事件

```kotlin
class TouchFourActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_touch_two)

        //View 7 分发事件 但是vp1在屏幕下半部分拦截move事件 从上往下滑动
        val view1 = MyView(this)
        val view2 = MyView(this)
        val view3 = MyView(this)
        val vp2 = MyFrameLayout(this)
        vp2.addView(view1)
        vp2.addView(view2)
        vp2.addView(view3)


        val view4 = MyView(this)
        val view5 = MyView(this)
        val view6 = MyView(this)

        val vp3 = MyFrameLayout(this)
        vp3.addView(view4)
        vp3.addView(view5)
        vp3.addView(view6)

        val view7 = MyView(this)
        view7.mOnTouchValue = true
        val view8 = MyView(this)
        val view9 = MyView(this)

        val vp4 = MyFrameLayout(this)
        vp4.addView(view7)
        vp4.addView(view8)
        vp4.addView(view9)

        val vp1 = MyFrameLayout(this)
        vp1.addView(vp2)
        vp1.addView(vp3)
        vp1.addView(vp4)

        setContentView(vp1)
        view1.name = "view1"
        view2.name = "view2"
        view3.name = "view3"
        view4.name = "view4"
        view5.name = "view5"
        view6.name = "view6"
        view7.name = "view7"
        view8.name = "view8"
        view9.name = "view9"
        vp1.name ="vp1"
        vp2.name ="vp2"
        vp3.name ="vp3"
        vp4.name ="vp4"
        vp1.mRegionInterceptor=true//下半屏幕拦截事件
    }
}
```

  
日志打印如下  

>>vp1 onInterceptTouchEvent ACTION\_DOWN  
>>
>>vp4 onInterceptTouchEvent ACTION\_DOWN  
>>
>>view9 onTouchEvent ACTION\_DOWN 
>>
>>view8 onTouchEvent ACTION\_DOWN  
>>
>>view7 onTouchEvent ACTION\_DOWN 
>>
>>vp1 onInterceptTouchEvent ACTION\_MOVE  
>>
>>vp4 onInterceptTouchEvent ACTION\_MOVE  
>>
>>view7 onTouchEvent ACTION\_MOVE 
>>
>>vp1 onInterceptTouchEvent ACTION\_MOVE  
>>
>>vp4 onInterceptTouchEvent ACTION\_MOVE  
>>
>>view7 onTouchEvent ACTION\_MOVE  
>>
>>vp1 onInterceptTouchEvent ACTION\_MOVE  
>>
>>vp4 onInterceptTouchEvent ACTION\_MOVE  
>>
>>view7 onTouchEvent ACTION\_MOVE  
>>
>>vp1 onInterceptTouchEvent ACTION\_MOVE  
>>
>>vp4 onInterceptTouchEvent ACTION\_MOVE  
>>
>>view7 onTouchEvent ACTION\_MOVE  
>>
>>vp1 onInterceptTouchEvent ACTION\_MOVE  
>>
>>vp4 onInterceptTouchEvent ACTION\_MOVE  
>>
>>view7 onTouchEvent ACTION\_MOVE  
>>
>>vp1 onInterceptTouchEvent ACTION\_MOVE  
>>
>>vp4 onInterceptTouchEvent ACTION\_MOVE  
>>
>>view7 onTouchEvent ACTION\_MOVE  
>>
>>vp1 onInterceptTouchEvent ACTION\_MOVE//此时移动到了屏幕下方  
>>
>>vp4 onInterceptTouchEvent ACTION\_CANCEL  
>>
>>view7 onTouchEvent ACTION\_CANCEL  
>>
>>vp1 onTouchEvent ACTION\_MOVE  
>>
>>vp1 onTouchEvent ACTION\_MOVE  
>>
>>vp1 onTouchEvent ACTION\_MOVE  
>>
>>vp1 onTouchEvent ACTION\_MOVE  
>>
>>vp1 onTouchEvent ACTION\_UP

  
在屏幕上半部分滑动事件分发图同场景二

在屏幕下半部分滑动分发图如下

![MOVE事件在屏幕下方被拦截](https://cdn.jsdelivr.net/gh/lizijin/bytestation@master/事件分发/byte_station_Move事件分发3.jpeg
)

![后续MOVE、UP事件分发](https://cdn.jsdelivr.net/gh/lizijin/bytestation@master/事件分发/byte_station_move-up.jpeg
)

后续的MOVE和UP都只会分发到vp1的onTouchEvent方法里。因为vp1.parent的ft不为空。vp1的ft为空。  
Android事件分发比较复杂。建议读者不要走马观花的看本篇文件。可以将树形图画在草稿本上。然后结合源码。挨个练习DOWN、MOVE、UP、CANCEL事件分发的流程。  

---  
#### Written By 多点-移动运营研发部-姜斌

如果你有任何问题，欢迎通过以下方式联系我
![欢迎扫码关注公众号](https://cdn.jsdelivr.net/gh/lizijin/bytestation@master/byte_station_微信公众号.jpeg)


![欢迎扫码加微信好友](https://cdn.jsdelivr.net/gh/lizijin/bytestation@master/byte_station_微信好友.jpeg)

