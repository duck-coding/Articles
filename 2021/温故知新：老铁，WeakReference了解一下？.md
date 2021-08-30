
> 弱引用是个什么鬼？大白话说就是不那么强的引用（哈哈，纯属玩笑，实际可不是这样滴），那强引用又是个什么鬼？他们有什么用处？问题有点迷，君阅完这篇文章后或许你心中就有答案了……

## 什么是弱引用

在解释弱引用之前，我们可以先来看看什么是`强引用`。以下是来自官方的定义：

> The garbage collector cannot collect an object in use by an application while the application's code can reach that object. The application is said to have a strong reference to the object.

翻译成大白话就是：应用程序的代码可以访问一个正由该程序使用的对象，垃圾回收器就不能回收该对象，就可以认为应用程序对该对象具有强引用。

我们平常用的都是对象的强引用。这种情况下，假如该对象的实例还在被其他地方所使用，那么`GC` 是不能回收当前对象的。

如果在实际开发中，你创建了一个很大的对象而且该对象还会不断的“生长”（比如一个拥有很大文本的string，被不断添加到一个静态的List中，而该List忘记在操作之后被清空）。

到最后你或许会得到一个OutOfMemoryException的异常，然后正在吃鸡的你突然接到老板的电话：“明天你可以不用来了，又出线上事故！”。

![OutOfMemoryException.png](https://i.loli.net/2021/08/23/YIO5rPigE6lw9Bn.png)

发生该惨案的原因是，大量的对象实例占用了大量的内存。

此时你可能会问：.NET不是自带了 `GC` （垃圾回收）吗？ 他不是会在某些时刻把这些对象给释放掉吗？

然而，就像上文所说，因为 `GC` 认定该对象正在被使用，所以就“不敢”释放该部分资源。

这也提醒了我们，在使用静态资源或者单例对象的时候（特别是静态List、Dictionary等集合）。要特别注意资源释放的问题。

那么有木有一个“神器”既可以保持对象的引用，在适当时候 `GC` 又 **“敢”** 回收掉这个对象呢？

此时就可以介绍我们本篇的主角了：`弱引用`。

> “ 弱引用允许应用程序访问对象，同时也允许垃圾回收器收集、回收相应的对象。”

在`.NET`中，微软给我们提供了`WeakReference`，来解决上述问题：

## WeakReference

我们以一个API服务以及结合本地缓存例子来玩一玩 `WeakReference`，看看他们会发生什么事情。详细代码如下：

```c#
[Route("api/cache")]
[ApiController]
public class CacheController : ControllerBase
{
    public CacheController()
    {
        Interlocked.Increment(ref DiagnosticsController.Requests);
    }

    // 不使用弱引用
    private static Dictionary<long, User> UserCacheStrongReference = new();

    // 使用弱引用
    private static Dictionary<long, WeakReference<User>> UserCacheWeakReference =
        new();

    [HttpGet]
    [Route("StrongReference")]
    public ActionResult<int> GetUserWithStrongReference()
    {
        User user = new User
        {
            Id = DateTime.Now.Ticks,
            Name = new String('dsx', 10 * 1024),
            Age = new Random().Next(),
            Birthday = DateTime.Now
        };

        UserCacheStrongReference.TryAdd(user.Id, user);

        return UserCacheStrongReference.Count;
    }

    [HttpGet]
    [Route("WeakReference")]
    public ActionResult<int> GetUserWithWeakReference()
    {
        User user = new User
        {
            Id = DateTime.Now.Ticks,
            Name = new String('dsx', 10 * 1024),
            Age = new Random().Next(),
            Birthday = DateTime.Now
        };

        UserCacheWeakReference.TryAdd(user.Id, new WeakReference<User>(user));

        return UserCacheWeakReference.Count;
    }
}

public class User
{
    public long Id { get; set; }
    public string Name { get; set; }
    public int Age { get; set; }
    public DateTime Birthday { get; set; }
}
```

> *温馨提示：例子使用了Github上提供的一个示例应用（[传送门](https://github.com/sebastienros/memoryleak)），它可以帮助我们用折线图的方式呈现实时内存和GC数据。*

上面的演示代码比较简单，是我们日常开发中使用本地缓存的常用方式，因此不在赘述。

当程序运行起来的时候，我们使用postjson工具做压力测试，以模拟大量用户请求上述接口的场景。

先看看我们没有任何请求的状态下GC和内存使用情况：

![正常状态下.png](https://i.loli.net/2021/08/23/4xFk3MVeLvuctOX.png)

无请求状态下应用已分配（托管对象占用的内存量）内存接近10M，工作集（进程的虚拟地址空间中当前驻留在物理内存中的页集）内存接近80M。

- 强引用测试

  我们模拟一个用户请求5000次的场景

  ![强引用测试环境.png](https://i.loli.net/2021/08/23/W8oGiUCshXq1AYL.png)

  待程序run一会儿，得到下面的一个折线图：

  ![StrongReference-2.png](https://i.loli.net/2021/08/23/wiBxfSbWdUELrks.png)

  可以看出，随着时间的推移，我们的内存呈现出了直线增长的状态。



- 弱应用测试

  为了测试公平起见，先停止应用程序然后再重新启动，模拟请求参数和上述保持一致：

  ![弱引用测试环境.png](https://i.loli.net/2021/08/23/uomLGjH6Jg5aQPZ.png)

  同样等程序run一会儿，得到下面的一个折线图：

  ![WeakReference-2.png](https://i.loli.net/2021/08/23/8B2KzuGPwF4bYMj.png)

相信看到这儿，您应该已经可以看出差距来了。

在强引用的折线图中，虽然间隔一段时间 `GC` 就会进行一次回收（图中中间位置的三角形）。，但是内存还是不断的在增长。

这也符合我们的预期，应该 `GC` 不敢对 `User` 实例进行回收，从而导致字典中的数据越来越多。

而弱引用却恰恰相反，每当 `GC` 进行一次回收，内存就像过山车一样往下掉。最终一直稳定在 20MB。

还有就是强引用的 `GC` 进行垃圾回收的频率比弱引用高很多。这是因为内存在往上增长时，`GC` 则会越频繁的工作，因为他想赶快帮我们释放资源，可哪知我们却不断创建大对象，“深深的伤害了它”。

## 从WeakReference中获取引用的对象

获取获取当前WeakReference对象引用的对象很简单，WeakReference提供了一个Target属性以及TryGetTarget方法。

我们通过这个属性就可以获取应用的对象，以上述示例为基础，我们从缓存中获取User对象：

```c#
public User GetUserFromWeakReferenceDic(long userId)
{
    if (UserCacheWeakReference.TryGetValue(userId, out WeakReference<User> user))
    {
        if (user.TryGetTarget(out User userInfo))
        {
            return userInfo;
        }else
        {
            //当实例没有的时候，证明它已经被GC所释放了，我们往往需要再次创建它
        }
    }else
    {
        // .....
    }
}
```

## 短弱引用和长弱引用

`WeakReference` 的另外一个构造函数 `WeakReference(Object, Boolean)`，需要我们传入一个bool类型的值，该值表示何时停止跟踪对象：

**长弱引用：** 在对象的Finalize方法被执行以后，长弱引用将获得保留，不过对象的某些成员变量或许已被回收，因此这种模式下需要谨慎使用这些变量。

**短弱引用：** 垃圾回收功能回收对象后，短弱引用的目标值会变为 `null`，因此对象只在目标被回收前有效。 弱引用本身是托管对象，与其他任何托管对象一样需要经过垃圾回收，需要注意的是，如果对象类型不包含Finalize方法，应用的是短弱引用功能。

## 总结

弱引用不是“银弹”，那么我们应该在什么时使用弱引用呢？

+ **当对象占用大量内存，但通过垃圾回收功能回收以后`很容易重新创建的对象`特别适合使用弱引用。**

因此在使用弱引用时我们需要关注以下准则：

- 仅在必要时使用长弱引用，因为在终结后对象的状态不可预知。

- 避免对小对象使用弱引用，因为指针本身可能和对象一样大，或者比对象还大。

- 避免将弱引用作为内存管理问题的自动解决方案， 而应开发一个有效的缓存策略来处理应用程序的对象。