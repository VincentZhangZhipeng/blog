## KVO底层原理小结
> KVO即键值对监听，是对键值监听，当键对应的属性值发生改变时，发送通知给监听者的方法。根据Apple官方文档，KVO是基于```isa-swizzling```技术来实现的。本文将会探讨KVO的底层实现原理，并根据自己的理解实现简单的KVO demo。

### Insight
下面将用自定义的Person类进行说明。Person类有name属性。

```objc
@interface Person: NSObject
@property (nonatomic, strong) NSString *name;
@end
```

为Person类的name添加KVO。

```objc
Person *person = [Person new];
NSLog(@"person isa is <%@: %p>", object_getClass(person) , object_getClass(person));
NSLog(@"====== after kvo ======");
[person addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:nil];
NSLog(@"person isa is <%@: %p>", object_getClass(person), object_getClass(person));
```
输出结果为：

```objc
person isa is <Person: 0x10149dac0>
====== after add kvo ======
person isa is <NSKVONotifying_Person: 0x60400010a5f0>
====== after remove kvo ======
person isa is <Person: 0x10149dac0>
```
从输出结果不难看出，person实例在添加监听以后，isa指针从原来的Person指向了新的中间类NSKVONotifying_Person，而NSKVONotifying_Person继承自Person，拥有person的属性和其他方法。这个过程就是```isa-swizzling```。


### isa-swizzling
在```addObserver:keyPath:options:context:```方法中，利用isa-swizzling，实现了如下效果：添加带有```NSKVONotifying_```前缀的中间类，并将调用该方法的实例的```isa```指向这个中间类。而该中间类继承自原来```isa```指向的类。由此，中间类也有了继承自父类的setter方法列表。可以在该中间类中添加新方法实现监听的通知。

以下代码是本人对isa-swizzling实现的过程。完整代码请看[这里](https://github.com/VincentZhangZhipeng/VCKVO)。

```objc
static void vcCreateInerClass(id target, NSObject * observer, NSString* getterName) {
	/** isa swizzling
	 * 1. 判断当前类的类型，以及中间类是否已经存在。如果既不是中间类，中间类也不存在，创建集成自原类的中间类
	 * 2. 根据keyPath拼接成setter样式的selector，根据selector获取原setter参数类型并为中间类动态添加新方法
	 * 3. 将self实例的isa指向中间类
	 **/
	Class originalClass = object_getClass(target);
	Class inerClass;
	if ([NSStringFromClass(originalClass) hasPrefix: prefix]) {
		inerClass = originalClass;
	} else if(NSClassFromString([prefix stringByAppendingString:NSStringFromClass(originalClass)])) {
		inerClass = NSClassFromString([prefix stringByAppendingString:NSStringFromClass(originalClass)]);
	} else {
		const char* inerClassName = [NSString stringWithFormat:@"%@%@", prefix, NSStringFromClass(originalClass)].UTF8String;
		inerClass = objc_allocateClassPair(originalClass, inerClassName, 0);
		// 注册class，注意未注销前只能注册一次
		objc_registerClassPair(inerClass);
	}
	
	// 添加方法来触发通知
	...
	
	// 把instance的isa指向inerClass
	object_setClass(target, inerClass);
}
```

### 添加方法

在为原始类（Person）添加的中间类（NSKVONotifying_Person）中，根据```keyPath```里面的属性名，替换其setter方法为新方法。在该新方法调用的setter，并调用```- (void)observeValueForKeyPath:ofObject:change:context:```以实现改变数值后，触发通知。于是，在上文中的```vcCreateInerClass```函数中动态添加方法:

```objc
NSString *setterSel = [NSString stringWithFormat:@"set%@%@:", [keyPath substringToIndex:1].uppercaseString, [keyPath substringFromIndex:1] ];
	
// 获取入参方法类型
Method method = class_getInstanceMethod(originalClass, NSSelectorFromString(setterSel));
	const char *types = method_getTypeEncoding(method);
	
class_addMethod(inerClass, NSSelectorFromString(setterSel), (IMP)VCSetter, types);
```

同时，用```NSMapTable```类设置value是weak的mapTable来存放observer，这样做的好处是当observer被销毁，mapTable会自动移除:

```objc
NSMapTable<NSString *, id> *map = objc_getAssociatedObject(target, _observer);
if(!map) {
	map = [NSMapTable mapTableWithKeyOptions:NSPointerFunctionsStrongMemory valueOptions:NSPointerFunctionsWeakMemory];
}
[map setObject:observer forKey:getterName];
	
// 动态将observer添加到实例
objc_setAssociatedObject(self, _observer, map, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
```

由于原始类中属性的setter方法的入参可能为基本数据类型，也可能为值类型。所以```VCSetter```为c语言的可变参数函数:

```objc
static void VCSetter(id self, SEL _cmd, ...)   {
	...
	// 获取参数的类型
	NSMethodSignature *methodSignature = [inerClass instanceMethodSignatureForSelector:_cmd];
	const char *argumentType = [methodSignature getArgumentTypeAtIndex:2];
	
	id obj;
	va_list args;
	va_start(args, _cmd);
	switch (argumentType[0]) {
		case 'c':
			obj = @(va_arg(args, char));
			break;
		case 'i':
			obj = @(va_arg(args, int));
			break;
		...
		default:
			obj = (va_arg(args, id));
			break;
	}
	va_end(args);
	
	Class inerClass = object_getClass(self);
	Class originalClass = class_getSuperclass(inerClass);
	struct objc_super superInfo = {
		self,
		originalClass
	};
	
	// 调用父类的setter方法
	((void (*) (void * , SEL, ...))objc_msgSendSuper)(&superInfo, _cmd, obj);
	
	// 获取观察者
	NSMapTable * observerMap = objc_getAssociatedObject(self, _observer);
	id observer = [observerMap objectForKey: getterName];
	SEL sel = NSSelectorFromString(@"vc_observeValueForKeyPath:ofObject:change:");
	if (observer) {
		if([observer respondsToSelector:sel]) {
			// objc_msgSend方法来调用vc_observeValueForKeyPath:ofObject:change:方法
			((void(*) (id, SEL, ...))objc_msgSend)(observer, sel, getterName, self, obj);
		}
	}
}
```
上文代码中用到```va_list```来获取动态的参数```obj```。然后调用```runtime```的```objc_msgSendSuper```方法，将obj作为setter的入参传给原始类进行set操作。

然而，在arm 64的机子中，```va_list```的数据结构发生了改变，使用上述方法的实现会导致crash，详见[这篇blog](https://blog.nelhage.com/2010/10/amd64-and-va_arg/)。在参考了JSPatch的实现方法后，决定将中间类的setter指向完整消息转发方法forwardInvocation，通过```NSInvocation```类来实现不确定是值类型还是指针类型的setter入参的获取。实现如下：

```objc
swizzleForwardInvocation(inerClass);
// 将setter的selector指向_objc_msgForward实现，触发完整的消息转发
class_replaceMethod(inerClass, NSSelectorFromString(setterSel), _objc_msgForward, types);
```

```objc
static void swizzleForwardInvocation(Class _Nonnull class) {
	/** Method Swizzling
	 * 1. 设置新的forwardInvocation方法--vcForwardInvocationImp
	 * 2. 在新forwardInvocation方法中，对非setter方法的SEL调用原forwardInvocation
	 * 3. 调用getArgument方法获得setter入参，并调用父类也即原始类的setter方法
	 * 4. 获取observe并使用objc_msgSend方法来调用vc_observeValueForKeyPath:ofObject:change:方法
	 * 5. 替换forwardInvocation SEL为vcForwardInvocationImp实现
	 **/
	
	id vcForwardInvocationImp = ^(id target, NSInvocation *invocation) {
		SEL selector = invocation.selector;
		NSString *setterName = NSStringFromSelector(selector);
		
		if(![setterName hasPrefix:@"set"]) {
			if(originalForwardInvocationImp) {
				originalForwardInvocationImp(target, selector, invocation);
			} else {
				[target doesNotRecognizeSelector:selector];
			}
			return;
		}
		
		// 3
		id value = getArgument(invocation);
		...
		
		// 4
		...
	
	// 5
	class_replaceMethod(class, forwardInvocationSelector, imp_implementationWithBlock(vcForwardInvocationImp), orginalEncodingTypes);
	imp_removeBlock(imp_implementationWithBlock(vcForwardInvocationImp));
}
```
其中，```getArgument```方法是以```NSInvocation```类为入参，调用invocation的```getArgument```方法来获取setter的入参。

```objc
static id getArgument(NSInvocation *invocation){
	id value;
	if (invocation.methodSignature.numberOfArguments > 2) {

		// 宏定义，根据type获得对应类型的参数
#define getValue(type) \
type arg;\
[invocation getArgument:&arg atIndex:2];\
value = @(arg);\

		switch ([invocation.methodSignature getArgumentTypeAtIndex:2][0]) {
			case 'c':
			{
				getValue(char)
				break;
			}
			case 'i':
			{
				getValue(int)
				break;
			}
			...
			default:
				[invocation getArgument:&value atIndex:2];
				break;
		}
	}
	return value;
}
```

### keyPath链
由于keyPath的链式字符串都是以```.```作为分割符。所以通过遍历分割keyPath得到的数组，可以获得真正```property```key对应的实例target。实现如下：

```objc
- (id)targetFromKeyPath:(NSString *)keyPath {
	id target = self;
	NSArray *keyPaths = [keyPath componentsSeparatedByString:@"."];
	if (keyPaths.count > 1) {
		// 根据链式keyPath遍历获取getter对应的classclass
		for (int i=0; i<keyPaths.count-1; i++) {
			target = [target valueForKey: [keyPaths objectAtIndex:i]];
		}
	}
	return target;
}
```

### 移除KVO
调用移除KVO的方法，主要是移除mapTable里面的监听者，并且将指向中间类的isa重新指向原始的类如```Person```。

```objc
- (void)vc_removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath {
	id target = [self targetFromKeyPath:keyPath];
	Class currentClass = object_getClass(target);
	NSMapTable *map = objc_getAssociatedObject(target, _observer);
	[map removeObjectForKey: [[keyPath componentsSeparatedByString:@"."] lastObject]];
	
	if([NSStringFromClass(currentClass) hasPrefix: prefix] && map.count <= 0) {
		// 重新指向未添加中间类之前的类
		Class superClass = class_getSuperclass(currentClass);
		object_setClass(target, superClass);
	}
}
```

## GNU中的实现思路
[GNU源码](https://github.com/gnustep/libs-base)对KVO的基本实现为：用模板类`GSKVOBase`定义一些基本行为，例如Class方法、`setValue:forKey:`方法，所有的派生类（replacement）的方法列表都来自于该模版类，这样派生类就可以做成一个轻量级的对象，不用重写特定的set方法，在调用被KVO的对象其它方法时，也能保证调用的是原Class方法列表中的IMP。

```objc
// GSKVOBase Implementaion
- (Class) class
{
// 返回KVOBase的父类，也即原类，所以[class description]返回的是原类名
  return class_getSuperclass(object_getClass(self));
}

- (void) setValue: (id)anObject forKey: (NSString*)aKey
{
  Class		c = [self class];
  void		(*imp)(id,SEL,id,id);

  imp = (void (*)(id,SEL,id,id))[c instanceMethodForSelector: _cmd];

// 判断实例的key有否被kvo
  if ([[self class] automaticallyNotifiesObserversForKey: aKey])
    {
      // 有则插入willChangeValueForKey和didChangeValueForKey两个方法
      [self willChangeValueForKey: aKey];
      imp(self,_cmd,anObject,aKey);
      [self didChangeValueForKey: aKey];
    }
  else
    {
      // 否则，执行原有的set方法即可
      imp(self,_cmd,anObject,aKey);
    }
}
```

## Reference
1. [透彻理解 KVO 观察者模式（附基于runtime实现代码](https://www.jianshu.com/p/7ea7d551fc69)
2. [JSPatch实现原理](https://github.com/bang590/JSPatch/wiki/JSPatch-%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E8%AF%A6%E8%A7%A3)
3. [带着问题读源码----KVO](https://suhou.github.io/2018/01/17/%E5%B8%A6%E7%9D%80%E9%97%AE%E9%A2%98%E8%AF%BB%E6%BA%90%E7%A0%81----KVO/)