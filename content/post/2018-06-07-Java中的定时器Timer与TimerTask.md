---
title: Java中的定时器Timer与TimerTask
categories:
  - Java
date: 2018-06-07 13:48:30
updated: 2018-06-07 13:48:30
tags: 
  - Java
---

定时器在我们需要循环执行，或者指定时间执行某个任务的时候非常的有用，所以Java其自身也提供了相应的类来实现这些功能。Java的 `java.util` 包内包含了 *Timer, TimerTask*。

# 简介
从命令上就可以看出，*Timer* 是一个定时器，其实质是单独开了一个线程来进行计时。

*TimerTask*是一个任务，其本质，是一个实现了 *Runnable* 接口的抽象类。其只定义了一个抽象方法 `run()`，所以我们在使用的时候，实现这个`run()`方法就行了。

我们主要关注的是，Timer的冲个调度方法：

```java
    public void schedule(TimerTask task, long delay);
    public void schedule(TimerTask task, Date time);
    public void schedule(TimerTask task, long delay, long period);
    public void schedule(TimerTask task, long delay, long period);      
```

其方便代表的意思是：

1. 指定延迟*delay* ms 后执行 *task*。
2. 在指定时间 *time* 时 执行 *task*。
3. 指定延迟*delay* ms 后，每 *period* ms 执行一次 *task* 任务
4. 在指定时间 *time* 时 延迟 *delay* ms 执行 *task*，之后每 *period* ms 执行一次任务。

# 实例

我们要实现一个模拟用户在线学习的功能，每个10秒钟，就给他增加学习时间：

```java
        new Timer().schedule(new TimerTask() {
            @Override
            public void run() {
            addLearnTime();
            }
        },10000,10000);
```

就是这么简单。