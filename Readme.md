1. [Getting Started](#getting-started)
   1. [Introduction and Concepts](#introduction-and-concepts)
      1. [Join And Sleep](#joind-and-sleep)
      2. [How Threading works](#how-threading-works)
      3. [Threads vs Processes](#threads-vs-processes)
   2. [Creating and Starting Threads](#creating-and-starting-threads)
      1. [Passing data to a Thread](#passing-data-to-a-thread)
      2. [Naming Threads](#naming-threads)
      3. [Foreground and Background Threads](#foreground-and-background-threads)
      4. [Thread Priority](#thread-priority)
      5. [Exception Handling](#exception-handling)
   3. [Thread Pooling](#thread-pooling)
      1. [Enter the Thread Pool](#enter-the-thread-pool)
      2. [Optimizing the Thread Pool](#optimizing-the-thread-pool)

# Getting Started
## Introduction and Concepts
A thread is an independant execution path, able to run simulateously with other threads.

A C# client program (Console, WPF or Windows Forms) starts in a single thread created automatically, called the "main thread" and is made multithreading by creatin additonal threads.

Once started, a thread is alive until the delegate associated finishes executing. Once ended, a thread cannot restart.

For each thread, the CLR assigns a memory stack so local variables are kept separate. 

Threads can share the same data:
- if they have a common reference on it.
- if the data is static

### Join and Sleep
Join waits the thread to end. A timeout can be included.
Sleep pauses the current thread for a specified period.

### How Threading works
Multithreading is managed internally by a thread scheduler, a function the CLR typically delegates to the operating system. A thread scheduler ensures all active threads are allocated appropriate execution time, and that threads that are waiting or blocked (for instance, on an exclusive lock or on user input) do not consume CPU time.

On a single-processor computer, a thread scheduler performs time-slicing — rapidly switching execution between each of the active threads.

On a multi-processor computer, multithreading is implemented with both time-slicing (operating system’s need to service its own threads) and concurrency, where different threads run code simultaneously on different CPUs. 

A thread is said to be preempted when its execution is interrupted due to an external factor such as time-slicing. In most situations, a thread has no control over when and where it’s preempted.

### Threads vs Processes
Processes run in parallel on a computer whereas Threads run in parallel within a single process.

Processes are fully isolated from each other whereas Threads are a bit isolated. They share (heap) memory with other threads running in the same application. 

Threading also incurs a resource and CPU cost in scheduling and switching threads (when there are more active threads than CPU cores) — and there’s also a creation/tear-down cost. Multithreading will not always speed up your application — it can even slow it down if used excessively or inappropriately.

## Creating and Starting Threads
Threads are created by creating an instance of Thread and passing the method to execute.
```C#
var thread = new Thread(Start);
thread.Start();

public void Start() {
    // Code executed in a thread
}
```

### Passing data to a Thread
To pass data to a thread, you can:
- Use a lambda expression when you create the thread. No limit of data you can pass. You can even wrap the entire implementation in a multi-statement lambda (() => {...})
```C#
var thread = new Thread(() => Start("I'm the first parameter", "I'm the second parameter"));
thread.Start();

public void Start(string first, stirng second) {
    // Code executed in a thread
}
```

If a variable is used as a parameter for several threads, you can have a nondeterministic result because this variable is shared.
```C#
for (var i = 0; i < 10; i++) 
{
    new Thread(() => Console.Write(i)).Start(); // output: 0223557799
}
```

- Use anonymous methods
```C#
var thread = new Thread(delegate()
{
    // Code executed in a thread
});
thread.Start();
```
- Use Thread's start method
```C#
var thread = new Thread(Start);
thread.Start("I'm a parameter");

public void Start(object parameter) {
    // Only one parameter is possible and needs to be cast
    // Code executed in a thread
}
```

### Naming Threads
Each thread has a Name property that you can set for the benefit of debugging.
```C#
static void Main()
{
    Thread.CurrentThread.Name = "main";

    Thread worker = new Thread (Go);
    worker.Name = "worker";
    worker.Start();
}
```

### Foreground and Background Threads
Foreground threads keep the application alive for as long as any one of them is running. The application ends only after all foreground threads have ended. By default, threads you create explicitly are foreground threads.

Background threads dont't keep the application alive. If the application ends all the background threads running are abruptly terminated, so if you have some finally block (or using block), they won't be executed.

You can query or change a thread’s background status using its IsBackground property.

### Thread Priority
A Thread's priority determines how much execution time it gets relative to other active threads in the operating system, on the following scale
enum ThreadPriority { Lowest, BelowNormal, Normal, AboveNormal, Highest }

Warning: elevating a Thread's priority can lead problems such as resource starvation for other threads.

If you want to perform real-time work, elevating the Thread's prioriy to Highest is not enough, you need to elevate the Process's priority to High. 

If you change it to Realtime, you instruct the OS that you never want the process to yield CPU time to another process. So if your program enters in an accidental infinite loop, the operating system will be locked out and the only solution you will have it is force the shutdown of your computer.

### Exception Handling
Any try/catch/finally blocks in the scope where the thread is created have no effects when it starts executing because each thread had an independent execution path.

To do that, several solution are available:
- Add an exception handler on all thread entry methods.
- For WPF and Windows Forms applications, you can use:
  - Application.DispatcherUnhandledException and Application.ThreadException: fire only for exceptions thrown on the main UI thread. You still must handle exceptions on worker threads manually.
  - AppDomain.CurrentDomain.UnhandledException: fires on any unhandled exception, but provides no means of preventing the application from shutting down afterward.

There are, however, some cases where you don’t need to handle exceptions on a worker thread, because the .NET Framework does it for you. These are covered in upcoming sections, and are:
- Asynchronous delegates
- BackgroundWorker
- The Task Parallel Library

## Thread Pooling
Whenever you start a thread, a few hundred microseconds are spent organizing such things as a memory stack. Each thread also consumes (by default) around 1 MB of memory. The thread pool cuts these overheads by sharing and recycling threads, allowing multithreading to be applied at a very granular level without a performance penalty. 

The thread pool have a limited number of worker threads that can be run simultaneously because too many active threads can throttle the operating system with administrative burden and render CPU caches ineffective. Once a limit is reached, jobs queue up and start only when another finishes.  

### Enter the Thread Pool
There are a number of ways to enter the thread pool:
- Via the Task Parallel Library (from Framework 4.0)
```C#
static void Main()
{
    // No generic construction
    Task.Factory.StartNew(Go);
  
    // Generic construction
    var task = Task.Factory.StartNew<string>(Go("thread pool"));

    // When we need the task's return value, we query its Result property:
    // If it's still executing, the current thread will now block (wait)
    // until the task finishes
    // Any unhandled exceptions are automatically rethrown when you query the task's Result property, wrapped in an AggregateException. 
    // However, if you fail to query its Result property (and don’t call Wait) any unhandled exception will take the process down.  
    var result = result = task.Result;
}
 
static void Go()
{
    Console.WriteLine("Hello from the thread pool!");
}

static void Go(string name)
{
    Console.WriteLine(String.Format("Hello from the {0}!", name));
}
```

- By calling ThreadPool.QueueUserWorkItem
It doesn't let you return data from the thread but marshal any exception back to the caller.
```C#
static void Main()
{
    ThreadPool.QueueUserWorkItem(Go);
    ThreadPool.QueueUserWorkItem(Go, 123);
}
 
static void Go(object data)
{
    Console.WriteLine("Hello from the thread pool! " + data);
}
```

- Via asynchronous delegates
It allows any number of typed arguments to be passed in both parameters and return values. Furthermore, unhandled exceptions are rethrown on the original thread (or more accurately, the thread that calls EndInvoke), and so they don’t need explicit handling.
```C#
static void Main()
{
    Func<string, int> method = Work;
    IAsyncResult cookie = method.BeginInvoke("test", null, null);

    // The final argument to BeginInvoke is a user state object that populates the AsyncState property of IAsyncResult. 
    // It can contain anything you like;
    method.BeginInvoke("test", Done, method);
  
    // ... here's where we can do other work in parallel...
  
    // Firstly, EndInvoke waits for the asynchronous delegate to finish executing, if it hasn’t already. 
    // Secondly, it receives the return value (as well as any ref or out parameters). 
    // Thirdly, it throws any unhandled worker exception back to the calling thread.
    int result = method1.EndInvoke(cookie);
    Console.WriteLine("String length is: " + result);
}
 
static int Work(string s) 
{ 
    return s.Length; 
}

static void Done (IAsyncResult asyncResult)
{
    var target = (Func<string, int>)asyncResult.AsyncState;
    int result = target.EndInvoke(asyncResult);
    Console.WriteLine("String length is: " + result);
}
```

- Via BackgroundWorker
```C#
static void Main()
{
    var worker = new BackgroundWorker();
    worker.DoWork += new DoWorkEventHandler(Work);
    worker.RunWorkerAsync(132);
}

static void Work(object sender, DoWorkEventArgs e)
{
    var parameters = (object[])e.Argument;
    Console.WriteLine("Hello from the thread pool! " + parameters[0]);
}
```

The following constructs use the thread pool indirectly:
- WCF, Remoting, ASP.NET, and ASMX Web Services application servers
- System.Timers.Timer and System.Threading.Timer
- Framework methods that end in Async, such as those on WebClient (the event-based asynchronous pattern), and most BeginXXX methods (the asynchronous programming model pattern)
- PLINQ

### Optimizing the Thread Pool
The thread pool starts out with one thread in its pool. The thread pool can create a new thread each time the current ones are working and the max limit are not reached.

You can set the upper limit of threads that the pool will create by calling ThreadPool.SetMaxThreads; the defaults are:
- 1023 in Framework 4.0 in a 32-bit environment
- 32768 in Framework 4.0 in a 64-bit environment
- 250 per core in Framework 3.5
- 25 per core in Framework 2.0

You can also set the lower limit (by calling ThreadPool.SetMinThreads) to avoid to delay the allocation of threads which cost some time. Raising the minimum thread count improves concurrency when there are blocked threads.
