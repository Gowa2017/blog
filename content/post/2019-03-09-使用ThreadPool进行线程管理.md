---
title: 使用ThreadPool进行线程管理
categories:
  - Java
date: 2019-03-09 15:03:15
updated: 2019-03-09 15:03:15
tags: 
  - Java
  - Android
  - Thread
---

从以前 Posix C 上的经验来说，线程是一个永远不会停止的话题啊。但是，线程的新建，销毁，初始化其实开销都是相对有点大的。同时，如果无限制的进行线程的抢占和使用，在资源受限（移动设备）时，就会影响性能了。所以才有了利用线程池来管理线程的这么一个做法。
<!--more-->

# 基本原理
对于线程池来说，我们设计我们想要执行的任务，但是具体怎么执行，什么时候执行，我们交给了线程池。这样就跟我们的任务的设计和执行相分离。

也即是说，我们把想要执行的 Runnable 丢给线程池就可以了。

# 相关的对象
- Executor 接口，只有一个方法 execute
- Callable 接口，类似 Runnable 但是可以返回值。
- Future 类似 JS 的 Promise ，异步任务的返回结果
- ExecutorService 接口，对 Executor 的扩展，Executor 太过简单了，所以这个接口增加了线程池中的线程管理功能。
- ThreadPoolExecutor 类。实现了 ExecutorService 。也就是说其实现了对线程池管理的很多方法。
- ScheduledThreadPoolExecutor。类，扩展了 ThreadPoolExecutor，增加了定时调度线程功能。
- Executors 类。一个工厂和常用方法的集合，一些特定类型的 Executor 由他可以直接实例化。

# Executors

此类可以直接利用其工厂方法，获取一些 ExecutorService 对象。

```java
static final int DEFAULT_THREAD_POOL_SIZE = 4;

ExecutorService executorService = Executors.newFixedThreadPool(DEFAULT_THREAD_POOL_SIZE);

ExecutorService executorService = Executors.newCachedThreadPool();

ExecutorService executorService = Executors.newSingleThreadExecutor();
```

- newFixedThreadPool 固定线程数量的线程池
- newCachedThreadPool 队列中有任务的时候，就会创建线程；队列空了大于60秒，就会被空闲线程回收。
- newSingleThreadExecutor 只有一个线程。

简单的向线程池添加任务：

```java
executorService.execute(new Runnable(){
  @Override
  public void run(){
    callBlockingFunction();
  }
});

Future future = executorService.submit(new Callable(){
  @Override
  public Object call() throws Exception {
    callBlockingFunction();
    return null;
  }
});
```

 在第二个方法中，Future 可以用来获取返回结果 `Future.get()`，还可以取消一个任务 `Future.cancel()`。

#ThreadPoolExecutor
我们可能并不满足于上面那些简单的使用，所以有了ThreadPoolExecutor。

我们在建立 ThreadPoolExecutor 的实例的时候，可以给予很多参数来控制我们所需要的线程数量，驻留时间等等。

```java
int NUMBER_OF_CORES = Runtime.getRuntime().availableProcessors();
int KEEP_ALIVE_TIME = 1;
TimeUnit KEEP_ALIVE_TIME_UNIT = TimeUnit.SECONDS;

BlockingQueue<Runnable> taskQueue = new LinkedBlockingQueue<Runnable>();

ExecutorService executorService = new ThreadPoolExecutor(NUMBER_OF_CORES, 
                                                          NUMBER_OF_CORES*2, 
                                                          KEEP_ALIVE_TIME, 
                                                          KEEP_ALIVE_TIME_UNIT, 
                                                          taskQueue, 
                                                          new BackgroundThreadFactory());
                                                          
private static class BackgroundThreadFactory implements ThreadFactory {
  private static int sTag = 1;

  @Override
  public Thread newThread(Runnable runnable) {
      Thread thread = new Thread(runnable);
      thread.setName("CustomThread" + sTag);
      thread.setPriority(Process.THREAD_PRIORITY_BACKGROUND);

      // A exception handler is created to log the exception from threads
      thread.setUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
          @Override
          public void uncaughtException(Thread thread, Throwable ex) {
              Log.e(Util.LOG_TAG, thread.getName() + " encountered an error: " + ex.getMessage());
          }
      });
      return thread;
  }
}
```

# 使用

我们来看一下 Google 官方架构栏图中 todo-mvp 中对 Executor 的使用。

在类 `app/src/main/java/com/example/android/architecture/blueprints/todoapp/util/AppExecutors.java` 中，定义了三种类型的 Executor。

- diskIO
- networkIO
- mainThread

```java
public class AppExecutors {

    private static final int THREAD_COUNT = 3;

    private final Executor diskIO;

    private final Executor networkIO;

    private final Executor mainThread;

    @VisibleForTesting
    AppExecutors(Executor diskIO, Executor networkIO, Executor mainThread) {
        this.diskIO = diskIO;
        this.networkIO = networkIO;
        this.mainThread = mainThread;
    }

    public AppExecutors() {
        this(new DiskIOThreadExecutor(), Executors.newFixedThreadPool(THREAD_COUNT),
                new MainThreadExecutor());
    }

    public Executor diskIO() {
        return diskIO;
    }

    public Executor networkIO() {
        return networkIO;
    }

    public Executor mainThread() {
        return mainThread;
    }

    private static class MainThreadExecutor implements Executor {
        private Handler mainThreadHandler = new Handler(Looper.getMainLooper());

        @Override
        public void execute(@NonNull Runnable command) {
            mainThreadHandler.post(command);
        }
    }
}
```

```java
public class DiskIOThreadExecutor implements Executor {

    private final Executor mDiskIO;

    public DiskIOThreadExecutor() {
        mDiskIO = Executors.newSingleThreadExecutor();
    }

    @Override
    public void execute(@NonNull Runnable command) {
        mDiskIO.execute(command);
    }
}
```

可以看到，diskIO 只启用了一个线程的线程池，而 networkIO，启了三个线程，主线程的是直接将要执行的代码 Post 过去。


## 使用

```java
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                final List<Task> tasks = mTasksDao.getTasks();
                mAppExecutors.mainThread().execute(new Runnable() {
                    @Override
                    public void run() {
                        if (tasks.isEmpty()) {
                            // This will be called if the table is new or just empty.
                            callback.onDataNotAvailable();
                        } else {
                            callback.onTasksLoaded(tasks);
                        }
                    }
                });
            }
        };

        mAppExecutors.diskIO().execute(runnable);
```

```java
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                final Task task = mTasksDao.getTaskById(taskId);

                mAppExecutors.mainThread().execute(new Runnable() {
                    @Override
                    public void run() {
                        if (task != null) {
                            callback.onTaskLoaded(task);
                        } else {
                            callback.onDataNotAvailable();
                        }
                    }
                });
            }
        };

        mAppExecutors.diskIO().execute(runnable);
```
