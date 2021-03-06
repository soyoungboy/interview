## Block 基础

### Block 语法

Block 可以认为是一种匿名函数，使用如下语法声明一个 Block 类型：

    return_type (^block_name)(parameters)


例如：

    double (^multiplyTwoValues)(double, double);

Block 字面值的写法如下：

    ^ (double firstValue, double secondValue) {
        return firstValue * secondValue;
    }

上面的写法省略了返回值的类型，也可以显式地指出返回值类型。

声明并且定义完一个Block之后，便可以像使用函数一样使用它：

```objective-c
double (^multiplyTwoValues)(double, double) =
                          ^(double firstValue, double secondValue) {
                              return firstValue * secondValue;
                          };
double result = multiplyTwoValues(2,4);

NSLog(@"The result is %f", result);
```

同时，Block 也是一种 Objective-C 对象，可以用于赋值，当做参数传递，也可以放入 NSArray 和 NSDictionary 中。

**注意**：当用于函数参数时，Block 应该放在参数列表的最后一个。

### Block 可以捕获外部变量

Block 可以来自外部作用域的变量，这是Block一个很强大的特性。

```objective-c
- (void)testMethod {
    int anInteger = 42;
    void (^testBlock)(void) = ^{
        NSLog(@"Integer is: %i", anInteger);
    };
    testBlock();
}
```


默认情况下，Block 中捕获的到变量是不能修改的，如果想修改，需要使用`__block`来声明：

```objective-c
__block int anInteger = 42;
```


### 使用 Block 时的注意事项

在非 ARC 的情况下，对于 block 类型的属性应该使用 `copy` ，因为 block 需要维持其作用域中捕获的变量。在 ARC 中编译器会自动对 block 进行 copy 操作，因此使用 `strong` 或者 `copy` 都可以，没有什么区别，但是苹果仍然建议使用 `copy` 来指明编译器的行为。

block 在捕获外部变量的时候，会保持一个强引用，当在 block 中捕获 `self` 时，由于对象会对 block 进行 `copy`，于是便形成了强引用循环：

```objective-c
@interface XYZBlockKeeper : NSObject
@property (copy) void (^block)(void);
@end
```

```objective-c
@implementation XYZBlockKeeper
- (void)configureBlock {
    self.block = ^{
        [self doSomething];    // capturing a strong reference to self
                               // creates a strong reference cycle
    };
}
...
@end
```

为了避免强引用循环，最好捕获一个 `self` 的弱引用：

```objective-c
- (void)configureBlock {
    XYZBlockKeeper * __weak weakSelf = self;
    self.block = ^{
        [weakSelf doSomething];   // capture the weak reference
                                  // to avoid the reference cycle
    }
}
```

使用弱引用会带来另一个问题，`weakSelf` 有可能会为 nil，如果多次调用 `weakSelf` 的方法，有可能在 block 执行过程中 `weakSelf` 变为 nil。因此需要在 block 中将 `weakSelf` “强化“

```objective-c
__weak __typeof__(self) weakSelf = self;
NSBlockOperation *op = [[[NSBlockOperation alloc] init] autorelease];
[ op addExecutionBlock:^ {
    __strong __typeof__(self) strongSelf = weakSelf;
    [strongSelf doSomething];
    [strongSelf doMoreThing];
} ];
[someOperationQueue addOperation:op];
```

这样上面的 `doSomething` 和 `doMoreThing` 要么全执行成功，要么全失败，不会出现一个成功一个失败，即执行到中间 `self` 变成 nil 的情况。

#### 参考资料

* https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/WorkingwithBlocks/WorkingwithBlocks.html#//apple_ref/doc/uid/TP40011210-CH8-SW16
* http://blog.waterworld.com.hk/post/block-weakself-strongself