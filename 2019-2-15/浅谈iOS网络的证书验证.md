# 浅谈iOS网络的证书验证

## HTTPS概念
> HTTPS是HTTP1.1在SSL/TSL层上建立连接进行加密通信的协议。在TCP握手过程中，利用非对称加密方式传送公钥，最后利用该公钥对称在建立好的连接中加密传送会话内容。


### TSL/SSL过程
1. 客户端向服务端索要公钥；
2. 客户端与服务端协商生成对话密钥，对话密钥是对称加密的；
3. 客户端用公钥加密对话密钥，发送给服务端，服务端用私钥解密；
4. 以后双方通信都用对话密钥进行。

### 四次握手的详细步骤
#### 一、 Client Hello
clinet向server发送以下信息：

1. 支持的TSL协议
2. 一个随机数，用作之后生成对话密钥
3. 支持的加密方法，如RSA
4. 支持的压缩方法

#### 二、 Server Hello

server会向client发送：

1. 数字证书，里面包含公钥
2. 另一个随机数，用作之后生成对话密钥
3. 确认的通信加密协议，如TSL 1.0
4. 确认client发送过来的加密方法（如RSA）

#### 三、 Client Response

client会再次向server发送：

1. 第三个随机数，该随机数用公钥加密
2. 编码改变通知，表示随后的信息都将用双方商定的加密方法和密钥发送
3. 	客户端握手结束通知，并且发送之前所用消息的hash值供服务端校验

#### 四、 Server Response

server发送：

1. 编码改变确认
2. 服务端结束握手通知，并且会将之前所有发送的信息的hash值供客户端校验

### TSL/SSL中的数字证书
> 由SSL证书Certificate Author(CA)颁发，包含了公钥和证书内容。若证书是可信，则其公钥也是可信，由此避免中间人攻击篡改证书问题。

1. 数字证书的结构是树，由顶层的根CA证书，可以授权予多个二级CA证书，而一个二级证书可以授权给多个三级证书，以此类推，所以所有次级的CA证书数字签名都是由上级的私钥加密而来的。而根CA证书是利用自己的私钥生成，信任了根CA证书，则其整个证书链都被信任。
2. CA证书的签发过程：将SSL证书内容作哈希加密得出摘要，然后再用CA证书的私钥（由服务器端保管）作非对称加密（RSA）得出CA数字签名。CA证书正是由公钥和CA数字签名组成。
3. CA证书的验证：客户端得到SSL证书后，对证书内容进行哈希加密，得到的内容摘要，再与用`CA的公钥`解码数字签名的结果作对比。如果两者相等，则表示证书没有被篡改。若想验证整个CA证书链是否合法，则再用同样的方法验证上级的CA证书，一直到根CA证书即可。



## iOS中的证书验证过程
在iOS中，对于TSL的验证证书过程，可以用`Security Framework`来建立SecTrust对象，然后利用SecTrust对象来进行验证。

### 为何客户端需要SSL Pinning？
1. 客户端要传输数据的sever domain一般是确定的；
2. 客户端为了节省成本而用自签名的证书时，SSL Pinning可以帮忙绕过CA机构颁发的证书验证，而只需要在客户端验证自签名证书即可。

### 创建SecTrust对象
> 在bundle中读取证书 -> 创建证书链 -> 创建验证规则 -> 创建SectTrust对象


### AFNetworking源码解读（节选）

1. 如何判断证书的规则枚举

	```objc
	typedef NS_ENUM(NSUInteger, AFSSLPinningMode) {
	    AFSSLPinningModeNone,				\\ 没有本地证书，只验证服务器证书是否系统受信任证书列表里的证书签发
	    AFSSLPinningModePublicKey,		\\ 预存服务器证书，但只验证证书里的公钥
	    AFSSLPinningModeCertificate,		\\ 服务器证书预先存于本地作为锚点证书，验证有效期，与服务器证书是否一致
	};
	
	```
2. 验证服务器证书是否可信的关键代码
	```evaluateServerTrust:forDomain```方法：

	```objc

	   //步骤：
	   // 1. 创建policy并将policy赋值给secTrust对象
	   // 2. 根据不同的SSL绑定mode来判断证书是否可用

	
	...
	
	// 根据是否需要判断domain可用，分别创建SSL和X509 policy
	NSMutableArray *policies = [NSMutableArray array];
	 if (self.validatesDomainName) {
        [policies addObject:(__bridge_transfer id)SecPolicyCreateSSL(true, (__bridge CFStringRef)domain)];
    } else {
        [policies addObject:(__bridge_transfer id)SecPolicyCreateBasicX509()];
    }

    SecTrustSetPolicies(serverTrust, (__bridge CFArrayRef)policies);
	
    if (self.SSLPinningMode == AFSSLPinningModeNone) {
        return self.allowInvalidCertificates || AFServerTrustIsValid(serverTrust);  // 若为自签名证书或者是系统信任的证书，返回true
    } else if (!AFServerTrustIsValid(serverTrust) && !self.allowInvalidCertificates) {
        return NO;
    }
        
        ...
	```
	
	对于```AFSSLPinningModeCetificate```，验证的过程是：利用从bundle中读取的DER编码证书数据表来新建绑定的证书列表，并将其设置为锚点证书链。验证成功后，再获取服务器的证书链，然后从根节点开始，看上面的新建绑定的证书列表是否包含。
	
	```objc
	NSMutableArray *pinnedCertificates = [NSMutableArray array];
   for (NSData *certificateData in self.pinnedCertificates) {
        [pinnedCertificates addObject:(__bridge_transfer id)SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificateData)];
    }
    SecTrustSetAnchorCertificates(serverTrust, (__bridge CFArrayRef)pinnedCertificates);

	if (!AFServerTrustIsValid(serverTrust)) {
	    return NO;
	}
	
    // obtain the chain after being validated, which *should* contain the pinned certificate in the last position (if it's the Root CA)
    NSArray *serverCertificates = AFCertificateTrustChainForServerTrust(serverTrust);
    
    for (NSData *trustChainCertificate in [serverCertificates reverseObjectEnumerator]) {
        if ([self.pinnedCertificates containsObject:trustChainCertificate]) {
            return YES;
        }
    }
    
    return NO;
	```
	
	对于```AFSSLPinningModePublicKey```，验证过程则是：从`AFPublicKeyTrustChainForServerTrust`方法中获取服务器证书的公钥链，然后再从本地证书中获取公钥集合，并一一对比。只要有相同的公钥，就会返回true。
	
	```
    NSUInteger trustedPublicKeyCount = 0;
    NSArray *publicKeys = AFPublicKeyTrustChainForServerTrust(serverTrust);

    for (id trustChainPublicKey in publicKeys) {
        for (id pinnedPublicKey in self.pinnedPublicKeys) {
            if (AFSecKeyIsEqualToKey((__bridge SecKeyRef)trustChainPublicKey, (__bridge SecKeyRef)pinnedPublicKey)) {
                trustedPublicKeyCount += 1;
            }
        }
    }
    return trustedPublicKeyCount > 0;
        
	```
	



## Reference
1. [AFNetWorking源码之AFSecurityPolicy](https://segmentfault.com/a/1190000009199444)
2. [验证 HTTPS 请求的证书（五）](https://draveness.me/afnetworking5)
3. [阮一峰TSL](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)