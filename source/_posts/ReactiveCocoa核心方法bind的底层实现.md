---
title: ReactiveCocoa核心方法bind的底层实现
date: 2016-05-01 23:35:21
tags: [Objective-C]
---

bind 是 ReactiveCocoa的核心方法, 顾名思义:绑定, 跟以往的赋值不同.
RAC 封装了很多方法, 底层都是用的 bind, 平常 bind 方法基本不用, 但是理解原理后其它方法很容易理解了, 比如: map, flattenMap等.

文章目录
1. 使用 bind 方法
2. bind 的实现原理
3. 总结

<!---more--->

### 使用 bind 方法

```Objc
typedef RACSignal * _Nullable (^RACSignalBindBlock)(ValueType _Nullable value, BOOL *stop);
- (RACSignal *)bind:(RACSignalBindBlock (^)(void))block;
```

上面是 bind 方法的声明, bind 方法只接收一个返回值是RACSignalBindBlock类型的无参 block. RACSignalBindBlock也是一个 block, 携带两个参数value和 stop, 返回值是 RACSignal类型的信号.
下面代码实现一下:

```Objc
- (void)testBind {
    // 创建源信号
    RACSignal *sourceSignal = [RACSignal createSignal:^RACDisposable * _Nullable(id<RACSubscriber>  _Nonnull subscriber) {
        
        [subscriber sendNext:@"hello, "];
        [subscriber sendCompleted];
        
        return [RACDisposable disposableWithBlock:^{
            NSLog(@"源信号被销毁, 这里进行收尾工作...");
        }];
        
    }];
    
    // 进行bind, 返回绑定信号
    RACSignal *bindSignal = [sourceSignal bind:^RACSignalBindBlock _Nonnull{
        
        RACSignalBindBlock bindBlock = ^RACSignal*(id value, BOOL *stop) {
            RACSignal *returnSignal = [RACSignal createSignal:^RACDisposable * _Nullable(id<RACSubscriber>  _Nonnull subscriber) {
                
                // 修改值
                NSString *newValue = [NSString stringWithFormat:@"%@%@", value, @"ReacticeCocoa bind"];
                
                [subscriber sendNext:newValue];
                
                return [RACDisposable disposableWithBlock:^{
                    NSLog(@"返回信号被销毁, 这里进行收尾工作...");
                }];
                
            }];
            
            return returnSignal;
        };
        
        return bindBlock;
    }];
    
    
    // 订阅绑定信号
    [bindSignal subscribeNext:^(id  _Nullable x) {
        NSLog(@"接收到信号传送的值: %@", x);
    } completed:^{
        NSLog(@"信号发送完毕");
    }];
}
```
console输出:

```
2016-05-01 18:18:20.874 ReactiveCocoaDemo[74310:880621] 接收到信号传送的值: hello, ReacticeCocoa bind
2016-05-01 18:18:20.875 ReactiveCocoaDemo[74310:880621] 返回信号被销毁, 这里进行收尾工作...
2016-05-01 18:18:20.875 ReactiveCocoaDemo[74310:880621] 源信号被销毁, 这里进行收尾工作...
```

从上面的结果可以看出,原本发送的”hello,”字符串,在 bind 方法中获取到发送的值, 进行修改重新发送一个新值 “hello,ReactiveCocoa bind”, 最后订阅 bindSignal,得到修改后的值.
bind是 ReactiveCocoa信号操作的核心,能够拿到上次信号发送的值,进行操作,传递给下一个信号发送,从基本的 map 方法就能体会.

使用 bind的时候会涉及到三个信号:

暂且命名为: 源信号(sourceSignal), 绑定信号(bindSignal), 返回信号(returnSignal)

那么这个三个信号有什么作用呢?
源信号(signal): 提供数据源的信号, 通过这个信号的订阅者发送原始数据.
绑定信号(sourceSignal): 通过订阅它来接收处理后的数据.
返回信号(returnSignal): 很明显要想形成一个完整的流程缺少一个”桥梁”, 缺少发送处理后的数据的信号, 即通过返回信号能够发送处理后的数据.

下面会讲解三者是怎么配合的.

### bind 的实现原理

进入 bind 方法的实现可以非常清晰的了解到 bind 的使用以及原理:
> -bind: should:
1. Subscribe to the original signal of values.
2. Any time the original signal sends a value, transform it using the binding block.
3. If the binding block returns a signal, subscribe to it, and pass all of its values through to the subscriber as they’re received.
4. If the binding block asks the bind to terminate, complete the original signal.
5. When all signals complete, send completed to the subscriber.
If any signal sends an error at any point, send that to the subscriber.

#### 创建绑定信号
调用 bind 方法会新创建一个 RACDynamicSignal信号, 这个信号就是绑定信号(bindSignal), 我们订阅绑定信号(bindSignal)就会执行绑定信号的 didSubscribe block.

```Objc
- (RACSignal *)bind:(RACSignalBindBlock (^)(void))block {
    NSCParameterAssert(block != NULL);
    return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
        
        // ......
        
    }] setNameWithFormat:@"[%@] -bind:", self.name];
}
```

按照调用步骤:
#### 订阅绑定信号(bindSignal)

```Objc
// 订阅绑定信号
[bindSignal subscribeNext:^(id  _Nullable x) {
	NSLog(@"接收到信号传送的值: %@", x);
}];
```
执行绑定信号(bindSignal)的didSubscribe block:

```Objc
RACDisposable *bindingDisposable = [self subscribeNext:^(id x) {
        // Manually check disposal to handle synchronous errors.
        if (compoundDisposable.disposed) return;
        
        BOOL stop = NO;
        id signal = bindingBlock(x, &stop);
        
        @autoreleasepool {
            if (signal != nil) addSignal(signal);
            if (signal == nil || stop) {
                [selfDisposable dispose];
                completeSignal(selfDisposable);
            }
        }
    } error:^(NSError *error) {
        [compoundDisposable dispose];
        [subscriber sendError:error];
    } completed:^{
        @autoreleasepool {
            completeSignal(selfDisposable);
        }
    }];
```
注意self调用了 subscribeNext 方法, 这个 self 是 源信号(sourceSignal), 也就说, 在订阅绑定信号的时候会订阅源信号,此时就是调用原信号的 didSubscribe block 发送数据:

```Objc
[subscriber sendNext:@"hello, "];
```

此时 x 的值为 “hello, “.

#### 调用 bindBlock, 得到返回信号(returnSignal)

```Objc
id signal = bindingBlock(x, &stop);
```

3.调用 addSignal block:

```Objc
if (signal != nil) 
	addSignal(signal);

void (^addSignal)(RACSignal *) = ^(RACSignal *signal) {
        OSAtomicIncrement32Barrier(&signalCount);
        
        RACSerialDisposable *selfDisposable = [[RACSerialDisposable alloc] init];
        [compoundDisposable addDisposable:selfDisposable];
        
        RACDisposable *disposable = [signal subscribeNext:^(id x) {
            [subscriber sendNext:x];
        } error:^(NSError *error) {
            [compoundDisposable dispose];
            [subscriber sendError:error];
        } completed:^{
            @autoreleasepool {
                completeSignal(selfDisposable);
            }
        }];
        
        selfDisposable.disposable = disposable;
    };
```
在这个 block 中会订阅返回信号,那么就会调用返回信号的 didSubscribe block, 在这个 block中对原始数据进行处理, 然后发送处理后的数据, 也就是:

```Objc
// 修改值
NSString *newValue = [NSString stringWithFormat:@"%@%@", value, @"ReacticeCocoa bind"];[subscriber sendNext:newValue];
                
return [RACDisposable disposableWithBlock:^{
           NSLog(@"返回信号被销毁, 这里进行收尾工作...");
       }];
```

#### 发送处理后数据

那么此时在 addSignal block中调用:

```Objc
[subscriber sendNext:x];
```

此时 x 的值就是经过返回信号处理后的数据,即”hello,ReactiveCocoa bind”. 注意此处的subscriber是绑定信号的订阅者, 使用绑定信号订阅者来发送数据,那么我们外面订阅绑定信号的地方就能收到数据:

```Objc
// 订阅绑定信号
[bindSignal subscribeNext:^(id  _Nullable x) {
    NSLog(@"接收到信号传送的值: %@", x);
}];
```

至此一个完成的发送数据流程走完, 源信号发送原始数据, 通过bind 方法传入返回信号,利用返回信号来处理原始数据,发送处理后的新数据, 在 bind 内部订阅返回信号,收到处理后的新数据, 使用绑定信号的订阅者来发送新数据, 那么我们在外面订阅绑定信号就能收到处理后的新数据.

### 总结一下

- bind涉及到三个信号: 源信号(signal), 绑定信号(bindSignal), 返回信号(returnSignal).
- 通过这三个信号能做什么: 发送源数据, 接收处理后的数据, 发送处理后的数据.
- 内部实现流程:
  + 创建绑定信号, 保存didScribeBlock, 只要外面订阅绑定信号就会执行block.
  + didScribeBlock中拿到返回处理数据的 block, bindingBlock.
  + 订阅源信号(signal), 拿到外面发送的源数据, 执行 bindingBlock处理源数据, 得到返回信号(returnSignal).
  + 订阅返回信号(returnSignal), 外面发送处理后的数据, 内部拿到处理后的数据, 拿绑定信号(bindSignal)的订阅者发送数据.
  + 外部订阅绑定信号的地方, 就能获取到处理的数据了, 完成的一个 Hook过程.

