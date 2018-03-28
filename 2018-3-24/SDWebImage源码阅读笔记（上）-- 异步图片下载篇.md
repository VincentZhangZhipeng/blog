# SDWebImage源码阅读笔记（上）-- 异步图片下载篇

## SDWebImage主要作用

异步下载图片（本文主要分析）、实现图片缓存。

## 异步下载图片实现流程
UIImageView通过SDWebImageManager发起下载请求 -> 创建自定义的SDWebImageDownloaderOperation -> 添加到SDWebImageDownloader中的downloadQueue -> downloadQueue触发NSURLSessionDataDelegate和NSURLSessionTaskDelegate异步调用progressCallBack和completeionCallBack

## 核心用法
````objc
  [cell.imageView sd_setImageWithURL:[NSURL URLWithString:@"http://www.domain.com/path/to/image.jpg"]
                    placeholderImage:[UIImage imageNamed:@"placeholder.png"]];
````

## 关键问题
1. 自己实现的想法：
	1. 将网络图片下载到本地；-- 如何下载，同步异步？
	2. 显示已缓存到本地的图片； -- 缓存策略如何选择？（将在下篇分析）
	3. 如无图片，显示placeholder。 -- 什么时候才显示placeholder？（将在下篇分析）
	
2. 更具体的问题：
	1. 如何实现异步下载图片，如何添加与管理任务？
	2. 如何实现进度监控回调，以及什么时候实现完成回调？

## 实现原理分析

1. SDWebImageDowloaderOperation
		
	这个是继承自NSOperation的图片下载任务类，每个operation作为一个任务添加到SDWebImageDownloader持有的downloadQueue中。同时，作为使用NSURLSesion下载图片的类，operation遵循NSURLSessionTaskDelegate和NSURLSessionDataDelegate。
		
	````objc
		#pragma mark NSURLSessionDataDelegate
		// 告诉delegate已经接受到服务器的初始应答, 准备接下来的数据任务的操作.
		- (void)URLSession:(NSURLSession *)session
          		dataTask:(NSURLSessionDataTask *)dataTask
		didReceiveResponse:(NSURLResponse *)response
 		completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler {
 			  ...
			 BOOL valid = statusCode < 400;
			   	 
			 if (statusCode == 304 && !self.cachedData) {
			   	// 对304 'not modified'的情况作特殊处理，如果没有缓冲数据，认为非法
			      valid = NO;
			  }
			    
			   if (valid) {
			       for (SDWebImageDownloaderProgressBlock progressBlock in [self callbacksForKey:kProgressCallbackKey]) {
			           // 开始返回进度相关的回调
			           progressBlock(0, expected, self.request.URL);
			       }
			   } else {
			       // Status code invalid and marked as cancelled. Do not call `[self.dataTask cancel]` which may mass up URLSession life cycle
			        disposition = NSURLSessionResponseCancel;
			    }
 			  ...
 			}
 			
 		// 告诉delegate已经接收到部分数据.
		- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data {
			...
			
			// 异步加载串行对列coderQueue，用于一点一点显示图片
			dispatch_async(self.coderQueue, ^{
	           // 对progress中的图片数据进行编码处理，一步一步把图片显示出来，其中用到SDWebImageManager和SDWebImageCodersManager的方法，稍后分析
	            }
	        });
	        
			...
		}
		for (SDWebImageDownloaderProgressBlock progressBlock in [self callbacksForKey:kProgressCallbackKey]) {
			// 更新进度的回调
	    	progressBlock(self.imageData.length, self.expectedSize, self.request.URL);
	    }
    }	
	````

	在task全部接收数据后，会触发NSURLSessionTaskDelegate的```URLSession:task:didCompleteWithError:```方法。在该方法中会调用[self done]方法以移除self.callbackBlocks保存的所有回调。
		在该方法内，当options不是SDWebImageDownloaderIgnoreCachedResponse时，默认会在异步的self.coderQueue对列内进行除GIFs和WebPs的图片压缩编码，并将压缩后的图片作为参数返回给completedBlock。
		
	operation会重写NSOperation的start方法，dataTask的获取与开始在start方法中完成。
		
		
2. SDWebImageDownloader

	SDWebImageDowlaoder是一个单例，持有下载队列downloadQueue。上面提及的SDWebImageDownloaderOperation就是添加到downloadQueue中执行。SDWebImageDowloader提供诸如最大并发数、下载顺序、下载超时等自定义操作。该类的核心方法是```downloadImageWithURL:options:progress:completed:``` 和```addProgressCallback:completedBlock:forURL:createCallback:```。通过前者来设置网络请求的超时、请求头部还有是否需要证书验证等操作。
	
	````objc
	
	- (nullable SDWebImageDownloadToken *)downloadImageWithURL:(nullable NSURL *)url
                                                   options:(SDWebImageDownloaderOptions)options
                                                  progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                                 completed:(nullable SDWebImageDownloaderCompletedBlock)completedBlock {
	 	
	 	...
	 	
	 	// 调用addProgressCallback:completedBlock:forURL:createCallback:返回SDWebImageDownloadToken类
	    return [self addProgressCallback:progressBlock completedBlock:completedBlock forURL:url createCallback:^SDWebImageDownloaderOperation *{
	        __strong __typeof (wself) sself = wself;
	        NSTimeInterval timeoutInterval = sself.downloadTimeout;
	        if (timeoutInterval == 0.0) {
	            timeoutInterval = 15.0; // 默认15秒超时
	        }
	
	        ...
	        
	        // SDWebImage的一个选项，选择任务执行顺序是根据先进先出还是后进先出
	        if (sself.executionOrder == SDWebImageDownloaderLIFOExecutionOrder) {
	            // 之前最后的operation依赖新加进来的operation，即等待新进来的operation先执行，然后新进来的operation会被设置成最后的operation
	            [sself.lastAddedOperation addDependency:operation];
	            sself.lastAddedOperation = operation;
	        }
	
	        return operation;
	    }];
	}
	
	````
	
	````objc
	
	- (nullable SDWebImageDownloadToken *)addProgressCallback:(SDWebImageDownloaderProgressBlock)progressBlock
                                           completedBlock:(SDWebImageDownloaderCompletedBlock)completedBlock
                                                   forURL:(nullable NSURL *)url
                                           createCallback:(SDWebImageDownloaderOperation *(^)(void))createCallback {
    	//对url合法性判断，因为url作为回调字典的key
	    if (url == nil) {
	        if (completedBlock != nil) {
	            completedBlock(nil, nil, nil, NO);
	        }
	        return nil;
	    }
	    
	    // 利用dispatch_semaphore_wait加锁，保证opeartion的读取和创建线程安全
	    LOCK(self.operationsLock);
	    SDWebImageDownloaderOperation *operation = [self.URLOperations objectForKey:url];
	    if (!operation) {
	        operation = createCallback();
	        __weak typeof(self) wself = self;
	        operation.completionBlock = ^{
	            __strong typeof(wself) sself = wself;
	            if (!sself) {
	                return;
	            }
	            LOCK(sself.operationsLock); // 再次加锁，保证URLOperations这个任务字典的移除操作线程安全
	            [sself.URLOperations removeObjectForKey:url];
	            UNLOCK(sself.operationsLock);
	        };
	        [self.URLOperations setObject:operation forKey:url];
	        // 保证所有配置完成后才调用addOperation方法
	        [self.downloadQueue addOperation:operation];
	    }
	    UNLOCK(self.operationsLock);
		
		// downloadeOperationCanelToken其实就是SDCallbacksDictionary的数组
	    id downloadOperationCancelToken = [operation addHandlersForProgress:progressBlock completed:completedBlock];
	    
	    // SDWebImage框架设计SDWebImageDownloadToken类，就是为了取消下载任务时，作为唯一标识
	    SDWebImageDownloadToken *token = [SDWebImageDownloadToken new];
	    token.downloadOperation = operation;
	    token.url = url;
	    token.downloadOperationCancelToken = downloadOperationCancelToken;
	
	    return token;
	}
	````
	
3. SDWebImageManager
	
	这个也是一个单例，里面包含对SDWebImageDowlaoder以及缓存相关操作的调用，该类将在下一遍再详细分析。


## Reference
1. [知其然亦知其所以然--NSOperation并发编程](http://www.cocoachina.com/game/20151201/14517.html)
2. [iOS 中对 HTTPS 证书链的验证](https://www.jianshu.com/p/31bcddf44b8d)
