# AFNetworking 3.0 源码浅析



> 摘要: AFNetworking 3.0 基于NSURLSession进行网络请求，最关键的地方其实还是NSURLRequest与NSURLSessionTask的构造。


## AFN结构图
![AFN structure](../assets/AFN_structure.png "AFN structure")


## 一次完整的AFN请求流程

1. 构造**AFURLSessionManager**:
<pre><code>- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration;
</code></pre>
在该构造方法中，会涉及到**NSURLSession**的构造，该过程中，将设置自己为**NSURLSessionDelegate**的代理，该代理的调用会调用到专门处理与sessionTask相关的 **AFURLSessionManagerTaskDelegate**的方法。  

  同时，会初始化遵循**AFURLResponseSerialization**的实例。


2. 构造**NSURLSessionTask** 
  - 利用AFURLSessionManager构造,其中需要**NSURLRequest**类型的参数:
<pre><code>- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                                completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler DEPRECATED_ATTRIBUTE;
</code></pre>

3. 构造**NSURLRequest**  
  - 遵循AFURLRequestSerialization协议的方法构造。其中会分为NSURLSessionDataTask, NSURLSessionUploadTask和NSURLSessionDownloadTask的构造。以NSURLSessionDataTask中的一个构造方法为例：
<pre><code>- (NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                                   withParameters:(id)parameters
                                            error:(NSError *__autoreleasing *)error;
</code></pre>


4. 最后，在构造出来的NSURLSessionTask中，执行[task resume]方法，实现一次网络请求。

5. 在请求完毕，返回response时，会根据不同的task调用**NSURLSessionDelegate**对应的方法，经过AFURLSessionManager后，最后执行的方法是**AFURLSessionManagerTaskDelegate**里面的代理方法。在该方法中，执行专门为AFN开辟的异步并行线程*url\_session\_manager\_processing\_queue*以回调completionHandler。

## AFURLRequestSerializer

AFURLRequestSerializer分为三个子类：AFHTTPRequestSerializer、AFJSONRequestSerializer和AFPropertyListRequestSerializer，分别对应入参为NSData、JSON和plist（特殊的XML）格式的数据，最后统一转换成NSData进行网络请求。  

详情可以参看[AFNetworking 3.0 源码解读（三）之 AFURLRequestSerialization](http://www.cnblogs.com/machao/p/5725874.html)这篇文章，里面有详细的解释，我就不多加赘述了。

## AFHTTPResponseSerializer
AFHTTPResponseSerializer有六个子类，分别对应JSON响应、XML响应、XML Doucment响应（仅限Mac OS）、plist响应、image响应以及复合响应。

详情可以参看[AFNetworking 3.0 源码解读（四）之 AFURLResponseSerialization](http://www.cnblogs.com/machao/p/5755947.html)这篇文章。

## AFSecurityPolicy
这是一个验证证书是否正确的类。   

1. AFSSLPinningMode这个枚举变量代表的意义如下：
<pre><code>
typedef NS_ENUM(NSUInteger, AFSSLPinningMode) {
    AFSSLPinningModeNone,          //无条件信任服务器的证书
    AFSSLPinningModePublicKey,     //对服务器返回的证书中的PublicKey进行验证，通过则通过，否则不通过
    AFSSLPinningModeCertificate,   //对服务器返回的证书同本地证书全部进行校验，通过则通过，否则不通过
};
</code></pre>
2. 利用下面的方法对证书进行校验。当pinnedCertificates集合中任一证书通过验证，该方法都会返回true。
<pre><code>- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                  forDomain:(NSString *)domain
</code></pre>

详情可以参看[AFNetworking 3.0 源码解读（二）之 AFSecurityPolicy](http://www.cnblogs.com/machao/p/5704201.html)这篇文章。