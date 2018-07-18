---
title: Objc Category
date: 2018-07-18 10:24:16
tags:
---
---
title: 使用category重写原方法的坑
date: 2018-07-18 10:21:09
tags:
---
好久没写blog了，有时候宁愿多花时间去研究新东西，也懒得写。最近在写category的时候发现了很多有意思的东西，想在这里跟大家分享一下。有时候我们在创建一个类别后，重写了原类的方法，这个并不会报错，但是会有警告“Category is implementing a method which will also be implemented by its primary class”，运行起来也可以执行，但是不会执行原类的方法，很多人认为覆盖执行了。

其实不然，category只是添加了一个新的方法，在执行的过程中，取的是方法列表的第一个方法就没有再去执行其他相同名称的方法了。我们看源码：
```
for (uint32_t m = 0;
             (scanForCustomRR || scanForCustomAWZ)  &&  m < mlist->count;
             m++)
        {
            SEL sel = method_list_nth(mlist, m)->name;
            if (scanForCustomRR  &&  isRRSelector(sel)) {
                cls->setHasCustomRR();
                scanForCustomRR = false;
            } else if (scanForCustomAWZ  &&  isAWZSelector(sel)) {
                cls->setHasCustomAWZ();
                scanForCustomAWZ = false;
            }
        }
        // Fill method list array
        newLists[newCount++] = mlist;
    .
    .
    .
    // Copy old methods to the method list array
    for (i = 0; i < oldCount; i++) {
        newLists[newCount++] = oldLists[i];
    }

```

源码中category新添加的方法是放在newLists前面的，其他方法在后面，我们总结一下：
1)、category的方法没有“替换掉”原类的方法，也就是说如果category和原来类都有methodA，那么category附加完成之后，类的方法列表里会有两个methodA
2)、category的方法被放到了新方法列表的前面，而原来类的方法被放到了新方法列表的后面，这也就是我们平常所说的category的方法会“覆盖”掉原来类的同名方法，这是因为运行时在查找方法的时候是顺着方法列表的顺序查找的，它只要一找到对应名字的方法，就不会继续查找了，但是后面可能还有一样名字的方法。

所以原类的方法我们还是可以找到的，一般存在于方法列表最后，我们是可以取出来的

```
if (currentClass) {
    unsigned int methodCount;
    Method *methodList = class_copyMethodList(currentClass, &methodCount);
    IMP lastImp = NULL;
    SEL lastSel = NULL;
    for (NSInteger i = 0; i < methodCount; i++) {
        Method method = methodList[i];
        NSString *methodName = [NSString stringWithCString:sel_getName(method_getName(method)) 
                                        encoding:NSUTF8StringEncoding];
        if ([@"targetName" isEqualToString:methodName]) {
            lastImp = method_getImplementation(method);
            lastSel = method_getName(method);
        }
    }
    typedef void (*fn)(id,SEL);

    if (lastImp != NULL) {
        fn f = (fn)lastImp;
        f(my,lastSel);
    }
    free(methodList);
}
```
利用这个原理，我们可以Hook原方法之后，找出原方法再执行。一般我们通过Swizzle去实现，现在呢，又多了一种Hook的方法了。
这种“覆盖”原方法的编码方式苹果本身并不建议，但是如果我们自己把握好节奏也是可以的。我个人也并不建议这样编写，因为当多人同时编写代码时，如果每个人都写了自己的一套category并且覆盖了同样的方法，编译并不会报错，那么到底执行谁写的代码呢，就会出现执行混乱。所以并不建议使用类别“覆盖”原方法。如果想要真正覆盖原方法，可以使用继承来实现。


我们再来看另一个问题，我现在创建了TestMethod和TestMethod2两个category，如图：
```
@implementation StrategyDemo (TestMethod)

- (void)printMethod
{
    NSLog(@"printMethod 1");
}
```

```

@implementation StrategyDemo (TestMethod2)

- (void)printMethod
{
    NSLog(@"printMethod 2");
}
```

并同时添加了printMethod的方法，然后我调用printMethod后到底是打印categoryA的方法还是打印categoryB的方法呢？还是都会打印呢？

![print@2x.png](https://upload-images.jianshu.io/upload_images/1313458-57f6daa9e1b5a962.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们上面讲过，category只是添加了一个新的方法,并没有覆盖原方法，只是在执行的时候是取的methodList里最靠前的执行。所以打印结果一定是编译时methodList最先匹配到方法名的方法。我们也可以手动调整编译顺序，进而改变methodList中方法的顺序，最后改变执行的方法。
![buildPhases@2x.png](https://upload-images.jianshu.io/upload_images/1313458-7780ea8bac25ed26.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图是调整前的顺序，当我调整后:
![buildPhases2@2x.png](https://upload-images.jianshu.io/upload_images/1313458-a7748868a4dc1512.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再看打印结果：
![print2@2x.png](https://upload-images.jianshu.io/upload_images/1313458-dda9ab9fbe5fa7ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

调整后确实是改变了编译时methodlist里方法的顺序。