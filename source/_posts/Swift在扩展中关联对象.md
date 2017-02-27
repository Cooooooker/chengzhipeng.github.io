---
title: Swift在扩展中关联对象
date: 2015-12-27 
categories: Swift
tags: [Swift, 关联对象]
---
Objective-C 最让人诟病的也许就是不能给已有类添加属性, 但是可以通过 Objective-C 的运行时机制关联自定义属性到对象上, 几乎弥补了这个痛点.  

Swift Extension 比 Objective-C Category 增色不少, extension 能够给已有类添加计算型属性, 这已经是很大的进步, 但是仍然不能添加存储属性. Swift 中也可以使用 Objective-C runtime 的关联对象([Associated Objects](http://nshipster.cn/associated-objects/))的方式添加属性, 弥补这一痛点.


<!---more--->

## 关联对象(Associated Objects)
Swift 中提供三个与 Objective-C 类似的方法将自定义的属性关联到对象上:
1. `objc_setAssociatedObject`
2. `objc_getAssociatedObject`
3. `objc_removeAssociatedObjects`

> 注意: 使用 objc_removeAssociatedObjects 时要小心, 这个方法会删除对象关联的所有属性, 就可能导致把别人添加的关联属性也删掉. 如果要删除某一个属性, 使用 objc_setAssociatedObject 方法, value 置为 nil.

下面给 UIView 添加三种不同类型的属性: isShow, displayName, width.

```Swift
extension UIView {
    // 嵌套结构体
    private struct AssociatedKeys {
        static var isShowKey = "isShowKey"
        static var displayNameKey = "displayNameKey"
        static var widthKey = "widthKey"
    }
    
    // Bool 类型
    var isShow: Bool {
        get {
            return objc_getAssociatedObject(self, &AssociatedKeys.isShowKey) as! Bool
        }
        set {
            objc_setAssociatedObject(self, &AssociatedKeys.isShowKey, newValue, objc_AssociationPolicy.OBJC_ASSOCIATION_RETAIN_NONATOMIC)
        }
    }
    
    // String 类型
    var displayName: String? {
        get {
            return objc_getAssociatedObject(self, &AssociatedKeys.displayNameKey) as? String
        }
        set {
            objc_setAssociatedObject(self, &AssociatedKeys.displayNameKey, newValue, objc_AssociationPolicy.OBJC_ASSOCIATION_COPY_NONATOMIC)
        }
    }
    
    // Float 类型
    var width: Float {
        get {
            return objc_getAssociatedObject(self, &AssociatedKeys.widthKey) as! Float
        }
        set {
            objc_setAssociatedObject(self, &AssociatedKeys.widthKey, newValue, objc_AssociationPolicy.OBJC_ASSOCIATION_RETAIN_NONATOMIC)
        }
    }
}
```

上述列子说明几点:
- 嵌套私有结构体, 声明与扩展属性对应的键(key). [Swift Extension](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Extensions.html#//apple_ref/doc/uid/TP40014097-CH24-ID151) 提供了丰富的功能, 可以在 Extension 中嵌套类型, 使用 private 私有访问控制, 不会污染整个命名空间, 而且能够统一管理关联对象键.
- Swift 的基本类型Int, Float, Double, Bool能够自动隐式地转换成 Objective-C 的 NSNumber 类型, 所以不需要显示的包装成 NSNumber 类型进行关联.
- 如果使用 `OBJC_ASSOCIATION_ASSIGN` 关联策略时要注意, 文档中指出是弱引用, 但不完全等同于 weak, 更像是 unsafe_unretained 引用, 关联对象被释放后,关联属性仍然保留被释放的地址, 如果不小心访问关联属性, 就会造成野指针访问出错.

>	Specifies a weak reference to the associated object.

## 抽取关联对象方法
我们可以把关联对象的方法提取成公共方法, 在 NSObject 类的 extension 里实现, 只要继承自 NSObject 的类就能够调用关联对象方法, 通过[Swift 泛型](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Generics.html#//apple_ref/doc/uid/TP40014097-CH26-ID179)来关联不同类型的属性.

```Swift
extension NSObject {
    func setAssociated<T>(value: T, associatedKey: UnsafeRawPointer, policy: objc_AssociationPolicy = objc_AssociationPolicy.OBJC_ASSOCIATION_RETAIN_NONATOMIC) -> Void {
        objc_setAssociatedObject(self, associatedKey, value, policy)
    }
    
    func getAssociated<T>(associatedKey: UnsafeRawPointer) -> T? {
        let value = objc_getAssociatedObject(self, associatedKey) as? T
        return value;
    }
}
```

我们只需要在 UIView+Extension.swift 中调动上面两个方法即可, 目前只支持有可选类型的属性.

```Swift
extension UIView {
    private struct AssociatedKeys {
        static var displayNameKey = "displayNameKey"
    }
    
    var displayName: String? {
        get {
            return getAssociated(associatedKey: &AssociatedKeys.displayNameKey)
        }
        set {
            setAssociated(value: newValue, associatedKey: &AssociatedKeys.displayNameKey)
        }
    }
}   
```

## 关联闭包属性
开发中有时会给已有类关联闭包属性, 比如给 UIViewController 类添加一个 pushCompletion 的闭包属性, 当导航控制器 push 动作完成后调用该控制器的 pushCompletion 闭包.  
先按照最基本的方式来关联对象, 如下:

```Swift
typealias pushCompletionClosure = ()->()

extension UIViewController {
    private struct AssociatedKeys {
        static var pushCompletionKey = "pushCompletionKey"
    }
    
    var pushCompletion: pushCompletionClosure? {
        get {
            return objc_getAssociatedObject(self, &AssociatedKeys.pushCompletionKey) as? pushCompletionClosure
        }
        set {
            objc_setAssociatedObject(self, &AssociatedKeys.pushCompletionKey, newValue, objc_AssociationPolicy.OBJC_ASSOCIATION_COPY_NONATOMIC)
        }
    }
}
```

开开心心编译一发, 发现编译报错:  
![关联闭包报错](http://upload-images.jianshu.io/upload_images/4238758-3381fdc45c5a8855.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
定位问题发现出在 objc_setAssociatedObject 这个方法上. 原来闭包属性需要包装一下才能进行关联, 下面给出两种解决办法:
1. 使用泛型包装闭包属性, 利用 NSObject+Extension.swift 中的 `setAssociated` 方法来关联闭包.
2. 创建私有闭包容器类, 利用闭包容器间接关联闭包属性.

### 泛型包装闭包属性
setAssociated 方法需要泛型参数, 当传入闭包后, 就会把闭包包装成泛型.

```Swift
set {
   setAssociated(value: newValue, associatedKey: &AssociatedKeys.pushCompletionKey)
}
```

### 闭包容器
使用闭包容器的方式关联闭包属性, 过程分为两步:
1. 在 extension 中嵌套创建容器类, 容器类中定义需要关联的闭包属性.
2. 关联对象时把容器类对象关联到已有类, 间接的就把闭包属性关联到已有类.

闭包容器的方式是把闭包属性包装到了容器中, 再把容器对象关联到已有类上, 跟泛型包装闭包有异曲同工之处, 因此必须通过容器对象来访问闭包, 如果需要给类关联的闭包属性相对较多, 这种方式也不失为一种好方法, 能统一管理闭包属性, 代码层级结构也比较清晰.

```Swift
typealias pushCompletionClosure = ()->()

extension UIViewController {
    private struct AssociatedKeys {
        static var pushCompletionKey = "pushCompletionKey"
    }
    
    // 嵌套闭包容器类
    class closureContainer {
        var pushCompletion: pushCompletionClosure?
    }
    
    // 关联容器属性
    var container: closureContainer? {
        get {
            return objc_getAssociatedObject(self, &AssociatedKeys.pushCompletionKey) as? closureContainer
        }
        set {
            objc_setAssociatedObject(self, &AssociatedKeys.pushCompletionKey, newValue, objc_AssociationPolicy.OBJC_ASSOCIATION_RETAIN_NONATOMIC)
        }
    }
}

```

`代码在这里:`  
[github](https://github.com/ChilliCheng/AssociatedObject)

欢迎大家留言斧正!

参考链接:  
<http://swift.gg/2016/10/11/swift-extensions-can-add-stored-properties/>
<http://stackoverflow.com/questions/24133058/is-there-a-way-to-set-associated-objects-in-swift/25428409#25428409>



