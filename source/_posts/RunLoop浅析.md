---
title: RunLoop浅析
date: 2016-04-8 
categories: Objective-C
tags: [RunLoop]
---

`RunLoop`虽然在平时开发过程中使用不多, 但是是非常重要的, 往往能够解决关键性问题, 比如计时器突然不准,  页面滑动有时会卡顿等问题, 都可以用`RunLoop`来解决, 本篇文章是总结性和实践性文章, 主要做个记录, 方便自己开发中查阅.

### 什么是RunLoop?
RunLoop从字面量意思看就是一个运行循环, 其实是一个事件处理的循环, 用来不停的调度任务和处理输入事件. RunLoop 内部其实是一个do-while大循环, 在这个循环里处理输入事件, 比如:点击事件, 滑动屏幕, 定时器等, 当处理完一个任务后RunLoop进入休眠, 有任务时又唤醒RunLoop处理事件.

简言之, `RunLoop是管理线程各类输入事件的对象`.

<!---more--->

基本作用:
1. `保持程序的持续运行`. 如果线程中没有RunLoop, 线程执行完任务队列中的任务后, 就会退出. 所以app的主线程必定有一个RunLoop, 一直让主线程处理不退出状态, 除非系统或者手动让RunLoop停止工作, 此时主线程退出, app挂掉.
2. `处理app中的各类输入事件`. 比如:点击事件,触摸事件, 方法调用(seletor)事件, 定时器等, RunLoop就是一直在等待处理这些事件.
3. `节省CPU资源`. 虽然RunLoop 是一个大循环, 但是不同于while(1)死循环, 当有任务的事件会`唤醒`RunLoop执行任务, 没有任务时`休眠`RunLoop.

在我们程序的main函数中, 返回`UIApplicationMain`方法调用的返回值, 这个方法程序运行时是不会被返回的, 因为在这个方法内部会创建一个RunLoop, 维持主线程的持续运行.

```Objc
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([CZPAppDelegate class]));
    }
}
```

类似于下面的伪代码:

```Objc
int main(int argc, char * argv[]) {
    BOOL isRunning = YES;
    
    do {
        // 通知观察者告知RunLoop的状态
        // ...
        // 处理各类事件
        // ...
        // 休眠, 等待被唤醒
        // 唤醒
        
        // 各类条件是否满足??
        BOOL conditions;
        if (conditions) {
            isRunning = YES;
        }else {
            isRunning = NO;
        }
        
    } while (isRunning);
    
    return 0;
}
```

### RunLoop对象
RunLoop是管理线程输入事件的对象

- `Core Foundation`框架中使用 `CFRunLoopRef`. 是纯C写的代码, 所以是线程安全的.
- 在`Foundation`框架中使用`NSRunLoop`, 是封装的 `CFRunLoopRef`中, 不是线程安全的. 两者都代表RunLoop对象, 是可以等价转换的. 

要想了解RunLoop很有必要知道底层的实现, 苹果公司开源了这个部分代码, 其中跟RunLoop相关的就两个文件: `CFRunLoop.h`, `CFRunLoop.c`.

[CFRunLoopRef开源代码下载地址](http://opensource.apple.com/source/CF/CF-1151.16/)
http://opensource.apple.com/source/CF/CF-1151.16/

获取 RunLoop 对象

1. 获取当前 RunLoop 对象:

    ```Objc
    // NSRunloop
     NSRunLoop * runloop = [NSRunLoop currentRunLoop];
     
    // CFRunLoopRef
    CFRunLoopRef runloop =   CFRunLoopGetCurrent();
    ```
2. 获取主线程的 RunLoop 对象:
    
    ```Objc
    // NSRunloop
     NSRunLoop * runloop = [NSRunLoop mainRunLoop];
    
    // CFRunLoopRef
     CFRunLoopRef runloop =   CFRunLoopGetMain();
    ```
注意:
> 创建 RunLoop 对象不是通过 alloc init 的方式创建的, 是直接获取, 没有的话就会创建. 主线程中也可以通过方式一来获取 RunLoop 对象.

### RunLoop 与线程
线程和RunLoop的关系:

- 每条线程都有唯一的一个RunLoop 对象.
- 主线程的RunLoop在程序启动时自动创建好了, 子线程的需要手动创建.
- RunLoop 在第一次获取时创建, 线程销毁时销毁.

线程和RunLoop的对应关系保存在一个全局的字典中, 通过key-value保持. 下面是CFRunLoop.m中的源代码, 无论是主线程还是子线程在获取RunLoop的时候会调用`_CFRunLoopGet0`函数, 该函数判断该线程RunLoop对象, 有则返回, 没有则创建并保存.

```C
// 创建字典
CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
// 创建主线程runloop
CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
// 保存主线程runloop
CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);

// 从字典中获取子线程的runloop
CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
__CFUnlock(&loopsLock);
if (!loop) {
// 如果子线程的runloop不存在,那么就为该线程创建一个对应的runloop
CFRunLoopRef newLoop = __CFRunLoopCreate(t);
__CFLock(&loopsLock);
loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
// 把当前子线程和对应的runloop保存到字典中
if (!loop) {
CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
loop = newLoop;
}
```
### RunLoop 运行原理

Runloop运行原理图中可以看到, 一个线程中的RunLoop接受 source 源的输入事件和 timer 定时器事件, 当有这些事件发生时, 就会唤醒 RunLoop 执行相应任务.

![运行原理图](http://oixseublm.bkt.clouddn.com/2.png)

### RunLoop 相关类

有5个相关类:

- `CFRunloopRef`
- `CFRunloopModeRef`: Runloop的运行模式
- `CFRunloopSourceRef`: Runloop要处理的事件源
- `CFRunloopTimerRef`: Timer事件
- `CFRunloopObserverRef`: Runloop的观察者（监听者）

Runloop和相关类之间的关系图:

![关系图](http://oixseublm.bkt.clouddn.com/1.png)

`CFRunloopModeRef`代表RunLoop的运行模式, 每次运行只能处于一个模式下, 每个mode下有很多source, timer, observer, 如果要切换模式, 只能退出当前循环, 重新指定一个新的mode后再次进入循环.  
RunLoop 这么设计其实是为了避免各个mode下的source, timer, observer相互影响.

`CFRunloopModeRef`有五种mode:

- `kCFRunLoopDefaultMode`: App的默认Mode，通常主线程是在这个Mode下运行.
- `UITrackingRunLoopMode`: 界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响.

- `UIInitializationRunLoopMode`: 在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用.
- `GSEventReceiveRunLoopMode`: 接受系统事件的内部 Mode，通常用不到
- `kCFRunLoopCommonModes`: 这是一个占位用的Mode，不是一种真正的Mode. 


`CFRunloopSourceRef`分为两种:

- source0: 非基于Port的, 把事件告诉RunLoop, 需要手动激活.
- Source1: 通过系统内核来唤醒.
- 可以通过打断点的方式查看一个方法的函数调用栈

`CFRunLoopObserverRef`观察者,监听RunLoop的状态:

监听的状态是枚举:

```Objc
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),           //即将进入Runloop
    kCFRunLoopBeforeTimers = (1UL << 1),    //即将处理NSTimer
    kCFRunLoopBeforeSources = (1UL << 2),   //即将处理Sources
    kCFRunLoopBeforeWaiting = (1UL << 5),   //即将进入休眠
    kCFRunLoopAfterWaiting = (1UL << 6),    //刚从休眠中唤醒
    kCFRunLoopExit = (1UL << 7),            //即将退出runloop
    kCFRunLoopAllActivities = 0x0FFFFFFFU   //所有状态改变
};
```

给RunLoop添加监听者:

```objc
//创建一个runloop监听者
CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(CFAllocatorGetDefault(),kCFRunLoopAllActivities, YES, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {

   NSLog(@"监听runloop状态改变---%zd",activity);
});

//为runloop添加一个监听者
CFRunLoopAddObserver(CFRunLoopGetCurrent(), observer, kCFRunLoopDefaultMode);

// 必须释放
CFRelease(observer);
```

`CFRunloopTimerRef`是定时器事件, `NSTimer`是基于`CFRunloopTimerRef`的封装, 是基于时间的触发器, 需要把timer加入到RunLoop里面去, RunLoop会在相应的时间点注册事件, 等时间到了触发唤醒RunLoop执行事件.  

定时器在开发中使用广泛, 但是很容易犯错, 有两种方式设置定时事件, `NSTimer`和`GCD`, 两者的定时器不同, `GCD`的更加精准, 不受RunLoop的影响. 而且RunLoop里面设置循环过期时间也是用的`GCD`的定时器, 由此可知, RunLoop 会使用 GCD 的部分功能.

注意:
> 使用timer 的时候一定要注意添加到哪种mode模式下, 一般标记为`NSRunLoopCommonModes`下, 也就是说任何模式下都可以正常运行定时器, 否则只能特定模式下运行, 当RunLoop切换模式后, 定时器不正常工作.

#### NSTimer + RunLoop用法:

```Objc
- (void)timer2 {
   //NSTimer 调用了scheduledTimer方法，那么会自动添加到当前的runloop里面去，而且runloop的运行模式kCFRunLoopDefaultMode
    
   NSTimer *timer = [NSTimer scheduledTimerWithTimeInterval:2.0 target:self selector:@selector(run) userInfo:nil repeats:YES];
    
   //更改模式
   [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
}
    
- (void)timer1 {
    NSTimer *timer = [NSTimer timerWithTimeInterval:2.0 target:self selector:@selector(run) userInfo:nil repeats:YES];
   
   // 定时器添加到UITrackingRunLoopMode模式，一旦runloop切换模式，那么定时器就不工作
   // [[NSRunLoop currentRunLoop] addTimer:timer forMode:UITrackingRunLoopMode];
   
   // 占位模式：common modes标记
   // 被标记为common modes的模式 kCFRunLoopDefaultMode  UITrackingRunLoopMode
   [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
   
   // NSLog(@"%@",[NSRunLoop currentRunLoop]);
}
    
- (void)run {
   NSLog(@"---run---%@", [NSRunLoop currentRunLoop].currentMode);
}
    
- (IBAction)btnClick {
   NSLog(@"---btnClick---");
}
```
    
还有一种办法是添加两次:

```Objc
[[NSRunLoop currentRunLoop] addTimer:timer forMode:UITrackingRunLoopMode];

[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

#### GCD 定时器用法:

```objc
//0.创建一个队列
dispatch_queue_t queue = dispatch_get_global_queue(0, 0);

//1.创建一个GCD的定时器
dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);

//2.设置定时器的开始时间，间隔时间以及精准度
dispatch_time_t start = dispatch_time(DISPATCH_TIME_NOW,3.0 *NSEC_PER_SEC);
//设置定时器工作的间隔时间
uint64_t intevel = 1.0 * NSEC_PER_SEC;

/*
第四个参数：定时器的精准度，如果传0则表示采用最精准的方式计算，如果传大于0的数值，则表示该定时切换i可以接收该值范围内的误差，通常传0
该参数的意义：可以适当的提高程序的性能
注意点：GCD定时器中的时间以纳秒为单位
*/
dispatch_source_set_timer(timer, start, intevel, 0 * NSEC_PER_SEC);

//3.设置定时器开启后回调的方法
dispatch_source_set_event_handler(timer, ^{
   NSLog(@"------%@", [NSThread currentThread]);
});

//4.执行定时器
dispatch_resume(timer);

//注意：dispatch_source_t本质上是OC类，在这里是个局部变量，需要强引用
self.timer = timer;
```

### RunLoop运行逻辑
运行逻辑就是一个do-while大循环, 在这个循环里面处理事件, 监听RunLoop的状态, 不停的`休眠-唤醒-处理-休眠`这个过程.

![处理过程](http://oixseublm.bkt.clouddn.com/3.png)

![逻辑图](http://oixseublm.bkt.clouddn.com/4.png)

### RunLoop应用
下面是开发中经常使用RunLoop的地方:
#### NSTimer

```Objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    NSTimer *timer = [NSTimer timerWithTimeInterval:3 repeats:YES block:^(NSTimer * _Nonnull timer) {
        NSLog(@"---timer block---");
    }];
    
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
}
```

控制台输出:

```
2017-02-28 11:17:46.102 test[2831:93953] ---timer block---
2017-02-28 11:17:49.102 test[2831:93953] ---timer block---
2017-02-28 11:17:52.102 test[2831:93953] ---timer block---
2017-02-28 11:17:55.102 test[2831:93953] ---timer block---
2017-02-28 11:17:58.102 test[2831:93953] ---timer block---
2017-02-28 11:18:01.175 test[2831:93953] ---timer block---
```

#### ImageView显示

```Objc
[self.imageView performSelector:@selector(setImage:) withObject:[UIImage imageNamed:@"abc"] afterDelay:2.0 inModes:@[NSDefaultRunLoopMode,UITrackingRunLoopMode]];
```
#### PerformSelector

有很多 PerformSelector 方法都是需要制定mode, date的, 正如上面所示.

当调用 NSObject 的 performSelecter:afterDelay: 后，实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。

当调用 performSelector:onThread: 时，实际上其会创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 RunLoop 该方法也会失效。

#### 常驻线程
有时需要在后台开启一个线程持续的做任务, 比如搜集数据, 语音唤起等, 无非就是在一个线程中开启一个RunLoop 让它维持线程的持续运行, 而不是执行完任务后退出.
AFN框架中也是用了:

```Objc
+ (void)networkRequestThreadEntryPoint:(id)__unused object {
    @autoreleasepool {
        [[NSThread currentThread] setName:@"AFNetworking"];
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}
 
+ (NSThread *)networkRequestThread {
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });
    return _networkRequestThread;
}
```

注意:
> 子线程的RunLoop里面至少要有一个source或者是timer, 只有observer不行的.

#### 自动释放池
自动释放池的第一次创建:
`第一次进入RunLoop时候自动创建`.

自动释放池的第一次销毁:
`RunLoop即将进入休眠的时候`

其它情况下创建和销毁:
`唤醒RunLoop的时候会创建新的自动释放池`
`RunLoop销毁时销毁自动释放池`

打印RunLoop信息可以看出:

```
_wrapRunLoopWithAutoreleasePoolHandler  activities = 0x1  1
_wrapRunLoopWithAutoreleasePoolHandler activities = 0xa0 160
```
160 是 kCFRunLoopBeforeWaiting + kCFRunLoopExit



