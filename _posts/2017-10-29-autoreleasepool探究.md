---
layout: post
title:  "autoreleasepool探究"
date:   2017-07-29
categories: iOS
---

## 引言

开始之前先看一段代码，猜猜输出结果是什么？

```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        __weak id refStr1 = nil;
        __weak id refStr2 = nil;
        @autoreleasepool {
            NSString *str = [NSString stringWithFormat:@"hello"];
            refStr1 = str;
            refStr2 = [str stringByAppendingString:@" world"];
        }
        NSLog(@"refStr1:%@",refStr1);
        NSLog(@"refStr2:%@",refStr2);
    }
    return 0;
}
```

测试输出结果如下：

```objc
refStr1:hello
refStr2:(null)
```

refStr1为何不是空，不合常理啊！难道我所理解的自动释放池是错的！autoreleasepool到底做了什么？

## autoreleasepool探究

我们知道自动释放池用于存放那些稍后在某个时刻需要释放的对象，清空自动释放池，会向其中的对象发送release消息，释放对象。上面测试中refStr1是个弱引用，不会递增str引用计数，autoreleasepool作用域结束后，str应该释放，但refStr1仍会输出"hello"。难道str没有被释放？str有没有被加入到自动释放池中？autoreleasepool本质又是什么？
使用clang命令转化为C++代码，如果当前环境不支持weak引用，可将**weak**声明改为**__unsafe_unretained**后再执行转换，**__unsafe_unretained**也不会增加对象的引用计数，但所指向的对象释放后，其值不会置为空，再访问可能导致意想不到的错误

```objc
 clang -rewrite-objc main.m
```

找到main入口函数

```objc
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool;

        __attribute__((objc_ownership(none))) id refStr1 = __null;
        __attribute__((objc_ownership(none))) id refStr2 = __null;


        /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool;
            NSString *str = ((NSString *(*)(id, SEL, NSString *, ...))(void *)objc_msgSend)((id)objc_getClass("NSString"), sel_registerName("stringWithFormat:"), (NSString *)&__NSConstantStringImpl__var_folders_1g_jhwq37q510q1d75qmq_bhrfc0000gn_T_main_8dcd7c_mi_0);
            refStr1 = str;
            refStr2 = ((NSString *(*)(id, SEL, NSString *))(void *)objc_msgSend)((id)str, sel_registerName("stringByAppendingString:"), (NSString *)&__NSConstantStringImpl__var_folders_1g_jhwq37q510q1d75qmq_bhrfc0000gn_T_main_8dcd7c_mi_1);
        }
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_1g_jhwq37q510q1d75qmq_bhrfc0000gn_T_main_8dcd7c_mi_2,refStr1);
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_1g_jhwq37q510q1d75qmq_bhrfc0000gn_T_main_8dcd7c_mi_3,refStr2);
    }
    return 0;
}
```

其中@autoreleasepool{}块被转换成了{__AtAutoreleasePool __autoreleasepool;}，那__AtAutoreleasePool又是什么呢？继续查找其定义

```c++
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```

__AtAutoreleasePool是个结构体，定义很简单，一个构造函数，一个析构函数，一个指针。其中在构造函数中调用了objc_autoreleasePoolPush()，在析构函数中调用了objc_autoreleasePoolPop(atautoreleasepoolobj)。至于atautoreleasepoolobj又是什么东东，下文再说。可以看出@autoreleasepool{}其实等价于下面代码

```C++
void* pt =objc_autoreleasePoolPush();
//your code
objc_autoreleasePoolPop(pt);
```

objc_autoreleasePoolPush与objc_autoreleasePoolPop这两个函数实现很简单，分别调用了AutoreleasePoolPage的静态方法push与pop

```C++
void *
objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}

void
objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}
```

终于揭开autoreleasepool的神秘面纱，原来AutoreleasePoolPage才是其中的核心，其主要结构如下：

```C++
class AutoreleasePoolPage
{
    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
}
```

至于AutoreleasePoolPage的具体实现，在此不再做详细介绍，具体可参考sunnyxx[《黑幕背后的Autorelease》](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)和Draveness的[《自动释放池的前世今生》](http://www.jianshu.com/p/32265cbb2a26)。这里主要引述其结论:

* AutoreleasePool与线程一一对应，结构中的thread指向当前线程
* AutoreleasePoolPage每个对象内存大小为4096字节，除了存储其本身的实例变量外，剩下的空间用来储存加入到自动释放池中的对象的地址
* AutoreleasePoolPage以双向链表的形式组合，parent指针指向上一个page，child指针指向下一个page
* 每次调用objc_autoreleasePoolPush时，会返回一个哨兵对象，也就是上文提到的autoreleasepoolobj，指向当前AutoreleasePoolPage中next指针指向的地址。
* 向一个对象发送autorelease消息，会把这个对象的地址加入到当前AutoreleasePoolPage中next指针指向的位置，之后next指针指向新加入对象的下一位置
* 一个AutoreleasePoolPage的空间被占满时，会新建一个AutoreleasePoolPage对象，child指针指向新建的page，后来加入到自动释放池中的对象添加到新的page
* 自动释放池释放时，根据push时创建的哨兵对象，找到对应的自动释放池。从最新加入的对象一直向前清理(发送release消息)，可以向前跨越若干个page，直至哨兵对象所指向的地址。

## AutoreleasePoolPage调试

了解autoreleasepool的原理后，回到开始的问题，我们的疑惑还没解决。结合调试，来看看到底发生了生么？调试需要编译后objc源码，有网友已编译好了，这里[下载](https://github.com/isaacselement/objc4-706)
修改开始的代码，输出refStr1、refStr2地址，便于对照

```objc
@autoreleasepool {
  NSString *str = [NSString stringWithFormat:@"hello"];
  refStr1 = str;
  refStr2 = [str stringByAppendingString:@" world"];
  NSLog(@"refStr1=%p,refStr2=%p",refStr1,refStr2);
}
```

在大括号结束前插入断点，在控制台执行下面命令

```text
expression AutoreleasePoolPage::hotPage() //获取当前AutoreleasePoolPage指针$0
p *$0 //查看AutoreleasePoolPage结构
p $0->printAll() //输出AutoreleasePoolPage信息
```

运行结果如下图
![](https://raw.githubusercontent.com/jiaxw32/MYImage/master/objc/AutoreleasePoolPage.png)
我们发现str并没有被加入到自动释放池中，所以refStr1最后仍能输出"hello"。为何str没有被加入到自动释放池中呢？
记得以前在《Effective Objective-C》中看过一段话（在"不要使用retainCount"一节），当时只做了标注，并未深思，至此方有所悟。

* **系统会尽可能把NSString实现成单例对象，这种对象的保留及释放操作都是'空操作'**。编译器会把NSString对象所表示的数据放到应用程序的二进制文件里，这样的话，运行程序时就可以直接使用了，无须再创建NSString对象。
* NSNumber也类似，它使用了一种叫做'标签指针'（tagged pointer）的概念来标注特定类型的数值。这种做法不使用NSNumber对象，而是把与数值有关的全部消息都放在指针里面。运行期系统会在消息派发期间检测到这种标签指针，并对它执行相应操作，使其行为看上去和真正的NSNumber对象一样。这种优化只在某些场合使用，同样是NSNumber对象，整数做了优化，浮点数对象就没有优化。

修改上面代码，又进行了测试，果然如此！

```objc
__weak id refStr1 = nil;
__weak id refStr2 = nil;
__weak id refNum1 = nil;
__weak id refNum2 = nil;
__weak id refObj = nil;
@autoreleasepool {
  NSString *str = [NSString stringWithFormat:@"hello"];
  refStr1 = str;
  refStr2 = [str stringByAppendingString:@" world"];

  NSNumber *number1 = [NSNumber numberWithInt:32];
  NSNumber *number2 = [NSNumber numberWithFloat:3.2];
  refNum1 = number1;
  refNum2 = number2;

  NSObject *obj = [NSObject new];
  refObj = obj;
}
NSLog(@"refStr1:%@",refStr1);
NSLog(@"refStr2:%@",refStr2);
NSLog(@"refNum1:%@",refNum1);
NSLog(@"refNum2:%@",refNum2);
NSLog(@"refObj:%@",refObj);
```

输出结果

```text
refStr1:hello
refStr2:(null)
refNum1:32
refNum2:(null)
refObj:(null)
```
至此，我们解决了开始的疑惑。

## 结论
 * autorelease块在开始和结束时，分别调用了objc_autoreleasePoolPush和objc_autoreleasePoolPop方法
* 自动释放池功能由AutoreleasePoolPage类实现，向一个对象发送autorelease消息，会把对象对象加入到当前的AutoreleasePoolPage中。
* 自动释放池清理时，会从当前最新加入的对象开始，直至push时创建的哨兵对象结束。
* NSString和**NSNumber部分对象**的保留及释放操作**可能**是空操作，释放时不会被加入到自动释放池。


## 结语

刚看sunnyxx的[《黑幕背后的Autorelease》](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)，感觉甚是深奥,不解其义，自己的测试结果也与文中开始实验的结果不同，未明白是怎么回事。其实sunnyxx的文章发布至今三年多都过去了，苹果说不一定已做了优化。其测试环境是真机还是模拟器，也不得而知，不同的测试环境，其结果也可能会有差异。
后来又读到Draveness的[《自动释放池的前世今生》](http://www.jianshu.com/p/32265cbb2a26)，学习了其中的调试技巧，结合调试、测试，终于搞明白了autoreleasepool的原理，解决了以前的困惑，但目前所知只是一角。
纸上得来终觉浅，绝知此事要躬行。自己动动手，你所学到的远比你看到的多！

## 思考

留个问题，ref1、ref2分别在什么时候释放？
在viewDidLoad方法结束时释放？还是在当前RunLoop即将休眠或结束时释放？或者不会释放？
诸位怎么看

```objc
__weak id ref1;
__weak id ref2;
- (void)viewDidLoad {
    [super viewDidLoad];
    NSString *str = @"haha";
    ref1 = str;
    NSObject *obj = [NSObject new];
    ref2 = obj;
}
```