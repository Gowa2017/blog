---
title: macOS中的服务管理launchdctl手册
categories:
  - macOS
date: 2018-01-16 23:09:43
updated: 2018-01-16 23:09:43
tags:
  - macOS
  - launchd
---
对于CentOS中用systemctl来进行服务管理，又或者Solaris用的是SMF，但是对于macOS呢，其用的就是launchd来进行服务管理的，而用launchctl命令来进行服务的设置。本文是对`launchctl`手册的一个翻译。
<!--more-->

# NAME
`launch` -- `launchd`的接口

# 命令格式

	launchctl *subcommand* *[arguments ...]*
	
# 描述
`launchctl`接口通过`launchd`服务来管理和检查守护进程，代理服务和XPC服务。
# SUBCOMMANDS
launchctl 允许对 launchd 端口的详细测试。 一个`domain（域）`管理了一系列服务的执行策略。 一个服务可能会考虑称为一个虚拟进程以便其总能对请求进行响应。 每个服务有一系列端点，对这些端点发送消息会让服务启动。 主机建议这些端点在一个共享的命名空间内，并作为 `Mach bootstrap`子集同义。  

很多子命令都用一个指示符来表明目标**域**或**服务**。指示符可能是以下形式： 

*system/[service-name]* 指定**域**或**域**中的服务。`system`域管理`Mach 根启动器`，其被认为是一个特权上下文。任何用户都可以读取或查询`system`域，但是修改的话就需要特权。   
*user/\<uid\>/[service-name]*   目标域为`UID`指定的`user`域，或为域中服务。每个登录的用户都有一个用户域独立存在。`User`域在iOS内不存在。  
*login/\<asid\>/[service-name]*  指定一个`user-login`域或此域中某一服务。当一个用户在GUI登录，并被与此次登录相关联的审计会话标识符所区别的时候进行创建一个`user-login`域。如果一个`user`域与一个`login`域相关联，`print`子命令会显示这个`login`域的**ASID**。`user-login`在iOS内不存在。  
*gui/\<uid\>/[service-name]*  `login`标识符的另外一种形式。这个标识符基于用户来指定目标**域**，而不是通过`ASID`来指定`user-login`，这会更加的方便。  
> `GUI`域和`user`域共享很多资源。为了对`Mach bootstrap`名字进行寻找，他们共享同样注册名字的集合。但他们依然有不同的服务。所以呢，当打印`user`域的内容时，可能会看到很多在那个用户`gui`域中存在的`Mach bootstrap`名字注册，但是却看不到服务本身。  

*session/\<asid\>/[service-name]*  指定目标域为给定的审计会话ID或者此域中的一个服务。对于更多关于审计会话的信息，参考`auditon(2)`和`libbsm(3)`.  
*pid/\<pid\>/[service-name]*  指定目标域为指定的PID或者是此域中某一服务。系统中每个进程都有一个PID域与其关联，这由`xpc_connection_create(3)函数`能到达的进程可见的XPC服务组成。  

* `bootstrap | bootout domain-target [service-path service-path2 ...] | service-target`    
引导或移除域和服务。服务可能被一系列的路径或服务标识符区分。路径可能会指向XPC服务集（`launchd.plist(5)`），或者一个包含两者集合目录。如果在启动或者移除服务的时候出现错误，出现错误的路径将会与其发生的错误一起被打印出来。若没有指定路径或目标服务，这些命令可以启动或者移除域。某些域会隐式的启动一些预定义的路径，这作为域创建的一部分。    
* `enable | disable service-target`   
在请求的域内启用/停用服务。一个服务被禁止后将不能在特定的域内载入，比如在重新启动后才能载入。这个状态持久存在，即使设备重启。这个子命令只能指定*system, user, user-login*域中的服务。    
* `uncache service-name`    
让 launchd 绕过服务缓存，直接从磁盘读取服务的配置文件。launchd 维护一个 XCP服务配置文件的缓存区来减少磁盘I/O。这个命令删除一个缓存对象，开发者可以快速的观察服务的配置文件。在生产环境中不要使用。  
* `kickstart [-kp] service-target`  
立即启动指定的服务。  
**-k**  服务在运行，kill 了后重启服务。  
**-p**  成功时打印新进程的PID，或者打印已运行的PID到标准输出  
* `attach [-ksx] service-target`  
将系统调试器附在指定服务的进程上。默认情况下，如果服务未运行，这个子命令会睡眠直到服务启动。   
**-k**  服务运行的话，杀死后重启。  
**-s**  强制服务启动。  
**-x**  在执行和成为服务进程前先附到`xpcproxy(3)`。这个标志一般没有，只对 launchd 维护者有意义。  
* ` debug service-target [--program <program path>] [--guard-malloc] [--malloc-stack-logging] [--debug-libraries] [--introspection-libraries] [--NSZombie] [--32] [--stdin] [--stdout] [--stderr] [--environment] [--] [argv0 argv1 argv2 ...]`  
配置下一个服务的调试。这个子命令允许将服务的可执行程序用另外一个路径进行替换，启动`libgmalloc(3)`，设置环境变量，设置参数向亮等等。这是对编辑`launchd.plist(5)`并进行重载的另外一个方便选择，附加的调试属性在服务运行一次后就进行清楚。  
**--program <program-path>**  `launchd(8)`使用 *program-path*作为服务的可执行程序。  
**--guard-malloc**  为服务启用`libgmalloc(3)`。  
**--malloc-stack-logging**  对服务开启`malloc(3)`的栈记录。  

* `kill signal-name | signal-number service-target`  
发送指定信号到服务。  
* `blame service-target`  
如果服务运行，以人类可读的形式打印出为什么 launchd 会运行这个服务。服务可能会因为多种原因而运行；这个命令只会打印最近的那个。如果一个服务是因为定时器超时，这个子命令就会打印此原因，而不会考虑此服务是否在多个端点上有信息。生产环境中不应该依赖这个命令的输出。  
* `print domain-target | service-target`  
打印**域**或**服务**的信息。**域**输出包括很多关于域的属性和服务列表与到达每个服务端点。**服务**输出包含很多属性，在磁盘上的原始信息，当前状态，执行上下文，最后退出状态。  
>IMPORTANT: 这些输出不是 **API**。**不要**依赖输出的结构。因为这可能在版本间变化而不会进行警告。  

* `print-cache`  
打印 launchd 服务的缓存内容。 
* `print-disabled domain-target`  
打印指定域中禁用的服务列表。  
* `plist [segment, section] Mach-0`  
打印嵌入到Mach-0目标中`__TEXT, __info_plist`段/节中的属性列表。   
* `proinfo pid`  
打印指定PID的执行上下文，需要特权。    
* `hostinfo`  
主机信息。需要特权。  
* `resolveport owner-pid port-name`     
需要特权。通过PID和`属性port`来获得端点。  
* `examine [tool arg0 arg1 @PID ...]`  
* `config system | user parameter value`  
对`launchd(8)`域设置持久的配置信息。只能配置`system, user`域。持久存储的位置因实现而不同，对存储的改动只能通过 子命令进行。 要让变动生效，需要进行 reboot。  
支持的配置参数是：  
**umask**  将目标域中服务的`umask(2)`设置为*value*。  
**path**  将目标域内的所有服务的`PATH`环境变量设置为*value*。格式的话和`environ(7)`保持一致。如果某个服务设置了自己的`PATH`，其优先级将高于这个设置。  
* `reboot [system | userspace | halt | logout | apps]`  
卸载用户空间。无参数或者提供的是`system`参数，launchd会在用户空间卸载后调用`reboot(2)`。指定`halt`参数，会在用户空间卸载后调用`reboot(2)`系统调用和传递`RB_HALT`标志，挂起系统而不初始化一次重启。  

指定`userspace`参数，launchd在用户空间卸载重新执行其自身，然后再唤醒用户空间。这在内核数据和硬件不需要重新初始化的时候进行快速启动非常有效。  

指定`logout`参数，launchd 会关闭调用者的GUI登录会话，就跟从Apple菜单按钮上注销一样。不同的是，这会比点菜单快，也不会给应用展示一些其他信息的机会，这有可能造成一切数据冲突的可能。当你在确定没有未存储数据的时候再使用这个命令。  

指定`apps`参数，launchd会终止所有才调用者GUI登录会话中运行的且不是来自硬盘上`launchd.plist(5)`中的程序。像*Finder, Dock, SystemUIServer*这些应用不受影响。
**-s**  重启机器的时候（无论是完整的重启或用户空间重启），让接下来的启动会话进入单用户模式。  

* `error [posix | mach | bootstrap] code`  
对给定错误码打印出人类可读的信息。默认情况下，launchctl会尝试猜测错误代码属于哪一个错误域。也可以指定一个 error 域来对指定错误代码进行解释。  
* `variant`  
打印 launchd 中在系统上活跃的 变量。可能包含**RELEASE, DEVELOPMENT, DEBUG**。  
* `version`  
打印版本。  

# 传统命令

* `load [-wF] [-S 会话类型] [-D domain] paths ...`
载入指定的配置文件或目录中的配置文件。 所有指定的工作会在允许启动前就被载入。Jobs that are not on-demand will be started as soon as possible.每个用户的配置文件`（LaunchAgents）`必须被其用户所拥有以便载入。所有系统级的服务`（LaunchDaemons）`必须被被`root`所拥有。配置文件不能是 组 或 其他可写的。基于安全目的进行这些限制，因为允许对launchd配置文件的写入也就允许某个用户指定启动其他的一些程序。 

> 允许非root用户对`/System/Library/LaunchDaemeons`目录具有写权限将会导致系统无法启动。

**-w**  覆盖`Disabled key`并设置其为`false`。早些版本中，这个选项会修改配置文件。现在`Disabled key`的状态存储在磁盘的其他地方。  
**-F**  强制载入`plist`。忽略`Disabled key`。   
**-S sessiontype**  某些工作只对特定上下文敏感。这个标志会让`launchctl`在加上`-D`标志的时候在不同的位置去寻找工作，并允许`launchctl`来限制工作会再入哪种会话类型。当前已知的会话类型有：*Aqua, LoginWindow, Background, StandardIO, System*    
**-D domain**  在给定的主机上寻找以 `*.plist`结尾的 `plist(5)`文件。有效的主机包括*system, local, network, all*。 在指定一个会话类型的时候，一个额外的主机*user*可以被使用。举个例子，在没有指定会话类型的时候，`-D system`会在`/System/Library/LaunchDaemons`这个目录中的 属性列表文件（plists, property list files）载入文件。如果指定了会话类型的话，这会载入`/System/Library/LaunchAgents`下的plist文件。

* `unload [-W] [-S sessiontype] [-D domain] paths ...`  

卸载指定的配置文件或者目录。这会停止正在运行的任务。  

**-w**  覆盖`Disabled key`并设置其为`true`。早些版本中，这个选项会修改配置文件。现在`Disabled key`的状态存储在磁盘的其他地方。  
**-S sessiontype**  某些工作只对特定上下文敏感。这个标志会让`launchctl`在加上`-D`标志的时候在不同的位置去寻找工作，并允许`launchctl`来限制工作会再入哪种会话类型。当前已知的会话类型有：*Aqua, LoginWindow, Background, StandardIO, System*    
**-D domain**  在给定的主机上寻找以 `*.plist`结尾的 `plist(5)`文件。有效的主机包括*system, local, network, all*。 在指定一个会话类型的时候，一个额外的主机*user*可以被使用。举个例子，在没有指定会话类型的时候，`-D system`会在`/System/Library/LaunchDaemons`这个目录中的 属性列表文件（plists, property list files）载入文件。如果指定了会话类型的话，这会载入`/System/Library/LaunchAgents`下的plist文件。  

* `submit -l *label* [-p executable] [-o path] [-e path] -- command [args]`  
一个不在配置文件内进行设置而使某个程序运行的简单方式。这会告诉`launchd`在出现失败事件时保持程序存活。  
**-l label**  为这个任务指定的唯一标签（传递给`launchd`）。  
**-p program**  要执行的程序。无论在submit子命令中 -- 后面的任何参数。  
**-o path**  指定程序输出位置。  
**-e path**  指定程序的错误输出位置。  

* `remove job_label` 移除一个任务（通过标签）。
* `start job_label` 通过标签启动特定任务。这个命令主要是用来调试和测试以便用户能手动启动某个被需要的服务。  
* `stop job_label` 通过标签来停止指定服务。如果这个任务是有需求的，`launchd`会在这个任务满足条件的时候立即重新启动。不是立即需求的基本工作总是会被重启。这个命令不推荐使用。任务自己会超时停止。
* `list [-x] [label]`  没有参数的时候，用三列（PID, Status, Label）列出所有 launchd 载入的任务。PID会在运行的时候显示，否则就是一个`-`号。第二列指出任务上次的退出状态，如果是负值，则指定的是杀死任务的信号。比如，`-15`代表任务是被`SIGTERM`信号终止。第二列指的是任务标签。  

可能某些任务的风格是`0xdeadbeef.anony-mous.program`。这些是没有被 launchd 管理的任务但在某些时候对其发生了请求。 launchd 对这些任务没有权限也不保证什么。这些任务单纯只是做了一个记录。  

某些标签是`0xdeadbeef.mach_init.program`的任务是在`mach_init`模拟器下运行的任务。这个特性会在将来版本移除，所有的`mach_init`任务都会转换到launchd。  

如果 `[label]` 指定，那就打印特定的任务信息。如果`[-x]`指定，那么将以`XML`属性列表的形式输出信息。  
* `setenv key valud`  为 launchd 程序设置环境变量  
* `unsetenv key`  移除　launchd 程序的环境变量  
* `getenv key`  获取环境变量的值  
* `export`  导出所有的环境变量，以便在 shell的一个 eval 声明中使用  
* `getusage self l children`  获取 launchd 的资源使用状态或 其子进程的资源使用状态  
* `log [level loglevel] [onley | mask loglevels...]`  获取及设置`syslog(3)` 日志等级掩码。可用的级别是7个：*debug, info, notice, warning, error, critical, alert, emergency*  
* `limit [cpu | filesize | data | stack | core | rss | memlock | maxproc | maxfiles] [both [sort | hard]]`  没有参数时，这打印出所有通过`getrlimit(2)系统调用`获取的资源限制信息。当指定了其中的一个资源作为参数，就打印对应参数的信息。当有第三个参数的时候，软、硬限制设置为同一值。当有第四个参数是，第三、四参数分别代表了要设置的软、硬限制。 参考`setrlimt(2)系统调用`  
* `shutdown`  通知 launchd 移除所有任务，准备关闭。  
* `umask [newmask]`  获取或设置 launchd 的`umask(2) 文件掩码`  
* `bslist [PID | ..] [-j]`  被`print`命令代替。  
* `bsexec PID command [args]`    
* `asuser PID command [args]`  
* `bstree`  `print`命令进行了替代。  
* `managerpid`  `manageruid`  `managername` `help`  

# 警告
命令的输出不应该被脚本或者程序所依赖，因为其格式和内容是会变化的。

# 反对和移除的功能
launchctl 不再有交互模式，也不从标准输出接受命令。`/etc/launchd.conf`文件不再查询子命令执行；这是因为安全原因而移除。  

launchd不再使用 Unix domain sockets来进行通信。 

launchd 不再从网络载入配置文件。  

# 文件

     ~/Library/LaunchAgents         Per-user agents provided by the user.
     /Library/LaunchAgents          Per-user agents provided by the administrator.
     /Library/LaunchDaemons         System wide daemons provided by the administrator.
     /System/Library/LaunchAgents   OS X Per-user agents.
     /System/Library/LaunchDaemons  OS X System wide daemons.
     
# 退出状态
成功返回0，如果失败的话，返回的错误码可以给 `error`子命令进行解析。

# 相关阅读
*launchd.plist(5), launchd(8), audit(8), setaudit_addr(2)*

# 配置文件(launchd.plist(5))
一个被launchd管理的守护进程或代理期望以特定的方式表现。

我们在这把 守护进程或代理 称为**服务**。  

一个通过 launchd 启动的服务的进程**必须不能**在其进程内：

* 调用`daemon(3)`  
* 与`daemon(3)`等价的事情，包括`fork(2), exit(3), _exit(2)`。  

一个服务在初始化的时候**不应该**执行下面的过程，因为launchd总是会自动在进程中进行：  

* 重定向`stdio(3)`到`/dev/null`  

一个服务在初始化的时候**不需要**进行下面的操作，launchd会根据`launchd.plist`设置的键来确定是否需要执行： 

* 设置 userID/group ID  
* 设置 CWD
* `chroot(2)`
* `setsid(2)`
* 关闭迷路的文件描述符
* 用`setrlimit(2)`设置资源限制
* 用`setpriority(2)` 设置调度优先基

一个服务应该：  

* 在XML 属性列表中给定条件满足的时候启动。更多信息见后面。
* 捕捉`SIGTERM`信号，首选的是`dispatch(3)`，完成工作后快速的退出。

## XML属性列表键
下面的键用来描述服务的配置细节。属性列表是苹果的标准配置文件格式。查看`plist(5)`获取更多信息。
> 属性列表文件的名字应该用`.plist`结尾。如果服务的标签是*com.apple.sshd*，那么plist文件应该是*com.apple.sshd.plist*。 

* Lable <string>  唯一的标识 launchd 中的一个任务。
* Disable <boolean> 设置默认是否载入。可以用 `launchctl(3) enable Lable`进行重新配置，但不会写入配置文件。
* UserName <string> 指定运行服务的用户。只有对载入到特权的系统域中的服务有效。
* GroupName <string> 指定运行组。特权系统域中有效。如果指定*UserName*而不指定*GroupName*，那么其会被设置为*UserName*的组。
* inetdCompatibility <dictionary>  用来表明是不是要从 inetd 启动那样运行服务。新的项目应该避免使用这个键。
> Wait <boolean> 对应 inetd中的*wait, nowait*。
* LimitLoadToHost <array of strings>  不再支持
* LimitLoadFromHost <array of strings>  不再支持
* LimitLoadToSessionType <string or array or strings>  此配置文件只对指定会话类型应用。这一般只对代理程序应用。在特权的系统上下文中没有区别会话。
* LimitLoadToHardware <dictionary of arrays>  配置文件只应用到指定的硬件上。在字典内的每个键定义了`sysctl(3) hw域`中的一个子域。例如，键*machine*的值是*MacBookPro4,2*只会在机器的*hw.machine*值为*MacBookPro4，2*时应用。
* Program <string>  可执行程序绝对路径。`execv(3)`的第一个参数。如果这个键不存在，那么提供给*ProgramArguments*键的第一个元素就会被作为程序使用。如果没有*ProgramArguments*键，这个值是必须的。
* ProgramArguments <array of strings> `execvp(3)`的第二个参数，指定了参数向量。没有*Program*键时是需要的。
> 如果有些迷惑的话，仔细阅读`execvp(3)`。  
> *Program*必须是绝对路径。

* EnableGlobbing <boolean>  使用`glob(3)`来更新程序参数后再执行。
* EnableTransactions <boolean>  

# 搞不动了，自己参考 man 文档吧。
