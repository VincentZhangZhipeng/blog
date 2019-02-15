# 创建CocoaPods库小结
> 最近公司需要用到cocopods私有的仓库，以此作为小结。

## 生成pod spec
1. 生成spec

	```shell
	pod spec create YourPodName
	```
	
2. 编辑YourPodName.podspec
	
	一个简单的例子：
	
	```shell
	Pod::Spec.new do |s|
	  s.name         = "YourPodName"
	  s.version      = "0.0.1"
	  s.summary      = "say something to smmarize your pod"
	  s.homepage     = "HOME_PAGE_URL"
	  s.license      = "LICENSE_NAME"
	  s.author       = "you"
	  s.source       = { :git => "GIT_URL", :tag => "#{s.version}" }
	  s.platform      = :ios, 'ios platform minimun version'
	  s.source_files  = 'PATH_FROM_GIT_ROOT_PATH_TO_YOUR_FILE_PATH/*.{.h, .m, .swift}'
	  s.dependency  'Socket.IO-Client-Swift'
	  s.dependency  'XCGLogger'	
	end
	```
	
## 验证podspec
1. 本地验证

	```shell
	pod lib lint --allow-warnings
	```
2. 远程验证

	```shell
	pod spec lint --allow-warnings
	```
	
## 推送到远程的cocoapods私有仓库
在推送之前，需要为GIT_URL对应的git仓库打上对应的version tag(如本文例子中应该为0.0.1)。

使用以下命令将你的pod库推送到私有仓库。

```shell
pod repo push LOCAL_SPEC_PATH YourPodName.podspec --allow-warnings
```
该LOCAL\_SPEC_PATH是用一下命令制造的私有仓库的本地路径名称。
	
```shell
pod repo add LOCAL_SPEC_PATH SOURCE_URL
```
	
## 更新和删除cocoapods仓库
1. 更新
	
	更新pods之前，需要增加podspec的s.version，推送到git仓库，并重新打上对应的version tag。之后，使用上面的推送命令就可以完成更新。
	
	```shell
	pod repo push LOCAL_SPEC_PATH YourPodName.podspec --allow-warnings
	```

2. 删除

	* 删除本地pod repo
	
		```shell
		pod repo remove SPEC_PATH_TO_DELETE
		```
	* 删除远程私有pod repo
	
		接上面删除本地repo操作
		
		```shell
		cd ~/.cocoapods/repos/ # 到cocoapods的本地仓库地址
		git status             # 查看git状态，会发现SPEC_PATH_TO_DELETE整个path下的文件会被移除掉
		git add .              # 添加git更改 
		git commit -a          # git commit并且需要添加commit信息
		git push               # 推送到podspec远端仓库
		```

## 遇到的问题及解决方法
1. 问题：本地验证时，有信息没有填写出现waring，如s.license = ""，导致验证失败。
	
	解决方法：使用```pod lib lint --allow-warnings```代替```pod lib lint```。
2. 问题：没有指定Swift版本，导致验证时编译失败。

	解决方法：使用 ```echo "x.x" > .swift-version```命令指定Swift版本。
	
3. podspec依赖的是一个私有库时，验证时，出现```- ERROR | [iOS] unknown: Encountered an unknown error (Unable to find a specification for `WCDB (= 1.0.4)` depended upon by `CRIM`) during validation.```。
	
	解决方法：使用```pod lib lint --sources='YOUR_PRIVATE_LIBRARY_PATH' --allow-warnings```
	
4. 使用wcdb时，因为wcdb有一个宏在做`ifndef __cplusplus`的类型判断，如果.m文件为非objective-c++ source，则会被抛出异常。所以，cocoapods库里面包含wcdb时，不能把引用<wcdb/wcdb.h>的头文件设置为public source header，否则会在umbrella header中对相关文件调用，从而导致非oc++的异常。

## Reference
1. [创建 podspec 文件，给自己写的框架添加 CocoaPos 支持](https://www.jianshu.com/p/4d73369b8cf9)
2. [创建私有cocoapods repo库 —— Private Pods](https://blog.csdn.net/andanlan/article/details/51713595)
3. [使用cocoapods打包静态库](https://www.jianshu.com/p/9096a2eb2804)
4. [制作Swift和Objective-C Mixed的Pod](https://www.jianshu.com/p/c7623c31d77b)
5. [动态库静态库优缺点](https://www.jianshu.com/p/4e0fd0214152)