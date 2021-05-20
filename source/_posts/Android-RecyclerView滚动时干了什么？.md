---
title: Android-RecyclerView滚动时干了什么？
tags: 底层原理
categories: 技术分享
---

# RecyclerView滚动时干了什么？

谈到RecyclerView的时候，复用机制是我们能脱口而出的优点之一。系统内置的ViewHolder避免了使用ListView时手动去创建ViewHolder的麻烦。关于何时回收View，何时复用View，我们能做到胸有成竹吗？当我们滑动一个RecyclerView时，是先回收View，再复用View？还是先复用View，再回收View呢？答案是都有可能。详情且看下面分析：

**名词解释**

**1. 回收：是指View不需要再展示在屏幕中，被回收到回收池中**

**2. 复用：本文中的复用是指调用了onCreateViewHolder或者onBindViewHolder方法**


**本文大纲**

**1. 滑动RV的两个场景**

**2. DEMO验证答案**

**3. 滑动原理讲解**

**4. 源码分析**

**5. 提问互动**


## 1.滑动RV的两个场景  

### 1.1 场景一 

RV中每个Item高度都为100px，最后一个Item超出屏幕50px。RV初始状态如下图        

![场景一](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9be83770378a4458b79a0cbf2dc365f7~tplv-k3u1fbpfcp-zoom-1.image)

Q1 假设向上滑动40px

请问是否有View发生回收和复用？如果有，先复用还是先回收？

$\color{red}{答    回收和复用都没有发生}$

  

Q2    假设向上滑动60px

请问是否有View发生回收和复用？如果有，先复用还是先回收？

$\color{red}{答    没有发生回收，发生了复用}$




Q3    假设向上滑动120px

请问是否有View发生回收和复用？如果有，先复用还是先回收？

$\color{red}{答    发生了回收和复用。先复用后回收}$



### 1.2 场景二

RV中第一个Item高度为50px，其它都为100px，最后一个Item超出屏幕95px。RV初始状态如下

![场景二](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cae471defcc04a93a7a0f11e3b85d621~tplv-k3u1fbpfcp-zoom-1.image)

Q1    假设向上滑动40px

请问是否有View发生回收和复用？如果有，先复用还是先回收？

$\color{red}{答    回收和复用都没有发生}$


  

Q2    假设向上滑动60px

请问是否有View发生回收和复用？如果有，先复用还是先回收？

$\color{red}{答    发生了回收，没有发生复用}$


  

Q3    假设向上滑动120px

请问是否有View发生回收和复用？如果有，先复用还是先回收？

$\color{red}{答    发生了回收和复用。先回收后复用}$

  

从答案可以看出。回收和复用并没有固定的答案。它因场景而异。下面我们通过案例验证答案真伪。  
  

## 2. DEMO验证答案 

### 2.1 我们来验证场景一  

程序运行图  

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5fee2c707893492aa3e158b1ce6a98d6~tplv-k3u1fbpfcp-zoom-1.image
)  
程序代码  

```kotlin
class RecyclerViewActivity1 : AppCompatActivity() {
    private lateinit var mRecyclerView: RecyclerView
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_recycler_view1)
        mRecyclerView = findViewById(R.id.recyclerview)
        mRecyclerView.setHasFixedSize(true)
        mRecyclerView.setItemViewCacheSize(0)
        mRecyclerView.layoutManager =
            LinearLayoutManager(this).apply {
                orientation = LinearLayoutManager.VERTICAL
                isItemPrefetchEnabled = false
            }
        val list: MutableList<String> =
            ArrayList()
        repeat(100) {
            list.add("item $it")
        }
        mRecyclerView.adapter = MyAdapter(list)
    }

    inner class MyAdapter(val mStrings: MutableList<String>) :
        RecyclerView.Adapter<RecyclerView.ViewHolder>() {
        override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
            println("RecyclerView 场景一 onCreateViewHolder ")
            val view = LayoutInflater.from(parent.context)
                .inflate(R.layout.view_item, parent, false)
            return object : RecyclerView.ViewHolder(view) {}
        }

        override fun getItemCount(): Int {
            return mStrings.size
        }

        override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
            println("RecyclerView 场景一 onBindViewHolder $position ")
            val textView = holder.itemView as TextView
            textView.layoutParams.height = (resources.displayMetrics.density * 100).toInt()
            textView.text = mStrings[position]
        }

        override fun onViewRecycled(holder: RecyclerView.ViewHolder) {
            println("RecyclerView 场景一 发生回收 " + (holder.itemView as TextView).text)

            super.onViewRecycled(holder)
        }

    }

    fun scroll120(view: View) {
        mRecyclerView.scrollBy(0, (resources.displayMetrics.density * 120).toInt())
    }

    fun scroll60(view: View) {
        mRecyclerView.scrollBy(0, (resources.displayMetrics.density * 60).toInt())

    }

    fun scroll40(view: View) {
        mRecyclerView.scrollBy(0, (resources.displayMetrics.density * 40).toInt())

    }
}
```

  
日志输出如下

>首先进入初始状态
>
>RecyclerView 场景一 onCreateViewHolder 
>
>RecyclerView 场景一 onBindViewHolder 0 
>
>RecyclerView 场景一 onCreateViewHolder 
>
>RecyclerView 场景一 onBindViewHolder 1 
>
>RecyclerView 场景一 onCreateViewHolder 
>
>RecyclerView 场景一 onBindViewHolder 2 
>
>RecyclerView 场景一 onCreateViewHolder 
>
>RecyclerView 场景一 onBindViewHolder 3 
>
>RecyclerView 场景一 onCreateViewHolder 
>
>RecyclerView 场景一 onBindViewHolder 4 
>
>RecyclerView 场景一 onCreateViewHolder 
>
>RecyclerView 场景一 onBindViewHolder 5
  
>点击上滑40px。打印日志不变。证明 回收和复用都没有发生  

>点击上滑60px。打印日志如下。证明 没有发生回收，发生了复用
>
>RecyclerView 场景一 onCreateViewHolder
>
>RecyclerView 场景一 onBindViewHolder 0
>
>RecyclerView 场景一 onCreateViewHolder
>
>RecyclerView 场景一 onBindViewHolder 1
>
>RecyclerView 场景一 onCreateViewHolder
>
>RecyclerView 场景一 onBindViewHolder 2
>
>RecyclerView 场景一 onCreateViewHolder
>
>RecyclerView 场景一 onBindViewHolder 3
>
>RecyclerView 场景一 onCreateViewHolder
>
>RecyclerView 场景一 onBindViewHolder 4
>
>RecyclerView 场景一 onCreateViewHolder
>
>RecyclerView 场景一 onBindViewHolder 5
>
>
>RecyclerView 场景一 onCreateViewHolder //只发生了复用  
>RecyclerView 场景一 onBindViewHolder 6

  
>点击上滑动120px。打印日志如下。证明 发生了回收和复用。先复用后回收
>
>RecyclerView 场景一 onCreateViewHolder 
>
>RecyclerView 场景一 onBindViewHolder 0 
>
>RecyclerView 场景一 onCreateViewHolder 
>
>RecyclerView 场景一 onBindViewHolder 1 
>
>RecyclerView 场景一 onCreateViewHolder 
>
>RecyclerView 场景一 onBindViewHolder 2 
>
>RecyclerView 场景一 onCreateViewHolder 
>
>RecyclerView 场景一 onBindViewHolder 3 
>
>RecyclerView 场景一 onCreateViewHolder 
>
>RecyclerView 场景一 onBindViewHolder 4 
>
>RecyclerView 场景一 onCreateViewHolder 
>
>RecyclerView 场景一 onBindViewHolder 5
>
>RecyclerView 场景一 onCreateViewHolder //先复用 
>
>RecyclerView 场景一 onBindViewHolder 6
>
>RecyclerView 场景一 发生回收 item 0  //后回收

  
### 2.2 我们来验证场景二  

程序运行图

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b2c035480b14deaa0a9023ba693fdff~tplv-k3u1fbpfcp-zoom-1.image
)  

程序代码

```kotlin
class RecyclerViewActivity2 : AppCompatActivity() {
    private lateinit var mRecyclerView: RecyclerView
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_recycler_view2)
        mRecyclerView = findViewById(R.id.recyclerview)
        mRecyclerView.setHasFixedSize(true)
        mRecyclerView.setItemViewCacheSize(0)
        mRecyclerView.layoutManager =
            LinearLayoutManager(this).apply {
                orientation = LinearLayoutManager.VERTICAL
                isItemPrefetchEnabled = false
            }
        val list: MutableList<String> =
            ArrayList()
        repeat(100) {
            list.add("item $it")
        }
        mRecyclerView.adapter = MyAdapter(list)
    }

    inner class MyAdapter(val mStrings: MutableList<String>) :
        RecyclerView.Adapter<RecyclerView.ViewHolder>() {
        override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
            println("RecyclerView 场景二 onCreateViewHolder ")
            val view = LayoutInflater.from(parent.context)
                .inflate(R.layout.view_item, parent, false)
            return object : RecyclerView.ViewHolder(view) {}
        }

        override fun getItemCount(): Int {
            return mStrings.size
        }

        override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
            println("RecyclerView 场景二 onBindViewHolder $position ")
            val textView = holder.itemView as TextView
            textView.layoutParams.height =
                if (position == 0) (resources.displayMetrics.density * 50).toInt() else (resources.displayMetrics.density * 100).toInt()
            textView.text = mStrings[position]
        }

        override fun onViewRecycled(holder: RecyclerView.ViewHolder) {
            println("RecyclerView 场景二 发生回收 " + (holder.itemView as TextView).text)
            super.onViewRecycled(holder)
        }

    }

    fun scroll120(view: View) {
        mRecyclerView.scrollBy(0, (resources.displayMetrics.density * 120).toInt())
    }

    fun scroll60(view: View) {
        mRecyclerView.scrollBy(0, (resources.displayMetrics.density * 60).toInt())

    }

    fun scroll40(view: View) {
        mRecyclerView.scrollBy(0, (resources.displayMetrics.density * 40).toInt())

    }
}
```

  
>日志输出首先进入初始状态
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 0
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 1
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 2
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 3
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 4
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 5
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 6

  
>点击上滑40px。打印日志不变。证明 回收和复用都没有发生  


>点击上滑60px。打印日志如下。证明 发生回收，没有发生复用
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 0
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 1
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 2
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 3
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 4
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 5
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 6
>
>RecyclerView 场景二 发生回收 item 0   //只发生了回收

  
>点击上滑动120px。打印日志如下。证明 发生了回收和复用。先回收后复用
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 0
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 1
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 2
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 3
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 4
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 5
>
>RecyclerView 场景二 onCreateViewHolder
>
>RecyclerView 场景二 onBindViewHolder 6
>
>RecyclerView 场景二 发生回收 item 0 //先回收
>
>RecyclerView 场景二 onBindViewHolder 7 //再复用

  

## 3. 滑动原理分析  

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4e9e34fd43034173912c8a7fff713f65~tplv-k3u1fbpfcp-zoom-1.image)

如图所示，介绍几个关于坐标的参数  

1.  delta：手指滑动的距离120px。  

2.  mOffset：RV最后一个子View的Bottom在屏幕坐标系的Y坐标600px。RV的下一个View\(Item7\)从mOffset处布局。

3.  mScrollingOffset：RV最后一个子view的Bottom距离RV Bottom的距离50px。向上滑动不超过该距离。如超过需创建新的View填充。

4.  mVailable：delta\-mScrollingOffset。可以填充View的空间。如果大于0表示有空间填充新的View

5. 如果delta\<mScrollingOffset，mScrollingOffset\=delta，mVailable\<0  

滑动逻辑如下  

1.   从RecyclerView的第0个View开始遍历，直到View的Bottom>mScrollingOffset,并记录该View的下标index，回收\[0,index\)区间的View，index为开区间，如果index>=1,则会将\[0,index\)区间的View移除屏幕，并按照回收算法放入回收池。具体回收算法先按下不表。

2.  如果mVailable\>0，则从mOffset处，用新的View填充。mOffset+=新View的高度，mVailable\-=新View的高度，mScrollingOffset+=新View的高度，如果mVailable\<0，mScrollingOffset+=mVailable。布局完成后用步骤1的算法按需回收上面的View。

3.  重复步骤2

4.  将RV整体，向上移动delta或者consumed距离(一般是delta距离，但是当RecyclerView下面没有Item时会是具体消耗掉的距离)  

  
根据此滑动逻辑，我们分析场景一中的向上滑动120px
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b422bb36e19245698c2c240890bb5c5f~tplv-k3u1fbpfcp-zoom-1.image)

mOffset = 600px  

mScrollingOffset = 50px

mAvailable = 70px

item1高度100px  
  

1.  首先从第0个View遍历Bottom>50px。找到item1.bottom=100px，记录index=0。因为index\<1。所以不发生回收  

2.  mAvailable\>0，从Item6的底部，增加View Item7\(此处发生复用逻辑\)高度为100px，mOffset\=700px，mAvailable=-30，mScrollingOffset=mScrollingOffset+100-30=120px。然后检查回收。首先从第0个View遍历Bottom>120px。找到item2.bottom=200px，记录index=1。回收\[0,1\)区间的View。即回收Item1

3.  mAvailable=-30\<0,退出填充逻辑

4.  整体向上移动120px

我们看到先创建Item7 然后回收Item1。跟日志相符合  

RecyclerView 场景一 onCreateViewHolder //先复用  
RecyclerView 场景一 onBindViewHolder 6

RecyclerView 场景一 发生回收 item 0  //后回收

  
  
同样的逻辑我们也可以分析场景二中的向上滑动120px的情况。场景二会先发生回收，再发生复用。读者可以自己去求证。  

## 4.源码分析  
RV的滑动，最终会调用LayoutManager的scrollBy方法。我们使用的是LinearLayoutManager。

```java
//LinearLayoutManager.java
int scrollBy(int delta, RecyclerView.Recycler recycler, RecyclerView.State state) {
        if (getChildCount() == 0 || delta == 0) {
            return 0;
        }
        ensureLayoutState();
        mLayoutState.mRecycle = true;
        final int layoutDirection = delta > 0 ? LayoutState.LAYOUT_END : LayoutState.LAYOUT_START;
        final int absDelta = Math.abs(delta);
        ##代码1 updateLayoutState方法，主要是计算mOffset等参数。
        updateLayoutState(layoutDirection, absDelta, true, state);
        ##代码2 fill方法，根据剩余空间，填充View
        final int consumed = mLayoutState.mScrollingOffset
                + 
                fill(recycler, mLayoutState, state, false);
        if (consumed < 0) {
            if (DEBUG) {
                Log.d(TAG, "Don't have any more elements to scroll");
            }
            return 0;
        }
        final int scrolled = absDelta > consumed ? layoutDirection * consumed : delta;
        ##代码3 offsetChildren，整体移动RV的子View
        mOrientationHelper.offsetChildren(-scrolled);
        if (DEBUG) {
            Log.d(TAG, "scroll req: " + delta + " scrolled: " + scrolled);
        }
        mLayoutState.mLastScrollDelta = scrolled;
        return scrolled;
    }
```

##代码1 updateLayoutState方法，主要是计算mOffset等参数。

##代码2 fill方法，根据剩余空间，填充View

##代码3 offsetChildren，整体移动RV的子View

```java
//主要是计算
private void updateLayoutState(int layoutDirection, int requiredSpace,
            boolean canUseExistingSpace, RecyclerView.State state) {
        // If parent provides a hint, don't measure unlimited.
        mLayoutState.mInfinite = resolveIsInfinite();
        mLayoutState.mLayoutDirection = layoutDirection;
        mReusableIntPair[0] = 0;
        mReusableIntPair[1] = 0;
        calculateExtraLayoutSpace(state, mReusableIntPair);
        int extraForStart = Math.max(0, mReusableIntPair[0]);
        int extraForEnd = Math.max(0, mReusableIntPair[1]);
        boolean layoutToEnd = layoutDirection == LayoutState.LAYOUT_END;
        mLayoutState.mExtraFillSpace = layoutToEnd ? extraForEnd : extraForStart;
        mLayoutState.mNoRecycleSpace = layoutToEnd ? extraForStart : extraForEnd;
        int scrollingOffset;
        if (layoutToEnd) {
            mLayoutState.mExtraFillSpace += mOrientationHelper.getEndPadding();
            // get the first child in the direction we are going
            final View child = getChildClosestToEnd();
            // the direction in which we are traversing children
            mLayoutState.mItemDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_HEAD
                    : LayoutState.ITEM_DIRECTION_TAIL;
            mLayoutState.mCurrentPosition = getPosition(child) + mLayoutState.mItemDirection;
            mLayoutState.mOffset = mOrientationHelper.getDecoratedEnd(child);
            // calculate how much we can scroll without adding new children (independent of layout)
            scrollingOffset = mOrientationHelper.getDecoratedEnd(child)
                    - mOrientationHelper.getEndAfterPadding();

        } else {
            final View child = getChildClosestToStart();
            mLayoutState.mExtraFillSpace += mOrientationHelper.getStartAfterPadding();
            mLayoutState.mItemDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_TAIL
                    : LayoutState.ITEM_DIRECTION_HEAD;
            mLayoutState.mCurrentPosition = getPosition(child) + mLayoutState.mItemDirection;
            mLayoutState.mOffset = mOrientationHelper.getDecoratedStart(child);
            scrollingOffset = -mOrientationHelper.getDecoratedStart(child)
                    + mOrientationHelper.getStartAfterPadding();
        }
        mLayoutState.mAvailable = requiredSpace;
        if (canUseExistingSpace) {
            mLayoutState.mAvailable -= scrollingOffset;
        }
        mLayoutState.mScrollingOffset = scrollingOffset;
    }
```

```java
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
            RecyclerView.State state, boolean stopOnFocusable) {
        // max offset we should set is mFastScroll + available
        final int start = layoutState.mAvailable;
        if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
            // TODO ugly bug fix. should not happen
            if (layoutState.mAvailable < 0) {
                layoutState.mScrollingOffset += layoutState.mAvailable;
            }
            ##代码1 首先判断是否需要回收View
            recycleByLayoutState(recycler, layoutState);
        }
        int remainingSpace = layoutState.mAvailable + layoutState.mExtraFillSpace;
        LayoutChunkResult layoutChunkResult = mLayoutChunkResult;
        ##代码2 根据剩余空间，判断是否需要填充View
        while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
            layoutChunkResult.resetInternal();
            if (RecyclerView.VERBOSE_TRACING) {
                TraceCompat.beginSection("LLM LayoutChunk");
            }
            ##代码3 是具体的layout方法
            layoutChunk(recycler, state, layoutState, layoutChunkResult);
            if (RecyclerView.VERBOSE_TRACING) {
                TraceCompat.endSection();
            }
            if (layoutChunkResult.mFinished) {
                break;
            }
            layoutState.mOffset += layoutChunkResult.mConsumed * layoutState.mLayoutDirection;
            /**
             * Consume the available space if:
             * * layoutChunk did not request to be ignored
             * * OR we are laying out scrap children
             * * OR we are not doing pre-layout
             */
            if (!layoutChunkResult.mIgnoreConsumed || layoutState.mScrapList != null
                    || !state.isPreLayout()) {
                layoutState.mAvailable -= layoutChunkResult.mConsumed;
                // we keep a separate remaining space because mAvailable is important for recycling
                remainingSpace -= layoutChunkResult.mConsumed;
            }

            if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
                layoutState.mScrollingOffset += layoutChunkResult.mConsumed;
                if (layoutState.mAvailable < 0) {
                    layoutState.mScrollingOffset += layoutState.mAvailable;
                }
                ##代码4 是layout完成后判断是否需要回收View
                recycleByLayoutState(recycler, layoutState);
            }
            if (stopOnFocusable && layoutChunkResult.mFocusable) {
                break;
            }
        }
        if (DEBUG) {
            validateChildOrder();
        }
        return start - layoutState.mAvailable;
    }
```

##代码1，首先判断是否需要回收View

##代码2，根据剩余空间，判断是否需要填充View

##代码3 是具体的layout方法

##代码4 是layout完成后判断是否需要回收View

本文主要讲解了滑动时的回收和复用的逻辑。具体如何如何回收，如何复用。RecyclerView的三级缓存是如何实现的。且听下回分解。

## 5. 提问互动
最后为了巩固大家对知识的理解，提出一个问题，请在评论区写出你的答案吧。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b422bb36e19245698c2c240890bb5c5f~tplv-k3u1fbpfcp-zoom-1.image)

**问题一 场景一的case3，向上滑动120px，120px大于第一个Item的高度100px，为何不先让Item1先回收掉呢？**

**问题二 谷歌这么设计的原因可能是什么？**

#### Written By 多点-移动运营研发部-姜斌



