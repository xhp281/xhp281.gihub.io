



title: Block本质
categories:

  - objc
tags:
  - Block
date: 2021-11-23 14:50:18
---
## Block本质
> 1、Block本质是一个对象，内部有一个isa指针
> 2、封装了函数调用以及函数调用环境的OC对象

### 探索本质
```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
         int age = 10; // 默认是 auto 类型变量
         void(^block)(int, int) = ^(int a, int b){
             NSLog(@"this is block a = %d, b = %d", a, b);
             NSLog(@"this is block, age = %d",age);
         };
         block(2,1);
    }
    return 0;
}
```

将示例代码转换为C++查看内部结构，与OC代码进行比较，转换代码如下:
`xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m`

编译之后的代码如下
```objc
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
         int age = 10;
         void(*block)(int, int) = ((void (*)(int, int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, age));

         ((void (*)(__block_impl *, int, int))((__block_impl *)block)->FuncPtr)((__block_impl *)block, 2, 1);
    }
    return 0;
}
```

#### 定义block变量
```c++
void(*block)(int, int) = ((void (*)(int, int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, age));
```
上述代码中，可以发现block的定义调用了 `__main_block_impl_0` 函数，并将 `__main_block_impl_0` 的地址赋值给了block。下面我们分析下 `__main_block_impl_0` 的内部结构。

##### __main_block_impl_0 结构体
```c++
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int age;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _age, int flags=0) : age(_age) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

在 `__main_block_impl_0` 结构体内部有一个同名构造函数 `__main_block_impl_0`，构造函数对变量进行了赋值，最终返回一个结构体。
构造函数中对一些变量进行了赋值（或者其他操作）最终会返回一个结构体。
`__main_block_impl_0` 内部的构造函数传入了四个参数，`(void *)__main_block_func_0`、`&__main_block_desc_0_DATA`、`age`、`flags`。其中 `flags` 有默认参数，在函数调用的时候可以省略不传。最后的 `age(_age)` 表示传入的 `_age` 会自动赋值给结构体内部的 `age` 成员，相当于 `age = _age`。

接下来分析一下前面三个参数的具体内容

##### (void *)__main_block_func_0
```c++
static void __main_block_func_0(struct __main_block_impl_0 *__cself, int a, int b) {
  int age = __cself->age; // bound by copy
  NSLog((NSString *)&__NSConstantStringImpl__var_folders_nf_7714tb2x50vctp5b0v2xmpt4nrww4l_T_main_6c06d8_mi_0, a, b);
  NSLog((NSString *)&__NSConstantStringImpl__var_folders_nf_7714tb2x50vctp5b0v2xmpt4nrww4l_T_main_6c06d8_mi_1,age);
}
```

在 `__main_block_func_0` 函数中首先取出block中 `age` 的值，然后是两个NSLog的打印信息。在 `__main_block_func_0` 内部存储了block引用的外部的变量的值。而 `__main_block_impl_0` 函数中传入的第二个参数是 `(void *)__main_block_func_0`,也就是说block块内部的代码封装成了 `__main_block_func_0` 函数，并将 `__main_block_func_0` 函数的地址传入了 `__main_block_impl_0` 的构造函数中保存在了结构体内。

##### &__main_block_desc_0_DATA
```c++
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
```

在 `__main_block_desc_0_DATA` 内部存储了两个参数，`reserved` 和 `Block_size`, `reserved` 直接赋值为0，`Block_size` 存储的是 `__main_block_impl_0` 所占用空间大小。最终将 `__main_block_desc_0` 结构体的地址传入 `__main_block_impl_0` 并赋值给 `Desc`

##### age
`age` 是我们定义的局部变量，因为在block内部使用到了局部变量，所以block声明的时候，这里会将 `age` 作为参数传入，也就是说block会捕获 `age`，如果没有在block中使用 `age` ，这里将只会传入 `(void *)__main_block_func_0` 和 `&__main_block_desc_0_DATA` 这两个参数，如果有多个参数，那么传入的时候就会变成多个

```c++
// 没有捕获的情况
void(*block)(int, int) = ((void (*)(int, int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
// 捕获多个变量
void(*block)(int, int) = ((void (*)(int, int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, age, name, numer));
```

#### 再次查看 __main_block_impl_0 
```c++
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int age;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _age, int flags=0) : age(_age) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;  // block内部代码块地址
    Desc = desc;        // block描述信息(占用内存大小)
  }
};
```
`__main_block_impl_0` 的第一个变量就是 `__block_impl` 结构体，内部代码如下：
```c++
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};
```

`__block_impl` 内部有一个 `isa` 指针。因此可以证明block本质上就是一个OC对象。而在构造函数中将函数传入的值分别存储在 `__main_block_impl_0` 结构体实例中，最终将结构体的地址赋值给block。

通过上面对 `__main_block_impl_0` 结构体构造函数三个参数的问题，可以得出结论：
1.`__block_impl`结构体中 `isa` 指针存储着 `&_NSConcreteStackBlock` 地址，可以暂时理解为其类对象地址，block 就是 `_NSConcreteStackBlock` 类型的。
2.block代码块中的代码呗封装成 `__main_block_func_0` 函数，`FuncPtr` 存储着 `__main_block_func_0` 函数的地址。
3.`Desc` 指向 `__main_block_desc_0`结构体对象，其中存储 `__main_block_impl_0` 结构体所占用的内存大小。

#### 调用block执行内部代码
```c++
// 执行block代码
((void (*)(__block_impl *, int, int))((__block_impl *)block)->FuncPtr)((__block_impl *)block, 2, 1);
```
通过上述代码可以发现，block 调用是通过 `block` 找到 `FuncPtr` 直接调用，通上面分析可以明确 block 执行的是 __main_block_impl_0 类型的结构体，但是在 `__main_block_impl_0` 内部并不能直接找到 `FuncPtr`，`FuncPtr` 是存在 `__block_impl` 中的。



`block` 可以直接调用 `__block_impl` 中的 `FuncPtr`原因，通过查看上述源码可以发现，`(__block_impl *)` 将 block 强制转换为了 `__block_impl` 类型，因为 `__block_impl` 是 `__main_block_impl_0` 结构体的第一个成员，相当于将 `__block_impl` 结构体的成员拿出来直接放在 `__main_block_impl_0` 中，也就说明 `__block_impl` 的内存地址就是 `__main_block_impl_0` 结构体内存地址的开头。所以可以转换成功，并找到 `FuncPtr` 成员。



通过上述可以知道，`FuncPtr` 存储着代码块的函数地址，调用此函数就会执行代码块中的代码，回头查看 `__main_block_func_0` ，可以发现第一个参数就是 `__main_block_impl_0` 类型的指针。也就是说将 `block` 传入 `__main_block_func_0` 函数中，便于从中捕获 `block` 的值。



#### 验证block的本质确实是 __main_block_impl_0 结构体类型

通过代码证明一下上述内容：通用使用之前的方法，我们按照上面分析block内部结构自定义结构体，并将block内部的结构体强制转换为自定义的结构体，转换成功说明底层结构体确实如之前分析的一样。

```objc
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int age;
};

int main(int argc, const char * argv[]) {
    @autoreleasepool {
         int age = 10; // 默认是 auto 类型变量
         void(^block)(int, int) = ^(int a, int b){
             NSLog(@"this is block a = %d, b = %d", a, b);
             NSLog(@"this is block, age = %d",age);
         };
        // 进行类型转换
        struct __main_block_impl_0 *blockStruct = (__bridge struct __main_block_impl_0 *)block;
         block(2,1);
    }
    return 0;
}
```

通过xcode断点可以看出自定义结构体可以被成功赋值。

 ![block_自定义构体验证](https://xhp281-blog.oss-cn-beijing.aliyuncs.com/ios_objc/block_%E8%87%AA%E5%AE%9A%E4%B9%89%E6%9E%84%E4%BD%93%E9%AA%8C%E8%AF%81.jpeg)

接下来断点来到block代码块中，看下对战信息中心的函数调用地址。`Debug -> Debug workflow -> always show Disassembly` 。

 ![block_自定义构体验证_汇编](https://xhp281-blog.oss-cn-beijing.aliyuncs.com/ios_objc/block_%E8%87%AA%E5%AE%9A%E4%B9%89%E6%9E%84%E4%BD%93%E9%AA%8C%E8%AF%81_%E6%B1%87%E7%BC%96.jpeg)

通过上图可以看到 `block` 的地址确实和 `FuncPtr` 的地址是一样的。

### 总结

通过上述分析已经对block底层有了一个基本认识，将上述代码转换为一张图片看下具体的关系。

![](https://xhp281-blog.oss-cn-beijing.aliyuncs.com/ios_objc/block_%E7%BB%93%E6%9E%84%E4%BD%93%E5%85%B3%E7%B3%BB%E5%9B%BE.png)

block的底层数据可以通过一张图来展示

![block_底层数据结构](https://xhp281-blog.oss-cn-beijing.aliyuncs.com/ios_objc/block_%E5%BA%95%E5%B1%82%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.png)

## block变量的捕获

为了保证block内部能正常访问外部的变量，block有一个变量捕获机制。

### 局部变量

#### auto变量

上述代码中已经了解过block对age变量的捕获。auto自动变量，离开作用域就销毁，通常局部变量前面自动添加 `auto` 关键字。自动变量会捕获到block内部，也就是说block会专门新增一个参数来存储变量的值。`auto` 只存在于局部变量中，访问方式为值传递，通过上述对age参数额解释，我们也可以确定确实是值传递。

#### static变量

static修饰的变量为指针传递，同样会被block捕获。

接下来分别添加auto修饰的局部变量和static修饰的局部变量，通过源码查看下他们的区别。

```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        auto int a = 66;
        static int b = 99;
        void(^block)(void) = ^{
            NSLog(@"this is block a = %d, b = %d", a, b);
        };
        a = 88;
        b = 77;
        block();
    }
    return 0;
}
// log：this is block a = 66, b = 77
// block中a的值并没有被改变，b的值被改变了
```

将源码转换成c++代码之后的两个参数的区别如下图：

![](https://xhp281-blog.oss-cn-beijing.aliyuncs.com/ios_objc/block_%E4%B8%8D%E5%90%8C%E5%8F%98%E9%87%8F%E6%8D%95%E8%8E%B7%E7%9A%84%E5%8C%BA%E5%88%AB.jpeg)

从上述源码中可以看出，a，b两个变量都有捕获到block内部。但是a传入的是值，b传入的是地址。

这两种变量产生差异的原因是，自动变量可能会销毁，block执行的时候自动变量有可能已经被销毁了，那么此时再去访问被销毁的地址肯定会发生坏的内存访问，因此对于自动变量一定是值传递而不是指针传递。而静态变量不会被销毁，所以完全可以传递地址。因为传递的是值的地址，所以block调用之前修改地址中保存的值，block中的地址是不会变的，所以值会随之改变。

### 全局变量

示例代码：

```objc
int a = 66;
static int b = 99;
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        void(^block)(void) = ^{
            NSLog(@"this is block a = %d, b = %d", a, b);
        };
        a = 88;
        b = 77;
        block();
    }
    return 0;
}
// log：this is block a = 88, b = 77
```

生成的c++代码看下全局调用方式，可以发现结构体中没有a，b的成员变量，传递的时候是直接调用。

![block_全局变量_c++源码](https://xhp281-blog.oss-cn-beijing.aliyuncs.com/ios_objc/block_%E5%85%A8%E5%B1%80%E5%8F%98%E9%87%8F_c%2B%2B%E6%BA%90%E7%A0%81.jpg)

通过上述代码可以发现，`__main_block_impl_0` 没有添加任何变量，因此block不需要捕获全局变量，因为全局变量在哪都可以访问。

**局部变量因为跨函数访问所以需要捕获，全局变量哪里都可以访问，因此不需要捕获。**

|      变量类型      | 捕获到block内部 | 访问方式 |
| :----------------: | :-------------: | :------: |
|  局部变量（auto）  |        ✅        |  值传递  |
| 局部变量（static） |        ✅        | 指针传递 |
|      全局变量      |        ❌        | 直接访问 |

**总结：局部变量都会被block捕获，自动变量是值捕获，静态变量是地址捕获，全局变量则不会被捕获。**

#### 疑问：以下代码中block是否会捕获变量呢？

1、在block中使用self会不会被捕获？

```objc
#import "Person.h"
@implementation Person
- (void)test {
    void(^block)(void) = ^{
        NSLog(@"%@",self);
    };
    block();
}
- (instancetype)initWithName:(NSString *)name {
    if (self = [super init]) {
        self.name = name;
    }
    return self;
}
+ (void) test2 {
    NSLog(@"类方法test2");
}
@end
```

同样转化为c++代码查看其内部结构

![block_捕获_示例代码](https://xhp281-blog.oss-cn-beijing.aliyuncs.com/ios_objc/block_%E6%8D%95%E8%8E%B7_%E7%A4%BA%E4%BE%8B%E4%BB%A3%E7%A0%81.jpg)

上图中可以发现，self同样被block捕获，通过test方法发可以发现，test方法默认传递了两个参数self和_cmd。类方法test也是同样的。

不论对象方法还是类方法都会默认将self作为参数传递给方法内部，既然是作为参数传入，那么self肯定是局部变量。局部变量肯定会被block捕获。

2、在block中使用成员变量或者调用实例的属性会有什么不同的结果？

```objc
- (void)test {
    void(^block)(void) = ^{
        NSLog(@"%@",self.name);
        NSLog(@"%@",_name);
    };
    block();
}
```

![block_自定义对象属性和成员变量](https://xhp281-blog.oss-cn-beijing.aliyuncs.com/ios_objc/block_%E8%87%AA%E5%AE%9A%E4%B9%89%E5%AF%B9%E8%B1%A1%E5%B1%9E%E6%80%A7%E5%92%8C%E6%88%90%E5%91%98%E5%8F%98%E9%87%8F.jpg)

通过上面图片可以发现，即使block中使用的是实例对象的属性，block中捕获的仍然是实例对象，并通过实例对象通过不同的方式去获取使用到的属性。

### block的类型

block对象是什么类型的，之前的代码中提到过，通过源码可以知道block中的isa指针指向的是 `_NSConcreteStackBlock` 类对象地址。那么block的类型是否就是 `_NSConcreteStackBlock`类型吗？

通过代码调用class方法或者isa指针查看具体类型。

```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        void (^block)(void) = ^{
            NSLog(@"Hello");
        };
        
        NSLog(@"%@", [block class]);
        NSLog(@"%@", [[block class] superclass]);
        NSLog(@"%@", [[[block class] superclass] superclass]);
        NSLog(@"%@", [[[[block class] superclass] superclass] superclass]);
    }
    return 0;
}
// log： __NSGlobalBlock__ : __NSGlobalBlock : NSBlock : NSObject
```

从上述打印内容可以看出block最终都是继承自NSBlock类型，而NSBlock继承于NSObjcet。那么block其中的isa指针其实是来自NSObject中的。证明了block的本质其实就是OC对象。

#### block的3种类型

block有3中类型

```objective-c
__NSGlobalBlock__ （ _NSConcreteGlobalBlock ）
__NSStackBlock__  （ _NSConcreteStackBlock ）
__NSMallocBlock__ （ _NSConcreteMallocBlock ）
```

通过代码查看一下block在什么情况下其类型会各不相同

```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // 1. 内部没有调用外部变量的block
        void (^block1)(void) = ^{
            NSLog(@"Hello");
        };
        // 2. 内部调用外部变量的block
        int a = 10;
        void (^block2)(void) = ^{
            NSLog(@"Hello - %d",a);
        };
       // 3. 直接调用的block的class
        NSLog(@"%@ %@ %@", [block1 class], [block2 class], [^{
            NSLog(@"%d",a);
        } class]);
    }
    return 0;
} 
```

通过打印内容确实发现block的三种类型

```shell
__NSGlobalBlock__ __NSMallocBlock__ __NSStackBlock__
```

上述代码转化为c++代码查看源码时却发现block的类型与打印出来的类型不一样，c++源码中三个block的isa指针全部都指向_NSConcreteStackBlock类型地址。

可以猜测runtime运行时过程中也许对类型进行了转变。最终类型当然以runtime运行时类型也就是我们打印出的类型为准。

#### block的内存中的存储



上图中可以发现，根据block的类型不同，block存放在不同的区域中。 数据段中的 `__NSGlobalBlock__` 直到程序结束才会被回收，不过我们很少使用到 `__NSGlobalBlock__` 类型的block，因为这样使用block并没有什么意义。

`__NSStackBlock__` 类型的block存放在栈中，我们知道栈中的内存由系统自动分配和释放，作用域执行完毕之后就会被立即释放，而在相同的作用域中定义block并且调用block似乎也多此一举。

#### block的如何定义其类型
