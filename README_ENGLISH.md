# LiteGo: "mini" Android asynchronous concurrent class library



LiteGo is an "asynchronous concurrency library" based on the Java language. Its core is a "mini" concurrer. It can freely set the maximum number of "concurrency" at the same time period, wait for the number of "queued" threads, and also Set "queuing strategy" and "overload strategy".
LiteGo can directly invest in Runnable, Callable, FutureTask and other types of implementations to run a task. Its core component is "SmartExecutor", which can be used as the only component in "App" that supports asynchronous concurrency.
There can be multiple instances of "SmartExecutor" in an App, and each instance has complete "independence", such as independent "core concurrency", "queuing and waiting" indicators, independent "operation scheduling and full load processing" strategy, But all instances "share a thread pool."
This mechanism not only meets the independent needs of different modules for thread control and task scheduling, but also shares a pool resource to save overhead, saves resources and reuses threads to the greatest extent, and helps improve performance.

--------------------------

Official website: [litesuits.com](` <a href="http://litesuits.com" title="Title">`)

QQ group: 42960650

------------------

### LiteGo background

About the status quo and problems of asynchronous and concurrent

- The cost of thread creation is relatively high, especially in scenarios that require a lot of concurrency in a short time, the problem is prominent, so Java has a thread pool to manage and reuse threads.
- Generally speaking, one thread pool per App is enough! There is no need to completely re-implement it yourself, and make full use of the concurrent library written by Doug Lea (the individual who has contributed the most to java).
- Now there are many frameworks, some are independent and powerful, and there are masters. It is recommended to read the source code. It is best to know the bottom line. It is possible that they have their own thread pool. At this time, if you don't pay attention to managing threads, it will be even worse.

Therefore, in view of this, I wrote this class library to unify the thread pool, clarify and control the management strategy.

### LiteGo philosophy

- Do not hold more threads when you are idle, and it is best not to exceed the number of CPUs, and make decisions based on specific application types and scenarios.
- Don't have too much concurrency at a moment, it's best to keep it around the number of CPUs, or it can be a few more problems.
- Pay attention to controlling the queuing and full load strategy, and it can be easily dealt with in the scene where a large number of concurrency rises instantly.

At the same time, the number of concurrent threads should not be too much. It is best to keep it around the number of CPU cores. Excessive CPU time slices and excessive rotation allocation will cause throughput to decrease. If too few, the CPU cannot be fully utilized. The number of concurrent threads can be appropriately greater than the number of CPU cores. A little more is no problem.

There is also a small personal suggestion to schedule tasks reasonably in business, optimize business logic, start by yourself, and don't mess around.

### LiteGo features

>  The number of core concurrent threads can be defined, that is, the number of concurrent requests at the same time.
>
> The number of threads waiting to be queued can be defined, that is, the number of requests that can be queued after exceeding the number of concurrent cores.
>
> The strategy for waiting for the queue to enter the execution state can be defined: first come first, execute first, then execute first.

You can define a strategy for processing new requests after the waiting queue is full:

- Discard the latest task in the queue
- Abandon the oldest task in the queue
- Abandon the current new task
- Direct execution (blocking the current thread)
- Throw an exception (interrupt the current thread)



### Used by LiteGo. OK, LET IT GO!

initialization:

```java
// put in 4 tasks at once
for (int i = 0; i <4; i++) {
     final int j = i;
     smallExecutor.execute(new Runnable() {
         @Override
         public void run() {
             HttpLog.i(TAG, "TASK" + j + "is running now ----------->");
             SystemClock.sleep(j * 200);
         }
     });
}

// Put in another task that may need to be cancelled
Future future = smallExecutor.submit(new Runnable() {
     @Override
     public void run() {
         HttpLog.i(TAG, "TASK 4 will be canceled... ------------>");
         SystemClock.sleep(1000);
     }
});

// Cancel this task at the right time
future.cancel(false);
```



The above code is designed to be able to concurrently "2" threads at the same time. After the concurrency is fully loaded, the waiting queue can accommodate "2" threads. The late tasks in the queue are executed first, and the new tasks will be discarded when the waiting queue is full. The oldest task.

Test the situation of multiple threads concurrency:

```java
// put in 4 tasks at once
for (int i = 0; i <4; i++) {
    final int j = i;
    smallExecutor.execute(new Runnable() {
        @Override
        public void run() {
            HttpLog.i(TAG, "TASK" + j + "is running now ----------->");
            SystemClock.sleep(j * 200);
        }
    });
}

// Put in another task that may need to be cancelled
Future future = smallExecutor.submit(new Runnable() {
    @Override
    public void run() {
        HttpLog.i(TAG, "TASK 4 will be canceled... ------------>");
        SystemClock.sleep(1000);
    }
});

// Cancel this task at the right time
future.cancel(false);
```



In the above code, five tasks of 0, 1, 2, 3, 4 are invested in sequence at a time. Note that task 4 is the last to be invested and returns a Future object.

According to the settings, 0 and 1 will be executed immediately. After the execution is full, 2, 3 will enter the queue. After the queue is full, the independently input task 4 will come. The oldest task 2 in the queue will be removed, and the queue will be 3 and 4.

Because 4 was subsequently cancelled, the final output:

```java
TASK 0 is running now ----------->
TASK 1 is running now ----------->
TASK 3 is running now ----------->
```



### Basic principles of LiteGO

Let's look at the main methods of SmartExecutor:

```java
public Future<?> submit(Runnable task)

public <T> Future<T> submit(Runnable task, T result)

public <T> Future<T> submit(Callable<T> task)

public <T> void submit(RunnableFuture<T> task)

public void execute(final Runnable command)

```

The most important is the execute method, and the other methods are to encapsulate the task as FutureTask and put it into the execute method. Because FutureTask is essentially a RunnableFuture object, which has both the characteristics and functions of Runnable and Future.

Then the key point is to look at the execute method:

```java
@Override
public void execute(final Runnable command) {
    if (command == null) {
        return;
    }
WrappedRunnable scheduler = new WrappedRunnable() {
    @Override
    public Runnable getRealRunnable() {
        return command;
    }

    public Runnable realRunnable;

    @Override
    public void run() {
        try {
            command.run();
        } finally {
            scheduleNext(this);
        }
    }
};

boolean callerRun = false;
synchronized (lock) {
    if (runningList.size() <coreSize) {
        runningList.add(scheduler);
        threadPool.execute(scheduler);
    } else if (waitingList.size() <queueSize) {
        waitingList.addLast(scheduler);
    } else {
        switch (overloadPolicy) {
            case DiscardNewTaskInQueue:
                waitingList.pollLast();
                waitingList.addLast(scheduler);
                break;
            case DiscardOldTaskInQueue:
                waitingList.pollFirst();
                waitingList.addLast(scheduler);
                break;
            case CallerRuns:
                callerRun = true;
                break;
            case DiscardCurrentTask:
                break;
            case ThrowExecption:
                throw new RuntimeException("Task rejected from lite smart executor. "+ command.toString());
            default:
                break;
        }
    }
    //printThreadPoolInfo();
}
if (callerRun) {
    command.run();
}
}
```

It can be seen that the whole process is briefly summarized as:

1. Encapsulate the task into a structure similar to a "linked list", execute one and schedule the next one.

2. Locking prevents resource grabbing during concurrency, and judges the number of currently running tasks.

3. If the current number of tasks is less than the maximum number of concurrent tasks, it will be put into operation, and if it is full, it will be put into the end of the waiting queue.

4. If the waiting queue is not full, the new task enters the queue, and if it is full, the full processing strategy is executed.

5. When a task is executed, its tail will schedule a new task to execute through a "link" method. If there is no task, it ends.
   Among them, "locking" and packaging tasks into "linked lists" are the key points.

   

[1]:  http://shang.qq.com/wpa/qunwpa?idkey=19bf15b9c85ec15c62141dd00618f725e2983803cd2b48566fa0e94964ae8370

