<p align="center">

<img src="Images/banner.jpeg" alt="Reveal" title="Reveal"/>

</p>

# 揭秘 APM iOS SDK 的核心技术

## 目录

* [前言](#前言)
* [页面渲染时间](#页面渲染时间)
* [启动时间](#启动时间)
* [网络](#网络)

## 前言

尽管 APM 越来越火爆，大大小小的专业 APM 厂商如雨后春笋般涌现出来，市面上有关 APM 的技术文章也非常多，但大部分都只是浅尝辄止，并未对实现细节进行深挖。本文旨在通过剖析 SDK 具体实现细节，揭露知名 APM 厂商的 iOS SDK 内部工作原理。我相信读者在阅读本文之前，也和笔者当初一样对 APM SDK 的实现细节充满了好奇。幸运的是，您正在读的这篇文章会带您一步步揭开 APM 的真实脉络，本文分析的 APM SDK 有**听云**, **OneAPM** 和 **Firebase Performance Monitoring** 等。笔者才疏学浅，若有讹误，不客斧正，以求再版，更臻完美。

> 本篇文章中分析的听云 SDK 版本是 2.3.5，与最新版本会存在些许差异，不过我大致翻看了新版的代码，差异不大，不影响分析。

## 页面渲染时间

页面渲染的监控，这个需求看似很简单，但是在实际开发的过程中还是会遇到不少问题。比较容易想到的是通过 hook 页面的几个关键的生命周期方法，例如 `viewDidLoad`、`viewDidAppear:` 等，从而计算出页面渲染时间，最终发现慢加载的页面。然而如果真正通过上述思路去着手实现的时候，便会遇到难题。在 APM SDK 中如何才能 hook 应用所有页面的生命周期的方法呢？如果尝试 hook `UIViewController` 的方法又会如何呢？hook `UIViewController` 的方法明显不可行，原因是他只会作用 `UIViewController` 的方法，而应用中大部分的视图控制器都是继承自 `UIViewController` 的，所以这种方法不可行。但是听云 SDK 却能够实现。页面 Hook 的逻辑主要是 `_priv_NBSUIAgent` 类中实现的，下面是 `_priv_NBSUIAgent` 类的定义，其中 `hook_viewDidLoad` 等几个方法便是线索。

```
@class _priv_NBSUIAgent : NSObject {
                       +hookUIImage
                       +hookNSManagedObjectContext
                       +hookNSJSONSerialization
                       +hookNSData
                       +hookNSArray
                       +hookNSDictionary
                       +hook_viewDidLoad:
                       +hook_viewWillAppear:
                       +hook_viewDidAppear:
                       +hook_viewWillLayoutSubviews:
                       +hook_viewDidLayoutSubviews:
                       +nbs_jump_initialize:
                       +hookSubOfController
                       +hookFMDB
                       +start
                   }
```

我们先将目光转到另外一个更可疑的方法：`hookSubOfController`，具体实现如下：

```
void +[_priv_NBSUIAgent hookSubOfController](void * self, void * _cmd) {
    r14 = self;
    r12 = [_subMetaClassNamesInMainBundle_c("UIViewController") retain];
    var_C0 = r12;
    if ((r12 != 0x0) && ([r12 count] != 0x0)) {
            var_C8 = object_getClass(r14);
            if ([r12 count] != 0x0) {
                    r15 = @selector(nbs_jump_initialize:);
                    rdx = 0x0;
                    do {
                            var_98 = rdx;
                            r12 = [[r12 objectAtIndexedSubscript:rdx, rcx, r8] retain];
                            [r12 release];
                            if ([r12 respondsToSelector:r15, rcx, r8] == 0x0) {
                                    _hookClass_CopyAMetaMethod();
                            }
                            r13 = class_getName(r12);
                            rax = [NSString stringWithFormat:@"nbs_%s_initialize", r13];
                            rax = [rax retain];
                            var_A0 = rax;
                            rax = NSSelectorFromString(rax);
                            var_B0 = rax;
                            rax = objc_retainBlock(__NSConcreteStackBlock);
                            var_A8 = rax;
                            r15 = objc_retainBlock(rax);
                            var_B8 = imp_implementationWithBlock(r15);
                            [r15 release];
                            rax = class_getSuperclass(r12);
                            r15 = objc_retainBlock(__NSConcreteStackBlock);
                            rbx = objc_retainBlock(r15);
                            r13 = imp_implementationWithBlock(rbx);
                            [rbx release];
                            rcx = r13;
                            r8 = var_B8;
                            _nbs_Swizzle_orReplaceWithIMPs(r12, @selector(initialize), var_B0, rcx, r8);
                            rdi = r15;
                            r15 = @selector(nbs_jump_initialize:);
                            [rdi release];
                            [var_A8 release];
                            [var_A0 release];
                            rax = [var_C0 count];
                            r12 = var_C0;
                            rdx = var_98 + 0x1;
                    } while (var_98 + 0x1 < rax);
            }
    }
    [r12 release];
    return;
}
```

从 `_subMetaClassNamesInMainBundle_c` 的命名和传入的 "UIViewController" 参数，基本可以推断这个 C 函数是获取 MainBundle 中所有 `UIViewController` 的子类。而事实上，如果通过 LLDB 在这个函数 Call 完之后的那行汇编代码下断点，会发现返回的确实是 `UIViewController` 子类的数组。下面的 `if` 语句判断 `r12` 寄存器不为 `nil` 并且 `r12` 寄存器的 `count` 不等于0才执行 `if` 里面的逻辑，而 `r12` 寄存器存放的正是 `_subMetaClassNamesInMainBundle_c` 函数的返回值，也就是 `UIViewController` 子类的数组。

`_subMetaClassNamesInMainBundle_c` 函数代码如下：

```
void _subMetaClassNamesInMainBundle_c(int arg0) {
    rbx = objc_getClass(arg0);
    rdi = 0x0;
    if (rbx == 0x0) goto loc_10001dbde;

loc_10001db4d:
    r15 = _classNamesInMainBundle_c(var_2C);
    var_38 = [NSMutableArray new];
    if (var_2C == 0x0) goto loc_10001dbd2;

loc_10001db77:
    r14 = 0x0;
    goto loc_10001db7a;

loc_10001db7a:
    r13 = objc_getClass(*(r15 + r14 * 0x8));
    r12 = r13;
    if (r13 == 0x0) goto loc_10001dbc9;

loc_10001db8e:
    rax = class_getSuperclass(r12);
    if (rax == rbx) goto loc_10001dba5;

loc_10001db9b:
    COND = rax != r12;
    r12 = rax;
    if (COND) goto loc_10001db8e;

loc_10001dbc9:
    r14 = r14 + 0x1;
    if (r14 < var_2C) goto loc_10001db7a;

loc_10001dbd2:
    free(r15);
    rdi = var_38;
    goto loc_10001dbde;

loc_10001dbde:
    [rdi autorelease];
    return;

loc_10001dba5:
    rax = class_getName(r13);
    rax = objc_getMetaClass(rax);
    [var_38 addObject:rax];
    goto loc_10001dbc9;
}
```

`_subMetaClassNamesInMainBundle_c` 函数中的 `loc_10001db4d` 子例程调用了 `_classNamesInMainBundle_c` 函数，该函数代码如下：

```
int _classNamesInMainBundle_c(int arg0) {
    rbx = [[NSBundle mainBundle] retain];
    r15 = [[rbx executablePath] retain];
    [rbx release];
    rbx = objc_retainAutorelease(r15);
    r14 = objc_copyClassNamesForImage([rbx UTF8String], arg0);
    [rbx release];
    rax = r14;
    return rax;
}
```

`_classNamesInMainBundle_c` 函数的实现显而易见，无非就是调用 `objc_copyClassNamesForImage` 以获取 `mainBundle` 可执行路径的所有类的名称，集合的数量赋给了 `outCount` 变量，调用方可以使用 `outCount` 来对其遍历。

``` objective-c
static inline char **WDTClassNamesInMainBundle(unsigned int *outCount) {
    NSString *executablePath = [[NSBundle mainBundle] executablePath];
    char **classNames = objc_copyClassNamesForImage([executablePath UTF8String], outCount);
    return classNames;
}
```

如果不在乎细节，那么 `_subMetaClassNamesInMainBundle_c` 函数的实现也很清晰，就是遍历 `objc_copyClassNamesForImage` 函数的返回值，如果 item 是 `UIViewController` 的子类，则取得该类的 `metaClass` 并添加到可变数组 `var_38` 中。

接下来再来重点看看里面的 `do-while` 循环语句，循环判断的语句为 `var_98 + 0x1 < rax`，`var_98` 在循环开始的位置赋值 `rdx` 寄存器，`rdx` 寄存器在循环外初始化为0，所以 `var_98` 就是计数器，而 `rax` 寄存器则是赋值为 `r12` 寄存器的 `count` 方法，依此得出这个 `do-while` 循环实际就是遍历 `UIViewController` 子类的数组。遍历的行为则是通过 `_nbs_Swizzle_orReplaceWithIMPs` 实现 `initialize` 和 `nbs_jump_initialize:` 的方法交换。

`nbs_jump_initialize` 的代码如下：

```
void +[_priv_NBSUIAgent nbs_jump_initialize:](void * self, void * _cmd, void * arg2) {
    rbx = arg2;
    r15 = self;
    r14 = [NSStringFromSelector(rbx) retain];
    if ((r14 != 0x0) && ([r14 isEqualToString:@""] == 0x0)) {
            [r15 class];
            rax = _nbs_getClassImpOf();
            (rax)(r15, @selector(initialize));
    }
    rax = class_getName(r15);
    r13 = [[NSString stringWithUTF8String:rax] retain];
    rdx = @"_Aspects_";
    if ([r13 hasSuffix:rdx] == 0x0) goto loc_100050137;

loc_10005011e:
    if (*(int8_t *)_is_tiaoshi_kai == 0x0) goto loc_100050218;

loc_10005012e:
    rsi = cfstring__V__A;
    goto loc_100050195;

loc_100050195:
    __NBSDebugLog(0x3, rsi, rdx, rcx, r8, r9, stack[2048]);
    goto loc_100050218;

loc_100050218:
    [r13 release];
    rdi = r14;
    [rdi release];
    return;

loc_100050137:
    rdx = @"RACSelectorSignal";
    if ([r13 hasSuffix:rdx] == 0x0) goto loc_10005016b;

loc_100050152:
    if (*(int8_t *)_is_tiaoshi_kai == 0x0) goto loc_100050218;

loc_100050162:
    rsi = cfstring__V__R;
    goto loc_100050195;

loc_10005016b:
    if (_classSelf_isImpOf(r15, "nbs_vc_flag") == 0x0) goto loc_1000501a3;

loc_10005017e:
    if (*(int8_t *)_is_tiaoshi_kai == 0x0) goto loc_100050218;

loc_10005018e:
    rsi = cfstring____Yh;
    goto loc_100050195;

loc_1000501a3:
    rbx = objc_retainBlock(void ^(void * _block, void * arg1) {
        return;
    });
    rax = imp_implementationWithBlock(rbx);
    class_addMethod(r15, @selector(nbs_vc_flag), rax, "v@:");
    [rbx release];
    [_priv_NBSUIAgent hook_viewDidLoad:r15];
    [_priv_NBSUIAgent hook_viewWillAppear:r15];
    [_priv_NBSUIAgent hook_viewDidAppear:r15];
    goto loc_100050218;
}
```
`nbs_jump_initialize` 的代码有点长，但是从 `loc_1000501a3` 的例程可以观察到主要逻辑会执行 `hook_viewDidLoad`、`hook_viewWillAppear` 和 `hook_viewDidAppear` 三个方法，从而 hook 住 `UIViewController` 子类的这三个方法。

先以 `hook_viewDidLoad:` 方法为例讲解，下面这段代码可能有点晦涩难懂，需要认真分析

```
void +[_priv_NBSUIAgent hook_viewDidLoad:](void * self, void * _cmd, void * arg2) {
    rax = [_priv_NBSUIHookMatrix class];
    var_D8 = _nbs_getInstanceImpOf();
    var_D0 = _nbs_getInstanceImpOf();
    rbx = class_getName(arg2);
    r14 = class_getSuperclass(arg2);
    rax = [NSString stringWithFormat:@"nbs_%s_viewDidLoad", rbx];
    rax = [rax retain];
    var_B8 = rax;
    var_C0 = NSSelectorFromString(rax);
    r12 = objc_retainBlock(__NSConcreteStackBlock);
    var_D0 = imp_implementationWithBlock(r12);
    [r12 release];
    rbx = objc_retainBlock(__NSConcreteStackBlock);
    r14 = imp_implementationWithBlock(rbx);
    [rbx release];
    _nbs_Swizzle_orReplaceWithIMPs(arg2, @selector(viewDidLoad), var_C0, r14, var_D0);
    [var_B8 release];
    return;
}

```

`hook_viewDidLoad:` 方法中的参数 `arg2` 即是要 hook 的 `ViewController` 的类，获取 `arg2` 的类名并赋给 `rbx` 寄存器，然后利用 `rbx` 构造字符串 `nbs_%s_viewDidLoad`，如 `nbs_XXViewController_viewDidLoad`，获得该字符串的 selector 后赋给 `var_C0`，下面几句中的 `__NSConcreteStackBlock` 是创建的存储栈的 block 对象，这个 block 之后会通过 `imp_implementationWithBlock` 方法获取到 IMP 函数指针，`_nbs_Swizzle_orReplaceWithIMPs` 是实现方法交换的函数，参数依次为：`arg2` 是 `ViewController` 的类；`@selector(viewDidLoad)` 是 `viewDidLoad` 的 selector；`var_C0` 是 `nbs_%s_viewDidLoad` 的 selector，`r14` 是第二个 `__NSConcreteStackBlock` 的 IMP；`var_D0` 是第一个 `__NSConcreteStackBlock` 的 IMP。

`hook_viewDidLoad:` 的整个逻辑大致清楚了，不过这里有个问题为什么不直接交换两个 IMP，而是要先构造两个 block，然后交换两个 block 的 IMP呢？原因是需要将 `ViewController` 的父类也就是 `class_getSuperclass` 的结果作为参数传递给交换后的方法，这样交换的两个 selector 签名的参数个数不一致，需要通过构造 block 去巧妙的解决这个问题，而事实上第一个 `__NSConcreteStackBlock` 的执行的就是 `_priv_NBSUIHookMatrix` 的 `nbs_jump_viewDidLoad:superClass:` 方法，正如之前所说的，这个方法的参数中有 `superClass`，至于为什么需要这个参数，稍后再做介绍。

为什么第二个 `__NSConcreteStackBlock` 的执行的是 `nbs_jump_viewDidLoad:superClass:` 方法呢？取消勾选 Hopper 的 `Remove potentially dead code` 选项，代码如下：

```
void +[_priv_NBSUIAgent hook_viewDidLoad:](void * self, void * _cmd, void * arg2) {
    rsi = _cmd;
    rdi = self;
    r12 = _objc_msgSend;
    rax = [_priv_NBSUIHookMatrix class];
    rsi = @selector(nbs_jump_viewDidLoad:superClass:);
    rdi = rax;
    var_D8 = _nbs_getInstanceImpOf();
    rdi = arg2;
    rsi = @selector(viewDidLoad);
    var_D0 = _nbs_getInstanceImpOf();
    rbx = class_getName(arg2);
    r14 = class_getSuperclass(arg2);
    LODWORD(rax) = 0x0;
    rax = [NSString stringWithFormat:@"nbs_%s_viewDidLoad", rbx];
    rax = [rax retain];
    var_B8 = rax;
    var_C0 = NSSelectorFromString(rax);
    var_60 = 0xc0000000;
    var_5C = 0x0;
    var_58 = ___37+[_priv_NBSUIAgent hook_viewDidLoad:]_block_invoke;
    var_50 = ___block_descriptor_tmp;
    var_48 = var_D8;
    var_40 = @selector(viewDidLoad);
    var_38 = var_D0;
    var_30 = r14;
    r12 = objc_retainBlock(__NSConcreteStackBlock);
    var_D0 = imp_implementationWithBlock(r12);
    r13 = _objc_release;
    rax = [r12 release];
    var_A8 = 0xc0000000;
    var_A4 = 0x0;
    var_A0 = ___37+[_priv_NBSUIAgent hook_viewDidLoad:]_block_invoke_2;
    var_98 = ___block_descriptor_tmp47;
    var_90 = rbx;
    var_88 = var_D8;
    var_80 = @selector(viewDidLoad);
    var_78 = r14;
    var_70 = arg2;
    rbx = objc_retainBlock(__NSConcreteStackBlock);
    r14 = imp_implementationWithBlock(rbx);
    rax = [rbx release];
    rax = _nbs_Swizzle_orReplaceWithIMPs(arg2, @selector(viewDidLoad), var_C0, r14, var_D0);
    rax = [var_B8 release];
    rsp = rsp + 0xb8;
    rbx = stack[2047];
    r12 = stack[2046];
    r13 = stack[2045];
    r14 = stack[2044];
    r15 = stack[2043];
    rbp = stack[2042];
    return;
}
```

再来看 `_nbs_getInstanceImpOf` 的代码：

```
void _nbs_getInstanceImpOf() {
    rax = class_getInstanceMethod(rdi, rsi);
    method_getImplementation(rax);
    return;
}
```
`_nbs_getInstanceImpOf` 函数的作用很明显，获取 `rdi` 类中 `rsi` selector 的 IMP，读者会发现在 `hook_viewDidLoad:` 方法中共调用了两次 `_nbs_getInstanceImpOf`，第一次 `rdi` 是 `_priv_NBSUIHookMatrix` 类，`rdx` 是 `@selector(nbs_jump_viewDidLoad:superClass:)`，第二次 `rdi` 是 `ViewController` 类，`rdx` 是 `@selector(viewDidLoad)`。

接下来看第一个 `__NSConcreteStackBlock`，也就是会调用 `nbs_jump_viewDidLoad:superClass:` 的 block，代码如下：

```
int ___37+[_priv_NBSUIAgent hook_viewDidLoad:]_block_invoke(int arg0, int arg1) {
    r8 = *(arg0 + 0x20);
    rax = *(arg0 + 0x28);
    rdx = *(arg0 + 0x30);
    rcx = *(arg0 + 0x38);
    rax = (r8)(arg1, rax, rdx, rcx, r8);
    return rax;
}
```

`r8` 寄存器是 `nbs_jump_viewDidLoad:superClass:` 的 IMP，这段代码只是调用这个 IMP。IMP 函数的参数与 `nbs_jump_viewDidLoad:superClass:` 相同。

```
void -[_priv_NBSUIHookMatrix nbs_jump_viewDidLoad:superClass:](void * self, void * _cmd, void * * arg2, void * arg3) {
    rbx = arg3;
    var_70 = arg2;
    var_68 = _cmd;
    r14 = self;
    rax = [self class];
    rax = class_getSuperclass(rax);
    if ((rbx != 0x0) && (rax != rbx)) {
            rax = var_70;
            if (rax != 0x0) {
                    rdi = r14;
                    (rax)(rdi, @selector(viewDidLoad));
            }
            else {
                    NSLog(@"");
                    [[r14 super] viewDidLoad];
            }
    }
    else {
            var_B8 = rbx;
            objc_storeWeak(_currentViewController, 0x0);
            r14 = 0x0;
            [[NSString stringWithFormat:@"%d#loading", 0x0] retain];
            r12 = 0x0;
            if (0x0 != 0x0) {
                    rcx = class_getName([r12 class]);
                    r14 = [[NSString stringWithFormat:@"MobileView/Controller/%s#%@", rcx, @"loading"] retain];
            }
            var_A0 = r14;
            r14 = [[_priv_NBSUILogCenter_assistant alloc] initWithControllerName:r14];
            var_80 = r14;
            var_60 = _objc_release;
            [r14 setTheVC:_objc_release];
            [r14 setVC_Address:_objc_release];
            [r14 setIsOther:0x0];
            [*_controllerStack push:r14];
            rbx = [_glb_all_activing_VCS() retain];
            var_98 = _objc_msgSend;
            [rbx setObject:r14 forKey:_objc_msgSend];
            [rbx release];
            r12 = [[NSDate date] retain];
            [r12 timeIntervalSince1970];
            xmm0 = intrinsic_mulsd(xmm0, *0x100066938);
            rbx = intrinsic_cvttsd2si(rbx, xmm0);
            [r12 release];
            [r14 setStartTime:rbx];
            rcx = class_getName([var_60 class]);
            r13 = [[NSString stringWithFormat:@"%s", rcx] retain];
            r14 = [NSStringFromSelector(var_68) retain];
            var_88 = [_nbs_embedIn_start() retain];
            [r14 release];
            [r13 release];
            rbx = [[NBSLensInterfaceEventLogger shareObject] retain];
            var_78 = rbx;
            rax = [NBSLensUITraceSegment new];
            var_58 = rax;
            rbx = [[rbx theStack] retain];
            [rbx push:rax];
            [rbx release];
            rcx = class_getName([var_60 class]);
            r13 = [[NSString stringWithFormat:@"%s", rcx] retain];
            r12 = [NSStringFromSelector(var_68) retain];
            r14 = [[NSString stringWithFormat:@"%@#%@", r13, r12] retain];
            var_A8 = r14;
            [r12 release];
            rdi = r13;
            [rdi release];
            [var_58 setSegmentName:r14];
            rax = [NSDictionary dictionary];
            rax = [rax retain];
            var_B0 = rax;
            [var_58 setSegmentParam:rax];
            rbx = [[NSThread currentThread] retain];
            rdx = rbx;
            [var_58 setThreadInfomation:rdx];
            [rbx release];
            rbx = [[NSDate date] retain];
            [rbx timeIntervalSince1970];
            xmm0 = intrinsic_mulsd(xmm0, *0x100066938);
            var_68 = intrinsic_movsd(var_68, xmm0);
            [rbx release];
            xmm0 = intrinsic_movsd(xmm0, var_68);
            [var_58 setStartTime:rdx];
            [var_58 setEntryTime:0x0];
            r14 = [NBSLensUITraceSegment new];
            var_90 = r14;
            xmm0 = intrinsic_movsd(xmm0, var_68);
            [r14 setStartTime:0x0];
            rcx = class_getName([var_60 class]);
            r15 = [[NSString stringWithFormat:@"%s", rcx] retain];
            rbx = [[NSString stringWithFormat:@"%@#viewLoading", r15] retain];
            [r14 setSegmentName:rbx];
            [rbx release];
            [r15 release];
            rcx = var_30;
            rax = [NSDictionary dictionaryWithObjects:rbx forKeys:rcx count:0x0];
            [r14 setSegmentParam:rax];
            rbx = [[NSThread currentThread] retain];
            [r14 setThreadInfomation:rbx];
            [rbx release];
            [r14 setEntryTime:0x0];
            rax = var_70;
            if (rax != 0x0) {
                    (rax)(var_60, @selector(viewDidLoad), 0x0, rcx, 0x0);
            }
            else {
                    NSLog(@"");
                    [[var_60 super] viewDidLoad];
            }
            _nbs_embedIn_finish();
            rdx = [var_88 mach_tm2];
            [var_80 setFinishTime:rdx];
            rbx = [[NSDate date] retain];
            [rbx timeIntervalSince1970];
            xmm0 = intrinsic_mulsd(xmm0, *0x100066938);
            var_70 = intrinsic_movsd(var_70, xmm0);
            [rbx release];
            xmm0 = intrinsic_movsd(xmm0, var_70);
            xmm0 = intrinsic_subsd(xmm0, var_68);
            rdx = intrinsic_cvttsd2si(rdx, xmm0);
            [var_58 setExitTime:rdx];
            rbx = [[var_78 theStack] retain];
            rax = [rbx pop];
            rax = [rax retain];
            [rax release];
            [rbx release];
            rbx = [[var_78 theStack] retain];
            r15 = [rbx isEmpty];
            [rbx release];
            if (r15 == 0x0) {
                    rbx = [[var_78 theStack] retain];
                    r14 = [[rbx peer] retain];
                    [rbx release];
                    [r14 startTime];
                    xmm1 = intrinsic_movsd(xmm1, var_68);
                    xmm1 = intrinsic_subsd(xmm1, xmm0);
                    rdx = intrinsic_cvttsd2si(rdx, xmm1);
                    [var_58 setEntryTime:rdx];
                    [r14 startTime];
                    rdx = intrinsic_cvttsd2si(rdx, intrinsic_subsd(intrinsic_movsd(xmm1, var_70), xmm0));
                    [var_58 setExitTime:rdx];
                    rbx = [[r14 childSegments] retain];
                    rdx = var_58;
                    [rbx addObject:rdx];
                    [rbx release];
                    [r14 release];
            }
            rbx = [[var_90 childSegments] retain];
            [rbx addObject:var_58];
            [rbx release];
            objc_setAssociatedObject(var_60, @"viewLoading", var_90, 0x1);
            rax = [*_controllerStack pop];
            rax = [rax retain];
            [rax release];
            rbx = [[_priv_NBSLENS_VCSBuffer sharedObj] retain];
            [rbx addObj:var_80];
            [rbx release];
            rbx = [_glb_all_activing_VCS() retain];
            [rbx removeObjectForKey:var_98];
            [rbx release];
            [var_90 release];
            [var_B0 release];
            [var_A8 release];
            [var_58 release];
            [var_78 release];
            [var_88 release];
            [var_80 release];
            [var_A0 release];
            [var_98 release];
    }
    return;
}
```

## 启动时间

启动时间以 **Firebase Performance Monitoring** SDK 为例进行讲解，下文以 FPM SDK 作为简称方便讲述，FPM SDK 实现的是冷启动时间的统计，主要逻辑在 `FPRAppActivityTracker` 类中实现。

首先看类的 `+load` 方法，反编译代码如下：

```
void +[FPRAppActivityTracker load](void * self, void * _cmd) {
    rax = [NSDate date];
    rax = [rax retain];
    rdi = *_appStartTime;
    *_appStartTime = rax;
    [rdi release];
    rbx = [[NSNotificationCenter defaultCenter] retain];
    [rbx addObserver:self selector:@selector(windowDidBecomeVisible:) name:*_UIWindowDidBecomeVisibleNotification object:0x0];
    rdi = rbx;
    [rdi release];
    return;
}	
```

显而易见，`_appStartTime` 是一个 static 的 `NSDate` 实例，用来保存整个应用启动的开始时间，所以 FPM SDK 是在 `FPRAppActivityTracker` 的 `+load` 标记应用启动的开始时间。了解 `+load` 方法的读者应该知道该方法是在 `main` 函数调用之前的钩子方法，准确的时间是当镜像加载到运行时、对 `+load` 方法的准备就绪之后，开始调用 `+load` 方法。此外不同类的 `+load` 方法还与 `Build Phases->Compile Sources` 的文件顺序有关，我们姑且认为这些对启动时间的统计没有显著的影响。

之后注册了 `UIWindowDidBecomeVisibleNotification` 的通知，这个通知是当 `UIWindow` 对象激活时并展示在界面的时候触发的。读者可以注册这个通知，然后用 LLDB 打印 notification 对象，示例如下:

```
NSConcreteNotification 0x7fc94a716f50 {name = UIWindowDidBecomeVisibleNotification; object = <UIStatusBarWindow: 0x7fc94a5092a0; frame = (0 0; 320 568); opaque = NO; gestureRecognizers = <NSArray: 0x7fc94a619f30>; layer = <UIWindowLayer: 0x7fc94a513f50>>}
```

第一次收到 `UIWindowDidBecomeVisibleNotification` 通知的时间早于 `- application:didFinishLaunchingWithOptions:` 回调，这个通知时状态栏的 `window` 创建时触发的，这个实现感觉有点取巧，不能确保未来 Apple 会不会调整调用的时序。

下面是 `UIWindowDidBecomeVisibleNotification` 的官方说明。

> Posted when an UIWindow object becomes visible.
The notification object is the window object that has become visible. This notification does not contain a userInfo dictionary.
Switching between apps does not generate visibility-related notifications for windows. Window visibility changes reflect changes to the window’s hidden property and reflect only the window’s visibility within the app.


下面是通知的处理方法，我将方法还原成 Objective-C 的伪代码，可以对比反编译的伪代码

```
void +[FPRAppActivityTracker windowDidBecomeVisible:](void * self, void * _cmd, void * arg2) {
    var_30 = self;
    r13 = _objc_msgSend;
    r12 = [[self sharedInstance] retain];
    [r12 startAppActivityTracking];
    rbx = [[FIRTrace alloc] initInternalTraceWithName:@"_as"];
    [r12 setAppStartTrace:rbx];
    [rbx release];
    r15 = @selector(appStartTrace);
    rbx = [_objc_msgSend(r12, r15) retain];
    [rbx startWithStartTime:*_appStartTime];
    [rbx release];
    rbx = [_objc_msgSend(r12, r15) retain];
    rcx = *_appStartTime;
    rdx = @"_astui";
    [rbx startStageNamed:rdx startTime:rcx];
    [rbx release];
    rax = *(int8_t *)_windowDidBecomeVisible:.FDDStageStarted;
    rax = rax & 0x1;
    COND = rax != 0x0;
    if (!COND) {
            r13 = _objc_msgSend;
            rcx = *_appStartTime;
            rbx = [_objc_msgSend(r12, r15, rdx, rcx) retain];
            rdx = @"_astfd";
            [rbx startStageNamed:rdx, rcx];
            [rbx release];
            *(int8_t *)_windowDidBecomeVisible:.FDDStageStarted = 0x1;
    }
    rbx = [(r13)(@class(NSNotificationCenter), @selector(defaultCenter), rdx, *_UIWindowDidBecomeVisibleNotification) retain];
    (r13)(rbx, @selector(removeObserver:name:object:), var_30, *_UIWindowDidBecomeVisibleNotification, 0x0);
    [rbx release];
    rdi = r12;
    [rdi release];
    return;
}
```

```
+ (void)windowDidBecomeVisible:(NSNotification *)notification {
    FPRAppActivityTracker *tracker = [self sharedInstance];
    [tracker startAppActivityTracking];
    FIRTrace *trace = [[FIRTrace alloc] initInternalTraceWithName:@"_as"];
    [tracker setAppStartTrace: trace];
    [[tracker appStartTrace] startWithStartTime:_appStartTime];
    [[tracker appStartTrace] startStageNamed:@"_astui" startTime:_appStartTime];
    
    if (_windowDidBecomeVisible:.FDDStageStarted) {
        [[tracker appStartTrace] startStageNamed:@"_astfd" startTime:_appStartTime];
        _windowDidBecomeVisible:.FDDStageStarted = 1;
    }
    
    [[NSNotificationCenter defaultCenter] removeObserver:self name:UIWindowDidBecomeVisibleNotification object:nil];
}

```

方法的最后会注销 `UIWindowDidBecomeVisibleNotification` 通知，因为该通知会调用多次，而我们只需要他执行一次。首先调用 `-startAppActivityTracking` 方法开始追踪 APP 的活动，这个方法稍后会深入讨论。

## 网络

首先明确一点，我们这里讨论的网络请求没有特殊说明指的都是 HTTP 请求。听云 SDK 实现网络监控主要使用了两种方式：第一种是通过 hook iOS 网络编程使用的 API，这种方式主要针对的是原生的网络请求；第二种是继承 `NSURLProtocol` 来实现网络请求的拦截，这种方式主要针对的是 UIWebView 中的网络请求。


### NSURLSession

SDK hook 了所有网络请求中构造 `NSURLSessionDataTask`, `NSURLSessionUploadTask` 和 `NSURLSessionDownloadTask` 的 API。 hook 的逻辑在 C 函数 `_nbs_hook_NSURLSession` 中，伪代码如下：

```
int _nbs_hook_NSURLSession() {
    _nbs_hook_NSURLSessionTask();
    r13 = [[_priv_NSURLSession_NBS class] retain];
    r14 = [objc_getClass("NSURLSession") retain];
    r15 = [objc_getMetaClass(class_getName(r13)) retain];
    r12 = [objc_getMetaClass("NSURLSession") retain];
    if ((((((_nbs_hookClass_CopyAMethod() != 0x0) && (_nbs_hookClass_CopyAMethod() != 0x0)) && (_nbs_hookClass_CopyAMethod() != 0x0)) && (_nbs_hookClass_CopyAMethod() != 0x0)) && (_nbs_hookClass_CopyAMethod() != 0x0)) && (_nbs_hookClass_CopyAMethod() != 0x0)) {
            if (_nbs_hookClass_CopyAMethod() != 0x0) {
                    if (_nbs_hookClass_CopyAMethod() != 0x0) {
                            if (_nbs_hookClass_CopyAMethod() != 0x0) {
                                    if (_nbs_hookClass_CopyAMethod() != 0x0) {
                                            _nbs_Swizzle(r14, @selector(dataTaskWithRequest:completionHandler:), @selector(nbs_dataTaskWithRequest:completionHandler:));
                                            _nbs_Swizzle(r14, @selector(downloadTaskWithRequest:completionHandler:), @selector(nbs_downloadTaskWithRequest:completionHandler:));
                                            _nbs_Swizzle(r14, @selector(downloadTaskWithResumeData:completionHandler:), @selector(nbs_downloadTaskWithResumeData:completionHandler:));
                                            _nbs_Swizzle(r14, @selector(uploadTaskWithRequest:fromData:completionHandler:), @selector(nbs_uploadTaskWithRequest:fromData:completionHandler:));
                                            _nbs_Swizzle(r14, @selector(uploadTaskWithRequest:fromFile:completionHandler:), @selector(nbs_uploadTaskWithRequest:fromFile:completionHandler:));
                                            _nbs_Swizzle(r14, @selector(downloadTaskWithRequest:), @selector(nbs_downloadTaskWithRequest:));
                                            _nbs_Swizzle(r14, @selector(uploadTaskWithRequest:fromFile:), @selector(nbs_uploadTaskWithRequest:fromFile:));
                                            _nbs_Swizzle(r14, @selector(uploadTaskWithRequest:fromData:), @selector(nbs_uploadTaskWithRequest:fromData:));
                                            _nbs_Swizzle(r12, @selector(sessionWithConfiguration:delegate:delegateQueue:), @selector(nbs_sessionWithConfiguration:delegate:delegateQueue:));
                                            _nbs_Swizzle(r14, @selector(uploadTaskWithStreamedRequest:), @selector(nbs_uploadTaskWithStreamedRequest:));
                                    }
                            }
                    }
            }
    }
    [r12 release];
    [r15 release];
    [r14 release];
    rdi = r13;
    rax = [rdi release];
    return rax;
}
```

> \_nbs_Swizzle 就是听云实现 Method Swizzling 的 C 函数。

从代码中可以看到除了对上面提到的 `NSURLSessionDataTask`, `NSURLSessionUploadTask` 和 `NSURLSessionDownloadTask` 的 API 使用了 `_nbs_Swizzle`，还替换了 `sessionWithConfiguration:delegate:delegateQueue:` 方法的实现，稍后会讲解为什么要 hook 这个方法。

所有 hook 的方法实现被定义在 `_priv_NSURLSession_NBS` 类中。

`nbs_dataTaskWithRequest:completionHandler:` 的核心代码如下：

```
typedef void (^nbs_URLSessionDataTaskCompletionHandler)(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error);

- (NSURLSessionDataTask *)nbs_dataTaskWithRequest:(NSURLRequest *)request
                                completionHandler:(void (^)(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error))completionHandler {
    _priv_NBSHTTPTransaction *httpTransaction = [_priv_NBSHTTPTransaction new];
    
    nbs_URLSessionDataTaskCompletionHandler wrappedCompletionHandler;
    __block NSURLSessionDataTask *dataTask;
    
    if (completionHandler) {
        wrappedCompletionHandler = ^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
            NSTimeInterval timeInterval = [[NSDate date] timeIntervalSince1970];
            [dataTask.httpTransaction finishAt:timeInterval];
            completionHandler(data, response, error);
        };
    }
    
    dataTask = [self nbs_dataTaskWithRequest:request
                           completionHandler:wrappedCompletionHandler];
    
    if (dataTask) {
        dataTask.httpTransaction = httpTransaction;
    }
    return dataTask;
}
```

`_priv_NBSHTTPTransaction` 是 SDK 中和 HTTP 请求相关的性能参数的 Model。该类结构如下：

```
@class _priv_NBSHTTPTransaction : NSObject {
    @property isFileURL
    @property tm_dur_dns
    @property tm_dur_cnnct
    @property tm_pnt_send
    @property tm_dur_firstP
    @property tm_dur_end
    @property tm_dur_ssl
    @property sendSize
    @property receiveSize
    @property headerSize
    @property dataSize
    @property statusCode
    @property errCode
    @property contentLength
    @property errText
    @property url
    @property ip
    @property contentType
    @property anyObj
    @property useContentLength
    @property netType
    @property appData
    @property request
    @property response
    @property responseData
    @property urlParams
    @property dataBody
    @property httpMethodNumber
    @property libClassId
    @property socketItem
    @property threadId
    @property cdn_associate
    @property connectType
    @property cdnVendorName
    @property cdn_flg
    ivar tm_dur_cnnct
    ivar tm_dur_dns
    ivar tm_dur_firstP
    ivar tm_dur_end
    ivar tm_dur_ssl
    ivar tm_pnt_send
    ivar sendSize
    ivar receiveSize
    ivar headerSize
    ivar dataSize
    ivar statusCode
    ivar errCode
    ivar errText
    ivar url
    ivar ip
    ivar contentType
    ivar contentLength
    ivar anyObj
    ivar useContentLength
    ivar netType
    ivar appData
    ivar response
    ivar responseData
    ivar urlParams
    ivar dataBody
    ivar httpMethodNumber
    ivar libClassId
    ivar socketItem
    ivar threadId
    ivar cdn_associate
    ivar cdn_flg
    ivar isFileURL
    ivar connectType
    ivar cdnVendorName
    ivar _request
    -clear
    -init
    -getText
    -addIntoArray:
    -startWithIP:DNSTime:atTimePoint:withObject:
    -updateWithResponse:timePoint:
    -updateWithReceiveData:
    -updateWithTotalReceiveData:
    -updateWithTotalReceiveSize:
    -updateSendSize:
    -updateWithError:
    -finishAt:
    -.cxx_destruct
    -tm_dur_dns
    -setTm_dur_dns:
    -tm_pnt_send
    -setTm_pnt_send:
    -tm_dur_firstP
    -setTm_dur_firstP:
    -tm_dur_end
    -setTm_dur_end:
    -tm_dur_cnnct
    -setTm_dur_cnnct:
    -tm_dur_ssl
    -setTm_dur_ssl:
    -sendSize
    -setSendSize:
    -receiveSize
    -setReceiveSize:
    -errCode
    -setErrCode:
    -contentLength
    -setContentLength:
    -statusCode
    -setStatusCode:
    -headerSize
    -setHeaderSize:
    -dataSize
    -setDataSize:
    -url
    -setUrl:
    -ip
    -setIp:
    -errText
    -setErrText:
    -contentType
    -setContentType:
    -useContentLength
    -setUseContentLength:
    -netType
    -setNetType:
    -appData
    -setAppData:
    -response
    -setResponse:
    -responseData
    -setResponseData:
    -anyObj
    -setAnyObj:
    -urlParams
    -setUrlParams:
    -dataBody
    -setDataBody:
    -httpMethodNumber
    -setHttpMethodNumber:
    -libClassId
    -setLibClassId:
    -isFileURL
    -setIsFileURL:
    -socketItem
    -setSocketItem:
    -threadId
    -setThreadId:
    -connectType
    -setConnectType:
    -cdnVendorName
    -setCdnVendorName:
    -cdn_associate
    -setCdn_associate:
    -cdn_flg
    -setCdn_flg:
    -request
    -setRequest:
}
```

下表列出了一些关键属性的含义：

|属性|含义|
|:---:|:----:|
|tm\_pnt_send|请求开始时间
|tm\_dur_dns|DNS 解析时间
|tm\_dur_cnnct|TCP 建立连接时间
|tm\_dur_firstP|首包时间
|tm\_dur_ssl|SSL 握手时间
|statusCode|HTTP 状态码

### 响应时间

响应时间是一个很好的量化指标，可以用来衡量用户请求服务的等待时间，一般将其定义为用户发送请求开始，到服务端的响应内容到达客户端的这段时间。

下图是 HTTP 请求详解图

<p align="center">

<img src="Images/http_request_process.jpg">

</p>

从上图可以观察到响应时间包括 DNS 域名解析时间，与服务器端建立连接时间，服务器处理时间和响应到达客户端的时间。

如果使用 Charles 拦截 HTTP 请求，可以在 OverView Tab 的 Timing 栏看响应时间的数据，如下图中的 `Duration` 表示就是该请求总的响应时间，其中还包含上文提及的 `DNS`（DNS 域名解析时间）、`Connect`（建立连接时间）和 `SSL Handshake`（SSL 握手时间），因为这个请求是 HTTP 请求，所以 `SSL Handshake` 这个字段留空。

<p align="center">

<img src="Images/charles_response_time.jpg">

</p>

事实上当开发完 SDK 中响应时间的 feature 后，我们也可以通过这种方式来验收结果的正确性，当然 SDK 中获取的时间与 Charles 不可能完全相等，因为两种的实现方式完全不同，但是他们之间差值应该在一个合理的范围内。下文会详细探讨这方面。

通过上文的介绍我们很容易想到的一个思路：通过 hook 请求发出时的函数，记录下请求的时间，再 hook iOS SDK 中响应的回调，记录下结束的时间，计算差值即可得到这次请求的响应时间。听云的大致思路也是如此，只不过其中还有许多细节需要注意，我们接下来详细讨论它的具体实现方案。

#### 请求开始

听云是在 `_nbs_hook_NSURLSessionTask` 函数中 hook NSURLSessionTask 的 `resume` 方法来达到记录请求开始的目的。

```
void _nbs_hook_NSURLSessionTask() {
    r14 = _objc_msgSend;
    rax = [NSURLSessionConfiguration ephemeralSessionConfiguration];
    rax = [rax retain];
    var_40 = rax;
    rax = [NSURLSession sessionWithConfiguration:rax];
    rax = [rax retain];
    rdx = 0x0;
    var_38 = rax;
    rax = [rax dataTaskWithURL:rdx];
    rax = [rax retain];
    var_30 = rax;
    rbx = [rax class];
    r12 = @selector(resume);
    if (class_getInstanceMethod(rbx, r12) != 0x0) {
            r15 = @selector(superclass);
            r13 = @selector(resume);
            var_48 = r15;
            do {
                    if (_nbs_slow_isClassItSelfHasMethod(rbx, r12) != 0x0) {
                            r15 = class_getInstanceMethod(rbx, r12);
                            rax = method_getImplementation(r15);
                            rax = objc_retainBlock(__NSConcreteStackBlock);
                            var_50 = imp_implementationWithBlock(rax);
                            r13 = r13;
                            [rax release];
                            rdi = r15;
                            r15 = var_48;
                            rax = method_getTypeEncoding(rdi);
                            rdx = var_50;
                            rcx = rax;
                            class_replaceMethod(rbx, r12, rdx, rcx);
                    }
                    r14 = _objc_msgSend;
                    rbx = _objc_msgSend(rbx, r15, rdx, rcx);
                    rax = class_getInstanceMethod(rbx, r13);
                    r12 = r13;
            } while (rax != 0x0);
    }
    (r14)(var_30, @selector(cancel), rdx);
    (r14)(var_38, @selector(finishTasksAndInvalidate), rdx);
    [var_30 release];
    [var_38 release];
    [var_40 release];
    return;
}
```

将上面伪代码还原为 Objective-C 代码如下：

```
void _nbs_hook_NSURLSessionTask() {
    NSURLSessionConfiguration *config = [NSURLSessionConfiguration ephemeralSessionConfiguration];
    NSURLSession *session = [NSURLSession sessionWithConfiguration:config];
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wnonnull"
    NSURLSessionDataTask *task = [session dataTaskWithURL:nil];
#pragma clang diagnostic pop
    Class cls = task.class;
    if (class_getInstanceMethod(cls, @selector(resume))) {
        Method method;
        do {
            if (_nbs_slow_isClassItSelfHasMethod(cls, @selector(resume))) {
                Method resumeMethod = class_getInstanceMethod(cls, @selector(resume));
                IMP imp = imp_implementationWithBlock(^(id self) {
                });
                class_replaceMethod(cls, @selector(resume), imp, method_getTypeEncoding(resumeMethod));
            }
            
            cls = [cls superclass];
            method = class_getInstanceMethod(cls, @selector(resume));
        } while (method);
        
    }
    [task cancel];
    [session finishTasksAndInvalidate];
}
```

我们知道在 `Foundation` 框架中有些类其实是类族（Class Cluster），比如 `NSDictionary` 和 `NSArray`。而 `NSURLSessionTask` 也是一个类族，在不同的系统版本中继承链不同，所以显然不能直接 hook `NSURLSessionTask` 类，这里采用了一个巧妙的方法，通过 `ephemeralSessionConfiguration` 方法构建一个 Ephemeral sessions（短暂的会话），它与默认会话类似，不过不会将任何数据存储到磁盘中，所有缓存，cookie 和凭据等都保存在 RAM 中并与会话相关联。这样一来当会话无效时，它们将自动清除。然后通过这个短暂会话创建了一个 session 对象，最后构建出 task 对象，并通过这个 task 对象获得真正的类。

上面这种巧妙的做法其实并非听云独创，他其实是参考了 [AFNetworking](https://github.com/AFNetworking/AFNetworking/blob/e8fde524d712e5d369c43f355bd2f01c91ad0359/AFNetworking/AFURLSessionManager.m) 的做法。AFNetworking 中为了加上通知，在 `AFURLSessionManager` 中也实现了 Hook `NSURLSessionTask` 的 `resume` 和 `suspend` 方法。

```
    if (NSClassFromString(@"NSURLSessionTask")) {
        /**
         iOS 7 and iOS 8 differ in NSURLSessionTask implementation, which makes the next bit of code a bit tricky.
         Many Unit Tests have been built to validate as much of this behavior has possible.
         Here is what we know:
            - NSURLSessionTasks are implemented with class clusters, meaning the class you request from the API isn't actually the type of class you will get back.
            - Simply referencing `[NSURLSessionTask class]` will not work. You need to ask an `NSURLSession` to actually create an object, and grab the class from there.
            - On iOS 7, `localDataTask` is a `__NSCFLocalDataTask`, which inherits from `__NSCFLocalSessionTask`, which inherits from `__NSCFURLSessionTask`.
            - On iOS 8, `localDataTask` is a `__NSCFLocalDataTask`, which inherits from `__NSCFLocalSessionTask`, which inherits from `NSURLSessionTask`.
            - On iOS 7, `__NSCFLocalSessionTask` and `__NSCFURLSessionTask` are the only two classes that have their own implementations of `resume` and `suspend`, and `__NSCFLocalSessionTask` DOES NOT CALL SUPER. This means both classes need to be swizzled.
            - On iOS 8, `NSURLSessionTask` is the only class that implements `resume` and `suspend`. This means this is the only class that needs to be swizzled.
            - Because `NSURLSessionTask` is not involved in the class hierarchy for every version of iOS, its easier to add the swizzled methods to a dummy class and manage them there.
        
         Some Assumptions:
            - No implementations of `resume` or `suspend` call super. If this were to change in a future version of iOS, we'd need to handle it.
            - No background task classes override `resume` or `suspend`
         
         The current solution:
            1) Grab an instance of `__NSCFLocalDataTask` by asking an instance of `NSURLSession` for a data task.
            2) Grab a pointer to the original implementation of `af_resume`
            3) Check to see if the current class has an implementation of resume. If so, continue to step 4.
            4) Grab the super class of the current class.
            5) Grab a pointer for the current class to the current implementation of `resume`.
            6) Grab a pointer for the super class to the current implementation of `resume`.
            7) If the current class implementation of `resume` is not equal to the super class implementation of `resume` AND the current implementation of `resume` is not equal to the original implementation of `af_resume`, THEN swizzle the methods
            8) Set the current class to the super class, and repeat steps 3-8
         */
        NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration ephemeralSessionConfiguration];
        NSURLSession * session = [NSURLSession sessionWithConfiguration:configuration];
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wnonnull"
        NSURLSessionDataTask *localDataTask = [session dataTaskWithURL:nil];
#pragma clang diagnostic pop
        IMP originalAFResumeIMP = method_getImplementation(class_getInstanceMethod([self class], @selector(af_resume)));
        Class currentClass = [localDataTask class];
        
        while (class_getInstanceMethod(currentClass, @selector(resume))) {
            Class superClass = [currentClass superclass];
            IMP classResumeIMP = method_getImplementation(class_getInstanceMethod(currentClass, @selector(resume)));
            IMP superclassResumeIMP = method_getImplementation(class_getInstanceMethod(superClass, @selector(resume)));
            if (classResumeIMP != superclassResumeIMP &&
                originalAFResumeIMP != classResumeIMP) {
                [self swizzleResumeAndSuspendMethodForClass:currentClass];
            }
            currentClass = [currentClass superclass];
        }
        
        [localDataTask cancel];
        [session finishTasksAndInvalidate];
    }
```

> `_nbs_slow_isClassItSelfHasMethod` 方法中会调用 `class_copyMethodList` 方法获取这个类的方法列表，注意这个方法获取的方法列表不包含父类的方法，所以 `_nbs_slow_isClassItSelfHasMethod` 方法其实就是判断 `cls` 这个类自己是不是包含 `@selector(resume)`。

事实上，上面的逻辑在开源库 [FLEX](https://github.com/Flipboard/FLEX/blob/29afa5e80f86be5373c11839ea3942722f67e696/Classes/Network/PonyDebugger/FLEXNetworkObserver.m) 中也有实现，只不过实现上略有区别，**FLEX** 会根据系统版本来区分是 hook `__NSCFLocalSessionTask`、NSURLSessionTask 和 __NSCFURLSessionTask，个人感觉听云的实现比 **FLEX** 的硬编码更优雅点。因为 `__NSCFLocalSessionTask` 和 `__NSCFURLSessionTask` 是私有类，所以采取了拆分再拼接的方式来避免审核被拒。

```
+ (void)injectIntoNSURLSessionTaskResume
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // In iOS 7 resume lives in __NSCFLocalSessionTask
        // In iOS 8 resume lives in NSURLSessionTask
        // In iOS 9 resume lives in __NSCFURLSessionTask
        Class class = Nil;
        if (![[NSProcessInfo processInfo] respondsToSelector:@selector(operatingSystemVersion)]) {
            class = NSClassFromString([@[@"__", @"NSC", @"FLocalS", @"ession", @"Task"] componentsJoinedByString:@""]);
        } else if ([[NSProcessInfo processInfo] operatingSystemVersion].majorVersion < 9) {
            class = [NSURLSessionTask class];
        } else {
            class = NSClassFromString([@[@"__", @"NSC", @"FURLS", @"ession", @"Task"] componentsJoinedByString:@""]);
        }
        SEL selector = @selector(resume);
        SEL swizzledSelector = [FLEXUtility swizzledSelectorForSelector:selector];

        Method originalResume = class_getInstanceMethod(class, selector);

        void (^swizzleBlock)(NSURLSessionTask *) = ^(NSURLSessionTask *slf) {
            [[FLEXNetworkObserver sharedObserver] URLSessionTaskWillResume:slf];
            ((void(*)(id, SEL))objc_msgSend)(slf, swizzledSelector);
        };

        IMP implementation = imp_implementationWithBlock(swizzleBlock);
        class_addMethod(class, swizzledSelector, implementation, method_getTypeEncoding(originalResume));
        Method newResume = class_getInstanceMethod(class, swizzledSelector);
        method_exchangeImplementations(originalResume, newResume);
    });
}
```

上面替换原来 `resume` 的实现是通过 `imp_implementationWithBlock`实现的，而替换后的 block 如下：

```
void ___nbs_hook_NSURLSessionTask_block_invoke(int arg0, int arg1, int arg2) {
    rbx = [[NSDate date] retain];
    [rbx timeIntervalSince1970];
    var_40 = intrinsic_movsd(var_40, xmm0);
    [rbx release];
    r15 = _is_tiaoshi_kai;
    COND = *(int8_t *)r15 == 0x0;
    var_50 = r13;
    if (!COND) {
            rax = [var_30 URL];
            rax = [rax retain];
            r14 = r15;
            r15 = rax;
            rbx = [[r15 absoluteString] retain];
            rdx = rbx;
            __NBSDebugLog(0x3, @"NSURLSession:start:url:%@", rdx, rcx, r8, r9, stack[2048]);
            [rbx release];
            rdi = r15;
            r15 = r14;
            [rdi release];
    }
    rbx = [objc_getAssociatedObject(r12, @"m_SessAssociatedKey") retain];
    if (rbx != 0x0) {
            xmm1 = intrinsic_movsd(xmm1, var_40);
            xmm1 = intrinsic_mulsd(xmm1, *0x1000b9990);
            xmm0 = intrinsic_movsd(xmm0, *0x1000b9da8);
            [rbx startWithIP:0x0 DNSTime:var_30 atTimePoint:r8 withObject:r9];
            [rbx setRequest:var_30];
            [rbx setLibClassId:0x1];
    }
    else {
            if (*(int8_t *)r15 != 0x0) {
                    __NBSDebugLog(0x3, cfstring_r, rdx, rcx, r8, r9, stack[2048]);
            }
    }
}
```

上面伪代码中将 `___nbs_hook_NSURLSessionTask_block_invoke` 中不相关的逻辑忽略了，可以看到其中生成了时间戳，并将该时间戳
当做 `[rbx startWithIP:0x0 DNSTime:var_30 atTimePoint:r8 withObject:r9]` 方法的入参，`rbx` 为 `_priv_NBSHTTPTransaction` 实例，而该实例是通过 `NSURLSessionDataTask` 的关联对象获取到的。而 `_priv_NBSHTTPTransaction` 实例的创建和关联对象的设置逻辑则在 `-[_priv_NSURLSession_NBS nbs_dataTaskWithRequest:completionHandler:]` 方法中。

```
r12 = [[var_30 nbs_dataTaskWithRequest:r13 completionHandler:0x0] retain];
r15 = [_priv_NBSHTTPTransaction new];
if (r12 != 0x0) {
        objc_setAssociatedObject(r12, @"m_SessAssociatedKey", r15, 0x301);
}
[r15 release];
```

`-[_priv_NBSHTTPTransaction startWithIP:DNSTime:atTimePoint:withObject:]` 方法会将参数的时间赋值给自己的 `tm_pnt_send` 属性。

```
-[_priv_NBSHTTPTransaction startWithIP:DNSTime:atTimePoint:withObject:] {
    var_30 = intrinsic_movsd(var_30, arg4, rdx, arg5);
    r12->tm_pnt_send = intrinsic_movsd(r12->tm_pnt_send, intrinsic_movsd(xmm0, var_30));

}
```

当然除了 `-[_priv_NSURLSession_NBS nbs_dataTaskWithRequest:completionHandler:]` 方法外，下面的方法也包含这段逻辑：

* `nbs_downloadTaskWithRequest:`
* `nbs_downloadTaskWithRequest:completionHandler:`
* `nbs_downloadTaskWithResumeData:completionHandler:`
* `nbs_uploadTaskWithRequest:fromData:completionHandler:`
* `nbs_uploadTaskWithRequest:fromFile:completionHandler:`
* `nbs_uploadTaskWithRequest:fromFile:`
* `nbs_uploadTaskWithRequest:fromData:`
* `nbs_uploadTaskWithStreamedRequest:`

#### 请求结束

在 `finishAt` 方法中计算得到最终的响应时间，并将其赋值给 `tm_dur_end` 属性。

```
void -[_priv_NBSHTTPTransaction finishAt:](void * self, void * _cmd, double arg2) {
	 r14 = self;
    rbx = [r14 retain];
    r14 = @selector(tm_pnt_send);
    _objc_msgSend(rbx, r14);
    xmm1 = intrinsic_xorpd(xmm1, xmm1);
    xmm0 = intrinsic_ucomisd(xmm0, xmm1);
    COND = xmm0 <= 0x0;
    if (!COND) {
            _objc_msgSend(rbx, r14);
            xmm1 = intrinsic_movsd(xmm1, var_30);
            xmm1 = intrinsic_subsd(xmm1, xmm0);
            xmm0 = intrinsic_movapd(xmm0, xmm1);
            [rbx setTm_dur_end:rdx];
    }
}
```

对于调用 `dataTaskWithRequest:completionHandler:` 方法发起的网络请求，请求完成的回调在 `completionHandler` 中，所以应该在完成的回调中调用 `finishAt` 方法。类似的还有 `___72-[_priv_NSURLSession_NBS nbs_downloadTaskWithRequest:completionHandler:]_block_invoke`、`___79-[_priv_NSURLSession_NBS nbs_uploadTaskWithRequest:fromData:completionHandler:]_block_invoke`等方法。

```
int ___68-[_priv_NSURLSession_NBS nbs_dataTaskWithRequest:completionHandler:]_block_invoke(int arg0, int arg1, int arg2, int arg3) {
    rdi = *(r12 + 0x20);
    xmm0 = intrinsic_movsd(xmm0, var_68);
    [rdi finishAt:rdx];
}
```

然而对于调用 `dataTaskWithRequest:` 方法的网络请求，则需要 hook `NSURLSessionTaskDelegate` 的 `URLSession:task:didCompleteWithError:` 方法。

```
void -[_priv_NBSLensAllMethodsDlgt_urlSess nbs_URLSession:task:didCompleteWithError:](void * self, void * _cmd, void * arg2, void * arg3, void * arg4) {
    rbx = [[NSDate date] retain];
    [rbx timeIntervalSince1970];
    xmm0 = intrinsic_mulsd(xmm0, *0x1000b9990);
    var_40 = intrinsic_movsd(var_40, xmm0);
    [rbx release];
    rax = objc_getAssociatedObject(r13, @"m_SessAssociatedKey");
    rax = [rax retain];
	 _objc_msgSend(r12, @selector(finishAt:), var_58);
}
```

## 致谢

* [林柏参](https://github.com/BaiCan)