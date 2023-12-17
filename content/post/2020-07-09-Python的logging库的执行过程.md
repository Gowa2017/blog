---
title: Python的logging库的执行过程
categories:
  - Python
date: 2020-07-09 21:02:45
updated: 2020-07-09 21:02:45
tags: 
  - Python
  - logging
---

经常有在用，不过没有仔细看过，对于如何比较合理的，结构化的，规范的，比如根据不同的进程和目标来记录的话就会比较苦恼。所以来看一下。

<!--more-->



# logging.info 看起

无论我们是用 logging 还是用 logging.getLogger() 返回的 logger 来记录，都会调用相关的方法。

```python
def info(msg, *args, **kwargs):
    """
    Log a message with severity 'INFO' on the root logger. If the logger has
    no handlers, call basicConfig() to add a console handler with a pre-defined
    format.
    """
    if len(root.handlers) == 0:
        basicConfig()
    root.info(msg, *args, **kwargs)
```

设置一下默认设置，我们看到，其实有一个作为根的 logger 存在。

```python
root = RootLogger(WARNING)
Logger.root = root
Logger.manager = Manager(Logger.root)
```

同时还有一个 Logger 的管理器。

如果这个管理器的 handlers 数量为0，那么就会进行一个默认设置 `basicConfig()`。当然，我们经常也会以一些键值参数来调用这个方法，具体就不说了。这个函数的默认行为是，如果 root 已经有 handler 了，就什么都不做；否则的话就以一个 StreamHandler 来写到 sys.stderr，使用  BASIC_FORMAT来作为格式化。

# getLogger()

这个函数，实际上就是从 Logger.manager 里面获取一个 Logger 实例，我们的 manager 内部保存了一个日志记录器的字典列表，根据键值来获取。如果已经存在就直接返回，没有的话就新建。

# 多个日志的两种思路

## 为 root 添加多个 handler 

这个个是比较简单，所有的地方都可以直接使用  logging 进行记录日志，但需要对特定的 handler 设置过滤器，记录对应的信息。

## 单独使用每个 logger

这样做的好处就是比较干净，不好的地方就是每个地方要使用都需要 logging.getLogger() 一下。