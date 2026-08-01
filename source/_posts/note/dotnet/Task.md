---
title: Task开始深入理解.NET异步编程
date: 2026-05-24 22:41:32
tags:
- 笔记
categories:
- [笔记,.NET]
---


# Intro

`Task` 是 .NET 异步编程的基础。

编写一个 `async` 方法时，编译器会将其转化为一个实现 `IAsyncStateMachine` 接口的结构体，但 `Task` 本身是一个 Class（引用类型）

- **分配机制**：每次调用异步方法，系统都会在堆（Heap）上实例化一个 `Task` 对象来追踪状态。如果异步方法频繁调用（如每帧执行），会造成明显的 GC（垃圾回收）压力
    
- **调度核心**：它依赖 `SynchronizationContext` 或 `TaskScheduler`。在默认情况下，它倾向于在线程池中运行
    
- **Promise 模型**：`Task` 就像一个承诺，它内部维护了结果、异常信息和回调列表。即使任务还没完成，你拿到的也是这个对象的引用



大致可理解为
```cs
await work();  
Console.WriteLine("Main 4");
```

```cs
var t = work();

if(!t.IsCompleted)
{
    保存状态;
    
    t.ContinueWith(_ =>
    {
        Console.WriteLine("Main 4");
    });//ContinueWith 默认线程池调度
    
    return;
}
```

| 情况                          | continuation位置 |
| --------------------------- | -------------- |
| WPF await                   | UI线程           |
| await ConfigureAwait(false) | 线程池            |
| Console App                 | 任意线程池线程        |
| ASP.NET Core                | 任意线程池线程        |
| Task 已完成                    | 同步直接继续         |
| Task.Run                    | 明确线程池          |

---
# Async

当你调用一个 `async` 方法并 `await` 一个 `Task` 时：

1. **编译器**将代码包装进 `IAsyncStateMachine`
    
2. **方法运行**到 `await` 处，状态机检查 `Task` 是否已完成
    
3. 若未完成，状态机捕获当前的 **`SynchronizationContext`** (或 `TaskScheduler`) 并挂起
    
4. **`Task`** 在后台（线程池或 I/O）异步运行
    
5. **`Task`** 完成后，通过之前捕获的上下文，将状态机的 `MoveNext()` 投递回原始线程
    
6. **状态机**恢复，从字段中取回变量值，继续执行后续代码

---

## IAsyncStateMachine (异步状态机)

当你使用 `async` 关键字时，编译器会将你的方法重写为一个实现了 `IAsyncStateMachine` 的**结构体**。

> Debug下为方便调试时class，Release一般为struct

- **字段保存**：方法内部的所有局部变量都会变成该结构体的字段，从而实现跨 `await` 的状态保持。

### 核心成员

- **`int <>1__state`**：核心字段。记录当前代码运行到了哪一个 `await`。`-1` 代表正在运行，`0, 1, 2...` 代表在特定的 `await` 处挂起，`-2` 代表结束
    
- **`AsyncMethodBuilder <>t__builder`**：辅助生成 Task 的工具人。
    
- **`MoveNext()`**：驱动状态机向后走的唯一动力源。
    
- **`SetStateMachine(IAsyncStateMachine stateMachine)`**：用于在某些特殊情况下（如堆栈重映射）关联状态机实例。

---

## AsyncMethodBuilder (异步方法构建器)

**角色：** 状态机与 Task 之间的“粘合剂”。 针对不同的返回类型（`Task`, `Task<T>`, `ValueTask`, `void`），会有不同的构建器（如 `AsyncTaskMethodBuilder`）。

### 核心成员

- **`Create()`**：静态方法，创建一个构建器实例。
    
- **`Start<TStateMachine>(ref TStateMachine stateMachine)`**：**启动**状态机。它会立即调用第一次 `MoveNext()`。
    
- **`SetResult(T result)` / `SetException(Exception exception)`**：当状态机运行完毕或报错时，通过这个方法来标记关联的 `Task` 为“已完成”或“失败”。
    
- **`AwaitOnCompleted` / `AwaitUnsafeOnCompleted`**：当代码遇到没完成的 `await` 时，状态机会调用这个方法。它负责把“恢复运行”的动作挂载到 Task 的回调列表里。

---

## TaskAwaiter (任务等待器)

**角色：** 协议适配器。 `await` 关键字背后并不直接操作 `Task`，而是操作一个符合 **Awaiter 模式** 的对象。

C# 的 `await` 并不是硬编码给 `Task` 专用的。只要一个类型满足以下条件，它就可以被 `await`：

1. 有一个 `GetAwaiter()` 方法。
    
2. 该方法返回的对象实现了 `INotifyCompletion` 或 `ICriticalNotifyCompletion` 接口。
    
3. 该对象具有特定的成员（`IsCompleted`, `OnCompleted`, `GetResult`）。
    

`TaskAwaiter` 就是 `Task` 类对应的 Awaiter 实现。

### 核心成员

#### ① IsCompleted (bool 属性)

- **作用：** 状态机在进入 `await` 逻辑时，首先检查这个属性。
    
- **逻辑：**
    
    - 如果为 `true`：说明任务已经完成（或者是同步完成的）。状态机**不会挂起**，直接同步执行后续代码。这是极其重要的性能优化。
        
    - 如果为 `false`：状态机准备挂起。
        

#### ② OnCompleted(Action continuation)

- **作用：** 注册回调。
    
- **逻辑：** 当 `IsCompleted` 为 `false` 时，状态机会把它的 `MoveNext()` 方法包装成一个 `Action` 传给这个方法。`TaskAwaiter` 负责把这个回调挂载到 `Task` 的内部列表中。当 `Task` 完成时，这个回调被触发，状态机恢复。
    

#### ③ GetResult()

- **作用：** 获取结果并处理异常。
    
- **逻辑：**
    
    - 它是 `await` 表达式最后一步调用的方法。
        
    - 如果 `Task` 成功，它返回结果（如果是 `Task<T>` 则返回 `T`）。
        
    - 如果 `Task` 失败，它会重新抛出（Rethrow）第一个异常，并保持原始的堆栈追踪。这也就是为什么你可以用 `try-catch` 捕获异步异常的原因。

---

我们可以把 `MoveNext()` 的调用简化为以下示意图：

| **触发点**        | **触发者**                  | **发生时机**                                    |
| -------------- | ------------------------ | ------------------------------------------- |
| **第一次调用**      | `AsyncMethodBuilder`     | `async` 方法被调用的瞬间（同步执行部分）。                   |
| **任务完成恢复**     | `TaskAwaiter`            | 异步操作结束，触发注册的回调。                             |
| **跨线程恢复**      | `SynchronizationContext` | 任务完成后，被重新投递到主线程队列。                          |
| **UniTask 驱动** | `PlayerLoop`             | 在 Unity 中，UniTask 可能会在特定的帧生命周期（如 Update）触发。 |

## 样例

#### 开发端代码

```cs
public class TestClass {
    public int Bv{get;set;}
    public async void Main() {
        int x=0;
        await workAsync();
        Bv++;
        await workAsync();
        Bv++;
        x++;
        await workAsync();
        await workAsync();
        Console.WriteLine(Bv);
        Console.WriteLine(x);
    }
    

    public async Task workAsync(){
        int av=0;
        if(Bv>2)
            av++;
        Bv=av;
        await Task.Delay(300);
        
    }
}
```

#### 编译器生成代码
##### 精简版

```cs
public class TestClass
{
    [CompilerGenerated]
    private sealed class <Main>d__4 : IAsyncStateMachine
    {
        public int <>1__state;

        public AsyncVoidMethodBuilder <>t__builder;

        public TestClass <>4__this;

        private int <x>5__1;

        private TaskAwaiter <>u__1;

        private void MoveNext()
        {
            int num = <>1__state;
            try
            {
                TaskAwaiter awaiter4;
                TaskAwaiter awaiter3;
                TaskAwaiter awaiter2;
                TaskAwaiter awaiter;
                int bv;
                switch (num)
                {
                    default:
                        <x>5__1 = 0;
                        awaiter4 = <>4__this.workAsync().GetAwaiter();
                        if (!awaiter4.IsCompleted)
                        {
                            num = (<>1__state = 0);
                            <>u__1 = awaiter4;
                            <Main>d__4 stateMachine = this;
                            <>t__builder.AwaitUnsafeOnCompleted(ref awaiter4, ref stateMachine);
                            return;
                        }
                        goto IL_0095;
                    case 0:
                        awaiter4 = <>u__1;
                        <>u__1 = default(TaskAwaiter);
                        num = (<>1__state = -1);
                        goto IL_0095;
                    case 1:
                        awaiter3 = <>u__1;
                        <>u__1 = default(TaskAwaiter);
                        num = (<>1__state = -1);
                        goto IL_0118;
                    case 2:
                        awaiter2 = <>u__1;
                        <>u__1 = default(TaskAwaiter);
                        num = (<>1__state = -1);
                        goto IL_01ab;
                    case 3:
                        {
                            awaiter = <>u__1;
                            <>u__1 = default(TaskAwaiter);
                            num = (<>1__state = -1);
                            break;
                        }
                        IL_0118:
	                        awaiter3.GetResult();
	                        bv = <>4__this.Bv;
	                        <>4__this.Bv = bv + 1;
	                        <x>5__1++;
	                        awaiter2 = <>4__this.workAsync().GetAwaiter();
	                        if (!awaiter2.IsCompleted)
	                        {
	                            num = (<>1__state = 2);
	                            <>u__1 = awaiter2;
	                            <Main>d__4 stateMachine = this;
	                            <>t__builder.AwaitUnsafeOnCompleted(ref awaiter2, ref stateMachine);
	                            return;
	                        }
	                        goto IL_01ab;
                        IL_0095:
	                        awaiter4.GetResult();
	                        bv = <>4__this.Bv;
	                        <>4__this.Bv = bv + 1;
	                        awaiter3 = <>4__this.workAsync().GetAwaiter();
	                        if (!awaiter3.IsCompleted)
	                        {
	                            num = (<>1__state = 1);
	                            <>u__1 = awaiter3;
	                            <Main>d__4 stateMachine = this;
	                            <>t__builder.AwaitUnsafeOnCompleted(ref awaiter3, ref stateMachine);
	                            return;
	                        }
	                        goto IL_0118;
                        IL_01ab:
	                        awaiter2.GetResult();
	                        awaiter = <>4__this.workAsync().GetAwaiter();
	                        if (!awaiter.IsCompleted)
	                        {
	                            num = (<>1__state = 3);
	                            <>u__1 = awaiter;
	                            <Main>d__4 stateMachine = this;
	                            <>t__builder.AwaitUnsafeOnCompleted(ref awaiter, ref stateMachine);
	                            return;
	                        }
	                        break;
	            }
                awaiter.GetResult();
                Console.WriteLine(<>4__this.Bv);
                Console.WriteLine(<x>5__1);
            }
            catch (Exception exception)
            {
                <>1__state = -2;
                <>t__builder.SetException(exception);
                return;
            }
            <>1__state = -2;
            <>t__builder.SetResult();
        }



        [DebuggerHidden]
        private void SetStateMachine([Nullable(1)] IAsyncStateMachine stateMachine)
        {
        }

    }


    [CompilerGenerated]
    private sealed class <workAsync>d__5 : IAsyncStateMachine
    {
        public int <>1__state;

        public AsyncTaskMethodBuilder <>t__builder;

        public TestClass <>4__this;

        private int <av>5__1;

        private TaskAwaiter <>u__1;

        private void MoveNext()
        {
            int num = <>1__state;
            try
            {
                TaskAwaiter awaiter;
                if (num != 0)
                {
                    <av>5__1 = 0;
                    if (<>4__this.Bv > 2)
                    {
                        <av>5__1++;
                    }
                    <>4__this.Bv = <av>5__1;
                    awaiter = Task.Delay(300).GetAwaiter();
                    if (!awaiter.IsCompleted)
                    {
                        num = (<>1__state = 0);
                        <>u__1 = awaiter;
                        <workAsync>d__5 stateMachine = this;
                        <>t__builder.AwaitUnsafeOnCompleted(ref awaiter, ref stateMachine);
                        return;
                    }
                }
                else
                {
                    awaiter = <>u__1;
                    <>u__1 = default(TaskAwaiter);
                    num = (<>1__state = -1);
                }
                awaiter.GetResult();
            }
            catch (Exception exception)
            {
                <>1__state = -2;
                <>t__builder.SetException(exception);
                return;
            }
            <>1__state = -2;
            <>t__builder.SetResult();
        }
        

        [DebuggerHidden]
        private void SetStateMachine([Nullable(1)] IAsyncStateMachine stateMachine)
        {
        }

    }

    [CompilerGenerated]
    [DebuggerBrowsable(DebuggerBrowsableState.Never)]
    private int <Bv>k__BackingField;

    public int Bv
    {
        [CompilerGenerated]
        get
        {
            return <Bv>k__BackingField;
        }
        [CompilerGenerated]
        set
        {
            <Bv>k__BackingField = value;
        }
    }

    [AsyncStateMachine(typeof(<Main>d__4))]
    [DebuggerStepThrough]
    public void Main()
    {
        <Main>d__4 stateMachine = new <Main>d__4();
        stateMachine.<>t__builder = AsyncVoidMethodBuilder.Create();
        stateMachine.<>4__this = this;
        stateMachine.<>1__state = -1;
        stateMachine.<>t__builder.Start(ref stateMachine);
    }

    [NullableContext(1)]
    [AsyncStateMachine(typeof(<workAsync>d__5))]
    [DebuggerStepThrough]
    public Task workAsync()
    {
        <workAsync>d__5 stateMachine = new <workAsync>d__5();
        stateMachine.<>t__builder = AsyncTaskMethodBuilder.Create();
        stateMachine.<>4__this = this;
        stateMachine.<>1__state = -1;
        stateMachine.<>t__builder.Start(ref stateMachine);
        return stateMachine.<>t__builder.Task;
    }
}
```
##### 完整版

```cs

[assembly: CompilationRelaxations(8)]
[assembly: RuntimeCompatibility(WrapNonExceptionThrows = true)]
[assembly: Debuggable(DebuggableAttribute.DebuggingModes.Default | DebuggableAttribute.DebuggingModes.IgnoreSymbolStoreSequencePoints | DebuggableAttribute.DebuggingModes.EnableEditAndContinue | DebuggableAttribute.DebuggingModes.DisableOptimizations)]
[assembly: SecurityPermission(SecurityAction.RequestMinimum, SkipVerification = true)]
[assembly: AssemblyVersion("0.0.0.0")]
[module: UnverifiableCode]
[module: RefSafetyRules(11)]

public class TestClass
{
    [CompilerGenerated]
    private sealed class <Main>d__4 : IAsyncStateMachine
    {
        public int <>1__state;

        public AsyncVoidMethodBuilder <>t__builder;

        public TestClass <>4__this;

        private int <x>5__1;

        private TaskAwaiter <>u__1;

        private void MoveNext()
        {
            int num = <>1__state;
            try
            {
                TaskAwaiter awaiter4;
                TaskAwaiter awaiter3;
                TaskAwaiter awaiter2;
                TaskAwaiter awaiter;
                int bv;
                switch (num)
                {
                    default:
                        <x>5__1 = 0;
                        awaiter4 = <>4__this.workAsync().GetAwaiter();
                        if (!awaiter4.IsCompleted)
                        {
                            num = (<>1__state = 0);
                            <>u__1 = awaiter4;
                            <Main>d__4 stateMachine = this;
                            <>t__builder.AwaitUnsafeOnCompleted(ref awaiter4, ref stateMachine);
                            return;
                        }
                        goto IL_0095;
                    case 0:
                        awaiter4 = <>u__1;
                        <>u__1 = default(TaskAwaiter);
                        num = (<>1__state = -1);
                        goto IL_0095;
                    case 1:
                        awaiter3 = <>u__1;
                        <>u__1 = default(TaskAwaiter);
                        num = (<>1__state = -1);
                        goto IL_0118;
                    case 2:
                        awaiter2 = <>u__1;
                        <>u__1 = default(TaskAwaiter);
                        num = (<>1__state = -1);
                        goto IL_01ab;
                    case 3:
                        {
                            awaiter = <>u__1;
                            <>u__1 = default(TaskAwaiter);
                            num = (<>1__state = -1);
                            break;
                        }
                        IL_0118:
                        awaiter3.GetResult();
                        bv = <>4__this.Bv;
                        <>4__this.Bv = bv + 1;
                        <x>5__1++;
                        awaiter2 = <>4__this.workAsync().GetAwaiter();
                        if (!awaiter2.IsCompleted)
                        {
                            num = (<>1__state = 2);
                            <>u__1 = awaiter2;
                            <Main>d__4 stateMachine = this;
                            <>t__builder.AwaitUnsafeOnCompleted(ref awaiter2, ref stateMachine);
                            return;
                        }
                        goto IL_01ab;
                        IL_0095:
                        awaiter4.GetResult();
                        bv = <>4__this.Bv;
                        <>4__this.Bv = bv + 1;
                        awaiter3 = <>4__this.workAsync().GetAwaiter();
                        if (!awaiter3.IsCompleted)
                        {
                            num = (<>1__state = 1);
                            <>u__1 = awaiter3;
                            <Main>d__4 stateMachine = this;
                            <>t__builder.AwaitUnsafeOnCompleted(ref awaiter3, ref stateMachine);
                            return;
                        }
                        goto IL_0118;
                        IL_01ab:
                        awaiter2.GetResult();
                        awaiter = <>4__this.workAsync().GetAwaiter();
                        if (!awaiter.IsCompleted)
                        {
                            num = (<>1__state = 3);
                            <>u__1 = awaiter;
                            <Main>d__4 stateMachine = this;
                            <>t__builder.AwaitUnsafeOnCompleted(ref awaiter, ref stateMachine);
                            return;
                        }
                        break;
                }
                awaiter.GetResult();
                Console.WriteLine(<>4__this.Bv);
                Console.WriteLine(<x>5__1);
            }
            catch (Exception exception)
            {
                <>1__state = -2;
                <>t__builder.SetException(exception);
                return;
            }
            <>1__state = -2;
            <>t__builder.SetResult();
        }

        void IAsyncStateMachine.MoveNext()
        {
            //ILSpy generated this explicit interface implementation from .override directive in MoveNext
            this.MoveNext();
        }

        [DebuggerHidden]
        private void SetStateMachine([Nullable(1)] IAsyncStateMachine stateMachine)
        {
        }

        void IAsyncStateMachine.SetStateMachine([Nullable(1)] IAsyncStateMachine stateMachine)
        {
            //ILSpy generated this explicit interface implementation from .override directive in SetStateMachine
            this.SetStateMachine(stateMachine);
        }
    }


    [CompilerGenerated]
    private sealed class <workAsync>d__5 : IAsyncStateMachine
    {
        public int <>1__state;

        public AsyncTaskMethodBuilder <>t__builder;

        public TestClass <>4__this;

        private int <av>5__1;

        private TaskAwaiter <>u__1;

        private void MoveNext()
        {
            int num = <>1__state;
            try
            {
                TaskAwaiter awaiter;
                if (num != 0)
                {
                    <av>5__1 = 0;
                    if (<>4__this.Bv > 2)
                    {
                        <av>5__1++;
                    }
                    <>4__this.Bv = <av>5__1;
                    awaiter = Task.Delay(300).GetAwaiter();
                    if (!awaiter.IsCompleted)
                    {
                        num = (<>1__state = 0);
                        <>u__1 = awaiter;
                        <workAsync>d__5 stateMachine = this;
                        <>t__builder.AwaitUnsafeOnCompleted(ref awaiter, ref stateMachine);
                        return;
                    }
                }
                else
                {
                    awaiter = <>u__1;
                    <>u__1 = default(TaskAwaiter);
                    num = (<>1__state = -1);
                }
                awaiter.GetResult();
            }
            catch (Exception exception)
            {
                <>1__state = -2;
                <>t__builder.SetException(exception);
                return;
            }
            <>1__state = -2;
            <>t__builder.SetResult();
        }

        void IAsyncStateMachine.MoveNext()
        {
            //ILSpy generated this explicit interface implementation from .override directive in MoveNext
            this.MoveNext();
        }

        [DebuggerHidden]
        private void SetStateMachine([Nullable(1)] IAsyncStateMachine stateMachine)
        {
        }

        void IAsyncStateMachine.SetStateMachine([Nullable(1)] IAsyncStateMachine stateMachine)
        {
            //ILSpy generated this explicit interface implementation from .override directive in SetStateMachine
            this.SetStateMachine(stateMachine);
        }
    }

    [CompilerGenerated]
    [DebuggerBrowsable(DebuggerBrowsableState.Never)]
    private int <Bv>k__BackingField;

    public int Bv
    {
        [CompilerGenerated]
        get
        {
            return <Bv>k__BackingField;
        }
        [CompilerGenerated]
        set
        {
            <Bv>k__BackingField = value;
        }
    }

    [AsyncStateMachine(typeof(<Main>d__4))]
    [DebuggerStepThrough]
    public void Main()
    {
        <Main>d__4 stateMachine = new <Main>d__4();
        stateMachine.<>t__builder = AsyncVoidMethodBuilder.Create();
        stateMachine.<>4__this = this;
        stateMachine.<>1__state = -1;
        stateMachine.<>t__builder.Start(ref stateMachine);
    }

    [NullableContext(1)]
    [AsyncStateMachine(typeof(<workAsync>d__5))]
    [DebuggerStepThrough]
    public Task workAsync()
    {
        <workAsync>d__5 stateMachine = new <workAsync>d__5();
        stateMachine.<>t__builder = AsyncTaskMethodBuilder.Create();
        stateMachine.<>4__this = this;
        stateMachine.<>1__state = -1;
        stateMachine.<>t__builder.Start(ref stateMachine);
        return stateMachine.<>t__builder.Task;
    }
}

```

# 其他

## SynchronizationContext (同步上下文)

### 解决的问题

异步任务往往在非 UI 线程完成，但 UI 元素的修改必须在主线程。**同步上下文解决了“如何跨线程安全地通信”以及“代码恢复时应该回到哪个线程”的问题。**

### 实现方式

它是一个抽象类，提供了一个 `Post` 方法（异步发送）和 `Send` 方法（同步发送）。

- **线程绑定**：不同的平台有不同的实现。例如，在 Unity 中是 `UnitySynchronizationContext`，在 WinForms 中是 `WindowsFormsSynchronizationContext`。
    
- **传递性**：`await` 默认会捕获当前的 `SynchronizationContext`。当异步任务完成后，它会通过 `context.Post(...)` 把后续逻辑“排队”回原始线程（比如 UI 主线程）执行。


## TaskScheduler (任务调度器)

### 解决的问题

并不是所有的异步逻辑都需要回到特定的线程（如 UI 线程）。有些纯计算任务只需要找个空闲的 CPU 核心运行即可。**任务调度器解决了“任务（Task）如何映射到线程池（ThreadPool）资源”的问题。**

### 实现方式

它负责处理 `Task` 对象的队列排队和执行。

- **默认调度器 (ThreadPoolTaskScheduler)**：使用 .NET 线程池。它采用“工作窃取（Work-Stealing）”算法来平衡多个 CPU 核心的负载。
    
- **层级关系**：如果在 `await` 时没有 `SynchronizationContext`（或者设置了 `ConfigureAwait(false)`），系统就会求助于 `TaskScheduler.Current` 来决定后续逻辑在哪运行。


