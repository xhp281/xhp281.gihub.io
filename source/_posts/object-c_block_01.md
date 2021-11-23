title: Block本质
categories:
  - test_categories
tags:
  - test_tag
date: 2021-11-23 14:50:18
---
### Block本质
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

将示例代码转换为C++查看内部结构，与OC代码进行比较，转换代码如下
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
1.`__block_impl`结构体中 `isa` 指针存储着 `&_NSConcreteStackBlock` 地址，可以暂时理解为其类对象地址，block就是 `_NSConcreteStackBlock` 类型的。
2.block代码块中的代码呗封装成 `__main_block_func_0` 函数，`FuncPtr` 存储着 `__main_block_func_0` 函数的地址。
3.`Desc` 指向 `__main_block_desc_0`结构体对象，其中存储 `__main_block_impl_0` 结构体所占用的内存大小。

#### 调用block执行内部代码
```c++
// 执行block代码
((void (*)(__block_impl *, int, int))((__block_impl *)block)->FuncPtr)((__block_impl *)block, 2, 1);
```
通过上述代码可以发现，block调用是通过 `block` 找到 `FuncPtr` 直接调用，通上面分析可以明确 block 执行的是 ```__main_block_impl_0``` 类型的结构体，但是在 `__main_block_impl_0` 内部并不能直接找到 `FuncPtr`，`FuncPtr` 是存在 `__block_impl` 中的。

<code>\`1111</code>

比如`突出背景色`来显示强调效果