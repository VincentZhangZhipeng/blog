# SDWebImage源码阅读笔记（下）-- 图片缓存篇

> SDWebImage核心功能是异步下载图片、实现图片缓存（本文主要分析）。本文主要阅读的源码基于[SDWebImage release 4.3.3](https://codeload.github.com/rs/SDWebImage/zip/4.3.3)。


## SDImageCache
SDImageCache是一个单例，内存缓存和硬盘缓存的实现都依赖于它。

其中包含的操作方法有：

	1. 实例化和初始化操作
	2. 生成和获取缓存路径操作
	3. 同步或异步缓存图片至内存或缓存
	4. 查询和获取缓存相关操作（提供同步和异步方法）
	5. 清除查询和获取缓存相关操作方法
	6. 清除缓存方法
	7. 获取缓存信息方法

### 1. 硬盘缓存管理
```SDImageCache```类持有SDMemoryCache和一条自定义的串行队列```ioQueue```。```ioQueue```顾名思义就是用作硬盘图片缓存的增删读写。

a.添加缓存

````objc
- (void)storeImage:(nullable UIImage *)image
         imageData:(nullable NSData *)imageData
            forKey:(nullable NSString *)key
            toDisk:(BOOL)toDisk
        completion:(nullable SDWebImageNoParamsBlock)completionBlock {
    ...

    if (self.config.shouldCacheImagesInMemory) {
        // 一个inline宏方法，Mac系统返回图片的像素，若为iOS系统则返回压缩过后的图片像素
        NSUInteger cost = SDCacheCostForImage(image);
        // 设置SDMemoryCache类缓存
        [self.memCache setObject:image forKey:key cost:cost];
    }
    
    if (toDisk) {
        dispatch_async(self.ioQueue, ^{
        	  // 使用autoreleasepool来避免临时图片数据变量过多使内存激增
            @autoreleasepool {
                NSData *data = imageData;
                if (!data && image) {
                    // 根据是否有透明度alpha来判断图片类型
                    SDImageFormat format;
                    if (SDCGImageRefContainsAlpha(image.CGImage)) {
                        format = SDImageFormatPNG;
                    } else {
                        format = SDImageFormatJPEG;
                    }
                    data = [[SDWebImageCodersManager sharedInstance] encodedDataWithImage:image format:format];
                }
                [self _storeImageDataToDisk:data forKey:key];
            }
			  ...
        });
    } 
    ...
}
````
b. 查询缓存的操作

````objc
- (nullable NSOperation *)queryCacheOperationForKey:(nullable NSString *)key options:(SDImageCacheOptions)options done:(nullable SDCacheQueryCompletedBlock)doneBlock {
    ...
        
    // 首先检查内存有否缓存，若符合只查询内存缓存option，返回完成回调以及空operation
    UIImage *image = [self imageFromMemoryCacheForKey:key];
    BOOL shouldQueryMemoryOnly = (image && !(options & SDImageCacheQueryDataWhenInMemory));
    if (shouldQueryMemoryOnly) {
        if (doneBlock) {
            doneBlock(image, nil, SDImageCacheTypeMemory);
        }
        return nil;
    }
    
    // 若需要查询硬盘缓存，新建operation作为返回值
    NSOperation *operation = [NSOperation new];
    void(^queryDiskBlock)(void) =  ^{
       ...
        // 使用autoreleasepool来避免临时图片数据变量过多使内存激增
        @autoreleasepool {
            NSData *diskData = [self diskImageDataBySearchingAllPathsForKey:key];
            UIImage *diskImage;
            SDImageCacheType cacheType = SDImageCacheTypeDisk;
            if (image) {
                // 若有内存缓存则使用内存缓存
                diskImage = image;
                cacheType = SDImageCacheTypeMemory;
            } else if (diskData) {
                // 若无内存缓存，diskImageForKey:方法中有对硬盘的图片数据进行压缩编码的操作
                diskImage = [self diskImageForKey:key data:diskData];
                if (diskImage && self.config.shouldCacheImagesInMemory) {
                   // 计算cost，即压缩后图片的像素值
                    NSUInteger cost = SDCacheCostForImage(diskImage);
                    // 存入内存缓存中
                    [self.memCache setObject:diskImage forKey:key cost:cost];
                }
            }
            
            if (doneBlock) {
            	   // 默认查询内存缓存时为同步查询，硬盘缓存为异步查询，若用SDImageCacheQueryDiskSync选项则硬盘缓存也为同步查询
                if (options & SDImageCacheQueryDiskSync) {
                    doneBlock(diskImage, diskData, cacheType);
                } else {
                    dispatch_async(dispatch_get_main_queue(), ^{
                        doneBlock(diskImage, diskData, cacheType);
                    });
                }
            }
        }
    };
    
    if (options & SDImageCacheQueryDiskSync) {
        queryDiskBlock();
    } else {
        dispatch_async(self.ioQueue, queryDiskBlock);
    }
    
    return operation;
}
````

c. 按照缓存策略删除缓存

````objc
- (void)deleteOldFilesWithCompletionBlock:(nullable SDWebImageNoParamsBlock)completionBlock {
    dispatch_async(self.ioQueue, ^{
        ...
        // maxCacheAge默认为一周
        NSDate *expirationDate = [NSDate dateWithTimeIntervalSinceNow:-self.config.maxCacheAge];
        NSMutableDictionary<NSURL *, NSDictionary<NSString *, id> *> *cacheFiles = [NSMutableDictionary dictionary];
        NSUInteger currentCacheSize = 0;

		 // 该数组用于存放过期的以及大于缓存最大数量的fileURL
        NSMutableArray<NSURL *> *urlsToDelete = [[NSMutableArray alloc] init];
        for (NSURL *fileURL in fileEnumerator) {
            
            ...
            
        	  // 根据最后修改日期与过期日期的对比，找出过期的文档的路径并存入urlsToDelete数组
            NSDate *modificationDate = resourceValues[NSURLContentModificationDateKey];
            if ([[modificationDate laterDate:expirationDate] isEqualToDate:expirationDate]) {
                [urlsToDelete addObject:fileURL];
                continue;
            }

            // 用于计算所有非过期文件的总缓存空间大小
            NSNumber *totalAllocatedSize = resourceValues[NSURLTotalFileAllocatedSizeKey];
            currentCacheSize += totalAllocatedSize.unsignedIntegerValue;
            cacheFiles[fileURL] = resourceValues;
        }
        
        // 清除已过期的文件
        for (NSURL *fileURL in urlsToDelete) {
            [self.fileManager removeItemAtURL:fileURL error:nil];
        }

		  // maxCacheSize默认为0，即默认缓存清除策略不包含缓存空间大小
		  // 如果自定义了maxCacheSize, 剩余的缓存文件size仍然大于maxCacheSize，则按最新存入硬盘的文件日期为先序，执行清除操作
        if (self.config.maxCacheSize > 0 && currentCacheSize > self.config.maxCacheSize) {
			  // 以自定义maxCacheSize的一半为阈值
            const NSUInteger desiredCacheSize = self.config.maxCacheSize / 2;

            // 按最新存入硬盘的文件日期为先序排序
            NSArray<NSURL *> *sortedFiles = [cacheFiles keysSortedByValueWithOptions:NSSortConcurrent
                                                                     usingComparator:^NSComparisonResult(id obj1, id obj2) {
                                                                         return [obj1[NSURLContentModificationDateKey] compare:obj2[NSURLContentModificationDateKey]];
                                                                     }];

            // 删除文件直到文件空间小于阈值
            for (NSURL *fileURL in sortedFiles) {
                if ([self.fileManager removeItemAtURL:fileURL error:nil]) {
                    NSDictionary<NSString *, id> *resourceValues = cacheFiles[fileURL];
                    NSNumber *totalAllocatedSize = resourceValues[NSURLTotalFileAllocatedSizeKey];
                    currentCacheSize -= totalAllocatedSize.unsignedIntegerValue;

                    if (currentCacheSize < desiredCacheSize) {
                        break;
                    }
                }
            }
        }
        if (completionBlock) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completionBlock();
            });
        }
    });
}

````

### 2. SDMemoryCache
```SDMemoryCache```用于内存缓存管理，它继承了```NSCache```，从而获得了```NSCache```自动删除策略和线程安全的优点。同时，使用强-弱引用的```NSMaptable```实现的```weakCache```来实现二级缓存。使用```NSMaptable```的好处是，映射中的```UIImage```类型的value值是弱引用，当其持有value的```UIImageView```或其他对象被销毁后，```weakCache```会自动移除对应的映射。

````objc
- (void)setObject:(id)obj forKey:(id)key cost:(NSUInteger)g {
    [super setObject:obj forKey:key cost:g];
    if (key && obj) {
        // 设置NSCache值的时候，把二级缓存的值也设置上。因为NSCache线程安全，但NSMaptable并非线程安全，所以二级缓存设置过程要加锁
        LOCK(self.weakCacheLock);
        [self.weakCache setObject:obj forKey:key];
        UNLOCK(self.weakCacheLock);
    }
}

- (id)objectForKey:(id)key {
    id obj = [super objectForKey:key];
    if (key && !obj) {
        // 加锁获取二级缓存中的图片
        LOCK(self.weakCacheLock);
        obj = [self.weakCache objectForKey:key];
        UNLOCK(self.weakCacheLock);
        if (obj) {
            // 从二级缓存中获取缓存图片，同步到NSCache的映射中，这样可以避免NSCache被移除后要从硬盘获取数据
            NSUInteger cost = 0;
            if ([obj isKindOfClass:[UIImage class]]) {
                cost = SDCacheCostForImage(obj);
            }
            [super setObject:obj forKey:key cost:cost];
        }
    }
    return obj;
}

// 被didReceiveMemoryWarning:调用，收到内存警告时，移除所有缓存数据
- (void)removeAllObjects {
    [super removeAllObjects];
    // weakCache需要手动清除
    LOCK(self.weakCacheLock);
    [self.weakCache removeAllObjects];
    UNLOCK(self.weakCacheLock);
}

````

## 总结
1. ```NSCache```的使用：自动管理缓存，当内存紧张时，丢弃NSCache，优化App性能。同时，NSCache是线程安全的，所以在多线程调用时不必加锁。
2. ```NSMaptable```的使用：创建strong-weak映射表，弱引用```UIImage```，达到当```UIImage```销毁时自动从映射表中移除的效果，实现图片缓存的二级缓存。
3. ```@autoreleasePool```的用途：主要用于防止临时变量过大，占用内存过多而导致内存激增使App崩溃的问题。


## Refernece
1. [NSCache原理](https://blog.csdn.net/u010165653/article/details/46472487)
2. [黑幕背后的Autorelease](http://www.cocoachina.com/ios/20141031/10107.html)