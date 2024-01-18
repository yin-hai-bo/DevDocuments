# ConfigureAwait FAQ
----

原文地址：[https://devblogs.microsoft.com/dotnet/configureawait-faq/](https://devblogs.microsoft.com/dotnet/configureawait-faq/) [Stephen Toub - MSFT]


.NET在七年多前引入了`async/await`到语言和库中。在这段时间里，它像野火一样迅速传播，不仅在.NET生态系统中得到广泛应用，而且在许多其他语言和框架中也被复制。在.NET中，它还经历了许多改进，包括利用异步性的附加语言构造、提供异步支持的API以及在使`async/await`生效的基础设施方面的根本性改进（特别是在 .NET Core 中的性能和诊断启用方面）。

然而，有关`async/await`的一个方面继续引发疑问，那就是`ConfigureAwait`。在这篇文章中，我希望回答其中许多问题。我打算让这篇文章既能够从头到尾阅读，又能够作为未来参考的常见问题解答（FAQ）列表。

要真正理解`ConfigureAwait`，我们需要稍微从早期开始...

### 什么是同步上下文（SynchronizationContext）？

`System.Threading.SynchronizationContext`文档指出，它“提供在各种同步模型中传播同步上下文的基本功能”。这并不是一个完全明显的描述。

对于 99.9% 的用例，`SynchronizationContext`只是一个提供了`Post`虚方法的类。`Post`虚方法接受一个要异步执行的委托（同步上下文还有一些其他虚成员，但它们使用较少且与本讨论无关）。基类型的`Post`实际上只是调用`ThreadPool.QueueUserWorkItem`以异步调用提供的委托。但是，派生类型会重写民`Post`，以使该委托在最合适的位置和最合适的时间执行。

例如，Windows Forms 有一个[派生自SynchronizationContext的类型](https://github.com/dotnet/winforms/blob/94ce4a2e52bf5d0d07d3d067297d60c8a17dc6b4/src/System.Windows.Forms/src/System/Windows/Forms/WindowsFormsSynchronizationContext.cs)，该类型重写了`Post`以执行`Control.BeginInvoke` 的等效操作。也就是说，对`Post`方法的调用，就是令该委托在`Control`的关联线程（即“UI线程”）上，于稍后的某个时间点上被调用。

Windows Forms 依赖于 Win32 消息处理，并在 UI 线程上运行一个“消息循环”，它简单地等待新消息的到达以进行处理。这些消息可能是“鼠标移动和点击”、“键盘输入”、“系统事件”、“委托可被调用”等，因此，给定一个 Windows Forms 应用程序 UI 线程的同步上下文实例，只需将委托传递给`Post`，即可使其在 UI 线程上执行。

对于 Windows Presentation Foundation（WPF）也是如此。它有一个[派生自SynchronizationContext的类型](https://github.com/dotnet/wpf/blob/ac9d1b7a6b0ee7c44fd2875a1174b820b3940619/src/Microsoft.DotNet.Wpf/src/WindowsBase/System/Windows/Threading/DispatcherSynchronizationContext.cs)，它对`Post`的重写，也是“调度”委托到 UI 线程（通过`Dispatcher.BeginInvoke`），这个行为由 WPF Dispatcher 进行管理（而不是 Windows Forms Control）。

同样，Windows RunTime（WinRT）也是[派生自SynchronizationContext的类型](https://github.com/dotnet/runtime/blob/60d1224ddd68d8ac0320f439bb60ac1f0e9cdb27/src/libraries/System.Runtime.WindowsRuntime/src/System/Threading/WindowsRuntimeSynchronizationContext.cs)，它重写了`Post`，通过`CoreDispatcher`将委托排队到 UI 线程。

这不仅仅是“在UI线程上运行此委托”。任何人都可以实现一个`SynchronizationContext`，通过重写`Post`方法来执行任何操作。例如，我可能不关心委托在哪个线程上运行，但我希望确保投递到我的`SynchronizationContext`的任何委托，都以某种有限程度的并发来执行。我可以通过一个自定义的`SynchronizationContext`来实现这一点：

```CSharp
internal sealed class MaxConcurrencySynchronizationContext : SynchronizationContext
{
    private readonly SemaphoreSlim _semaphore;

    public MaxConcurrencySynchronizationContext(int maxConcurrencyLevel) =>
        _semaphore = new SemaphoreSlim(maxConcurrencyLevel);

    public override void Post(SendOrPostCallback d, object state) =>
        _semaphore.WaitAsync().ContinueWith(delegate
        {
            try { d(state); } finally { _semaphore.Release(); }
        }, default, TaskContinuationOptions.None, TaskScheduler.Default);

    public override void Send(SendOrPostCallback d, object state)
    {
        _semaphore.Wait();
        try { d(state); } finally { _semaphore.Release(); }
    }
}

```

事实上，单元测试框架 xunit 提供了[一个与此非常相似的 SynchronizationContext](https://github.com/xunit/xunit/blob/d81613bf752bb4b8774e9d4e77b2b62133b0d333/src/xunit.execution/Sdk/MaxConcurrencySyncContext.cs)，它用于限制与可以同时运行的测试相关的代码的数量。

所有这些的好处与任何抽象一样：它提供了一个单一的API，可以用于排队处理委托，无需了解该实现的详细信息，只需按照实现者的意愿进行操作。因此，如果我正在编写一个库，我想去做一些工作，然后将委托排队回原始位置的“上下文”，我只需获取它们的`SynchronizationContext`，保存它，然后在完成工作后，调用该上下文的`Post`方法以传递我想要调用的委托。我不需要知道现在是 Windows Forms（我应该获取一个 Control 并使用其`BeginInvoke`），还是 WPF（我应该获取一个 Dispatcher 并使用其`BeginInvoke`），或者对于 xunit 我应该以某种方式获取其上下文并排队给它……我只需获取当前的`SynchronizationContext`，然后在稍后使用它。为了实现这一点，`SynchronizationContext`提供了一个`Current`属性。为了实现上述目标，我可能会编写如下代码：

```CSharp
public void DoWork(Action worker, Action completion)
{
    SynchronizationContext sc = SynchronizationContext.Current;
    ThreadPool.QueueUserWorkItem(_ =>
    {
        try { worker(); }
        finally { sc.Post(_ => completion(), null); }
    });
}
```

如果想让`Current`返回我自定义的`SynchronizationContext`，可以使用`SynchronizationContext.SetSynchronizationContext`方法，来将我的自定义派生类，设置给`SynchronizationContext.Current`。

### TaskScheduler是什么？

`SynchronizationContext`是一个对“调度器”的通用抽象。各个框架有时会有自己的调度器抽象，`System.Threading.Tasks`也不例外。为了让`Task`可以排队和执行，定义了一个名为`System.Threading.Tasks.TaskScheduler`的东西。

`SynchronizationContext`提供了`Post`虚方法，在`Post`中用标准方法的对委托进行排除和调用，而`TaskScheduler`提供了一个抽象的`QueueTask`方法做类似的事情（通过`ExecuteTask`方法调用Task）。

由`TaskScheduler.Default`返回的默认调度器是线程池，但可以从`TaskScheduler`派生并重写相关方法，以实现自定义行为（比如，精确地控制在何时，以及何地调用任务）。

例如，核心库中包含`System.Threading.Tasks.ConcurrentExclusiveSchedulerPair`类型。这个类的一个实例公开两个`TaskScheduler`属性，一个叫`ExclusiveScheduler`，另一个叫`ConcurrentScheduler`。安排给`ConcurrentScheduler`的任务可以并发运行，但受限于在构造`ConcurrentExclusiveSchedulerPair`时提供的限制（类似于前面的`MaxConcurrencySynchronizationContext`），当由`ExclusiveScheduler`调度的任务正在运行时，不会运行`ConcurrentScheduler`的任务，只允许一个独占任务一次运行… 这样，它的行为很像一个读/写锁。

像`SynchronizationContext`一样，`TaskScheduler`也有一个`Current`属性，它返回“当前”的`TaskScheduler`。然而，与`SynchronizationContext`不同，没有设置当前调度程序的方法。相反，当前调度程序是与当前运行的任务关联的调度程序，并且调度程序是作为启动任务的一部分提供给系统的。因此，例如，这个程序将输出“True”，因为与`StartNew`一起使用的 lambda 在`ConcurrentExclusiveSchedulerPair`的`ExclusiveScheduler`上执行，将看到`TaskScheduler.Current`设置为该调度程序：

```CSharp
using System;
using System.Threading.Tasks;

class Program
{
    static void Main()
    {
        var cesp = new ConcurrentExclusiveSchedulerPair();
        Task.Factory.StartNew(() =>
        {
            Console.WriteLine(TaskScheduler.Current == cesp.ExclusiveScheduler);
        }, default, TaskCreationOptions.None, cesp.ExclusiveScheduler).Wait();
    }
}
```

有趣的是，`TaskScheduler`提供了一个静态的`FromCurrentSynchronizationContext`方法，该方法创建一个新的`TaskScheduler`，将任务排队在`SynchronizationContext.Current`返回的上下文中运行，使用其`Post`方法来排队任务。

### SynchronizationContext 和 TaskScheduler 与 await 有什么关系？

在考虑使用`await`时，可以将`SynchronizationContext`和`TaskScheduler`与异步编程相关联。在编写一个带有按钮的UI应用程序时，例如，我们希望在点击按钮时从网站下载一些文本，并将其设置为按钮的内容。**按钮应该仅从拥有它的 UI 线程访问**，因此当我们成功下载新的日期和时间文本，并希望将其存储回按钮的内容时，我们需要从拥有控件的线程（UI线程）执行此操作。如果不这样做，将会出现异常，例如：

```
System.InvalidOperationException: 'The calling thread cannot access this object because a different thread owns it.'
```

如果我们手动编写这部分代码，我们可以像之前展示的那样使用`SynchronizationContext`那样，将内容的设置工作，投递到原始上下文中来执行。例如通过`TaskScheduler`：

```CSharp
private static readonly HttpClient s_httpClient = new HttpClient();

private void downloadBtn_Click(object sender, RoutedEventArgs e)
{
    s_httpClient.GetStringAsync("http://example.com/currenttime").ContinueWith(downloadTask =>
    {
        downloadBtn.Content = downloadTask.Result;
    }, TaskScheduler.FromCurrentSynchronizationContext());
}
```

或直接使用`SynchronizationContext`：

```CSharp
private static readonly HttpClient s_httpClient = new HttpClient();

private void downloadBtn_Click(object sender, RoutedEventArgs e)
{
    SynchronizationContext sc = SynchronizationContext.Current;
    s_httpClient.GetStringAsync("http://example.com/currenttime").ContinueWith(downloadTask =>
    {
        sc.Post(delegate
        {
            downloadBtn.Content = downloadTask.Result;
        }, null);
    });
}
```

然而，这两种方法都明确使用了回调。现在，我们希望使用`async/await`来自然地编写代码：

```CSharp
private static readonly HttpClient s_httpClient = new HttpClient();

private async void downloadBtn_Click(object sender, RoutedEventArgs e)
{
    string text = await s_httpClient.GetStringAsync("http://example.com/currenttime");
    downloadBtn.Content = text;
}
```

这样“就可以工作”，成功在 UI 线程上设置 Content。因为与上面手动实现的版本一样，`await`一个`Task`，关注的是`SynchronizationContext.Current`以及`TaskScheduler.Current`。

在 C# 中，当你`await`任何东西时，编译器会将代码转换为：

- 调用被等待对象（awaitable）的`GetAwaiter()`方法，来得到一个awaiter。在本例中，awaitable对象就是 Task 本身，得到的 awaiter 是`TaskAwaiter<string>`。
- 这个 awaiter 负责关联一个通常名为 "continuation" 的回调，当等待的对象完成时，它使用在注册回调时捕获的任何上下文/调度程序，来执行回调。

虽然这并不完全是使用的代码（还有其他优化和调整），但大致是这样的：

```CSharp
object scheduler = SynchronizationContext.Current;
if (scheduler is null && TaskScheduler.Current != TaskScheduler.Default)
{
    scheduler = TaskScheduler.Current;
}
```

换句话说，它首先检查是否设置了`SynchronizationContext`，如果没有，则检查是否有非默认的`TaskScheduler`在起作用。如果找到一个，当回调准备好被调用时，它将使用捕获的调度程序；否则，它通常只是在完成等待的任务的操作中执行回调。

### ConfigureAwait(false) 做了什么？

ConfigureAwait 方法并不特殊：它不以任何特殊的方式被编译器或运行时识别。它只是一个返回结构体`ConfiguredTaskAwaitable`的方法，该结构体包装了调用它的原始任务以及指定的布尔值。

请记住，`await`可以用于任何类型，只要这些类型符合`await`要求的模式。这意味着当编译器访问实例的`GetAwaiter`方法（模式的一部分）时，它是从`ConfigureAwait`返回的类型（`ConfiguredTaskAwaitable`）而不是直接从`Task`中获取的，这提供了一种通过这个自定义`awaiter`更改`await`行为的方法。

具体来说，与直接等待`Task`不同，等待从`ConfigureAwait(continueOnCapturedContext: false)`返回的类型，最终影响了先前显示的有关如何捕获目标上下文/调度程序的逻辑。它实际上使先前显示的逻辑更像是这样的：

```CSharp
object scheduler = null;
if (continueOnCapturedContext)
{
    scheduler = SynchronizationContext.Current;
    if (scheduler is null && TaskScheduler.Current != TaskScheduler.Default)
    {
        scheduler = TaskScheduler.Current;
    }
}
```

反过来说，通过指定`false`，即使有当前的上下文或调度程序可以回调，它也会假装不存在。

### 使用 `ConfigureAwait(false)` 有什么好处？

`ConfigureAwait(continueOnCapturedContext: false)`用于避免强制回调在原始上下文或调度程序上调用。这有一些好处：

#### 1. 提高性能：

将回调排队而不仅仅是调用它会有一些成本，因为涉及到额外的工作（通常还有额外的分配），而且因为这意味着运行时中我们本来想要使用的某些优化不能使用（当我们确切地知道回调将如何被调用时，我们可以进行更多的优化，但如果它被交给抽象的任意实现，有时我们可能受限）。对于非常频繁的路径，即使检查当前的 SynchronizationContext 和当前的 TaskScheduler （两者都涉及访问线程静态变量）的额外成本也可能增加可测量的开销。如果在 await 之后的代码实际上不需要在原始上下文中运行，使用 ConfigureAwait(false) 可以避免所有这些成本：它不需要不必要地排队，它可以利用它能够集结的所有优化，并且可以避免不必要的线程静态访问。

#### 2. 避免死锁：

考虑一个使用 await 等待某些网络下载结果的库方法。你调用这个方法并阻塞等待它完成，比如通过使用返回的 Task 对象的 `.Wait()`、`.Result` 或 `.GetAwaiter().GetResult()` 等同步方法。现在考虑一下，如果你调用它的时候，当前的 SynchronizationContext 是一个限制在其上运行的操作数量为 1 的上下文，无论是通过类似前面显示的 MaxConcurrencySynchronizationContext 这样的显式方式，还是通过这是一个只有一个可以使用的线程的上下文（例如，UI 线程）这样的隐式方式。所以你在那一个线程上调用方法，然后阻塞它等待操作完成。该操作启动了网络下载并等待它。由于默认情况下等待 Task 会捕获当前的 SynchronizationContext，它这样做，当网络下载完成时，它将回调排队到 SynchronizationContext，以调用操作的其余部分。但唯一能够处理排队回调的线程当前被你的代码阻塞，等待操作完成。而操作不会完成，直到回调被处理。死锁！这甚至在上下文不限制并发性为 1 时也适用，但资源以任何方式有限制时。想象一下相同的情况，除了使用最大并发数为 4 的 MaxConcurrencySynchronizationContext。而且不仅仅是对该操作进行一次调用，我们将4个调用排队到该上下文，每个调用都进行调用并阻塞等待它完成。现在，我们仍然阻塞了所有资源，等待异步方法完成，唯一能够允许这些异步方法完成的是它们的回调能够被已经完全占用的这个上下文处理。同样，死锁！如果相反地库方法使用了 ConfigureAwait(false)，它将不会将回调排队到原始上下文，避免了死锁情况。

### 使用`ConfigureAwait(true)`有什么好处吗？

实际上，并没有什么好处，除非你纯粹是用它作为明确表示你故意没有使用`ConfigureAwait(false)`的标志（例如，为了消除静态分析警告等）。`ConfigureAwait(true)`没有任何有意义的作用。在比较`await task`和`await task.ConfigureAwait(true)`时，它们在功能上是相同的。如果在生产代码中看到`ConfigureAwait(true)`，可以安全地删除它而不会产生不良影响。

ConfigureAwait 方法接受一个布尔值，因为有一些特定情况下，你可能希望传递一个变量来控制配置。但 99% 的情况是使用硬编码的 false 参数值，即`ConfigureAwait(false)`。

### 何时应该使用`ConfigureAwait(false)`？

这取决于你是在实现应用程序级别的代码还是通用库代码。

在编写应用程序时，通常希望使用默认行为（这就是为什么它是默认行为的原因）。如果应用程序模型/环境（例如 Windows Forms、WPF、ASP.NET Core 等）发布了自定义的 SynchronizationContext，几乎肯定有一个非常好的原因：它为关心同步上下文的代码提供了一种与应用程序模型/环境适当交互的方式。因此，如果你在 Windows Forms 应用程序中编写事件处理程序，在 xunit 中编写单元测试，在 ASP.NET MVC 控制器中编写代码，无论应用程序模型是否实际发布了 SynchronizationContext，你都希望使用该 SynchronizationContext（如果存在的话）。这意味着默认`ConfigureAwait(true)`。你简单地使用 await，并且在回调/继续方面发生了正确的事情，如果原始上下文存在的话，回调/继续会被发布回原始上下文。这导致了这样的一般指导原则：如果你正在编写应用程序级别的代码，请不要使用`ConfigureAwait(false)`。如果回顾一下本文先前的 Click 事件处理程序代码示例：

```CSharp
private static readonly HttpClient s_httpClient = new HttpClient();

private async void downloadBtn_Click(object sender, RoutedEventArgs e)
{
    string text = await s_httpClient.GetStringAsync("http://example.com/currenttime");
    downloadBtn.Content = text;
}
```

设置 downloadBtn.Context = text 需要在原始上下文（UI 线程）中执行。如果代码违反了这个准则，而是在不应该使用ConfigureAwait(false)的地方使用了它：

```CSharp
private static readonly HttpClient s_httpClient = new HttpClient();

private async void downloadBtn_Click(object sender, RoutedEventArgs e)
{
    string text = await s_httpClient.GetStringAsync("http://example.com/currenttime").ConfigureAwait(false); // bug
    downloadBtn.Content = text;
}
```

上面的 `ConfigureAwait(false)` 会导致问题。同样的情况也适用于依赖于`HttpContext.Current`的经典 ASP.NET 应用程序的代码；使用`ConfigureAwait(false)`，然后尝试使用`HttpContext.Current`可能会导致问题。

相反，通用库是“通用目的”的一部分，因为它们不关心它们被使用的环境。你可以从Web应用程序、客户端应用程序或测试中使用它们，这并不重要，因为库代码对可能被用于的应用程序模型是不可知的。因为它是不可知的，这也意味着它不会执行任何需要以特定方式与应用程序模型进行交互的操作，例如，它不会访问UI控件，因为通用库对UI控件一无所知。因为我们不需要在任何特定的环境中运行代码，我们可以避免强制将继续/回调返回到原始上下文，方法是使用`ConfigureAwait(false)`，并获得它带来的性能和可靠性的好处。这导致了这样的一般指导原则：如果你正在编写通用库代码，请使用`ConfigureAwait(false)`。这就是为什么，例如，在 .NET Core 运行时库中，几乎每个 await 都使用`ConfigureAwait(false)`的原因；在不使用它的情况下，这很可能是一个需要修复的错误。例如，这个PR 修复了 HttpClient 中缺少的 `ConfigureAwait(false)` 调用。

像所有指导一样，当然会有例外情况，有些情况下可能没有意义。例如，在通用库中的一个较大的例外情况（或者至少是需要考虑的类别之一）是当这些库有“带有回调委托的API”。在这种情况下，库的调用者正在传递可能是应用程序级别代码的委托，这实际上使库的这些“通用目的”的假设失去了意义。例如，考虑 LINQ 的 Where 方法的异步版本，例如

```CSharp
public static async IAsyncEnumerable<T> WhereAsync(
    this IAsyncEnumerable<T> source,
    Func<T, bool> predicate);
```

这里的 predicate 是否需要在调用者的原始 SynchronizationContext 上被调用？这取决于 WhereAsync 的实现来决定，并且这是它可能选择不使用`ConfigureAwait(false)`的原因。

即使在这些特殊情况下，一般的指导原则仍然是一个非常好的起点：如果你正在编写通用库/不关心应用程序模型的代码，请使用`ConfigureAwait(false)`，否则不要使用。

### `ConfigureAwait(false)`是否保证回调不会在原始上下文中运行？

不是的。它保证不会将回调排队回原始上下文... 但这并不意味着在等待任务之后的代码（例如`await task.ConfigureAwait(false)`）不会在原始上下文中运行。这是因为对已经完成的可等待对象的等待会在等待后立即同步运行，而不会强制将任何内容排队回去。因此，如果等待的任务在等待时已经完成，无论是否使用了`ConfigureAwait(false)`，此后紧随其后的代码仍将在当前线程中以之前存在的任何上下文中执行。

### 只在方法的第一个 await 上使用`ConfigureAwait(false)`而不在其他地方使用是可以的吗？

一般来说，不行。请参考前面的FAQ。如果`await task.ConfigureAwait(false)`涉及到在等待时已经完成的任务（实际上非常常见），那么`ConfigureAwait(false)`将是无意义的，因为线程将继续在此之后的方法代码中执行，并仍然在之前存在的相同上下文中执行。

一个显著的例外是，如果你知道第一个await将始终异步完成，并且被等待的事物将在不带自定义 SynchronizationContext 或 TaskScheduler 的环境中调用其回调。例如，.NET运行时库中的 CryptoStream 希望确保其潜在的计算密集型代码不作为调用者的同步调用的一部分运行，因此它使用了一个自定义的 awaiter 来确保第一个 await 之后的所有内容都在线程池线程上运行。然而，即使在这种情况下，你会注意到接下来的 await 仍然使用`ConfigureAwait(false)`；从技术上讲，这是不必要的，但这样做会使代码审查变得更容易，否则每次查看此代码时都需要分析为什么省略了`ConfigureAwait(false)`。

### 可以使用`Task.Run`来避免使用`ConfigureAwait(false)`吗？

可以。如果你这样写：

```CSharp
Task.Run(async delegate
{
    await SomethingAsync(); // won't see the original context
});
```

如果在 SomethingAsync() 调用上使用了`ConfigureAwait(false)`，那么这个调用将无意义，因为传递给`Task.Run`的委托将在一个线程池线程上执行，没有更高层次的用户代码，因此`SynchronizationContext.Current`将返回`null`。此外，`Task.Run`隐式使用`TaskScheduler.Default`，这意味着在委托内部查询`TaskScheduler.Current`也将返回`Default`。这意味着无论是否使用了`ConfigureAwait(false)`，await 的行为都将是相同的。它还不对此 lambda 内部的代码可能做什么做出任何保证。如果你有以下代码：

```CSharp
Task.Run(async delegate
{
    SynchronizationContext.SetSynchronizationContext(new SomeCoolSyncCtx());
    await SomethingAsync(); // will target SomeCoolSyncCtx
});
```

那么 SomethingAsync 内部的代码实际上会将`SynchronizationContext.Current`视为 SomeCoolSyncCtx 实例，而 SomethingAsync 内部的所有非配置的 await 都会将回调/继续发布回该上下文。因此，要使用这种方法，你需要了解你正在排队的所有代码可能做什么，以及它的行为是否可能破坏你的行为。

这种方法还需要创建/排队额外的任务对象，这可能会根据你的应用程序或库的性能敏感性而有所影响。

此外，请记住，这种技巧可能会带来比它们所值得的更多问题，并且可能导致其他意外后果。例如，一些静态分析工具（例如Roslyn分析器）已被编写成标记不使用`ConfigureAwait(false)`的 await，比如CA2007。如果你启用了这样的分析器，然后采用了类似的技巧来避免使用 ConfigureAwait，那么该分析器有很大机会标记它，并实际上为你带来更多的工作。因此，你可能会因为其喧扰性而禁用分析器，现在你可能会错过代码库中其他地方，其中实际上应该使用`ConfigureAwait(false)`。

### 能用`SynchronizationContext.SetSynchronizationContext`来避免使用`ConfigureAwait(false)`吗?

不一定，这取决于涉及到的代码。

有些程序员编写这样的代码：

```CSharp
Task t;
SynchronizationContext old = SynchronizationContext.Current;
SynchronizationContext.SetSynchronizationContext(null);
try
{
    t = CallCodeThatUsesAwaitAsync(); // awaits in here won't see the original context
}
finally { SynchronizationContext.SetSynchronizationContext(old); }
await t; // will still target the original context
```

希望这样的代码能够使 CallCodeThatUsesAwaitAsync 内部的代码将当前上下文视为 null。确实会这样。然而，如果 await 看到的是 `TaskScheduler.Current`，则上述代码对这个 await 不起作用。因此如果此代码在某个自定义 TaskScheduler 上运行，CallCodeThatUsesAwaitAsync 内部的 await（且未使用`ConfigureAwait(false)`）仍然会看到并排队回到该自定义 TaskScheduler。

与前一个与`Task.Run`相关的 FAQ 中相同的注意事项也适用：这样的解决方法存在性能问题，并且 try 块内部的代码也可能通过设置不同的上下文（或者使用非默认的 TaskScheduler 调用代码）来挫败这些尝试。

在这样的模式中，你还需要注意一种轻微的变化：

```CSharp
SynchronizationContext old = SynchronizationContext.Current;
SynchronizationContext.SetSynchronizationContext(null);
try
{
    await t;
}
finally { SynchronizationContext.SetSynchronizationContext(old); }
```

看到问题了吗？这有点难以察觉，但也可能产生很大的影响。不能保证`await`最终会在原始线程上调用回调/继续，这意味着将`SynchronizationContext`重置回原始上下文的操作实际上可能不会在原始线程上发生，这可能导致该线程上的后续工作项看到错误的上下文（为了抵消这一点，良好编写的应用程序模型通常会在调用任何进一步的用户代码之前手动重置上下文）。即使它确实在同一线程上运行，也可能要等一段时间，因此上下文不会在一段时间内适当地恢复。如果在不同的线程上运行，它可能会将错误的上下文设置到该线程上。依此类推。离理想还有很大的差距。

### 我正在使用`GetAwaiter().GetResult()`。我需要使用`ConfigureAwait(false)`吗？

不需要。`ConfigureAwait`只影响回调。具体来说，`awaiter`模式要求`awaiter`公开`IsCompleted`属性、`GetResult`方法和`OnCompleted`方法（可选包括`UnsafeOnCompleted`方法）。`ConfigureAwait`只影响`{Unsafe}OnCompleted`的行为，因此如果你只是直接调用`awaiter`的`GetResult()`方法，无论是在`TaskAwaiter`上还是在`ConfiguredTaskAwaitable.ConfiguredTaskAwaiter`上做，都不会有任何行为差异。因此，如果你在代码中看到`task.ConfigureAwait(false).GetAwaiter().GetResult()`，你可以将其替换为`task.GetAwaiter().GetResult()`（并考虑是否真的想要阻塞）。

### 我知道我运行在永远不会有自定义`SynchronizationContext`或自定义`TaskScheduler`的环境中。我可以跳过使用`ConfigureAwait(false)`吗？

也许。这取决于你对“永远不会”部分的确信程度。正如前面的FAQ中所提到的，仅因为你正在工作的应用程序模型没有设置自定义`SynchronizationContext`并且不在自定义`TaskScheduler`上调用你的代码，不意味着其他用户或库代码没有这样做。因此，你需要确保这不是情况，或者至少认识到如果可能是这样的话存在的风险。

### 听说在.NET Core中不再需要使用ConfigureAwait(false)。是真的吗？

不是的。在.NET Core上运行时，出于与在.NET Framework上运行时相同的原因，它仍然是必需的。在这方面没有发生任何变化。

然而，发生变化的是某些环境是否发布它们自己的SynchronizationContext。特别是，与在.NET Framework上的经典ASP.NET有自己的SynchronizationContext相反，在ASP.NET Core中没有。这意味着在ASP.NET Core应用程序中默认情况下运行的代码将看不到自定义的SynchronizationContext，这减少了在这种环境中运行时需要使用ConfigureAwait(false)的情况。

然而，这并不意味着永远不会存在自定义的SynchronizationContext或TaskScheduler。如果某些用户代码（或您的应用程序使用的其他库代码）设置了自定义上下文并调用您的代码，或者在计划到自定义TaskScheduler的Task中调用您的代码，那么即使在ASP.NET Core中，您的await可能会看到一个非默认的上下文或调度程序，这将导致您希望使用ConfigureAwait(false)。当然，在这种情况下，如果您避免同步阻塞（无论如何，您都应该在Web应用程序中避免这样做），并且如果您不介意在这种有限的情况下存在的小的性能开销，您可能可以不使用ConfigureAwait(false)。

### 在await foreach遍历IAsyncEnumerable时，我能使用ConfigureAwait吗？

可以。请参阅此MSDN杂志文章以获取示例。

await foreach绑定到一种模式，因此它不仅可以用于枚举IAsyncEnumerable&lt;T&gt;，还可以用于枚举暴露正确API表面的内容。 .NET运行时库包括在IAsyncEnumerable&lt;T&gt;上的ConfigureAwait扩展方法，它返回一个包装IAsyncEnumerable&lt;T&gt;和一个布尔值的自定义类型，并暴露正确的模式。当编译器生成对枚举器的MoveNextAsync和DisposeAsync方法的调用时，这些调用是对返回的配置枚举器结构类型的调用，它依次以期望的配置方式执行await。

### 在await using时，我能使用ConfigureAwait吗？

可以，尽管有一个小小的复杂性。

与前一个FAQ中描述的IAsyncEnumerable&lt;T>一样，.NET运行时库在IAsyncDisposable上暴露了一个ConfigureAwait扩展方法，await using将愉快地与之一起工作，因为它实现了适当的模式（即暴露适当的DisposeAsync方法）：

```CSharp
await using (var c = new MyAsyncDisposableClass().ConfigureAwait(false))
{
    ...
}
```

这里的问题是`c`的类型现在不是MyAsyncDisposableClass，而是`System.Runtime.CompilerServices.ConfiguredAsyncDisposable`，这是从IAsyncDisposable上的ConfigureAwait扩展方法返回的类型。

要解决这个问题，你需要多写一行代码：

```CSharp
var c = new MyAsyncDisposableClass();
await using (c.ConfigureAwait(false))
{
    ...
}
```

现在`c`的类型是 MyAsyncDisposableClass 了.

当然，这样的写法扩大了`c`的作用范围，如果不爽的话，可以再在外面套一个大括号。

### 我使用了ConfigureAwait(false)，但我的AsyncLocal仍然在await之后的代码中传递。这是一个错误吗？

不，这是预期的行为。AsyncLocal&lt;T> 数据作为ExecutionContext 的一部分流动，这与SynchronizationContext 是分开的。除非你使用了`ExecutionContext.SuppressFlow()`明确禁用了ExecutionContext 的流动，否则ExecutionContext（因此AsyncLocal&lt;T> 数据）将始终在await之间流动，而不管是否使用ConfigureAwait 来避免捕获原始SynchronizationContext。有关更多信息，请参阅[这篇博文](https://devblogs.microsoft.com/pfxteam/executioncontext-vs-synchronizationcontext/)。

### 是否能从语言层面帮助我避免显式使用ConfigureAwait(false)？

有时库开发人员会表达对需要使用`ConfigureAwait(false)`的不满，并寻求更少侵入性的替代方案。

目前，至少在语言/编译器/运行时中没有任何内置的替代方案。但是有许多提案，探讨了这样一个解决方案可能是什么样子，例如

https://github.com/dotnet/csharplang/issues/645

https://github.com/dotnet/csharplang/issues/2542

https://github.com/dotnet/csharplang/issues/2649

https://github.com/dotnet/csharplang/issues/2746

如果这对你很重要，或者如果你觉得在这方面有新的有趣的想法，我鼓励你将你的想法贡献给这些或新的讨论。

----

（翻译：YHB 2024.1.）