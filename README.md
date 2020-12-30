# Performance

这是一个性能检测库，可以检测以下问题

- [x] ANR
- [x] FPS
- [x] 线程和线程池的监控
- [x] IPC(进程间通讯)
- [x] 主线程耗时任务检测

同时还计划实现以下功能

- [x] 实时通过 logcat 打印问题
- [ ] 高效保存检测信息到本地
- [ ] 提供上报到指定服务器接口

# 接入指南

1 Project 根目录下的 build.gradle 文件下添加如下内容

```groovy
buildscript {

  dependencies {

    classpath "com.xander.performance.plugin:performance:0.1.3"

  }

}
```

2 在 APP 工程目录下面的 build.gradle 添加如下内容

```groovy
apply plugin: "com.xander.performance.performance"

dependencies {

  debugImplementation "com.xander.performance:p-tool:0.1.4"
  releaseImplementation "com.xander.performance:p-tool-noop:0.1.4"
}
```

3 APP 工程的 Application 类新增类似如下初始化代码

```java

    private void initPerformanceTool(Context context) {
        pTool.init(
            new pTool.Builder()
                .globalTag("pTool") // 全局 log 日志 tag ，可以快速过滤日志
                .checkUIThread(true, 500) // 检查 ui 线程, 超过指定时间未响应，会被认为 ui 线程 delay
                .checkThread(false) // 检查线程和线程池的创建
                .checkFps(true) // 检查 Fps
                .checkIPC(false) // 检查 IPC 调用
                .checkHandlerCostTime(100) // 检查 Handler 处理 message 的时间, 超过时间指定时间，会被捕获并打印
                .appContext(context)
                .build()
        );
    }

```

# 原理介绍

## 检测 ANR 的原理

主要参考了 ANR-WatchDog 的思路，利用一个线程，先向主线程投放一个 msg 。
然后 sleep 指定的时间，时间到了之后，检测这个 msg 是否被处理过，
如果被处理过，说明这段时间内没有阻塞。
如果这个 msg 没有被处理，那么说明这段时间内有阻塞，可能发生了 ANR 。
如果发生了 ANR 可以把主线程的调用栈打印出来，作为一个 ANR 问题的 log 信息参考。

检测完后，这个线程继续投放下一个 msg ，然后重复做之前的检测。这样就可以监控 ANR 是否发生了。

但是这个检测有个问题，就是打印的堆栈不一定是指定的时间段内最耗时的堆栈，
这个时候，可以考虑缩短检测时间段，多次采样来提高准确率。

## 检测 FPS 的原理

FPS 检测的原理，利用了 Android 的屏幕绘制原理。

这里简单说下 Android 的屏幕绘制原理。

系统每隔 16 ms 就会发送一个 VSync 信号，告诉应用，该开始准备绘制画面了。
如果准备顺利，也就是 cpu 准备好数据， gpu 栅格化完成。如果这些任务在 16 ms 之内完成，
那么下一个 VSync 信号到来的时候就可以绘制这一帧界面了。这个准备好的画面就会被显示出来。
如果没准备好，可能就需要 32 ms 后或者更久的时间后，才能准备好，这个画面才能显示出来，
这种情况下就发生了丢帧。

上面提到了 VSync 信号，当 VSync 信号到来的时候会通知应用开始准备绘制，
具体的通知细节不做表述，大概就是用到了 Handler ，往 MessageQueue 里面放
一个异步屏障，然后注册 VSync 信号监听，当 VSync 信号到达的时候，发送一个异步 msg 
来通知应用开始 measure layout and  draw。

检测 FPS 的原理其实挺简单的，就是通过一段时间内，
比如 1s，统计绘制了多少个画面，就可以计算出 FPS 。

那如何知道应用 1s 内绘制了多少个界面呢？这个就要靠 VSync 信号监听了。假设 1s 内，
应用没有做任何绘制，我们通过 Choreographer 注册 VSync 信号监听。 16ms 后，
我们收到了 VSync 的信号，我们不做特别处理，只是做一个计数，然后监听下一次的 VSync 
信号，这样，我们就可以知道 1s 内我们监听到了多少个 VSync 信号，就可以得出帧率。

为什么监听到的 VSync 信号数量就是帧率呢？由于 looper 处理 msg 是串型的，
一次只能处理一个 msg ，而绘制的时候，绘制任务 msg 是异步消息，
会优先执行，绘制任务 msg 执行完成后，才会执行上面说的统计 VSync 的任务，
所以 VSync 信号数量可以认为是某段时间内绘制的帧数。然后就可以通过这段时间的长度
和 VSync 信号数量来计算帧率了。


## 线程和线程池的监控的原理

线程和线程池的监控，主要是监控线程和线程池在哪里创建和执行的，如果我们可以知道这些信息，
我们就可以比较清楚线程和线程池的创建和启动时机是否合理。从而得出优化方案。

一个比较容易想到的方法就是，应用代码里面的所有线程和线程池继承同一个线程基类和线程池基类。
然后在构造函数和启动函数里面打印方法调用栈，这样我们就知道哪里创建和执行了线程或者线程池。

让应用所有的线程和线程池继承同一个基类，可以通过编译插件来实现，定制一个特殊的 Transform ，
通过 ASM 编辑生成的字节码来改变继承关系。但是，这个方法有一定的上手难度，不太适合新手。

除了这个方法，我们还有另外一种方法，就是 `hook` 。通过 hook 线程或者线程池的构造方法和启动方法，
在原本调用每个线程和线程池的实例的构造方法和启动方法的时候，实际调用 hook 后的方法。hook 后的
方法，我们可以先打印方法调用栈，然后继续 hook 之前的逻辑，这样我们也可以知道线程和线程池的
创建和启动了。

本项目采用的 hook 方案是 epic 库，由于一些原因，本人对 epic 库做了一点小修改，主要是
注释了一个 log 的打印，因为本人在实际操作的时候，发现在 hook 线程池的构造方法的时候，
epic 框架调用线程池的 toString() 方法打印调试信息，但是线程池的 toString() 方法里面
会使用线程池的一个 final 的属性，由于这个属性没有初始化，epic 框架报错了，
所以我就找了下 epic 的源码，注释掉了调试信息的打印，其他的未作任何修改。

由于线程池会创建线程，所以线程池和其创建的线程需要关联下，否则会干扰我们分析线程的创建问题。
我是通过如下的方法来创建关联的。

线程池执行 task 的时候，如果需要创建线程，会先创建一个内部类 Worker 的实例，由于是内部类，
Worker 实际的构造函数会传入外部类的一个实例，也就是线程池的实例，通过这个关系，我们可以
知道 Worker 和线程池的关联。然后 Worker 里面创建线程池的线程的时候会把 Worker 实例
作为 Runnable 传入线程，这样 Worker 里面创建的线程和 Worker 就对应上了，然后
通过 Worker 的关联，就把线程池和线程关联上了。

之前看到一个方案是通过线程的 ThreadGroup 来建立关联，本来我也是打算按照这个关系来关联的，
但是我发现，我们工程里面的代码，线程池在创建线程的时候，并没有传入 ThreadGroup ，
这种情况下，就统计的不准确了。所以用了上面的方法来建立线程和线程池的关联。

## 检测 IPC(进程间通讯)的原理

进程间通讯的具体原理，也就是 Binder 机制，这里不做详细的说明，也不是这个框架库的原理。

检测进程间通讯的方法和前面检测线程的方法类似，就是找到所有的进程间通讯的方法的共同点，然后
对共同点做一些修改或者说切片，让应用在进行进程间通讯的时候，打印一下调用栈，然后继续做原来
的事情。

那如何找到共同点，或者说切片，就是本节的重点。进程间通讯离不开 Binder ，需要从 Binder 入手。

写一个 AIDL demo 后发现，自动生成的代码里面接口 A 继承自 IInterface 接口，然后接口里面有个
内部抽象类 Stub 类，继承自 Binder ，实现了接口 A。这个 Stub 类里面还有一个内部类 Proxy ，实现了
接口 A ，并持有一个 IBinder 实例和一个接口 A 的实例。

我们在使用 AIDL 的时候，会用到 Stub 类的 asInterFace 的方法，这个方法会新建一个 Proxy 实例，
并传入 IBinder , 或者传入的如果是接口 A 的话，就强制转化为接口 A 实例。不管是新建的 Proxy 实例
还是强转的实例，反正最后都会调用接口方法，最终还是要调用接口 A 定义的方法，查看 Proxy 类里面的
方法，可以看到，最终调用了 IBinder 的 transact 方法。

到这里，这个共同的方法，或者说切片就找到了。所以我们只需要 hook 住 Binder 的 transact 方法，就可以
知道应用里面的进程间调用了。

## 主线程耗时任务检测的原理

这个的话，和 ANR 检测有点相关，都是用到了 Handler ，
主线程如果耗时了，就会导致界面卡顿。AndroidPerformanceMonitor(BlockCanary) 
虽然能检测到哪些任务耗时，但是无法检测到时谁，在哪里往主线程放了这个耗时的任务。

通过对 Handler 的分析发现，往主线程放入耗时操作，
一定会调用 Handler.sendMessageAtTime 方法，如果我们在这个方法里面记录调用堆栈，
然后在 Handler.dispatchMessage 方法里面统计耗时，超过阈值的任务，
打印之前记录的放入这个 msg 的调用堆栈，我们就可以知道是谁在哪里往主线程里面放入了这个耗时的任务。