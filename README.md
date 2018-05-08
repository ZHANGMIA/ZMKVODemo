# ZMKVODemo
KVO底层原理及自定义KVO

### 一、KVO
KVO是Key Value ObserVing 的简称，也称为键值监听，当指定对象的属性被修改之后，主动通知观察者对象。

即指定一个被观察对象，当对象某个属性发生更改时，对象会发送一个通知给监听者，以便监听者执行回调操作。常见的KVO应用例如监听scrollView的contentOffset属性。

### 二、如何使用系统KVO
KVO的使用非常简单，只需要为需要被监听的对象添加监听，并在回调方法内进行处理即可。
```
Person *p = [[Person alloc] init];

// 给Person对象的属性name添加监听

// @param observer ： 观察者，处理监听事件的对象
// @param keyPath ： 需要监听的属性
// @param options ： 要监听新值还是旧值 也可以都观察
// @param context ： 上下文，用于传递数据，可以利用上下文区分不同的监听
[p addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:nil];
```
当person对象的属性发生变化时，调用此方法
```
// @param keyPath ： 监听的属性名
// @param object ： 属性所属的对象
// @param change ： 属性的修改情况
// @param context ： 上下文，用于传递数据，可以利用上下文区分不
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context
{
    // 执行操作
    if ([keyPath isEqualToString:@"name"]) {
        
        NSLog(@"== %@",change[NSKeyValueChangeNewKey]);
    }
}
```


### 三、KVO的底层实现原理
当给对象Person对象添加监听时，系统内部会动态生成一个Person类的子类NSKVONotifying_Person类，并为这个新的子类重写了被观察属性keyPath的setter 方法， 以及替换原对象的isa指针。

######<1 NSKVONotifying_Person类剖析：
在这个过程，被观察对象的 isa 指针从指向原来的Person类，被修改为指向系统新创建的子类 NSKVONotifying_Person类，来监听当前类属性值的改变

```
    // 实例化Person对象
    Person *p = [[Person alloc] init];

    // 断点1 在控制台打印p的类Person
    p.name = @"July";

    // 断点2 在控制台打印p的类为NSKVONotifying_Person
    [p addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:nil];

```


![image.png](https://upload-images.jianshu.io/upload_images/2474916-dcd22ccb0a0dbe33.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以当我们从应用层面上看来，完全没有意识到有新的类出现，这是系统“隐瞒”了对KVO的底层实现过程，让我们误以为还是原来的类。但是此时如果我们新建一个名为NSKVONotifying_Person的类，控制台会输出观察者注册监听失败。
![image.png](https://upload-images.jianshu.io/upload_images/2474916-bb6e4d3a1b8fb1ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

######<2 子类setter方法剖析：
KVO的键值观察通知依赖于 NSObject 的两个方法,
```
// 被观察属性发生改变之前，willChangeValueForKey 被调用
// 通知系统该 keyPath 的属性值即将改变
- (void)willChangeValueForKey:(NSString *)key;
```
```
// 当改变发生后， didChangeValueForKey 被调用
// 通知系统该 keyPath 的属性值已经改变
- (void)didChangeValueForKey:(NSString *)key;
```

举个栗子：
```
-(void)setName:(NSString *)newName
{
     //KVO在调用存取方法之前调用
    [self willChangeValueForKey:@"name"];   

     //调用父类的存取方法
    [super setValue:newName forKey:@"name"]; 
   
     //KVO在调用存取方法之后调用
    [self didChangeValueForKey:@"name"];    
}
```

之后会调用observeValueForKey:ofObject:change:context:方法 。

只需要在Person类里重写这两个方法，则可以证明是否会执行这两个方法了。

```
- (void)willChangeValueForKey:(NSString *)key {

    NSLog(@"%s", __func__);
    [super willChangeValueForKey:key];
}

- (void)didChangeValueForKey:(NSString *)key {
    
    NSLog(@"%s", __func__); 
    [super didChangeValueForKey:key];
}
```

![image.png](https://upload-images.jianshu.io/upload_images/2474916-de61387428d72078.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 四、自己动手实现KVO


##### <1 为什么要自己实现KVO呢
相信用过KVO的童鞋都能感受到KVO的不便吧，例如

```
<1 当监听属性过多时，所有判断都写在 -observeValueForKeyPath:ofObject:change:context:里，内部非常混乱
<2 keyPath是以NSString字符串格式定义，容易出错且不会有警告
<3 忘记移除观察者
```

##### <2 KVO实现

```
#import <Foundation/Foundation.h>

typedef void(^kvoBlock)(NSString *value);

@interface NSObject (ZMKVO)

- (void)zm_Observer:(NSObject *)object keyPath:(NSString *)keyPath block:(kvoBlock)block;

@end
```

```
#import "NSObject+ZMKVO.h"
#import <objc/runtime.h>

typedef void(^deallocBlock)(void);

@interface ZMKVOController : NSObject

@property(nonatomic, strong)NSObject *observedObject;

@property(nonatomic, strong)NSMutableArray <deallocBlock>*blockArr;

@end

@implementation ZMKVOController

-(NSMutableArray<deallocBlock> *)blockArr {
    
    if (!_blockArr) {
        _blockArr = [NSMutableArray array];
    }
    return _blockArr;
}

//nextviewController -> kvoController
- (void)dealloc {
    
    ///对observedObject  removeObserver
    NSLog(@"kvoController dealloc");
    
    [_blockArr enumerateObjectsUsingBlock:^(deallocBlock _Nonnull block, NSUInteger idx, BOOL * _Nonnull stop) {
        
        block();
        
    }];
}

@end

@interface NSObject()

@property(nonatomic, strong)NSMutableDictionary <NSString *, kvoBlock>*dict;
@property(nonatomic, strong)ZMKVOController *kvoController;

@end

@implementation NSObject (ZMKVO)

- (void)zm_Observer:(NSObject *)object keyPath:(NSString *)keyPath block:(kvoBlock)block
{
    self.dict[keyPath] = block;
    
    self.kvoController.observedObject = object;
    
    __unsafe_unretained typeof(self) weakSelf = self;
    
    [self.kvoController.blockArr addObject:^{
        //
        [object removeObserver:weakSelf forKeyPath:keyPath];
    }];
    
    // 添加监听
    [object addObserver:self forKeyPath:keyPath options:NSKeyValueObservingOptionNew context:NULL];
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context
{
    kvoBlock block = self.dict[keyPath];
    
    if (block) {
        block(change[NSKeyValueChangeNewKey]);
    }
}

////getter 和 setter方法
- (NSMutableDictionary<NSString *,kvoBlock> *)dict
{
    NSMutableDictionary *tempDict = objc_getAssociatedObject(self, @selector(dict));
    
    if (!tempDict) {
        tempDict = [NSMutableDictionary dictionary];
        objc_setAssociatedObject(self, @selector(dict), tempDict, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    return tempDict;
}

- (ZMKVOController *)kvoController
{
    ZMKVOController *tempKvoController = objc_getAssociatedObject(self, @selector(kvoController));
    
    if (!tempKvoController) {
        tempKvoController = [[ZMKVOController alloc] init];
        
        objc_setAssociatedObject(self, @selector(kvoController), tempKvoController, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    
    return tempKvoController;
}

@end

```
[附上我的简书地址](https://www.jianshu.com/p/9308cc001114)

如果你看完有些小的收获，请为我Star哦，蟹蟹
