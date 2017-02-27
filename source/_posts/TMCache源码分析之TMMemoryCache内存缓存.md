---
title: TMCache源码分析<一> TMMemoryCache内存缓存
date: 2016-2-27 
categories: Objective-C
tags: [TMCache, 缓存]
---
缓存是我们移动端开发必不可少的功能, 目前提及的缓存按照存储形式来分主要分为:
- 内存缓存: 快速, 读写数据量小
- 磁盘缓存: 慢速, 读写数据量大(慢速是相对于内存缓存而言)

那缓存的目的是什么呢? 大概分为以下几点:
+ 复用数据,避免重复计算.
+ 缓解服务端压力.
+ 提高用户体验,比如离线浏览, 节省流量等等.

简言之,缓存的目的就是:
> 以空间换时间.  
  
<!---more--->
  
目前 gitHub 上开源了很多缓存框架, 著名的 [TMCache](https://github.com/tumblr/TMCache), [PINCache](https://github.com/pinterest/PINCache), [YYCache](https://github.com/ibireme/YYCache)等, 接下来我会逐一分析他们的源码实现, 对比它们的优缺点.  

[TMCache](https://github.com/tumblr/TMCache), [PINCache](https://github.com/pinterest/PINCache), [YYCache](https://github.com/ibireme/YYCache)基本框架结构都相同, 接口 API 类似, 所以只要会使用其中一个框架, 另外两个上手起来非常容易, 但是三个框架的内部实现原理略有不同.

### TMMemoryCache
`TMMemoryCache` 是 `TMCache` 框架中针对内存缓存的实现, 在系统 `NSCache` 缓存的基础上增加了很多方法和属性, 比如数量限制、内存总容量限制、缓存存活时间限制、内存警告或应用退到后台时清空缓存等功能. 并且`TMMemoryCache`能够同步和异步的对内存数据进行操作,最重要的一点是`TMMemoryCache`是线程安全的, 能够确保在多线程情况下数据的安全性.  

首先来看一下 `TMMemoryCache` 提供什么功能, 按照功能来分析它的实现原理:
1. 同步/异步的存储对象到内存中.
2. 同步/异步的从内存中获取对象.
3. 同步/异步的从内存中删除指定 key 的对象,或者全部对象.
4. 增加/删除数据, 内存警告, 退回后台的异步回调事件.
5. 设置内存缓存使用上限.
6. 设置内存缓存过期时间.
7. 内存警告或退到后台清空缓存.
8. 根据时间或缓存大小来清空指定时间段或缓存范围的数据.

### 同步/异步的存储对象到内存中
相关 API:

```Objc
// 同步
- (void)setObject:(id)object forKey:(NSString *)key;
- (void)setObject:(id)object forKey:(NSString *)key withCost:(NSUInteger)cost;

// 异步
- (void)setObject:(id)object forKey:(NSString *)key block:(TMMemoryCacheObjectBlock)block;
- (void)setObject:(id)object forKey:(NSString *)key withCost:(NSUInteger)cost block:(TMMemoryCacheObjectBlock)block;
```

#### 异步存储
首先看一下异步存储对象, 因为同步存储里面会调用异步存储操作, 采用 `dispatch_semaphore` 信号量的方式强制把异步操作转换成同步操作.  
内存缓存的核心是创建字典把需要存储的对象按照 key, value的形式存进字典中, 这是一条主线, 然后在主线上分发出许多分支, 比如:缓存时间, 缓存大小, 线程安全等, 都是围绕着这条主线来的. TMMemoryCache 也不例外, 在调用`+ (instancetype)sharedCache`方法创建并初始化的时候会创建三个可变字典`_dictionary`, `_dates`, `_costs`,这三个字典分别保存三种键值对:   
- | Key |value
-------------|-------------|-------------
_dictionary | 存储对象的 key | 存储对象的值
_dates | 存储对象的 key | 存储对象时的时间
_costs | 存储对象的 key | 存储对象所占内存大小

实现数据存储的核心方法:

```Objc  
- (void)setObject:(id)object forKey:(NSString *)key withCost:(NSUInteger)cost block:(TMMemoryCacheObjectBlock)block {
    NSDate *now = [[NSDate alloc] init];

    if (!key || !object)
        return;

    __weak TMMemoryCache *weakSelf = self;

    // 0.竞态条件下, 在并发队列中保护写操作
    dispatch_barrier_async(_queue, ^{
        TMMemoryCache *strongSelf = weakSelf;
        if (!strongSelf)
            return;

        // 1.调用 will add block
        if (strongSelf->_willAddObjectBlock)
            strongSelf->_willAddObjectBlock(strongSelf, key, object);

        // 2.存储 key 对应的数据,时间,缓存大小到相应的字典中
        [strongSelf->_dictionary setObject:object forKey:key];
        [strongSelf->_dates setObject:now forKey:key];
        [strongSelf->_costs setObject:@(cost) forKey:key];

        _totalCost += cost;

        // 3.调用 did add block
        if (strongSelf->_didAddObjectBlock)
            strongSelf->_didAddObjectBlock(strongSelf, key, object);

        // 4.根据时间排序来清空指定缓存大小的内存
        if (strongSelf->_costLimit > 0)
            [strongSelf trimToCostByDate:strongSelf->_costLimit block:nil];

        // 5.异步回调
        if (block) {
            __weak TMMemoryCache *weakSelf = strongSelf;
            dispatch_async(strongSelf->_queue, ^{
                TMMemoryCache *strongSelf = weakSelf;
                if (strongSelf)
                    block(strongSelf, key, object);
            });
        }
    });
}
```

在上面的代码中我标出了核心存储方法做了几件事, 其中最为核心的是保证线程安全的`dispatch_barrier_async`方法, 在 GCD 中称之为`栅栏`方法, 一般跟`并发队列`一起用, 在多线程中对同一资源的竞争条件下保护共享资源, 确保在同一时间片段只有一个线程`写`资源, 这是不扩展讲 GCD 的相关知识.
> dispatch_barrier_async 方法一般都是跟并发队列搭配使用,下面的图解很清晰(`侵删`), 在并发队列中有很多任务(block), 这些block都是按照 FIFO 的顺序执行的, 当要执行用 dispatch_barrier_async 方法提交到并发队列queue的 block 的时候, 该并发队列暂时会'卡住', 等待之前的任务 block 执行完毕, 再执行dispatch_barrier_async 提交的 block, 在此 block 之后提交到并发队列queue的 block 不会被执行,会一直等待 dispatch_barrier_async block 执行完毕后才开始并发执行, 我们可以看出, 在并发队列遇到 dispatch_barrier_async block 时就处于一直串行队列状态, 等待执行完毕后又开始并发执行.  
由于TMMemoryCache中所有的读写操作都是在一个 concurrent queue(并发队列)中, 所以使用 `dispatch_barrier_async` 能够保证写操作的线程安全, 在同一时间只有一个写任务在执行, 其它读写操作都处于等待状态, 这是 TMMemoryCache 保证线程安全的核心, 但也是它最大的毛病, 容易造成性能下降和死锁.
> ![dispatch_barrier_async](https://camo.githubusercontent.com/9cb07ac740e4a46fb69777e3ccd982ef23072403/687474703a2f2f63646e312e72617977656e6465726c6963682e636f6d2f77702d636f6e74656e742f75706c6f6164732f323031342f30312f44697370617463682d426172726965722d343830783237322e706e67)

从上面代码中可以看出, 在该方法中把需要存储的数据按照 key-value 的形式存储进了`_dictionary`字典中, 其它操作无非就是增加功能的配料,后面会抽丝剥茧的捋清楚, 到此处我们的任务完成, 知道是怎么存储数据的, 非常简单:
1. 使用 GCD 的 `dispatch_barrier_async` 方法保证写操作线程安全.
2. 把需要存储的数据存进可变字典中.

#### 同步存储
根据上文所说, 同步存储中会调用异步存储操作, 来看一下代码:  

```Objc
- (void)setObject:(id)object forKey:(NSString *)key withCost:(NSUInteger)cost {
    if (!object || !key)
        return;

    // 1.创建信号量
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);

    // 2.异步存数据
    [self setObject:object forKey:key withCost:cost block:^(TMMemoryCache *cache, NSString *key, id object) {
        
        // 3.异步存储完毕发送 signal 信号
        dispatch_semaphore_signal(semaphore);
    }];

    // 4.等待异步存储完毕
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);

}
```

从上面代码可以看出,同步的存储数据使用了 GCD 的 `dispatch_semaphore_t` 信号量, 这是一个非常古老又复杂的线程概念, 有兴趣的话可以看看 `<<UNIX 环境高级编程>>` 这本经典之作, 因为它的复杂是建立在操作系统的复杂性上的.但是这并不影响我们使用 dispatch_semaphore_t 信号量. 怎么使用 GCD 的信号量以及原理下面大概描述一下:
> 信号量在竞态条件下能够保证线程安全,在创建信号量 dispatch_semaphore_create 的时候设置信号量的值, 这个值表示允许多少个线程可同时访问公共资源, 就好比我们的车位一样, 线程就是我们的车子,这个信号量就是停车场的管理者, 他知道什么时候有多少车位, 是不是该把车子放进停车场, 当没有车位或者车位不足时, 这个管理员就会把司机卡在停车场外不准进, 那么被拦住的司机按照 FIFO 的队列排着队, 有足够位置的时候,管理员就方法闸门, 大吼一声: 孩子们去吧. 那么肯定有司机等不耐烦了, 就想着等十分钟没有车位就不等了,就可以在 dispatch_semaphore_wait 方法中设置等待时间, 等待超过设置时间就不等待.    
> 那么把上面的场景应用在 dispatch_semaphore_create 信号量中就很容易理解了, 创建信号量并设置最大并发线程数, dispatch_semaphore_wait 设置等待时间,在等待时间未到达或者信号量值没有达到初始值时会一直等待, 调用 dispatch_semaphore_wait 方法会使信号量的值+1, 表示增加一个线程等待处理共用资源, 当 dispatch_semaphore_signal 时会使信号量的值-1, 表示该线程不再占用共用资源.

根据上面对 dispatch_semaphore_t 信号量的描述可知, 信号量的初始值为0,当前线程执行 dispatch_semaphore_wait 方法就会一直等待, 此时就相当于同步操作, 当在并发队列中异步存储完数据调用dispatch_semaphore_signal 方法, 此时信号量的值变成0,跟初始值一样,当前线程立即结束等待, 同步设置方法执行完毕.  

其实同步实现存储数据的方式很多, 主要就是要串行执行写操作采用 dispatch_sync的方式, 但是基于 TMMemoryCache 所有的操作都是在并发队列上的, 所以才采用信号量的方式.

其实只要知道`dispatch_barrier_async`, `dispatch_semaphore_t` 的用法,后面的都可以不用看了, 自己去找源码看看就明白了.

---

休息一下吧,后面的简单了

---

### 同步/异步的从内存中获取对象.
有了上面的同步/异步存储的理论, 那么同步/异步获取对象简直易如反掌, 不就是从`_dictionary`字典中根据 key 取出对应的 value 值, 在取的过程中加以线程安全, will/did 之类辅助处理的 block 操作.

#### 异步获取数据

```Objc
- (void)objectForKey:(NSString *)key block:(TMMemoryCacheObjectBlock)block {
    NSDate *now = [[NSDate alloc] init];
    
    if (!key || !block)
        return;

    __weak TMMemoryCache *weakSelf = self;

    // 1.异步加载存储数据
    dispatch_async(_queue, ^{
        TMMemoryCache *strongSelf = weakSelf;
        if (!strongSelf)
            return;
        
        // 2.根据 key 找到value
        id object = [strongSelf->_dictionary objectForKey:key];

        if (object) {
            __weak TMMemoryCache *weakSelf = strongSelf;
            // 3.也用栅栏保护写操作, 保证在写的时候没有线程在访问共享资源
            dispatch_barrier_async(strongSelf->_queue, ^{
                TMMemoryCache *strongSelf = weakSelf;
                if (strongSelf)
                    // 4.更新数据的最后操作时间(当前时间)
                    [strongSelf->_dates setObject:now forKey:key];
            });
        }

        // 5.回调
        block(strongSelf, key, object);
    });
}
```

根据代码中注释可知,除了拿到 key 值对应的 value, 还更新了此数据最后操作时间, 这有什么用呢? 其实是为了记录数据最后的操作时间, 后面会根据这个最后操作时间来删除数据等一系列根据时间排序的操作.最后一步是回调, 我们可以看到, TMMemoryCache所有的读写和回调操作都放在同一个并发队列中,这就为以后性能下降和死锁埋下伏笔.

#### 同步获取数据

```Objc
- (id)objectForKey:(NSString *)key {
    if (!key)
        return nil;

    __block id objectForKey = nil;

    // 采用信号量强制转化成同步操作
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);

    [self objectForKey:key block:^(TMMemoryCache *cache, NSString *key, id object) {
        objectForKey = object;
        dispatch_semaphore_signal(semaphore);
    }];

    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);

    return objectForKey;
}
```

同步获取数据也是通过 `dispatch_semaphore_t` 信号量的方式,把异步获取数据的操作强制转成同步获取, 跟同步存储数据的原理相同.

### 同步/异步的从内存中删除指定 key 的对象,或者全部对象.
删除操作也不例外:

```Objc
- (void)removeObjectForKey:(NSString *)key block:(TMMemoryCacheObjectBlock)block {
    if (!key)
        return;

    __weak TMMemoryCache *weakSelf = self;

    // 1."栅栏"方法,保证线程安全
    dispatch_barrier_async(_queue, ^{
        TMMemoryCache *strongSelf = weakSelf;
        if (!strongSelf)
            return;

        // 2.根据 key 删除 value
        [strongSelf removeObjectAndExecuteBlocksForKey:key];

        if (block) {
            __weak TMMemoryCache *weakSelf = strongSelf;
            
            // 3.完成后回调
            dispatch_async(strongSelf->_queue, ^{
                TMMemoryCache *strongSelf = weakSelf;
                if (strongSelf)
                    block(strongSelf, key, nil);
            });
        }
    });
}

// private API
- (void)removeObjectAndExecuteBlocksForKey:(NSString *)key {
    id object = [_dictionary objectForKey:key];
    NSNumber *cost = [_costs objectForKey:key];

    if (_willRemoveObjectBlock)
        _willRemoveObjectBlock(self, key, object);

    if (cost)
        _totalCost -= [cost unsignedIntegerValue];

    // 删除所有跟此数据相关的缓存: value, date, cost
    [_dictionary removeObjectForKey:key];
    [_dates removeObjectForKey:key];
    [_costs removeObjectForKey:key];

    if (_didRemoveObjectBlock)
        _didRemoveObjectBlock(self, key, nil);
}

```

需要注意的是 `- (void)removeObjectAndExecuteBlocksForKey` 是共用私有方法, 删除跟 key 相关的所有缓存, 后面的删除操作还会用到此方法.

### 设置内存缓存使用上限 
TMMemoryCache 提供`costLimit`属性来设置内存缓存使用上限, 这个也是 NSCache 不具备的功能,来看一下跟此属性相关的方法以及实现,代码中有详细解释:


{% codeblock lang:Objc %}
// getter
- (NSUInteger)costLimit {
    __block NSUInteger costLimit = 0;

    // 要想通过函数返回值传递回去,那么必须同步执行,所以使用dispatch_sync同步获取内存使用上限
    dispatch_sync(_queue, ^{
        costLimit = _costLimit;
    });

    return costLimit;
}

// setter
- (void)setCostLimit:(NSUInteger)costLimit {
    __weak TMMemoryCache *weakSelf = self;

    // "栅栏"方法保护写操作
    dispatch_barrier_async(_queue, ^{
        TMMemoryCache *strongSelf = weakSelf;
        if (!strongSelf)
            return;

        // 设置内存上限
        strongSelf->_costLimit = costLimit;

        if (costLimit > 0)
            // 根据时间排序来削减内存缓存,以达到设置的内存缓存上限的目的
            [strongSelf trimToCostLimitByDate:costLimit];
    });
}

- (void)trimToCostLimitByDate:(NSUInteger)limit {
    if (_totalCost <= limit)
        return;

    // 按照时间的升序来排列 key
    NSArray *keysSortedByDate = [_dates keysSortedByValueUsingSelector:@selector(compare:)];

    // oldest objects first
    for (NSString *key in keysSortedByDate) {
        [self removeObjectAndExecuteBlocksForKey:key];

        if (_totalCost <= limit)
            break;
    }
}
{% endcodeblock %}



在`- (void)trimToCostLimitByDate:(NSUInteger)limit` 方法的作用:
1. 如果目前已使用的内存大小小于需要设置的内存上线,则不删除数据,否则删除'最老'的数据,让已使用的内存大小不超过设置的内存上限.
2. 按照存储的数据最近操作的最后时间进行升序排序,即最近操作的数据对应的 key 排最后.
3. 如果已经超过内存上限, 则根据 key 值删除数据, 先删除操作时间较早的数据.


从这里就会恍然大悟, 之前设置的 `_date` 数组终于派上用场了,如果需要删除数据则按照时间的先后顺序来删除,也算是一种优先级策略吧.

### 设置内存缓存过期时间
TMMemoryCache 提供`ageLimit`属性来设置缓存过期时间,根据上面`costLimit`属性可以猜想一下`ageLimit`是怎么实现的,既然是要设置缓存过期时间, 那么我设置缓存过期时间 `ageLimit = 10` 10秒钟,说明距离当前时间之前的10秒的数据已经过期, 需要删除掉; 再过10秒又要当前时间删除之前10秒存的数据,我们知道删除只需要找到 key 就行,所以就必须通过`_date`字典找到过期的 key, 再删除数据.由此可知需要一个定时器,每过10秒删除一次,完成一个定时任务. 
上面只是我们的猜想,来看看代码是不是这么实现的呢?我们只需看核心的操作方法

```Objc
- (void)trimToAgeLimitRecursively {
    if (_ageLimit == 0.0)
        return;

    // 说明距离现在 ageLimit 秒的缓存应该被清除掉了
    NSDate *date = [[NSDate alloc] initWithTimeIntervalSinceNow:-_ageLimit];
    [self trimMemoryToDate:date];
    
    __weak TMMemoryCache *weakSelf = self;
    
    // 延迟 ageLimit 秒, 又异步的清除缓存
    dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(_ageLimit * NSEC_PER_SEC));
    dispatch_after(time, _queue, ^(void){
        TMMemoryCache *strongSelf = weakSelf;
        if (!strongSelf)
            return;
        
        __weak TMMemoryCache *weakSelf = strongSelf;
        
        dispatch_barrier_async(strongSelf->_queue, ^{
            TMMemoryCache *strongSelf = weakSelf;
            [strongSelf trimToAgeLimitRecursively];
        });
    });
}

;``  
上面的代码验证了我们的猜想,但是在不断的创建定时器,不断的在并行队列中使用`dispatch_barrier_async`栅栏方法提交递归 block, 天啦噜...如果设置的 ageLimit 很小,可想而知性能消耗会非常大!
 
 
### 内存警告或退到后台清空缓存
内存警告和退到后台需要监听系统通知,`UIApplicationDidReceiveMemoryWarningNotification`和`UIApplicationDidEnterBackgroundNotification`, 然后执行清除操作方法`removeAllObjects`,只不过在相应的位置执行对应的 will/did 之类的 block 操作.

### 根据时间或缓存大小来清空指定时间段或缓存范围的数据
这两类方法主要是为了更加灵活的使用 TMMemoryCache,指定一个时间或者内存大小,会自动删除时间点之前和大于指定内存大小的数据.
相关 API:

```Objc
// 清空 date 之前的数据
- (void)trimToDate:(NSDate *)date block:(TMMemoryCacheBlock)block;
// 清空数据,让已使用内存大小为cost 
- (void)trimToCost:(NSUInteger)cost block:(TMMemoryCacheBlock)block;
```

删除指定时间点有两点注意:
- 如果指定的时间点为 `[NSDate distantPast]` 表示最早能表示的时间,说明清空全部数据.
- 如果不是最早时间,把`_date`中的 key 按照升序排序,再遍历排序后的 key 数组,判断跟指定时间的关系,如果比指定时间更早则删除, 即删除指定时间节点之前的数据.

```Objc
- (void)trimMemoryToDate:(NSDate *)trimDate {
    // 字典中存放的顺序不是按照顺序存放的, 所以按照一定格式排序, 根据 value 升序的排 key 值顺序, 也就是说根据时间的升序来排 key, 数组中第一个值是最早的时间的值.
    NSArray *keysSortedByDate = [_dates keysSortedByValueUsingSelector:@selector(compare:)];
    
    for (NSString *key in keysSortedByDate) { // oldest objects first
        NSDate *accessDate = [_dates objectForKey:key];
        if (!accessDate)
            continue;
        
        // 找出每个时间的然后跟要删除的时间点进行比较, 如果比删除时间早则删除
        if ([accessDate compare:trimDate] == NSOrderedAscending) { // older than trim date
            [self removeObjectAndExecuteBlocksForKey:key];
        } else {
            break;
        }
    }
}
```

### 总结
内存缓存是很简单的, 核心就是 key-value 的形式存储数据进字典,再辅助设置内存上限,缓存时间,各类 will/did block 操作, 最重要的是要实现线程安全.

欢迎大家斧正!



