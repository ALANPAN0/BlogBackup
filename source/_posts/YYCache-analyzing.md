---
title: YYKit源码分析---YYCache
date: 2016-07-18 22:47:35
categories: iOS
tags: [YYKit, YYCache]
---
[YYCache](https://github.com/ibireme/YYCache)是用于Objective-C中用于缓存的第三方框架。此文主要用来讲解该框架的实现细节，性能分析、设计思路ibireme已经讲得很清楚了，我这边就不在分析了。

<!--more-->

## 文件结构
![](http://7xq5ax.com1.z0.glb.clouddn.com/yycache-tree.png)
1. YYCache：同时实现内存缓存和磁盘缓存且是线程安全的
2. YYMemoryCache：实现内存缓存，所有的API都是线程安全的，与其他缓存方式比较不同的是内部利用LRU淘汰算法（后面会介绍）来提高性能
3. YYDiskCache：实现磁盘缓存，所有的API都是线程安全的，内部也采用了LRU淘汰算法，主要SQLite和文件存储两种方式
4. YYKVStorage：实现磁盘存储，不推荐直接使用该类，该类不是线程安全的

## LRU
LRU(Least recently used，最近最少使用)算法，根据访问的历史记录来对数据进行淘汰
<p>
![](http://7xq5ax.com1.z0.glb.clouddn.com/lru.png)
<p>
简单的来说3点：

1. 有新数据加入时添加到链表的头部
2. 每当缓存命中（即缓存数据被访问），则将数据移到链表头部
3. 当链表满的时候，将链表尾部的数据丢弃

**在YYMemoryCache中使用来双向链表和NSDictionary实现了LRU淘汰算法，后面会介绍**

## 关于锁
YYCache 使用到两种锁

1. OSSpinLock ：自旋锁，上一篇博客也提及到[pthread_mutex](http://iipanda.com/2016/06/21/YYKit-pthread/) 
2. dispatch_semaphore：信号量，当信号量为1的时候充当锁来用

**内存缓存用的pthread_mutex：由于pthread_mutex相当于do while忙等，等待时会消耗大量的CPU资源<br>磁盘缓存使用的dispatch_semaphore：优势在于等待时不会消耗CPU资源**
> 简单的科普就到这，现在来开始源码的探索

## _YYLinkedMap
```
@interface _YYLinkedMapNode : NSObject {
    @package
    __unsafe_unretained _YYLinkedMapNode *_prev; // retained by dic
    __unsafe_unretained _YYLinkedMapNode *_next; // retained by dic
    id _key;
    id _value;
    NSUInteger _cost;
    NSTimeInterval _time;
}
@end
```
**_YYLinkedMapNode**：链表的节点<br>
\_prev、\_next：分别表示指向上一个节点、下一个节点<br>
\_key：缓存的key<br>
\_value：缓存对象<br>
\_cost：内存消耗<br>
\_time：缓存时间<br>

```
@interface _YYLinkedMap : NSObject {
    @package
    CFMutableDictionaryRef _dic; // do not set object directly
    NSUInteger _totalCost;
    NSUInteger _totalCount;
    _YYLinkedMapNode *_head; // MRU（最近最常使用算法）, do not change it directly
    _YYLinkedMapNode *_tail; // LRU（最近最少使用算法-清除较不常使用数据）, do not change it directly
    BOOL _releaseOnMainThread;
    BOOL _releaseAsynchronously;
}

```
**_YYLinkedMap**：链表<br>
\_dic：用来保存节点<br>
\_totalCost：总缓存开销<br>
\_head、\_tail：头节点、尾节点<br>
\_releaseOnMainThread：是否在主线程释放\_YYLinkedMapNode<br>
\_releaseAsynchronously：是否异步释放\_YYLinkedMapNode<br>


**双向链表**<br>
![](http://7xq5ax.com1.z0.glb.clouddn.com/LinkedMap@1x.png)

1. 插入节点到头部
2. 将除两边的节点移到头部
3. 移除除两边的节点
4. 移除尾部节点
5. 移除所有节点

看下移除所有节点的代码：

```
- (void)removeAll {
    _totalCost = 0;
    _totalCount = 0;
    _head = nil;
    _tail = nil;
    if (CFDictionaryGetCount(_dic) > 0) {
        CFMutableDictionaryRef holder = _dic;
        _dic = CFDictionaryCreateMutable(CFAllocatorGetDefault(), 0, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
        
        if (_releaseAsynchronously) {
            dispatch_queue_t queue = _releaseOnMainThread ? dispatch_get_main_queue() : YYMemoryCacheGetReleaseQueue();
            dispatch_async(queue, ^{
                CFRelease(holder); // hold and release in specified queue
            });
        } else if (_releaseOnMainThread && !pthread_main_np()) {
            dispatch_async(dispatch_get_main_queue(), ^{
                CFRelease(holder); // hold and release in specified queue
            });
        } else {
            CFRelease(holder);
        }
    }
}


```



这边通过双向链表来对数据进行操作，和NSDictionary实现了LRU淘汰算法。时间复杂度0（1），5种操作基本上都是对头尾节点和链表节点的上一个节点和下一个节点进行操作。


## YYMemoryCache
这边介绍两个主要的操作：添加缓存，查找缓存<p>

- **添加缓存**

```
- (void)setObject:(id)object forKey:(id)key withCost:(NSUInteger)cost {
    if (!key) return;
    if (!object) {
        // 缓存对象为nil，直接移除
        [self removeObjectForKey:key];
        return;
    }
    // 为了保证线程安全，数据操作前进行加锁
    pthread_mutex_lock(&_lock);
    // 查找缓存
    _YYLinkedMapNode *node = CFDictionaryGetValue(_lru->_dic, (__bridge const void *)(key));
    // 当前时间
    NSTimeInterval now = CACurrentMediaTime();
    if (node) {
        // 缓存对象已存在，更新数据，并移到栈顶
        _lru->_totalCost -= node->_cost;
        _lru->_totalCost += cost;
        node->_cost = cost;
        node->_time = now;
        node->_value = object;
        [_lru bringNodeToHead:node];
    } else {
        // 缓存对象不存在，添加数据，并移到栈顶
        node = [_YYLinkedMapNode new];
        node->_cost = cost;
        node->_time = now;
        node->_key = key;
        node->_value = object;
        [_lru insertNodeAtHead:node];
    }
    // 判断当前的缓存进行是否超出了设定值，若超出则进行整理
    if (_lru->_totalCost > _costLimit) {
        dispatch_async(_queue, ^{
            [self trimToCost:_costLimit];
        });
    }
    
    // 每次添加数据仅有一个，数量上超出时，直接移除尾部那个object即可
    if (_lru->_totalCount > _countLimit) {
        _YYLinkedMapNode *node = [_lru removeTailNode];
        if (_lru->_releaseAsynchronously) {
            dispatch_queue_t queue = _lru->_releaseOnMainThread ? dispatch_get_main_queue() : YYMemoryCacheGetReleaseQueue();
            dispatch_async(queue, ^{
                [node class]; //hold and release in queue
            });
        } else if (_lru->_releaseOnMainThread && !pthread_main_np()) {
            dispatch_async(dispatch_get_main_queue(), ^{
                [node class]; //hold and release in queue
            });
        }
    }
    // 操作结束，解锁
    pthread_mutex_unlock(&_lock);
}

```

- **异步线程释放**
![](http://7xq5ax.com1.z0.glb.clouddn.com/%E5%BC%82%E6%AD%A5%E9%87%8A%E6%94%BE.png)
里面很多都用到类似的方法，将一个对象在异步线程中释放，来分析下：<br>


		- p
		1. 首先通过node来对其进行持有，以至于不会在方法调用结束的时候被销毁
		2. 我要要在其他线程中进行销毁，所以将销毁操作放在block中，block就会对其进行持有
		3. 这边在block中随便调用了个方法，保证编译器不会优化掉这个操作
		4. 当block结束后，node没有被持有的时候，就会在当前线程被release掉了



- **添加缓存**

```
// 这边从memory中取数据时，根据LRU原则，将最新取出的object放到栈头
- (id)objectForKey:(id)key {
    if (!key) return nil;
    pthread_mutex_lock(&_lock);
    _YYLinkedMapNode *node = CFDictionaryGetValue(_lru->_dic, (__bridge const void *)(key));
    if (node) {
        node->_time = CACurrentMediaTime();
        [_lru bringNodeToHead:node];
    }
    pthread_mutex_unlock(&_lock);
    return node ? node->_value : nil;
}

```

## YYKVStorage

该文件主要以两种方式来实现磁盘存储：SQLite、File，使用两种方式混合进行存储主要为了提高读写效率。写入数据时，SQLite要比文件的方式更快；读取数据的速度主要取决于文件的大小。据测试，在iPhone6中，当文件大小超过20kb时，File要比SQLite快的多。所以当大文件存储时建议用File的方式，小文件更适合用SQLite。<p>
下边分别对Save、Remove、Get分别进行分析


- **Save**

```
- (BOOL)saveItemWithKey:(NSString *)key value:(NSData *)value filename:(NSString *)filename extendedData:(NSData *)extendedData {
    // 条件不符合
    if (key.length == 0 || value.length == 0) return NO;
    if (_type == YYKVStorageTypeFile && filename.length == 0) {
        return NO;
    }
    
    if (filename.length) {    // filename存在 SQLite File两种方式并行
        // 用文件进行存储
        if (![self _fileWriteWithName:filename data:value]) {
            return NO;
        }
        // 用SQLite进行存储
        if (![self _dbSaveWithKey:key value:value fileName:filename extendedData:extendedData]) {
            // 当使用SQLite方式存储失败时，删除本地文件存储
            [self _fileDeleteWithName:filename];
            return NO;
        }
        return YES;
    } else {               // filename不存在 SQLite
        if (_type != YYKVStorageTypeSQLite) {
            // 这边去到filename后，删除filename对应的file文件
            NSString *filename = [self _dbGetFilenameWithKey:key];
            if (filename) {
                [self _fileDeleteWithName:filename];
            }
        }
        // SQLite 进行存储
        return [self _dbSaveWithKey:key value:value fileName:nil extendedData:extendedData];
    }
}


```

- **Remove**

```
- (BOOL)removeItemForKey:(NSString *)key {
    if (key.length == 0) return NO;
    switch (_type) {
        case YYKVStorageTypeSQLite: {
            // 删除SQLite文件
            return [self _dbDeleteItemWithKey:key];
        } break;
        case YYKVStorageTypeFile:
        case YYKVStorageTypeMixed: {
            // 获取filename
            NSString *filename = [self _dbGetFilenameWithKey:key];
            if (filename) {
                // 删除filename对的file
                [self _fileDeleteWithName:filename];
            }
            // 删除SQLite文件
            return [self _dbDeleteItemWithKey:key];
        } break;
        default: return NO;
    }
}

```

- **Get**

```
- (NSData *)getItemValueForKey:(NSString *)key {
    if (key.length == 0) return nil;
    NSData *value = nil;
    switch (_type) {
        case YYKVStorageTypeFile: { //File
            NSString *filename = [self _dbGetFilenameWithKey:key];
            if (filename) {
                // 根据filename获取File
                value = [self _fileReadWithName:filename];
                if (!value) {
                    // 当value不存在，用对应的key删除SQLite文件
                    [self _dbDeleteItemWithKey:key];
                    value = nil;
                }
            }
        } break;
        case YYKVStorageTypeSQLite: {
            // SQLite 方式获取
            value = [self _dbGetValueWithKey:key];
        } break;
        case YYKVStorageTypeMixed: {
            NSString *filename = [self _dbGetFilenameWithKey:key];
            // filename 存在文件获取，不存在SQLite方式获取
            if (filename) {
                value = [self _fileReadWithName:filename];
                if (!value) {
                    [self _dbDeleteItemWithKey:key];
                    value = nil;
                }
            } else {
                value = [self _dbGetValueWithKey:key];
            }
        } break;
    }
    if (value) {
        // 更新文件操作时间
        [self _dbUpdateAccessTimeWithKey:key];
    }
    return value;
}

```

File方式主要使用的writeToFile进行存储，SQLte直接使用的sqlite3来对文件进行操作，具体数据库相关的操作这边就不在进行分析了，感兴趣的自己可以阅读下

## YYDiskCache

YYDiskCache是对YYKVStorage进行的一次封装，是线程安全的，这边使用的是dispatch_semaphore_signal来确保线程的安全。另外他结合LRU算法，根据文件的大小自动选择存储方式来达到更好的性能。

```
- (instancetype)initWithPath:(NSString *)path
             inlineThreshold:(NSUInteger)threshold {
    self = [super init];
    if (!self) return nil;
    
    // 获取缓存的 YYDiskCache
    YYDiskCache *globalCache = _YYDiskCacheGetGlobal(path);
    if (globalCache) return globalCache;
    
    // 确定存储的方式
    YYKVStorageType type;
    if (threshold == 0) {
        type = YYKVStorageTypeFile;
    } else if (threshold == NSUIntegerMax) {
        type = YYKVStorageTypeSQLite;
    } else {
        type = YYKVStorageTypeMixed;
    }
    
    // 初始化 YYKVStorage
    YYKVStorage *kv = [[YYKVStorage alloc] initWithPath:path type:type];
    if (!kv) return nil;
    
    // 初始化数据
    _kv = kv;
    _path = path;
    _lock = dispatch_semaphore_create(1);
    _queue = dispatch_queue_create("com.ibireme.cache.disk", DISPATCH_QUEUE_CONCURRENT);
    _inlineThreshold = threshold;
    _countLimit = NSUIntegerMax;
    _costLimit = NSUIntegerMax;
    _ageLimit = DBL_MAX;
    _freeDiskSpaceLimit = 0;
    _autoTrimInterval = 60;
    
    // 递归的去整理文件
    [self _trimRecursively];
    // 对当前对象进行缓存
    _YYDiskCacheSetGlobal(self);
    
    // 通知 APP即将被杀死时
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(_appWillBeTerminated) name:UIApplicationWillTerminateNotification object:nil];
    return self;
}

```

其他的一些操作基本上都是对YYKVStorage的一些封装，这边就不一一分析了。

## 参考文献

1. http://blog.ibireme.com/2015/10/26/yycache/
2. http://blog.csdn.net/yunhua_lee/article/details/7599671
