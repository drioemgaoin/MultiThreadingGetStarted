1. [Getting Started](#getting-started)
   1. [Introduction and Concepts](#introduction-and-concepts) 
      1. [Join And Sleep](#joind-and-sleep)
      2. [How Threading works](#how-threading-works)  
      3. [Threads vs Processes](#threads-vs-processes) 
   2. [Creating and Starting Threads](#creating-and-starting-threads)
      1. [Passing data to a Thread](#passing-data-to-a-thread)   

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

## Passing data to a Thread
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