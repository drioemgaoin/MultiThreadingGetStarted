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
