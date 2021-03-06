1. [Getting Started](#getting-started)
   1. [Introduction and Concepts](#introduction-and-concepts)
      1. [Join And Sleep](#join-and-sleep)
      2. [How Threading works](#how-threading-works)
      3. [Thread-Local Storage](#thread-local-storage)
      4. [Context Switching](#context-switching)
      5. [Threads vs Processes](#threads-vs-processes)
   2. [Creating and Starting Threads](#creating-and-starting-threads)
      1. [Passing data to a Thread](#passing-data-to-a-thread)
      2. [Naming Threads](#naming-threads)
      3. [Foreground and Background Threads](#foreground-and-background-threads)
      4. [Thread Priority](#thread-priority)
      5. [Exception Handling](#exception-handling)
   3. [Thread Pooling](#thread-pooling)
      1. [Enter the Thread Pool](#enter-the-thread-pool)
      2. [Optimizing the Thread Pool](#optimizing-the-thread-pool)
2. [Basic Synchronization](#basic-synchronization) 
   1. [Synchronization Essentials](#synchronization-essentials)
      1. [Blocking](#blocking)
      2. [Locking](#locking)
         1. [Exclusive Locking](#exclusive-locking)
            1. [Lock](#lock)
            2. [Monitor.Enter and Monitor.Exit](#monitorenter-and-monitorexitvis)
            3. [SpinLock](#spinlock)
            4. [Mutex](#mutex)
            5. [Choosing the Synchronization Object](#choosing-the-synchronization-object)
            6. [When to Lock](#when-to-lock)
            7. [Locking and Atomicity](#locking-and-atomicity)
            8. [Nested Locking](#nested-locking)
            9. [Deadlocks](#deadlocks)
            10. [Performance](#performance)
         2. [Nonexclusive Locking](#nonexclusive-locking)
            1. [Semaphore](#semaphore)
            2. [SemaphoreSlim](#semaphoreslim)
            3. [Reader and Writer Lock](#reader-and-writer-lock)
               1. [ReadWriteLockSlim](#readwritelockslim)
               2. [Upgradable Locks and Recursion](#upgradable-locks-and-recursion)
      3. [Signaling](#signaling)
         1. [Event Wait Handles](#event-wait-handles)
            1. [AutoResetEvent](#autoresetevent) 
            3. [ManualResetEvent](#manualresetevent) 
            4. [CountdownEvent](#countdownevent) 
            5. [Barrier](#barrier) 
            6. [Creating a Cross-Process](#creating-a-cross-process) 
            7. [Pooling Wait Handles](#pooling-wait-handles) 
            8. [WaitAny, WaitAll and SignalAndWait](#waitany-waitall-and-signalandwait) 
            9. [Monitor Wait and Pulse](#monitor-wait-and-pulse) 
               1. [How to use them](#how-to-use-them) 
         2. [Monitor Wait and Pulse](#monitor-wait-and-pulse)
         3. [CountdownEvent](#countdownevent)
         4. [Nonblocking Synchronization](#nonblocking-synchronization)
            1. [Volatile](#volatile)
            2. [Interlocked](#interlocked)
            3. [Thread.MemoryBarrier](#thread.memorybarrier)
            4. [Thread.VolatileRead](#thread.volatileread)
            5. [Thread.VolatileWrite](#thread.volatilewrite)
   3.  [Timers](#timers)
       1. [Multi-Threaded Timers](#multi-threaded-timers)
       2. [Single-Threaded Timers](#single-threaded-timers)

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
Multithreading is managed internally by a Thread Scheduler, a function the CLR typically delegates to the operating system. 

A thread scheduler is an oparating system's functionality that ensures all active threads (ready threads) are allocated appropriate execution time (micropocessor time), and that threads that are waiting or blocked (for instance, on an exclusive lock or on user input) do not consume CPU time.

On a single-processor computer, a thread scheduler performs time-slicing � rapidly switching execution between each of the active threads (it is called Context Switching)

On a multi-processor computer, multithreading is implemented with both time-slicing (operating system�s need to service its own threads) and concurrency, where different threads run code simultaneously on different CPUs. 

A thread is said to be preempted when its execution is interrupted due to an external factor such as time-slicing. In most situations, a thread has no control over when and where it�s preempted.

A program or method is thread-safe if it can be executed by multiple threads at the same time without side effects.

### Thread-Local Storage
Sometimes, you want to keep data isolated, ensuring that each thread has a separate copy. Local variables achieve exactly this, but they are useful only with transient data.

There are three ways to implement thread-local storage:
- [ThreadStatic]: mark a static field with the ThreadStatic attribute
```C#
[ThreadStatic] static int x;
```
Unfortunately, [ThreadStatic] doesn�t work with instance fields; nor does it play well with field initializers � they execute only once on the thread that's running when the static constructor executes. 

- ThreadLocal<T>: provides thread-local storage for both static and instance fields � and allows you to specify default values.
```C#
// static field
static ThreadLocal<int> x = new ThreadLocal<int> (() => 3); 
```

```C#
// instance field
var localRandom = new ThreadLocal<Random>(() => new Random());
Console.WriteLine (localRandom.Value.Next());
```
The Random class is not thread-safe, so we have to either lock around using Random (limiting concurrency) or generate a separate Random object for each thread.

- GetData and SetData: These store data in thread-specific �slots�. Thread.GetData reads from a thread�s isolated data store; Thread.SetData writes to it. Both methods require a LocalDataStoreSlot object to identify the slot. The same slot can be used across all threads and they�ll still get separate values.
```C#
// The same LocalDataStoreSlot object can be used across all threads.
LocalDataStoreSlot secSlot = Thread.GetNamedDataSlot("securityLevel"); // creates a named slot. Thread.AllocateDataSlot to create an unnamed slot
 
// This property has a separate value on each thread.
int SecurityLevel
{
    get
    {
        object data = Thread.GetData secSlot);
        return data == null ? 0 : (int)data;    // null == uninitialized
    }
    set { Thread.SetData (secSlot, value); }
}
```

### Context Switching
A context switch (also sometimes referred to as a process switch or a task switch) is the switching of the CPU (central processing unit) from one process or thread to another. It is done by the scheduler itself.

The steps in a context switch are:
- Save the context of the thread that just finished executing.
- Place the thread that just finished executing at the end of the queue for its priority.
- Find the highest priority queue that contains ready threads.
- Remove the thread at the head of the queue, load its context, and execute it.

Threads waiting for a synchronization object or input are not considered as ready thread so they won't be used by the Thread Scheduler so they won't consume any time CPU.

### Threads vs Processes
Processes run in parallel on a computer whereas Threads run in parallel within a single process.

Processes are fully isolated from each other whereas Threads are a bit isolated. They share (heap) memory with other threads running in the same application. 

Threading also incurs a resource and CPU cost in scheduling and switching threads (when there are more active threads than CPU cores) � and there�s also a creation/tear-down cost. Multithreading will not always speed up your application � it can even slow it down if used excessively or inappropriately.

![Thread vs Process](/img/thread-process.jpg)

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
### Thread state
![Thread State](/img/thread-state.jpg)

You can query a thread's execution status via its ThreadState property. This returns a flags enum of type ThreadState, which combines three �layers� of data in a bitwise fashion.

The ThreadState property is useful for diagnostic purposes, but unsuitable for synchronization, because a thread�s state may change in between testing ThreadState and acting on that information.

### Foreground and Background Threads
Foreground threads keep the application alive for as long as any one of them is running. The application ends only after all foreground threads have ended. By default, threads you create explicitly are foreground threads.

Background threads dont't keep the application alive. If the application ends all the background threads running are abruptly terminated, so if you have some finally block (or using block), they won't be executed.

You can query or change a thread�s background status using its IsBackground property.

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

There are, however, some cases where you don�t need to handle exceptions on a worker thread, because the .NET Framework does it for you. These are covered in upcoming sections, and are:
- Asynchronous delegates
- BackgroundWorker
- The Task Parallel Library


## Thread Pooling
A Thread Pool is a limited number of worker threads that can be run simultaneously to perform tasks.

Whenever you start a thread, a few hundred microseconds are spent organizing such things as a memory stack. Each thread also consumes (by default) around 1 MB of memory. The thread pool cuts these overheads by sharing and recycling threads, allowing multithreading to be applied at a very granular level without a performance penalty. 

The thread pool have a limited number of worker threads that can be run simultaneously because too many active threads can throttle the operating system with administrative burden and render CPU caches ineffective. Once a limit is reached, jobs queue up and start only when another finishes.  

Pros: 
- Reduce overhead of thread management (creation, schedule, release).
- Number of thread is limited, there is no worries about creating too many threads and hence affecting system performance.
 
Cons: 
- Thread can't be name so difficult to debug
- No control over the state and priority of the thread.
- When submitting a process to the thread pool, you have no idea when the process will be executed. Your process may be delayed when there are high demand on the thread pool.
- The thread pool is not suitable when you want to run two tasks or processes using two threads, and need these two tasks to be processed simultaneously in a deterministic fashion.
- The .NET framework uses the thread pool for asynchronous operations, and this places additional demand on the limited number of available threads.

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
    // However, if you fail to query its Result property (and don�t call Wait) any unhandled exception will take the process down.  
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
It allows any number of typed arguments to be passed in both parameters and return values. Furthermore, unhandled exceptions are rethrown on the original thread (or more accurately, the thread that calls EndInvoke), and so they don�t need explicit handling.
```C#
static void Main()
{
    Func<string, int> method = Work;
    IAsyncResult cookie = method.BeginInvoke("test", null, null);

    // The final argument to BeginInvoke is a user state object that populates the AsyncState property of IAsyncResult. 
    // It can contain anything you like;
    method.BeginInvoke("test", Done, method);
  
    // ... here's where we can do other work in parallel...
  
    // Firstly, EndInvoke waits for the asynchronous delegate to finish executing, if it hasn�t already. 
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


# Basic Synchronization
## Synchronization Essentials
The synchronization is responsible to coordinqte the qctions of threads for a predictable outcome. Synchronization is particularly important when threads access the same data.

Synchronization constructs can be divided into four categories:
- Simple blocking methods: These wait for another thread to finish or for a period of time to elapse. Sleep, Join, and Task.Wait are simple blocking methods.
- Locking constructs: These limit the number of threads that can perform some activity or execute a section of code at a time. Exclusive locking constructs are most common � these allow just one thread in at a time. The standard exclusive locking constructs are lock (Monitor.Enter/Monitor.Exit), Mutex, and SpinLock. The nonexclusive locking constructs are Semaphore, SemaphoreSlim, and the reader/writer locks.
- Signaling constructs: These allow a thread to pause until receiving a notification from another, avoiding the need for inefficient polling. There are two commonly used signaling devices: event wait handles and Monitor�s Wait/Pulse methods. Framework 4.0 introduces the CountdownEvent and Barrier classes.
- Nonblocking synchronization constructs: These protect access to a common field by calling upon processor primitives. The CLR and C# provide the following nonblocking constructs: Thread.MemoryBarrier, Thread.VolatileRead, Thread.VolatileWrite, the volatile keyword, and the Interlocked class.

### Blocking
A thread is blocked when its execution is paused for some reason, such as when Sleeping or waiting for another to end via Join or EndInvoke. 

A blocked thread immediately yields its processor time slice, so it doesn't consume any processor time until its blocking condition is satisfied.

When a thread blocks or unblocks, the operating system performs a context switch (This incurs an overhead of a few microseconds). Context switching is when the CPU is switched from one process or thread to another. Switching the CPU from one thread to another involves suspending the current thread, saving its state (e.g., registers), and then restoring the state of the thread being switched to.

Unblocking happens in one of four ways:
- by the blocking condition being satisfied
- by the operation timing out (if a timeout is specified)
- by being interrupted via Thread.Interrupt
- by being aborted via Thread.Abort

Sometimes a thread is blocked until a certain condition is met. Signaling and locking constructs achieve this efficiently by blocking until a condition is satisfied. However you can do the same by spinning in a polling loop. It is very wasteful on processor time because CLR and operating system keep performing important calculation and allocation resources. However spinning very briefly can be effective when you expect a condition to be satisfied soon (perhaps within a few microseconds) because it avoids the overhead and latency of a context switch. 

### Locking
#### Exclusive Locking
Exclusive locking is used to ensure that only one thread can enter particular sections of code at a time. The two main exclusive locking constructs are lock and Mutex.

##### Lock
 The lock construct is faster and more convenient.
```C#
public class ThreadSafe
{
    static readonly object locker = new object();
    static int val1, val2;
 
    static void Go()
    {
        lock (locker)
        {
            if (val2 != 0) Console.WriteLine(val1 / val2);
            val2 = 0;
        }
    }
}
```

Only one thread can lock the synchronizing object (in this case locker) at a time, the others threads are blocked and push in a FIFO queue until the lock is released. 

A Lock can be released only from the same thread that obtained it.

##### Monitor.Enter and Monitor.Exit
C#�s lock statement is in fact a shortcut for a call to the methods Monitor.Enter and Monitor.Exit, with a try/finally block. 
```C#
Monitor.Enter(locker);
try
{
    if (val2 != 0) Console.WriteLine(val1 / val2);
    val2 = 0;
}
finally 
{ 
    Monitor.Exit(locker); 
}
```

Calling Monitor.Exit without first calling Monitor.Enter on the same object throws an exception.

The code below is exactly what the C# 1.0, 2.0, and 3.0 compilers produce in translating a lock statement. However there is a vulnerability. If there is an exception within the implementation of Monitor.Enter, or between the call to Monitor.Enter and the try block and the lock is taken, it won�t be released � because we�ll never enter the try/finally block. This will result in a leaked lock.

To avoid this vulnerability, CLR 4.0 changed the Enter's signature
```C#
bool lockTaken = false;
try
{
  Monitor.Enter(locker, ref lockTaken);
  // Do your stuff...
}
finally 
{ 
    if (lockTaken) Monitor.Exit(locker); 
}
```

lockTaken will be false after calling Enter if (and only if) the Enter method throws an exception and the lock was not taken.

Monitor also provides a TryEnter method that allows a timeout to be specified, either in milliseconds or as a TimeSpan. The method then returns true if a lock was obtained, or false if no lock was obtained because the method timed out. As with the Enter method, it�s overloaded in CLR 4.0 to accept a lockTaken argument.

##### SpinLock
The SpinLock struct let you lock without incurring the cost of context switching, at the expense of keeping a thread spinning. This approach is valid in high-contention scenarios when locking will be very brief.

If you leave a spinlock contended for too long (we�re talking milliseconds at most), it will yield its time slice, causing a context switch just like an ordinary lock. When rescheduled, it will yield again � in a continual cycle of �spin yielding.� This consumes far fewer CPU resources than outright spinning � but more than blocking.

Using a SpinLock is like using an ordinary lock, except:
- Spinlocks are structs.
- Spinlocks are not reentrant, meaning that you cannot call Enter on the same SpinLock twice in a row on the same thread. If you violate this rule, it will either throw an exception (if owner tracking is enabled) or deadlock (if owner tracking is disabled). You can specify whether to enable owner tracking when constructing the spinlock. Owner tracking incurs a performance hit.
- SpinLock lets you query whether the lock is taken, via the properties IsHeld and, if owner tracking is enabled, IsHeldByCurrentThread.
- SpinLock follow the robust pattern of providing a lockTaken argument

```C#
var spinLock = new SpinLock(true);   // Enable owner tracking
bool lockTaken = false;
try
{
    spinLock.Enter(ref lockTaken);
    // Do stuff...
}
finally
{
    if (lockTaken) spinLock.Exit();
}
```

##### Mutex
A Mutex is like a C# lock, but it can work across multiple processes. In other words, Mutex can be computer-wide as well as application-wide.

Acquiring and releasing an uncontended Mutex takes a few microseconds � about 50 times slower than a lock.

With a Mutex class, you call the WaitOne method to lock and ReleaseMutex to unlock. Closing or disposing a Mutex automatically releases it. 

A Mutex can be released only from the same thread that obtained it.

```C#
class OneAtATimePlease
{
  static void Main()
  {
    // Naming a Mutex makes it available computer-wide. Use a name that's
    // unique to your company and application (e.g., include your URL).
 
    using (var mutex = new Mutex(false, "my name"))
    {
      // Wait a few seconds if contended, in case another instance
      // of the program is still in the process of shutting down.
 
      if (!mutex.WaitOne(TimeSpan.FromSeconds (3), false))
      {
        Console.WriteLine("Another app instance is running. Bye!");
        return;
      }

      RunProgram();
    }
  }
 
  static void RunProgram()
  {
    Console.WriteLine("Running. Press Enter to exit");
    Console.ReadLine();
  }
}
```

##### Choosing the Synchronization Object

You can use any object visible from each threads on condition that be a reference type. 

The synchronizing object is typically private (because this helps to encapsulate the locking logic) and an instance or static field. 

However, you can use the containing object (this) or its type but you're not encapsulating the locking logic, so it becomes harder to prevent deadlocking and excessive blocking.

##### When to Lock
As a basic rule, you need to lock around accessing any writable shared field. Even in the simplest case � an assignment operation on a single field � you must consider synchronization.

##### Locking and Atomicity
If a group of variables are always read and written within the same lock, you can say the variables are read and written atomically.

The atomicity provided by a lock is violated if an exception is thrown within a lock block
```C#
decimal _savingsBalance, _checkBalance;

void Transfer (decimal amount)
{
    lock (locker)
    {
        savingsBalance += amount;
        checkBalance -= amount + GetBankFee();
    }
}
```

If an exception was thrown by GetBankFee(), the bank would lose money. In this case, we could avoid the problem by calling GetBankFee earlier. A solution for more complex cases is to implement �rollback� logic within a catch or finally block.

##### Nested Locking
A thread can repeatedly lock the same object in a nested (reentrant) fashion
```C#
lock (locker)
  lock (locker)
    lock (locker)
    {
       // Do something...
    }
```

or

```C#
Monitor.Enter(locker); Monitor.Enter(locker);  Monitor.Enter(locker); 
// Do something...
Monitor.Exit(locker);  Monitor.Exit(locker);   Monitor.Exit(locker);
```

In these scenarios, the object is unlocked only when the outermost lock statement has exited � or a matching number of Monitor.Exit statements have executed.

##### Deadlocks
A deadlock happens when two threads each wait for a resource held by the other, so neither can proceed.
```C#
object locker1 = new object();
object locker2 = new object();
 
new Thread (() => {
    lock (locker1)
    {
        Thread.Sleep(1000);
        lock(locker2); // Deadlock
    }
}).Start();

lock (locker2)
{
    Thread.Sleep(1000);
    lock(locker1); // Deadlock
}
```

A threading deadlock causes participating threads to block indefinitely, unless you�ve specified a locking timeout.

##### Performance
Locking is fast: you can expect to acquire and release a lock in as little as 20 nanoseconds if the lock is uncontended. If it is contended, the consequential context switch moves the overhead closer to the microsecond region, although it may be longer before the thread is actually rescheduled. You can avoid the cost of a context switch with the SpinLock class � if you�re locking very briefly.

Locking can degrade concurrency if locks are held for too long. This can also increase the chance of deadlock.

#### Nonexclusive Locking
##### Semaphore
A semaphore is like a nightclub: it has a certain capacity, enforced by a bouncer. Once it�s full, no more people can enter, and a queue builds up outside. Then, for each person that leaves, one person enters from the head of the queue. The constructor requires a minimum of two arguments: the number of places currently available in the nightclub and the club�s total capacity.

A semaphore with a capacity of one is similar to a Mutex or lock, except that the semaphore has no �owner� � it�s thread-agnostic. Any thread can call Release on a Semaphore, whereas with Mutex and lock, only the thread that obtained the lock can release it.

Semaphores can be useful in limiting concurrency � preventing too many threads from executing a particular piece of code at once. 

```C#
class TheClub
{
    static SemaphoreSlim sem = new SemaphoreSlim(3);    // Capacity of 3
 
    static void Main()
    {
        for (int i = 1; i <= 5; i++) new Thread(Enter).Start (i);
    }
 
    static void Enter (object id)
    {
        Console.WriteLine(id + " wants to enter");
        sem.Wait();
        Console.WriteLine(id + " is in!");           // Only three threads
        Thread.Sleep(1000 * (int) id);               // can be here at
        Console.WriteLine(id + " is leaving");       // a time.
        sem.Release();
    }
}
```

A Semaphore, if named, can work across multiple processes in the same way as a Mutex.

##### SemaphoreSlim
SemaphoreSlim is a lightweight implementation of a Semaphore. The real purpose of the SemaphoreSlim is to supply a faster Semaphore (typically a Semaphore might take 1 ms per WaitOne and per Release, the SemaphoreSlim takes a quarter of this time, source ).

SemaphoreSlim is based on SpinWait and Monitor, so the thread that waits to acquire the lock is burning CPU cycles for some time in hope to acquire the lock before letting it to another thread. If the thread can't acquire the lock, it lets the systems to switch context and tries again (by burning some CPU cycles) once it is scheduled again by the OS. With long waits this pattern can burn through a substantial amount of CPU cycles. 

So the beast case scenario for such implementation is when most of the time there is no wait time and you can almost instantly acquire the lock.

##### Reader and Writer Lock
###### ReadWriteLockSlim
The ReadWriteLockSlim is used to protected a resource that is read by multiple threads and written by a small number of threads at a time. ReaderWriterLockSlim allows more concurrent Read activity than a simple lock.

The ReadWriteLockSlim was introduced in Framework 3.5 in a replacement of ReaderWriterLock which is several times slower and has an inherent design fault in its mechanism for handling lock upgrades.

There are two basic kinds of lock � a read lock and a write lock:
- A write lock is universally exclusive.
- A read lock is compatible with other read locks.

So, a thread holding a write lock blocks all other threads trying to obtain a read or write lock (and vice versa). But if no thread holds a write lock, any number of threads may concurrently obtain a read lock.

ReaderWriterLockSlim defines the following methods for obtaining and releasing read/write locks:
- public void EnterReadLock();
- public void ExitReadLock();
- public void EnterWriteLock();
- public void ExitWriteLock();

Additionally, there are �Try� versions of all EnterXXX methods that accept timeout arguments in the style of Monitor.TryEnter.

The following program demonstrates ReaderWriterLockSlim. Three threads continually enumerate a list, while two further threads append a random number to the list every second. A read lock protects the list readers, and a write lock protects the list writers:
```C#
class ReaderWriterLockSlimDemo
{
    static ReaderWriterLockSlim rw = new ReaderWriterLockSlim();
    static List<int> items = new List<int>();
    static Random rand = new Random();
 
    static void Main()
    {
        new Thread(Read).Start();
        new Thread(Read).Start();
        new Thread(Read).Start();
 
        new Thread(Write).Start("A");
        new Thread(Write).Start("B");
    }
 
    static void Read()
    {
        while (true)
        {
            rw.EnterReadLock();
            foreach (int i in items) Thread.Sleep(10);
            rw.ExitReadLock();
        }
    }
 
    static void Write (object threadID)
    {
        while (true)
        {
            int newNumber = GetRandNum(100);
            rw.EnterWriteLock();
            items.Add(newNumber);
            rw.ExitWriteLock();
            Console.WriteLine("Thread " + threadID + " added " + newNumber);
            Thread.Sleep(100);
        }
    }
 
    static int GetRandNum (int max) { lock (rand) return rand.Next(max); }
}
```

###### Upgradable Locks and Recursion
Sometimes it�s useful to swap a read lock for a write lock in a single atomic operation. 

For instance, suppose you want to add an item to a list only if the item wasn�t already present. Ideally, you�d want to minimize the time spent holding the (exclusive) write lock, so you might proceed as follows:
- Obtain a read lock.
- Test if the item is already present in the list, and if so, release the lock and return.
- Release the read lock.
- Obtain a write lock.
- Add the item.

The problem is that another thread could sneak in and modify the list (e.g., adding the same item) between steps 3 and 4. ReaderWriterLockSlim addresses this through a third kind of lock called an upgradeable lock. An upgradeable lock is like a read lock except that it can later be promoted to a write lock in an atomic operation. 

Here�s how you use it:
- Call EnterUpgradeableReadLock.
- Perform read-based activities (e.g., test whether the item is already present in the list).
- Call EnterWriteLock (this converts the upgradeable lock to a write lock -> releases the read lock and obtains a fresh write lock, atomically).
- Perform write-based activities (e.g., add the item to the list).
- Call ExitWriteLock (this converts the write lock back to an upgradeable lock).
- Perform any other read-based activities.
- Call ExitUpgradeableReadLock.

```C#
while (true)
{
    int newNumber = GetRandNum(100);
    rw.EnterUpgradeableReadLock();
    if (!items.Contains(newNumber))
    {
        rw.EnterWriteLock();
        items.Add(newNumber);
        rw.ExitWriteLock();
        Console.WriteLine("Thread " + threadID + " added " + newNumber);
    }
    rw.ExitUpgradeableReadLock();
    Thread.Sleep(100);
}
```

An upgradeable lock can coexist with any number of read locks, only one upgradeable lock can itself be taken out at a time.

Ordinarily, nested and recursive locking is prohibited with ReaderWriterLockSlim, an exception will be throw. But if you construct your ReaderWriterLockSlim as followed, you can do recursive locking. Recursive locking can create undesired complexity because it�s possible to acquire more than one kind of lock.
```C#
rw.EnterWriteLock();
rw.EnterReadLock();
Console.WriteLine (rw.IsReadLockHeld);     // True
Console.WriteLine (rw.IsWriteLockHeld);    // True
rw.ExitReadLock();
rw.ExitWriteLock();
```

The basic rule is that once you�ve acquired a lock, subsequent recursive locks can be less, but not greater, on the following scale:

    Read Lock, Upgradeable Lock, Write Lock

A request to promote an upgradeable lock to a write lock, however, is always legal.

### Signaling
Signaling is when one thread waits until it receives notification from another. 

#### Event Wait Handles
![Thread State](/img/event-wait-handles.jpg)

#### AutoResetEvent
An AutoResetEvent is like a ticket turnstile: inserting a ticket lets exactly one person through. The �auto� in the class�s name refers to the fact that an open turnstile automatically closes or �resets� after someone steps through. 

A thread waits, or blocks, at the turnstile by calling WaitOne (wait at this �one� turnstile until it opens), and a ticket is inserted by calling the Set method. If a number of threads call WaitOne, a queue builds up behind the turnstile.

Any (unblocked) thread with access to the AutoResetEvent object can call Set on it to release one blocked thread.

```C#
class AutoResetEvent
{
  static EventWaitHandle waitHandle = new AutoResetEvent(false); // true is equivalent to call Set immediately
 
  static void Main()
  {
    new Thread(Waiter).Start();
    Thread.Sleep(1000);                 // Pause for a second...
    waitHandle.Set();                    // Wake up the Waiter.
  }
 
  static void Waiter()
  {
    Console.WriteLine("Waiting...");
    waitHandle.WaitOne();                // Wait for notification
    Console.WriteLine("Notified");
  }
}
```

![Thread State](/img/autoresetevent.jpg)

If Set is called when no thread is waiting, the handle stays open for as long as it takes until some thread calls WaitOne. Even if you call several time Set, only the next one thread is letted through.

Calling Reset on an AutoResetEvent closes the turnstile (should it be open) without waiting or blocking.

#### ManualResetEvent
A ManualResetEvent functions like an ordinary gate. 

Calling Set opens the gate, allowing any number of threads calling WaitOne to be let through. Calling Reset closes the gate. Threads that call WaitOne on a closed gate will block; when the gate is next opened, they will be released all at once.

A ManualResetEvent is useful in allowing one thread to unblock many other threads.

#### CountdownEvent
CountdownEvent lets you wait on more than one thread.

You can reincrement a CountdownEvent�s count by calling AddCount. However, if it has already reached zero, this throws an exception: you can�t �unsignal� a CountdownEvent by calling AddCount. To avoid the possibility of an exception being thrown, you can instead call TryAddCount, which returns false if the countdown is zero.

To unsignal a countdown event, call Reset: this both unsignals the construct and resets its count to the original value.

```C#
static CountdownEvent countdown = new CountdownEvent(3); // Initialize with "count" of 3.
 
static void Main()
{
  new Thread (SaySomething).Start("I am thread 1");
  new Thread (SaySomething).Start("I am thread 2");
  new Thread (SaySomething).Start("I am thread 3");
 
  countdown.Wait();   // Blocks until Signal has been called 3 times
  Console.WriteLine("All threads have finished speaking!");
}
 
static void SaySomething (object thing)
{
  Thread.Sleep (1000);
  Console.WriteLine (thing);
  countdown.Signal();
}
```

#### Barrier
It implements a thread execution barrier, which allows many threads to rendezvous at a point in time. The class is very fast and efficient, and is built upon Wait, Pulse, and spinlocks.

To use this class:
- Instantiate it, specifying how many threads should partake in the rendezvous (you can change this later by calling AddParticipants/RemoveParticipants).
- Have each thread call SignalAndWait when it wants to rendezvous.

```C#
static Barrier barrier = new Barrier (3);
 
static void Main()
{
    new Thread(Speak).Start();
    new Thread(Speak).Start();
    new Thread(Speak).Start();
}
 
static void Speak()
{
    for (int i = 0; i < 5; i++)
    {
        Console.Write (i + " ");
        barrier.SignalAndWait();
    }
}
```

![Barrier](/img/barrier.jpg)

#### Creating a Cross-Process
You can create an Event wait handle cross-process by using the EventWaitHandle�s constructor
```C#
EventWaitHandle wh = new EventWaitHandle (false, EventResetMode.AutoReset, "MyCompany.MyApp.SomeName");
```

If two applications each ran this code, they would be able to signal each other: the wait handle would work across all threads in both processes.

#### Pooling Wait Handles
If your application has lots of threads that spend most of their time blocked on a wait handle, you can reduce the resource burden by calling ThreadPool.RegisterWaitForSingleObject.

For example, if 100 clients called the method "WaitHandle", 100 server threads would be tied up for the duration of the blockage. Replacing WaitOne with RegisterWaitForSingleObject (method WaitHandleReplaced) allows the method to return immediately
```C#
static ManualResetEvent waitHandle = new ManualResetEvent(false);

void WaitHandle()
{
    _wh.WaitOne();
    // ... continue execution
}

void WaitHandleReplaced
{
    var reg = ThreadPool.RegisterWaitForSingleObject(waitHandle, Resume, null, -1, true);
    // ... continue execution
}
 
static void Resume(object data, bool timedOut)
{
    // ... continue execution
}
```

#### WaitAny, WaitAll and SignalAndWait
In addition to the Set, WaitOne, and Reset methods, there are static methods on the WaitHandle:
- WaitAny: waits for any one of an array of wait handles.
- WaitAll: waits on all of the given handles, atomically.
- SignalAndWait: calls Set on one WaitHandle, and then calls WaitOne on another WaitHandle. After signaling the first handle, it will jump to the head of the queue in waiting on the second handle.

#### Monitor Wait and Pulse
It allow to wirte the signaling logic yourself using custom flags and fields (enclosed in lock statements), and then introduce Wait and Pulse commands to prevent spinning. With this logic you can achieve the functionality of every event wait handle we saw previously.

However, it has some disadvantages over event wait handles:
- Wait/Pulse cannot span application domains or processes on a computer.
- You must remember to protect all variables related to the signaling logic with locks.

Pulse executes asynchronously, meaning that it doesn't itself block or pause in any way. If another thread is waiting on the pulsed object, it�s unblocked. Otherwise the pulse has no effect and is silently ignored.

Pulse provides one-way communication: a pulsing thread (potentially) signals a waiting thread. Pulse does not return a value indicating whether or not its pulse was received. Further, when a notifier pulses and releases its lock, there�s no guarantee that an eligible waiter will kick into life immediately. There can be a small delay, at the discretion of the thread scheduler, during which time neither thread has a lock. This means that the pulser cannot know if or when a waiter resumes � unless you code something specifically (for instance with another flag and another reciprocal, Wait and Pulse).

The Pulse and PulseAll methods release threads blocked on a Wait statement. Pulse releases a maximum of one thread; PulseAll releases them all.

##### How to use them
- Define the synchronization object
- Define field(s) for use in your custom blocking condition(s).
- Use Monitor.Wait whenever you want to block.
- Use Monitor.Pulse whenever you change (or potentially change) a blocking condition.

```C#
class SimpleWaitPulse
{
    static readonly object locker = new object();
    static bool go;
 
    static void Main()
    {                                 // The new thread will block
        new Thread(Work).Start();     // because go==false.
 
        Console.ReadLine();           // Wait for user to hit Enter
 
        lock (locker)                 // Let's now wake up the thread by
        {                             // setting go=true and pulsing.
            go = true;
            Monitor.Pulse(locker);
        }
    }
 
    static void Work()
    {
        lock (locker)
            while (!go)
            Monitor.Wait(locker);    // Lock is released while we�re waiting. Timeout is available as a second parameter
 
        Console.WriteLine ("Woken!!!");
    }
}
```

### Nonblocking Synchronization
It can perform simple operations without ever blocking, pausing, or waiting. So, the thread doesn't suffer the overhead of a context switch and the latency of being descheduled. 

The nonblocking approaches also work across multiple processes. 

#### Volatile
It is a keyword indicating that a field might be modified by multiple threads that are executing at the same time. 

Fields that are declared volatile are not subject to compiler optimizations (caching data and re-ordering) that assume access by a single thread. This ensures that the most up-to-date value is present in the field at all times.

A processor might cache a piece of data from the main memory and use the cached data in the execution of a thread, modify it, and only update the main memory at a later time.
![Volatile](/img/volatile.jpg)

Marking a field as volatile would make sure that it is not cached during the execution of a thread.

#### Interlocked
It allows to perform a couple of simple operations atomically. 

For example, Reading and writing 64-bit fields is nonatomic on 32-bit environments (but it is on 64-bit environments) because it requires two separate instructions: one for each 32-bit memory location. So, if thread X reads a 64-bit value while thread Y is updating it, thread X may end up with a bitwise combination of the old and new values.

```C#
class Program
{
    static long sum;
 
    static void Main()
    {                                                            
        // Simple increment/decrement operations:
        Interlocked.Increment(ref sum);                              // 1
        Interlocked.Decrement(ref sum);                              // 0
 
        // Add/subtract a value:
        Interlocked.Add(ref sum, 3);                                 // 3
 
        // Read a 64-bit field:
        Console.WriteLine(Interlocked.Read (ref sum));               // 3
 
        // Write a 64-bit field while reading previous value:
        // (This prints "3" while updating sum to 10)
        Console.WriteLine(Interlocked.Exchange(ref sum, 10));       // 10
 
        // Update a field only if it matches a certain value (10):
        Console.WriteLine(Interlocked.CompareExchange(ref sum, 123, 10);      // 123
    }
}
```

#### Thread.MemoryBarrier
It is a full memory barrier (full fence i.e. release and acquire semantics) which prevents any kind of instruction reordering (improve efficiency) or caching around that fence.

```C#
class Foo
{
    int answer1, answer2, answer3;
    bool complete;
 
    void A()
    {
        answer1 = 1; answer2 = 2; answer3 = 3;
        Thread.MemoryBarrier();     // Barrier 1
        complete = true;
        Thread.MemoryBarrier();     // Barrier 2
    }
 
    void B()
    {
        Thread.MemoryBarrier();     // Barrier 3
        if (complete)
        {
            Thread.MemoryBarrier(); // Barrier 4
            Console.WriteLine (answer1 + answer2 + answer3);
        }
    }
}
```

Barriers 1 and 4 prevent this example from writing �0�. Barriers 2 and 3 ensure that if B ran after A, reading "complete" would evaluate to true.

#### Thread.VolatileRead
It is a static method in the Thread class that read a field and guarantee to the most up-to-date value.

#### Thread.VolatileWrite
It is a static method in the Thread class that write a value to a field immediately. The value is not written in the cache then later in the main memory.

### Synchronization Contexts
An alternative to locking manually is to lock declaratively. By deriving from ContextBoundObject and applying the Synchronization attribute, you instruct the CLR to apply locking automatically.
```C#
using System;
using System.Threading;
using System.Runtime.Remoting.Contexts;
 
[Synchronization]
public class AutoLock : ContextBoundObject
{
    public void Demo()
    {
        Console.Write("Start...");
        Thread.Sleep(1000);           // We can't be preempted here
        Console.WriteLine("end");     // thanks to automatic locking!
    } 
}
 
public class Test
{
    public static void Main()
    {
        AutoLock safeInstance = new AutoLock();
        new Thread(safeInstance.Demo).Start();     // Call the Demo
        new Thread(safeInstance.Demo).Start();     // method 3 times
        safeInstance.Demo();                       // concurrently.
    }
}
```

The CLR ensures that only one thread can execute code in safeInstance at a time. It does this by creating a single synchronizing object � and locking it around every call to each of safeInstance's methods or properties. The scope of the lock � in this case, the safeInstance object � is called a synchronization context.

A ContextBoundObject can be thought of as a �remote� object, meaning all method calls are intercepted. To make this interception possible, when we instantiate AutoLock, the CLR actually returns a proxy � an object with the same methods and properties of an AutoLock object, which acts as an intermediary. It's via this intermediary that the automatic locking takes place. Overall, the interception adds around a microsecond to each method call.

A synchronization context can extend beyond the scope of a single object. By default, if a synchronized object is instantiated from within the code of another, both share the same context (in other words, one big lock!) This behavior can be changed by specifying an integer flag in Synchronization attribute�s constructor:
| Constant              | Meaning                                                                                                                       |
| NOT_SUPPORTED	        | Equivalent to not using the Synchronized  attribute                                                                           |
| SUPPORTED             | Joins the existing synchronization context if instantiated from another synchronized object, otherwise remains unsynchronized |
| REQUIRED (default)    | Joins the existing synchronization context if instantiated from another synchronized object, otherwise creates a new context  |
| REQUIRES_NEW          | Always creates a new synchronization context                                                                                  |

```C#
[Synchronization (SynchronizationAttribute.REQUIRES_NEW)]
```

The bigger the scope of a synchronization context, the easier it is to manage, but the less the opportunity for useful concurrency. At the other end of the scale, separate synchronization contexts invite deadlocks.

#### Reentrancy
A thread-safe method is sometimes called reentrant, because it can be preempted part way through its execution, and then called again on another thread without ill effect.

If you use small scopes of synchronization, you can have some deadlocks. To avoid it, you can use the reentrancy by using Synchronization(true) attribute which allows to temporarily release the synchronization context's lock when execution leaves the context. The side effect of that is, during the release, any thread is free to call any method on the original object and change its state. 

# Timers
It allows to execute some method repeatedly at regular intervals. 

The .NET Framework provides four timers. Two of these are general-purpose multithreaded timers:
- System.Threading.Timer
- System.Timers.Timer

The other two are special-purpose single-threaded timers:
- System.Windows.Forms.Timer (Windows Forms timer)
- System.Windows.Threading.DispatcherTimer (WPF timer)

## Multi-Threaded Timers
### System.Timers.Timer
Fires an event and executes code on the event handler at regular intervals. This one is intended to be used as a sever-based or service component in a multithreaded environment.

Runs in UI or worker thread
ThreadSafe, each eventhandler has explicit locks
Can be run in any thread using ISynchronizeObject
Intuitive to use
Good metronome-quality beat
Support inheritance

```C#
using System;
using System.Threading.Tasks;
using System.Timers;

class Example
{
   static void Main()
   {
      Timer timer = new Timer(1000);
      timer.Elapsed += async ( sender, e ) => await HandleTimer();
      timer.Start();
      Console.Write("Press any key to exit... ");
      Console.ReadKey();
   }

   private static Task HandleTimer()
   {
     Console.WriteLine("\nHandler not implemented..." );
     throw new NotImplementedException();
   }
}
```

### System.Threading.Timer
Runs worker thread (Thread pool)
Good metronome-quality beat
Timer event supports state object
Initial timer event can be scheduled
Purely used for numerical timing, where UI update is not or very less required.

```C#
using System;
using System.Threading;
 
class Program
{
    static void Main()
    {
        // First interval = 5000ms; subsequent intervals = 1000ms
        Timer timer = new Timer(Tick, "tick...", 5000, 1000);
        Console.ReadLine();
        timer.Dispose();         // This both stops the timer and cleans up.
    }
 
    static void Tick (object data)
    {
        // This runs on a pooled thread
        Console.WriteLine (data);          // Writes "tick..."
    }
}
```

## Single-Threaded Timers
### System.Windows.Forms.Timer
Runs in UI Thread
Intuitive to use
Support inheritance

```C#
private System.Windows.Forms.Timer WinTimer = new System.Windows.Forms.Timer();
ObservableCollection<string> WinTimerList = new ObservableCollection<string>();

private void btnwft_Click(object sender, RoutedEventArgs e)
{
    this.WinTimer.Interval = 1000; //1 sec
    this.WinTimer.Tick += new EventHandler(WinTimer_Tick);vis
    this.WinTimer.Start();
    this.WinTimerList.Clear();
    this.lstwft.DataContext = this.WinTimerList;
}

void WinTimer_Tick(object sender, EventArgs e)
{
    this.WinTimerList.Add(string.Format("Tick Generated from {0}", Thread.CurrentThread.Name));
}
```

### System.Windows.Threading.DispatcherTimer
Runs on the Dispatcher Thread by default (UI Thread)

```C#

```
