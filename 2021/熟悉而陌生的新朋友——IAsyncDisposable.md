#   熟悉而陌生的新朋友——IAsyncDisposable

在`.NET Core 3.0`的版本更新中，官方我们带来了一个新的接口 `IAsyncDisposable`。

小伙伴一看肯定就知道，它和.NET中原有的`IDisposable`接口肯定有着密布可分分的关系，且一定是它的异步实现版本。

那么.NET是为什么要在 **.NET Core 3.0 （伴随C# 8）** 发布的同时，带来该接口呢？ 还有就是该异步版本和原来的`IDispose`有着什么样的区别呢？ 到底在哪种场景下我们能使用它呢？

带着这些问题，我们今天一起来认识一下这位"新朋友" —— `IAsyncDisposable`。 

为了更好的了解它，让我们先来回顾一下.NET中的资源释放：

## .NET的资源释放

由于.NET强大的GC，对于托管资源来说（比如C#的类实例），它的释放往往不需要开发人员来操心。  

但是在开发过程中，有时候我们需要涉及到非托管的资源，比如I/O操作，将缓冲区中的文本内容保存到文件中、网络通讯，发送数据包等等。  

由于这些操作GC没有办法控制，所以也就没有办法来管理它们的生命周期。如果使用了非托管资源之后，没有及时进行释放资源，那么就会造成内存的泄漏问题。

而.NET为我们提供了一些手段来进行资源释放的操作：

### 析构函数 

析构函数在C#中是一个语法糖，在构造函数前方加一个`～`符号即代表使用析构函数 。

```csharp
public class ExampleClass
{
	public ExampleClass()
	{
	}

	~ExampleClass()	// 析构函数
	{
		// 释放非托管资源
	}
}
```

当一个类申明了析构函数了之后，GC将会对它进行特殊的处理，当该实例的资源被GC回收之前会调用析构函数。（该部分内容本文将不做过多介绍）

虽然析构函数方法在某些需要进行清理的情况下是有效的，但它有下面两个严重的缺点：

+ 只有在GC检测到某个对象可以被回收时才会调用该对象的终结方法，这发生在不冉需要资源之后的某个不确定的时间。这样一来，开发人员可以或希望释放资源的时刻与资源实际被终结方法释放的时刻之间会有一个延迟。如果程序需要使用许多稀缺资源（容易耗尽的资源）或不释放资源的代价会很高（例如，大块的非托管内存），那么这样的延迟可能会让人无法接受。
+ 当CLR需要调用终结方法时，它必须把回收对象内存的工作推迟到垃圾收集的下一轮（终结方法会在两轮垃圾收集之间运行）。这意味着对象的内存会在很长一段时间内得不到释放。

因此，如果需要尽快回收非托管资源，或者资源很稀缺，或者对性能要求极高以至于无法接受在GC时增加额外开销，那么在这些情况下完全依靠析构函数的方法可能不太合适。

而框架提供了`IDisposable`接口，该接口为开发人员提供了一种手动释放非托管资源的方法，可以用来立即释放不再需要的非托管资源。

### IDisposable

从`.NET Framework 1.1`开始 ，.NET就为我们提供了`IDispose`接口。

使用该接口，我们可以实现名为`Dispose`的方法，进行一些手动释放资源的操作（包括托管资源和非托管资源）。 

```csharp

public class ExampleClass:IDisposable
{
	private Stream _memoryStream = new MemoryStream();

	public ExampleClass()
	{
	}
	
	public void Dispose()
	{
		// 释放资源
		myList.Clear();
		myData = null;
		_memoryStream.Dispose();	
	}
}

```

在C#中，我们除了可以手动调用 `xx.Dispose()`方法来触发释放之外，还可以使用`using`的语法糖。 

当我们在 visual studio 中添加`IDisposable`接口时，它会提示我们使用是否使用“释放模式”：

![101.png](https://i.loli.net/2021/08/24/UmVcsXute6ZQhJM.png)

“释放模式”所生成的代码如下：

```csharp

protected virtual void Dispose(bool disposing)
{
	if (!disposedValue)
	{
		if (disposing)
		{
			// TODO: 释放托管状态(托管对象)
		}

		// TODO: 释放未托管的资源(未托管的对象)并重写终结器
		// TODO: 将大型字段设置为 null
		disposedValue = true;
	}
}

// // TODO: 仅当“Dispose(bool disposing)”拥有用于释放未托管资源的代码时才替代终结器
// ~ExampleClass()
// {
//     // 不要更改此代码。请将清理代码放入“Dispose(bool disposing)”方法中
//     Dispose(disposing: false);
// }

public void Dispose()
{
	// 不要更改此代码。请将清理代码放入“Dispose(bool disposing)”方法中
	Dispose(disposing: true);
	GC.SuppressFinalize(this);
}

```

释放资源的代码被放置在 `Dispose(bool disposing)` 方法中，你可以选用 析构函数 或者 IDisposable 来进行调用该方法。

这里说一下：在 IDisposable 的实现中，有一句 `GC.SuppressFinalize(this);`。 这句话的意思是，告诉GC，不需要对该类的析构函数进行单独处理了。也就是说，该类的析构函数将不会被调用。因为资源已经在 `Dispose()` 中被我清理了。

## 异步时代

从`.NET Core`开始，就意味着.NET来到了一个全新的异步时代。无论是各种基础类库（比如System.IO）、AspNet Core、还是EFCore.....  它们都支持异步操作，应该说是推荐异步操作。

在今天，假如一个新项目没有使用 `await` 和 `async`。你都会觉得自己在写**假代码😂**。

现在越来越多的开发者都爱上了这种异步方式：不阻止线程的执行，带来高性能的同时还完全不需要更改原有的编码习惯，可谓是两全其美。

所以从`.NET Core` 开始到现在的`.NET 5` ，每一次版本更迭都会有一批API提供了异步的版本。  

### IAsyncDisposable的诞生

为了提供这样一种机制让使用者能够执行资源密集型的处置操作，而不会长期阻塞GUI应用程序的主线程，我们让操作成为了异步。

同样，释放资源的时候我们能否成为异步呢？  假如一次释放操作会占耗费太多的时间，那为什么我们不让它去异步执行呢？ 

为了解决这一问题，同时更好的完善`.NET`异步编程的体验，`IAsyncDisposable`诞生了。

它的用法与`IDisposable`非常的类似：

```csharp

public class ExampleClass : IAsyncDisposable
{
	private Stream _memoryStream = new MemoryStream();

	public ExampleClass()
	{

	}

	public async ValueTask DisposeAsync()
	{
		await _memoryStream.DisposeAsync();
	}
}

```

当然，`using`的语法糖同样适用于它。不过，由于它是异步编程的风格，在使用时记得添加`await`关键字：

```csharp

await using var s = new ExampleClass()
{
	// doing
};

```

当然在 `C# 8` 以上，我们可以使用`using作用域`的简化写法：

```csharp

await using var s = new ExampleClass();
// doing

```

### IAsyncDisposable与IDisposable的选择

有一个关键点是： `IAsyncDisposable` 其实并没有继承于 `IDisposable`。

这就意味着，我们可以选择两者中的任意一个，或者同时都要。

那么我们到底该选择哪一个呢？

这个问题其实很类似于EF刚为大家提供`SaveChangesAsync`方法的时候，到底我们该选用`SaveChangesAsync`还是`SaveChanges`呢？

在以往同步版本的代码中，我们往往会选择`SaveChanges`同步方法。 当来到了异步的环境，我们往往会选择`SaveChangesAsync`。 

所以在`AspNet Core`这个全流程异步的大环境下，我们的代码潜移默化的就会更改为`SaveChangesAsync`。

而`IAsyncDisposable`也是同理的，当我们处于异步的环境中，所使用的资源提供了异步释放的接口，那么我们肯定就会自然而然的使用`IAsyncDisposable`。

在`.NET 5` 之后，大部分的类都具有了`IAsyncDisposable`的实现。比如： 

+ `Utf8JsonWriter`、`StreamWriter`这些于文件操作有关的类；
+ `DbContext`这类数据库操作类
+ `Timer`
+ 依赖注入的`ServiceProvider`
+ ………………

接下来的`.NET`版本中，我们也会看到`AspNet Core`中的`Controller` 等对于`IAsyncDisposable`提供支持。

![102.png](https://i.loli.net/2021/08/24/7qeP1LYkfglFNDZ.png)

可以预测是，在未来的`.NET`发展中，全异步的发展是必然的。后面越来越的已有库会支持异步的所有操作，包括`IAsyncDisposable`的使用也会越来越频繁。

### Asp Net Core 依赖注入中的IAsyncDisposable

对于咱们使用`AspNet Core`的开发人员来说，我们在大多数情况下都会依赖于框架所提供的依赖注入功能。 

而依赖注入框架，会在作用域释放的时候，自动去调用所注入服务的释放接口`IDisposable`。

比如我们把 `DbContext` 注入之后，其实就只管使用就行了，从来不会关心它的`Dispose`问题。 相对于传统`using(var dbContext = new MyDbContext)`的方式要省心很多，也不会担心忘记写释放而导致的数据库连接未释放的问题。

那么，当`IAsyncDisposable`出现之后呢？会出现什么情况：

```csharp

public void ConfigureServices(IServiceCollection services)
{
	services.AddControllers();

	services.AddScoped<DemoDisposableObject>();	// 注入测试类	
}


public class DemoDisposableObject : IAsyncDisposable
{
	public ValueTask DisposeAsync()
	{
		 code here  
		// 当完成一次http 请求后，该方法会自动调用
	}
}
```

当我们实现了`IAsyncDisposable`之后，会被自动调用。

那么如果 `IAsyncDisposable` 和 `IDisposable` 一同使用呢？

```csharp

public class DemoDisposableObject : IAsyncDisposable,IDisposable
{
	public void Dispose()
	{
		code here  
	}

	public ValueTask DisposeAsync()
	{
		code here  
	}
}

```

这样的结果是：*只有`DisposeAsync`方法会被调用*。

为什么会有这样的结果呢？ 让我们一起来扒开它的面纱。

*以下代码位于 [AspNet Core源码](https://github.com/dotnet/aspnetcore/blob/8b30d862de6c9146f466061d51aa3f1414ee2337/src/Http/Http/src/Features/RequestServicesFeature.cs#L59)*

```csharp
public class RequestServicesFeature : IServiceProvidersFeature, IDisposable, IAsyncDisposable
{
	public IServiceProvider RequestServices
	{
		get
		{
			if (!_requestServicesSet && _scopeFactory != null)
			{
				_scope = _scopeFactory.CreateScope();
				……………………
			}
			return _requestServices!;
		}
	}

	public ValueTask DisposeAsync()
	{
		switch (_scope)
		{
			case IAsyncDisposable asyncDisposable:
				var vt = asyncDisposable.DisposeAsync();
				………………
				break;
			case IDisposable disposable:
				disposable.Dispose();
				break;
		}

		……………………
		return default;
	}

	public void Dispose()
	{
		DisposeAsync().AsTask().GetAwaiter().GetResult();
	}
}
```

为了方便起见，我省略了部分代码。 这里的关键代码在于： `DisposeAsync()`方法，它会在内部进行判断，`IServiceScope`是否为`IAsyncDisposable`类型。如果是，则会采用它的`IServiceScope`的异步释放方法。

所以本质上还是回到了官方依赖注入框架中`IServiceScope`的实现：

*以下代码位于 [DependencyInjection源码](https://github.com/dotnet/runtime/blob/b7f505dcb13f0fd80a33c962446bf13f87a31b33/src/libraries/Microsoft.Extensions.DependencyInjection/src/ServiceLookup/ServiceProviderEngineScope.cs#L112)*

```csharp

internal sealed class ServiceProviderEngineScope : IServiceScope, IServiceProvider, IAsyncDisposable, IServiceScopeFactory
{
	public ValueTask DisposeAsync()
	{
		List<object> toDispose = BeginDispose();

		if (toDispose != null)
		{
			try
			{
				for (int i = toDispose.Count - 1; i >= 0; i--)
				{
					object disposable = toDispose[i];
					if (disposable is IAsyncDisposable asyncDisposable)
					{
						ValueTask vt = asyncDisposable.DisposeAsync();
						if (!vt.IsCompletedSuccessfully)
						{
							return Await(i, vt, toDispose);
						}

						// If its a IValueTaskSource backed ValueTask,
						// inform it its result has been read so it can reset
						vt.GetAwaiter().GetResult();
					}
					else
					{
						((IDisposable)disposable).Dispose();
					}
				}
			}
			catch (Exception ex)
			{
				return new ValueTask(Task.FromException(ex));
			}
		}

		return default;
}
```

可以看出`新版本的IServiceScope实现一定是继承了IAsyncDisposable接口`，所以在上面的`AspNet Core`的代码里，它一定会调用`IServiceScope`的`DisposeAsync()`方法。

而`IServiceScope`的默认实现在异步释放时会进行判断：如果注入的实例为`IAsyncDisposable`则调用`DisposeAsync()`，否则判断是否为`IDisposable`。

这也解释了为什么我们在上面同时实现两个释放接口，却只有异步版本的会被调用。

## 总结

在上面的文章中，我们了解到`IAsyncDisposable`作为`.NET`异步发展中一个重要的新接口，在应用上会被越来越频繁的使用，它将逐步完善`.NET`的异步生态。

当存在下方的情况时，我们应该优先考虑来使用它：

+ 当内部拥有的资源具有对`IAsyncDisposable`的实现（比如`Utf8JsonWriter`等），我们可以采用使用`IAsyncDisposable`来对他们进行释放。
+ 当在异步的大环境下，新编写一个需要释放资源的类，可以优先考虑使用`IAsyncDisposable`。

现在`.NET`的很多类库都已经同时支持了`IDisposable`和`IAsyncDisposable`。而从使用者的角度来看，其实调用任何一个释放方法都能够达到释放资源的目的。就好比`DbContext`的`SaveChanges`和`SaveChangesAsync`。

但是从未来的发展角度来看，`IAsyncDisposable`会成使用的更加频繁。因为它应该能够优雅地处理托管资源，而不必担心死锁。

而对于现在已有代码中实现了`IDisposable`的类，如果想要使用`IAsyncDisposable`。建议您同时实现两个接口，已保证使用者在使用时，无论调用哪个接口都能达到效果，而达到兼容性的目的。

类似于下方代码：

*节选自**Stream**类的[源码](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Private.CoreLib/src/System/IO/Stream.cs)*
```csharp
public void Dispose() => Close();

public virtual void Close()
{
   
    Dispose(true);
    GC.SuppressFinalize(this);
}

public virtual ValueTask DisposeAsync()
{
    try
    {
        Dispose();
        return default;
    }
    catch (Exception exc)
    {
        return ValueTask.FromException(exc);
    }
}
```

最后的最后，希望 **点赞，关注，一键三连** 走一波。