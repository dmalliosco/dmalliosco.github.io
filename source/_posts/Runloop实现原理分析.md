---
title: RunLoop实现原理分析
tags: 源码分析
categories: 技术分享
---
# Runloop 
## 一、概念
RunLoop是通过内部维护的事件循环(Event Loop)来对事件/消息进行管理的一个对象。App主线程不退出就是用到了Event Loop。

- 保持程序持续运行
- 监听输入源，进行调度处理app各种事件（touch/timer/performSelector/异步回调）
- 节省cpu资源

![截屏2020-09-03 上午9.15.01](https://tva1.sinaimg.cn/large/007S8ZIlly1gietsitlo3j31l60t41kx.jpg)

RunLoop 会接收两种类型的输入源：一种是来自另一个线程或者来自不同应用的异步消息；另一种是来自预订时间或者重复间隔的同步事件。

>macOS/iOS系统中提供了两个对象：**NSRunLoop**和**CFRunLoopRef**。
>
>- **CFRunLoopRef**在**CoreFoundation**框架中，提供了纯C函数的API，并且所有API都是**线程安全**的。
>- **NSRunLoop**则是基于**CFRunLoopRef**的封装，提供面向对象的API，但是这些API是**非线程安全**的。

#### 和线程关系

- 线程和RunLoop是一一对应的,其映射关系是保存在一个全局的 Dictionary 里

- 自己创建的线程默认是没有开启RunLoop的

- UIApplicationMain内部默认开启了主线程的RunLoop，不断地接收处理消息以及等待休眠，所以运行程序之后会保持持续运行状态。

  线程和runloop映射关系关键代码如下（[Runloop源码](https://opensource.apple.com/tarballs/CF/ )）

```c
CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
    CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
    // 绑定
    CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);
```



## 二、数据结构

内部结构如下： 

```c
struct __CFRunLoop {
    ...
    pthread_t _pthread;                  //持有的线程对象             
    CFMutableSetRef  _commonModes;       //模式名称字符串集合
    CFMutableSetRef  _commonModeItems;   //由Observer,Timer,Source集合构成
    CFRunLoopModeRef _currentMode;
    CFMutableSetRef  _modes;             //多个运行模式集合
		...
};

struct __CFRunLoopMode {
    ...
    CFStringRef _name;                   //// Mode Name, 例如 @"kCFRunLoopDefaultMode"
    CFMutableSetRef _sources0;           //处理APP内部事件，APP自己负责，管理触发
    CFMutableSetRef _sources1;           //由runloop和内核管理，包含一个  mach_port（处理端口类的消息）和一个回调（函数指针），被用于通过内核和其他线程互发消息，这种source能主动唤醒runloop的线程
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers; 
    ...
};
```

**CommonModes**：一个`Mode`可以将自己标记为`Common`属性（通过将其`ModeName`添加到`RunLoop`的`commonModes`中）。每当`RunLoop`的内容发生变化时，`RunLoop`都会自动将 `_commonModeItems`里的`Source/Observer/Timer`同步到具有`Common`标记的所有`Mode`里。

> 主线程的`RunLoop`里有两个公开预置的`Mode`,可以用这两个`Mode Name`来操作其对应的`Mode`：
>
> - `kCFRunLoopDefaultMode`
> - `UITrackingRunLoopMode`
>
> 这两个`Mode`都已经被标记为`Common`属性。`DefaultMode`是App平时所处的状态，`TrackingRunLoopMode`是追踪ScrollView滑动时的状态。当你创建一个`Timer`并加到`DefaultMode`时，`Timer`会得到重复回调，但此时滑动一个TableView时，`RunLoop`会将`mode`切换为`TrackingRunLoopMode`，这时`Timer`就不会被回调，并且也不会影响到滑动操作。
>
> 有时你需要一个`Timer`，在两个`Mode`中都能得到回调，一种办法就是将这个`Timer`分别加入这两个`Mode`。还有一种方式，就是将`Timer`加入到顶层的`RunLoop`的`commonModeItems`中。`commonModeItems`被`RunLoop`自动更新到所有具有`Common`属性的`Mode`里去。
>
> ```c
> [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
> ```



#### CFRunLoopMode(CFRunLoopModeRef)

总共是有五种`CFRunLoopMode`:

- `kCFRunLoopDefaultMode`：默认模式，主线程是在这个运行模式下运行
- `UITrackingRunLoopMode`：跟踪用户交互事件（用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他Mode影响）
- `UIInitializationRunLoopMode`：在刚启动App时第进入的第一个 Mode，启动完成后就不再使用
- `GSEventReceiveRunLoopMode`：接受系统内部事件，通常用不到
- `kCFRunLoopCommonModes`：伪模式，不是一种真正的运行模式，是同步Source/Timer/Observer到多个Mode中的一种解决方案



一个RunLoop 对象中可能包含多个Mode，每次调用 RunLoop 的主函数时，只能指定其中一个 Mode(CurrentMode)。切换 Mode需要退出loop重新指定一个 Mode进入 。主要是为了分隔开不同的 Source、Timer、Observer，让它们之间互不影响。

![截屏2020-09-03 上午9.15.40](https://tva1.sinaimg.cn/large/007S8ZIlly1gietsxahcdj31gk0s4dj9.jpg)



##### CFRunLoopSource

分为source0和source1两种

- sources0 :用户触发的事件。不能主动触发事件，需要手动唤醒线程，将当前线程从内核态切换到用户态。

  使用时，需要先调用`CFRunLoopSourceSignal(source)`，将这个`Source`标记为待处理，然后手动调用`CFRunLoopWakeUp(runloop)`来唤醒`RunLoop`，让其处理这个事件。

- sources1 :基于port的，包含一个 mach_port 和一个回调，可监听系统端口和通过内核和其他线程发送的消息，能主动唤醒RunLoop，接收分发系统事件

##### CFRunLoopTimer

在预设的时间点唤醒RunLoop执行回调。基于RunLoop的，因此它不是实时的.

> CADisplayLink 是一个用于显示的定时器， 它可以让用户程序的显示与屏幕的硬件刷新保持同步，iOS系统中正常的屏幕刷新率为60Hz（60次每秒）。
>
> FPS监测就是利用屏幕刷新的频率调用CADisplayLink指定的selector，就是说每次屏幕刷新的时候就调用selector，那么只要在selector方法里面统计每秒这个方法执行的次数，通过次数/时间就可以得出当前屏幕的刷新率了。

##### CFRunLoopObserver

包含了一个回调(函数指针)，当`RunLoop`的状态发生变化时，观察者就能通过回调接受到这个变化。

可以观测的时间点有以下几个：

```c
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),         //RunLoop准备启动，即将进入loop
    kCFRunLoopBeforeTimers = (1UL << 1),  //触发 Timer 回调，RunLoop将要处理一些Timer相关事件
    kCFRunLoopBeforeSources = (1UL << 2), //触发 Source0 回调，RunLoop将要处理一些Source事件
    kCFRunLoopBeforeWaiting = (1UL << 5), //等待 mach_port 消息，RunLoop将要进行休眠状态,即将由用户态切换到内核态
    kCFRunLoopAfterWaiting = (1UL << 6),  //接收 mach_port 消息，RunLoop被唤醒，即从内核态切换到用户态
    kCFRunLoopExit = (1UL << 7),          //RunLoop退出
    kCFRunLoopAllActivities = 0x0FFFFFFFU //监听所有状态
};
```

上面的`Source/Timer/Observer`被统称为 **mode item**，一个`item`可以被同时加入多个`mode`。如果一个`mode`中一个`item`都没有，则`RunLoop`会直接退出，不进入`Loop`。

`CFRunLoop`对外暴露的管理 Mode 接口只有下面2个:

```c
CFRunLoopAddCommonMode(CFRunLoopRef runloop, CFStringRef modeName);
CFRunLoopRunInMode(CFStringRef modeName, ...);
```

Mode 暴露的管理 mode item 的接口有下面几个：

```c
CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef modeName);
CFRunLoopAddObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef modeName);

CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
CFRunLoopRemoveSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef modeName);

CFRunLoopRemoveObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef modeName);
CFRunLoopRemoveTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
```

`mode`并不像`Source/Timer/Observer`一样有 Remove 方法，所以`mode`只能增加，不能减少。你只能通过 `modeName` 来操作内部的 mode，当你传入一个新的`modeName` 但 RunLoop 内部没有对应 mode 时，RunLoop会自动帮你创建对应的 `CFRunLoopModeRef`。



### 三、内部实现

![3344530-0f941859d3fbd597](https://tva1.sinaimg.cn/large/007S8ZIlly1giettavhy8j30xc0poahf.jpg)

大致逻辑为：
1、通知observers RunLoop 即将启动。

```c
// **1.通知Observers：RunLoop即将进入loop
    if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
    //进入loop循环
    result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
```

开启一个 do while 来保活线程.

2、通知观察者即将要处理Timer事件。

```c
// 通知 Observers RunLoop 会触发 Timer 回调
if (currentMode->_observerMask & kCFRunLoopBeforeTimers)
    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeTimers);
```

3、通知观察者即将要处理source0事件。

```
// 通知 Observers RunLoop 会触发 Source0 回调
if (currentMode->_observerMask & kCFRunLoopBeforeSources)
    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeSources);
    // 执行 block
__CFRunLoopDoBlocks(runloop, currentMode);
```

4、处理source0事件。

```c
// RunLoop触发Source0(非port)回调
        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
```

5、如果基于端口的源(Source1)准备好并处于等待状态，进入步骤9。

```c
if (__CFRunLoopServiceMachPort(dispatchPort, &msg,       sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) {
                goto handle_msg;
}
```

6、通知观察者线程即将进入休眠状态。

```c
Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);
if (!poll && (currentMode->_observerMask & kCFRunLoopBeforeWaiting)) {
    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);
}
```

7、将线程置于休眠状态，由用户态切换到内核态，直到下面的任一事件发生才唤醒线程。

- 一个基于 port 的Source1 的事件。

- Timer 时间到了。

- RunLoop超时。

- 被其他调用者手动唤醒。

  ```c
  do {
      __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort) {
          // 基于 port 的 Source 事件、调用者唤醒
          if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
              break;
          }
          // Timer 时间到、RunLoop 超时
          if (currentMode->_timerFired) {
              break;
          }
  } while (1);
  ```

8、通知观察者线程将被唤醒。

```c
if (!poll && (currentMode->_observerMask & kCFRunLoopAfterWaiting))
    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopAfterWaiting);
```

9、处理唤醒时收到的事件。

- 如果用户定义的定时器启动，处理定时器事件并重启RunLoop。进入步骤2。

- 如果输入源启动，传递相应的消息。

- 如果RunLoop被显示唤醒而且时间还没超时，重启RunLoop。进入步骤2

  ```c
  handle_msg:
  // 如果 Timer 时间到，就触发 Timer 回调
  if (msg-is-timer) {
      __CFRunLoopDoTimers(runloop, currentMode, mach_absolute_time())
  } 
  // 如果 dispatch 就执行 block
  else if (msg_is_dispatch) {
      __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
  } 
  
  // Source1 事件的话，就处理这个事件
  else {
      CFRunLoopSourceRef source1 = __CFRunLoopModeFindSourceForMachPort(runloop, currentMode, livePort);
      sourceHandledThisLoop = __CFRunLoopDoSource1(runloop, currentMode, source1, msg);
      if (sourceHandledThisLoop) {
          mach_msg(reply, MACH_SEND_MSG, reply);
      }
  }
  ```

判断当前RunLoop状态值，是否停止或继续下一个loop。

```
if (sourceHandledThisLoop && stopAfterHandle) {
     // 事件已处理完
    retVal = kCFRunLoopRunHandledSource;
} else if (timeout) {
    // 超时
    retVal = kCFRunLoopRunTimedOut;
} else if (__CFRunLoopIsStopped(runloop)) {
    // 外部调用者强制停止
    retVal = kCFRunLoopRunStopped;
} else if (__CFRunLoopModeIsEmpty(runloop, currentMode)) {
    // mode 为空，RunLoop 结束
    retVal = kCFRunLoopRunFinished;
}
```

10、通知观察者RunLoop结束

```c
// **10.通知Observers：RunLoop即将退出loop
    if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);

        __CFRunLoopModeUnlock(currentMode);
        __CFRunLoopPopPerRunData(rl, previousPerRun);
    rl->_currentMode = previousMode;
    __CFRunLoopUnlock(rl);
```

整个流程如下图：

![5f51c5e05085badb689f01b1e63e1c7d](https://tva1.sinaimg.cn/large/007S8ZIlly1giettlm3udj31hc0u00x8.jpg)



## 四、应用
#### 监测卡顿
导致卡顿问题主要几个原因：
* 复杂 UI 、图文混排的绘制量过大；
* 在主线程上做网络同步请求；
* 在主线程做大量的 IO 操作；
* 运算量过大，CPU 持续高占用；
* 死锁和主子线程抢锁。

线程的消息事件是依赖于 NSRunLoop 的，所以从 NSRunLoop 入手，就可以知道主线程上都调用了哪些方法。我们通过监听 NSRunLoop 的6种状态，就能够发现调用方法是否执行时间过长，从而判断出是否会出现卡顿。

如果 RunLoop 的线程，进入睡眠前方法的执行时间过长而导致无法进入睡眠，或者线程唤醒后接收消息时间过长而无法进入下一步的话，就可以认为是线程受阻了。如果这个线程是主线程的话，表现出来的就是出现了卡顿。

所以，如果我们要利用 RunLoop 原理来监控卡顿的话，就是要关注这两个阶段。RunLoop 在进入睡眠之前和唤醒后的两个 loop 状态定义的值，分别是 `kCFRunLoopBeforeSources `（触发 Source0 回调，RunLoop将要处理一些Source事件）和 `kCFRunLoopAfterWaiting` （接收 mach_port 消息，RunLoop被唤醒，即从内核态切换到用户态），也就是要触发 Source0 回调和接收 mach_port 消息两个状态。

> 监测FPS也可以粗略监测卡顿，FPS 是一秒显示的帧数，也就是一秒内画面变化数量。
>
> 人眼在24帧就感觉还是很流畅，感受不到卡顿，所以通过FPS监控卡顿不太合适。

#### 步骤

创建一个观察者 runLoopObserver 添加到主线程 RunLoop 的 common 模式下观察。

然后，创建一个持续的子线程专门用来监控主线程的 RunLoop 状态。

一旦发现进入睡眠前的 kCFRunLoopBeforeSources 状态，或者唤醒后的状态 kCFRunLoopAfterWaiting，在设置的时间阈值内一直没有变化，即可判定为卡顿。接下来，我们就可以打印堆栈的信息。

> ```c
> long semaphoreWait = dispatch_semaphore_wait(dispatchSemaphore, dispatch_time(DISPATCH_TIME_NOW, 3 * NSEC_PER_SEC));
> ```
>
> NSEC_PER_SEC，代表的是触发卡顿的时间阈值，单位是秒,这里设置成3s。
>
> 触发卡顿的时间阈值根据 WatchDog 机制来设置.
>
> WatchDog 在不同状态下设置的不同时间，如下所示：启动（Launch）：20s；恢复（Resume）：10s；挂起（Suspend）：10s；退出（Quit）：6s；后台（Background）：3min（在 iOS 7 之前，每次申请 10min； 之后改为每次申请 3min，可连续申请，最多申请到 10min）。
>
> 通过 WatchDog 设置的时间，可以把启动的阈值设置为 10 秒，其他状态则都默认设置为 3 秒。总的原则就是，要小于 WatchDog 的限制时间。这个阈值也不用小得太多，原则就是要优先解决用户感知最明显的体验问题。



## 五、底层实现

RunLoop在没有消息处理时，休眠已避免资源占用，由用户态切换到内核态(CPU-内核态和用户态)；当有消息需要处理时，立刻被唤醒，由内核态切换到用户态。

`RunLoop`的核心是基于`mach port`的，其进入休眠时调用的函数是`mach_msg()`。

内核态：系统中既有操作系统的程序，也由普通用户的程序。为了安全和稳定性操作系统的程序不能随便访问,这就是内核态,内核态可以使用所有的硬件资源
用户态：不能直接使用系统资源，也不能改变CPU的工作状态，并且只能访问这个用户程序自己的存储空间

RunLoop这个机制是依靠系统内核来完成。

![1782258-e344f28cdf417b49](https://tva1.sinaimg.cn/large/007S8ZIlly1gietu0neptj30rw0i2acx.jpg)

> 深入了解可以参考如下文章：
>
>  [mach port使用](https://nshipster.com/inter-process-communication/)（[中文](https://segmentfault.com/a/1190000002400329)）
>
> [Mach](https://segmentfault.com/a/1190000002400329)



## 六、文章参考

> [深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)


#### Written By 多点-移动运营研发部-白迎春
