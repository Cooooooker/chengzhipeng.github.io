---
title: TMCache源码分析<二> TMDiskCache磁盘缓存
tags:
---
[上篇分析](http://www.jianshu.com/p/a8a45c12d2d2)了 `TMCache`中内存缓存`TMMemoryCache`的实现原理, 这篇文章将详细分析磁盘缓存的实现原理.  

磁盘缓存,顾名思义:将数据存储到磁盘上,由于需要储存的数据量比较大,所以一般读写速度都比内存缓存慢, 但也是非常重要的一项功能, 比如能够实现离线浏览等提升用户体验.  

磁盘缓存的实现形式大致分为三种:
- 基于文件读写.
- 基于数据库.  
- 基于 mmap 文件内存映射.

前面两种使用的比较广泛, `SDWebImage`和`TMDiskCache`都是基于文件 I/O 进行存储的, 也就是一个 value 对应一个文件, 通过读写文件来缓存数据. 根据上篇可以知道`TMMemoryCache`内存缓存的主线是按照 key-value的形式把数据存进可变字典中, 那么磁盘缓存的主线也是按照 key-value的形式进行对应的, 只不过 value 对应的是一个文件, 换汤不换药.  

通过`TMDiskCache`的接口 API 可以看到, `TMDiskCache`提供以下功能:  
- 同步/异步的进行读写数据.
- 同步/异步的进行删除数据.
- 同步/异步的获取缓存路径.
- 同步/异步的根据缓存时间或者缓存大小来削减磁盘空间.
- 设置磁盘缓存空间上限, 磁盘缓存时间上限.
- 各类 will / did block, 以及监听后台操作.
- 清空临时存储区.

`TMDiskCache`的同步操作是跟`TMMemoryCache`操作一样,都是采用`dispatch_semaphore_t`信号量的形式来强制把异步转成同步操作,后面同步操作就一步带过,除非特别说明.   其实`TMDiskCache`的难点不在于线程安全,因为它所有的操作都在一个 serial queue 串行队列中, 不存在竞态情况, 难点在于文件的操作, 了解 Linux 文件系统操作的同学应该知道文件 I/O 的概念, iOS 封装了操作文件的类, 使用这些高级 API 能更好的操作文件.

### 初始化方法
在操作之前先看一下`TMDiskCache`的初始化方法, 提供一个类方法, 两个实例方法:

```
+ (instancetype)sharedCache;

- (instancetype)initWithName:(NSString *)name;
- (instancetype)initWithName:(NSString *)name rootPath:(NSString *)rootPath;
```

从名字应该能猜测出最终调用的方法应该是`- (instancetype)initWithName:(NSString *)name rootPath:(NSString *)rootPath`, 传磁盘缓存所在目录的名字和绝对路径, 如果调用前两个方法,在方法内部将默认设置好路径或者缓存文件夹名字. 我们主要看终极方法主要做了几件事:
- 创建串行队列,是单例,即一个单例缓存对象有一个单例串行队列.
- 初始化两个可变字典`_dates`, `_sizes`, 分别用于存数据最后操作时间和数据占用磁盘空间大小.
- 创建缓存文件, 设置缓存文件操作时间.

其余的比较简单, 这里主要说一下`设置缓存文件操作时间`的相关 API, 首先是处理 key 的方法, 这两个方法分别对传入的 key 进行编码和解码, 比如在调用`setObject:forKey:`的时候 key 值传入了中文字符, 就会调用`encodedString`和`decodedString`来编解码, 可以进入沙盒中看到对应的缓存文件名字是这类编码后的字符, 形如:`%E7%A8%8B%E5%85%88%E7%94%9F`.

```
- (NSString *)encodedString:(NSString *)string {
    if (![string length])
        return @"";
    
    CFStringRef static const charsToEscape = CFSTR(".:/");
    CFStringRef escapedString = CFURLCreateStringByAddingPercentEscapes(kCFAllocatorDefault,
                                                                        (__bridge CFStringRef)string,
                                                                        NULL,
                                                                        charsToEscape,
                                                                        kCFStringEncodingUTF8);
    return (__bridge_transfer NSString *)escapedString;
}

- (NSString *)decodedString:(NSString *)string {
    if (![string length])
        return @"";
    
    CFStringRef unescapedString = CFURLCreateStringByReplacingPercentEscapesUsingEncoding(kCFAllocatorDefault,
                                                                                          (__bridge CFStringRef)string,
                                                                                          CFSTR(""),
                                                                                          kCFStringEncodingUTF8);
    return (__bridge_transfer NSString *)unescapedString;
}

```

下面这个初始化设置方法, 只做了一件事:
> 遍历缓存文件夹下面所有的已缓存的文件, 更新的操作时间数组`_dates`, 文件大小数组`_sizes`以及更新磁盘总使用大小.

这么做的目的是什么呢?第一次创建磁盘缓存目录肯定是空的文件夹, 里面铁定没有缓存文件, 那为什么要遍历一次所有的缓存文件并更新其操作时间和大小呢? 其实是为了防止不小心再次调用`- (instancetype)initWithName:(NSString *)name rootPath:(NSString *)rootPath`创建了一个名字和路径都相同的缓存目录, 避免里面已经缓存的数据`脱离控制`. 用心良苦呀!

```
- (void)initializeDiskProperties {
    NSUInteger byteCount = 0;
    NSArray *keys = @[ NSURLContentModificationDateKey, NSURLTotalFileAllocatedSizeKey ];
    
    NSError *error = nil;
    NSArray *files = [[NSFileManager defaultManager] contentsOfDirectoryAtURL:_cacheURL
                                                   includingPropertiesForKeys:keys
                                                                      options:NSDirectoryEnumerationSkipsHiddenFiles
                                                                        error:&error];
    TMDiskCacheError(error);
    
    for (NSURL *fileURL in files) {
        NSString *key = [self keyForEncodedFileURL:fileURL];
        
        error = nil;
        NSDictionary *dictionary = [fileURL resourceValuesForKeys:keys error:&error];
        TMDiskCacheError(error);
        
        NSDate *date = [dictionary objectForKey:NSURLContentModificationDateKey];
        if (date && key)
            [_dates setObject:date forKey:key];
        
        NSNumber *fileSize = [dictionary objectForKey:NSURLTotalFileAllocatedSizeKey];
        if (fileSize) {
            [_sizes setObject:fileSize forKey:key];
            byteCount += [fileSize unsignedIntegerValue];
        }
    }
    
    if (byteCount > 0)
        self.byteCount = byteCount; // atomic
}

- (NSString *)keyForEncodedFileURL:(NSURL *)url {
    NSString *fileName = [url lastPathComponent];
    if (!fileName)
        return nil;

    return [self decodedString:fileName];
}

```  

由此看出, 对于缓存数据来说, ` key 经过编码后设为缓存文件名,  value 经过归档后写入文件`.  

至此, 所有的准备工作都基本做完, 下面开始存取数据了.

### 同步/异步的进行读写数据

#### 异步的进行读写数据
相关 API:

```
- (void)objectForKey:(NSString *)key block:(TMDiskCacheObjectBlock)block;
- (void)setObject:(id <NSCoding>)object forKey:(NSString *)key block:(TMDiskCacheObjectBlock)block;
```

先来看看写操作如何实现的, 我就不贴源码具体实现了, 省的看的费劲, 只看关键部位吧~~~你懂的, 嘻嘻.

##### 写入缓存 
1. 写操作被 commit 到串行队列中, 保证了写缓存的时候线程安全:
 
    ```
    dispatch_async(_queue, ^{ 
        // 写操作
        // ...
    }
    ```

2. 将传入的对象进行归档处理, 所以要缓存的对象一定要遵守`NSCoding`协议, 并实现相关方法: 

    ```
    BOOL written = [NSKeyedArchiver archiveRootObject:object toFile:[fileURL path]];
    ```

3. 更新缓存文件的修改时间, 不管是新加入的缓存数据还是已有的缓存数据进行更新, 都会修改对应的时间为当前时间:
    
    ```
    [strongSelf setFileModificationDate:now forURL:fileURL];
    ```

4. 下面是针对缓存空间大小的处理, 比较重要的一步, 根据最新缓存的数据更新总共已经使用的磁盘空间大小, 如果超过预设磁盘空间上限, 则需要删除一些数据以达到不超过上限的目的, 那以什么规则来删除超过缓存上限的部分数据呢? `TMMemoryCache`的优化策略是根据操作时间的先后顺序, 即操作时间早的数据, 认为你使用的概率比较低, 所以就优先删除掉, `TMDiskCache`优化策略跟`TMMemoryCache`相同, 先删除最早的数据. 这也是以文件系统的形式缓存数据的缺点, 不能进行有效的算法.
 - 更新缓存空间大小.
 
     ```
        NSNumber *oldEntry = [strongSelf->_sizes objectForKey:key];
        
        if ([oldEntry isKindOfClass:[NSNumber class]]){
            strongSelf.byteCount = strongSelf->_byteCount - [oldEntry unsignedIntegerValue];
        }
        
        [strongSelf->_sizes setObject:diskFileSize forKey:key];
        strongSelf.byteCount = strongSelf->_byteCount + [diskFileSize unsignedIntegerValue]; // atomic
     ```
 - 删除超出部分空间的缓存数据.
 
     ```
     if (strongSelf->_byteLimit > 0 && strongSelf->_byteCount > strongSelf->_byteLimit)
                    [strongSelf trimToSizeByDate:strongSelf->_byteLimit block:nil];
     ```

至此异步写入缓存数据完成, 注意:
> `_dates`, `_sizes`中的 key 并没有经过编码, 只有缓存文件名才是经过编码的.

##### 读取缓存
相关 API:

```
- (id <NSCoding>)objectForKey:(NSString *)key;
- (void)objectForKey:(NSString *)key block:(TMDiskCacheObjectBlock)block;
```
也是看看异步的读取缓存, 根据上面写入缓存的步骤可以推测读取的步骤, 无非就是把 key 进行编码, 找到缓存文件, 再解档缓存文件内容, 最后更新操作时间, 主线就这几步, 其余的就是加点"配料" - will / did block 之类的时序控制类操作.

```
dispatch_async(_queue, ^{
        TMDiskCache *strongSelf = weakSelf;
        if (!strongSelf)
            return;
        
        NSURL *fileURL = [strongSelf encodedFileURLForKey:key];
        id <NSCoding> object = nil;
        
        if ([[NSFileManager defaultManager] fileExistsAtPath:[fileURL path]]) {
            @try {
                object = [NSKeyedUnarchiver unarchiveObjectWithFile:[fileURL path]];
            }
            @catch (NSException *exception) {
                NSError *error = nil;
                [[NSFileManager defaultManager] removeItemAtPath:[fileURL path] error:&error];
                TMDiskCacheError(error);
            }
            
            [strongSelf setFileModificationDate:now forURL:fileURL];
        }
        
        block(strongSelf, key, object, fileURL);
    });
```

代码中通过`@ try`, `@catch`抛出异常, 如果解档缓存文件内容失败, 直接删除该缓存文件, 简单不做作, 直接了当! 额, 也许不近人情, 好歹你告诉我错误信息是什么, 让我来决定删不删嘛. 

#### 同步的写入/读取缓存
都是采用`dispatch_semaphore_t`信号量的形式来实现的.

### 同步/异步的进行删除数据
相关 API:

```
- (void)removeObjectForKey:(NSString *)key;
- (void)removeObjectForKey:(NSString *)key block:(TMDiskCacheObjectBlock)block;
```

我们只分析异步的删除缓存数据, 同步的跟其它同步操作一样.
既然知道怎么写入缓存, 那删除应该也没什么问题了, 找到要删除的文件路径, 删除该缓存文件即可. 所以步骤应该是:
1. key 进行编码, 再拼接成完整的缓存文件的绝对路径.
2. 删除文件, 其中删除文件做了特殊的步骤, 但是不影响整个删除流程, 后面会讲解.
3. 删除`_dates`,`_sizes`中的键值对, 更新总用使用的缓存空间大小.

> 注意删除文件的时候并没有直接删除, 而是把待删除文件移到临时目录 `tmp`下的缓存目录里, 创建了一个新的串行队列进行删除操作.

```
BOOL trashed = [TMDiskCache moveItemAtURLToTrash:fileURL];
if (!trashed)
     return NO;

[TMDiskCache emptyTrash];
```

### 同步/异步的获取缓存路径
相关 API:

```
- (void)fileURLForKey:(NSString *)key block:(TMDiskCacheObjectBlock)block;
- (NSURL *)fileURLForKey:(NSString *)key;
```
实现非常简单:
1. 对 key 进行编码, 拼接完整缓存文件路径.
2. 更新缓存文件操作时间.

```
    NSURL *fileURL = [strongSelf encodedFileURLForKey:key];
    
    if ([[NSFileManager defaultManager] fileExistsAtPath:[fileURL path]]) {
        [strongSelf setFileModificationDate:now forURL:fileURL];
    } else {
        fileURL = nil;
    }
```

### 同步/异步的根据缓存时间或者缓存大小来削减磁盘空间
这部分操作跟`TMMemoryCache`的实现类似, 相关 API: 

```
- (void)trimToDate:(NSDate *)date;
- (void)trimToDate:(NSDate *)date block:(TMDiskCacheBlock)block;

- (void)trimToSize:(NSUInteger)byteCount;
- (void)trimToSize:(NSUInteger)byteCount block:(TMDiskCacheBlock)block;

- (void)trimToSizeByDate:(NSUInteger)byteCount;
- (void)trimToSizeByDate:(NSUInteger)byteCount block:(TMDiskCacheBlock)block;
```
第一组, 根据缓存时间来削减缓存空间, 如果缓存数据的缓存时间超过了设置的`date`, 则会被删除.
第二组, 根据缓存大小来削减缓存空间, 如果缓存数据的缓存大小超过了指定的`byteCount`, 则会被删除.
第三组, 根据操作时间的先后顺序, 来削减超过了指定缓存大小的空间.

实现大致都相同, 无非就是对时间进行排序, 然后把 key 进行编码, 拼接路径, 移动缓存文件到 tmp目录下, 再清空 tmp 目录. 注意一点, 无论是按照缓存时间还是缓存大小, 都是升序排序, 最先删除的都是`最早的或最小的`数据.

### 设置磁盘缓存空间上限, 磁盘缓存时间上限
源码实现:

```
- (NSUInteger)byteLimit {
    __block NSUInteger byteLimit = 0;
    
    dispatch_sync(_queue, ^{
        byteLimit = _byteLimit;
    });
    
    return byteLimit;
}

- (void)setByteLimit:(NSUInteger)byteLimit {
    __weak TMDiskCache *weakSelf = self;
    
    dispatch_barrier_async(_queue, ^{
        TMDiskCache *strongSelf = weakSelf;
        if (!strongSelf)
            return;
        
        strongSelf->_byteLimit = byteLimit;
        
        if (byteLimit > 0)
            [strongSelf trimDiskToSizeByDate:byteLimit];
    });
}
```

设置缓存空间上限的时候采用`dispatch_barrier_async `栅栏方法, 我不知道作者为何这么写, 多此一举! 本来就是串行队列了, 就能够保证线程安全, 加`栅栏`方法没什么意义. 现在应该注意的不是线程安全, 而是`线程死锁`的问题. 所以在 API 接口中有个⚠️警告
> @warning Do not read this property on the <sharedQueue> (including asynchronous method blocks).

意思是不要在 `shareQueue` 和接口里面的任何 API 的异步 block 中去读这个属性, 为什么呢?  因为`TMDiskCache`所有的读写删除操作都是放在`Serial Queue`串行队列中的, 也就是`shareQueue`队列, 天啦噜...这不造成死锁才怪呢! 警告还写这么不明显.形如下面的是`错误❌`的用法:

```
[diskCache removeObjectForKey:@"profileKey" block:^(TMDiskCache *cache, NSString *key, id<NSCoding> object, NSURL *fileURL) {
        NSLog(@"%ld", diskCache.byteLimit);
 }];
```

因为在`removeObjectForKey`之类的方法中会同步执行传入的 block 操作, 如果在 block 里面再提交新的任务到串行队列中, 再同步执行, 必然死锁. 因为外层的 block 需要等待新提交的 block 执行完毕才能执行完成, 然而新提交的 block 需要等待外层 block 执行完才能执行, 两者相互依赖对方执行完才能执行完成, 就造成`死锁`了.   

```
if (block)
    block(strongSelf, key, nil, fileURL);
``` 

上一篇分析了 `TMMemoryCache` 容易造成性能消耗严重, 而`TMDiskCache`使用不当容易造成`死锁.`

### 各类 will / did block, 以及后台操作

will / did block 穿插在各类异步操作中, 非常简单, 看看即可.

```
if (strongSelf->_willAddObjectBlock)
    strongSelf->_willAddObjectBlock(strongSelf, key, object, fileURL);
```

其中后台操作有点意思, 创建一个全局的`后台管理者`遵守`TMCacheBackgroundTaskManager`协议, 实现其中的两个方法:

```
- (UIBackgroundTaskIdentifier)beginBackgroundTask;
- (void)endBackgroundTask:(UIBackgroundTaskIdentifier)identifier;
```

然后调用设置方法, 给 `TMDiskCache`对象设置后台管理者.

```
+ (void)setBackgroundTaskManager:(id <TMCacheBackgroundTaskManager>)backgroundTaskManager;
```

在后台任务开始之前调用 `beginBackgroundTask` 方法, 结束后台任务之前调用 `endBackgroundTask`, 就能在后台管理者里面监听到什么时候进入后台操作, 什么时候结束后台操作了.
具体做法:

```
UIBackgroundTaskIdentifier taskID = [TMCacheBackgroundTaskManager beginBackgroundTask];

dispatch_async(_queue, ^{ 
      TMDiskCache *strongSelf = weakSelf;
        if (!strongSelf) {
            [TMCacheBackgroundTaskManager endBackgroundTask:taskID];
            return;
        }

      // 执行后台任务
      // 比如: 写缓存, 取缓存, 删除缓存等等.

     [TMCacheBackgroundTaskManager endBackgroundTask:taskID];
}
```

因为磁盘的操作可能耗时非常长, 不可能一直等待, 因此通过这种全局的方式来感知异步操作的开始和结束, 从而执行响应事件. 

### 清空临时存储区
根据上面可以知道, 删除缓存文件的时候, 先会在`tmp`下创建"回收目录", 需要删除的缓存文件统一放进回收目录下, 下面是获取回收目录的URL 路径, 没有就创建, 有则返回, 只创建一次:

```
+ (NSURL *)sharedTrashURL {
    static NSURL *sharedTrashURL;
    static dispatch_once_t predicate;
    
    dispatch_once(&predicate, ^{
        sharedTrashURL = [[[NSURL alloc] initFileURLWithPath:NSTemporaryDirectory()] URLByAppendingPathComponent:TMDiskCachePrefix isDirectory:YES];
        
        if (![[NSFileManager defaultManager] fileExistsAtPath:[sharedTrashURL path]]) {
            NSError *error = nil;
            [[NSFileManager defaultManager] createDirectoryAtURL:sharedTrashURL
                                     withIntermediateDirectories:YES
                                                      attributes:nil
                                                           error:&error];
            TMDiskCacheError(error);
        }
    });
    
    return sharedTrashURL;
}
```

创建一个清空操作专属的串行队列`TrashQueue`, 并且使用`dispatch_set_target_queue`方法修改`TrashQueue`的优先级, 并与全局并发队列`global_queue` 的后台优先级一致. 因为`tmp`目录的情况操作不是那么的重要, 即使我们不手动清除, 系统也会在恰当的时候清除, 所以这里把`TrashQueue`队列的优先级降低.

```
+ (dispatch_queue_t)sharedTrashQueue {
    static dispatch_queue_t trashQueue;
    static dispatch_once_t predicate;
    
    dispatch_once(&predicate, ^{
        NSString *queueName = [[NSString alloc] initWithFormat:@"%@.trash", TMDiskCachePrefix];
        trashQueue = dispatch_queue_create([queueName UTF8String], DISPATCH_QUEUE_SERIAL);
        dispatch_set_target_queue(trashQueue, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0));
    });
    
    return trashQueue;
}
```

类方法, 把原本在`Caches`下的缓存文件移动进`tmp`目录下的回收目录.

```
+ (BOOL)moveItemAtURLToTrash:(NSURL *)itemURL {
    if (![[NSFileManager defaultManager] fileExistsAtPath:[itemURL path]])
        return NO;
    
    NSError *error = nil;
    NSString *uniqueString = [[NSProcessInfo processInfo] globallyUniqueString];
    NSURL *uniqueTrashURL = [[TMDiskCache sharedTrashURL] URLByAppendingPathComponent:uniqueString];
    BOOL moved = [[NSFileManager defaultManager] moveItemAtURL:itemURL toURL:uniqueTrashURL error:&error];
    TMDiskCacheError(error);
    return moved;
}
```

把清除操作添加到`TrashQueue`中异步执行, 在该方法中遍历回收目录下所有的缓存文件, 依次进行删除:

```
+ (void)emptyTrash {
    UIBackgroundTaskIdentifier taskID = [TMCacheBackgroundTaskManager beginBackgroundTask];
    
    dispatch_async([self sharedTrashQueue], ^{
        NSError *error = nil;
        NSArray *trashedItems = [[NSFileManager defaultManager] contentsOfDirectoryAtURL:[self sharedTrashURL]
                                                              includingPropertiesForKeys:nil
                                                                                 options:0
                                                                                   error:&error];
        TMDiskCacheError(error);
        
        for (NSURL *trashedItemURL in trashedItems) {
            NSError *error = nil;
            [[NSFileManager defaultManager] removeItemAtURL:trashedItemURL error:&error];
            TMDiskCacheError(error);
        }
        
        [TMCacheBackgroundTaskManager endBackgroundTask:taskID];
    });
}
```

其实我们只要看一下删除操作在哪里执行的, 就能明白为何作者要创建一个专门用于删除数据的串行队列了. `emptyTrash`方法调用是在读写操作的串行队列`queue`中, 方法调用后面还有`_didRemoveObjectBlock`等待执行, 如果删除数据量比较大且删除操作在`queue`中, 将阻塞当前线程, 那么`_didRemoveObjectBlock`会等待许久才能回调, 况且删除操作对于响应用户事件而言不是那么的重要, 所以把需要删除的缓存文件放进`tmp`目录下, 创建新的低优先级的串行队列来进行删除操作.  这点值得学习!

```
[TMDiskCache emptyTrash];```
```

### 总结
1. 使用`TMDiskCache`姿势要正确, 否则容易造成`死锁`.
2. 删除缓存的思路值得借鉴.  

欢迎大家斧正!



