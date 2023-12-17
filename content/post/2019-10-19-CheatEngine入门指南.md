---
title: CheatEngine入门指南
categories:
  - [Asm]
  - [Windows]
date: 2019-10-19 23:16:05
updated: 2019-10-19 23:16:05
tags:
  - 逆向
---

展示了一下如何使用 CheatEngine 来进行内存的查找工作。

<!--more-->

我们可以在 CE 主窗口的 help 菜单，打开一个示例程序。

[![Tutorials.CETutorialx32.01.png](https://wiki.cheatengine.org/images/f/f2/Tutorials.CETutorialx32.01.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.01.png)

然后点击左边的那个有放大镜的图标，选择'Tutorial-i386.exe'. 进程

# 文前

首先，我们要明确一个问题，就是对于一个程序来说，无论是其代码，还是数据，都是存储在内存中的。对于一个内存地址处的内容，其属于什么，是由程序来定的。同时，Windows 的程序是没有分段的，，CS,DS,ES,SS 都是指向的同一个地址。

另外，和我们用 C 写程序的时候不同，通过 C 生成的汇编代码，对于符号是什么，内存处存的是什么类型的东西，没有概念，他处理都只是字节数据。所以说，对于在 C 代码中我们可以明确使用的变量，在汇编后，其都只是一个地址。

对于自动变量，实际上所有的符号都会没有了，比如 `int a = 0`，转换成汇编代码的话就是这样的：

```asm
mov DWORD PTR -4[ebp],19
```

会从栈顶往下移 4 字节来保存值（因为栈是从高地址往低地址方向扩展，所以这实际上是指向栈底往上的双字（4 字节））。

寄存器只能存两种有意义的数据 ：1. 立即数。2. 内存地址。

一个内存地址是否是一个指针，就看汇编代码是否要对齐进行间接寻址了。

## Step 1 启动示例程序

当我们的示例程序启动，我们可以看到图片中的内容，我们只需要 **点击 next 按钮就行了**。

保存好我们设置的密码，也方便在我们崩溃（当进行注入的时候）后，能进行恢复到先前执行到的步骤。

[![Tutorials.CETutorialx32.02.png](https://wiki.cheatengine.org/images/b/be/Tutorials.CETutorialx32.02.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.02.png)

## Step 2 找值

步骤二有一些简单的说明

[![Tutorials.CETutorialx32.step01.01.png](https://wiki.cheatengine.org/images/a/ac/Tutorials.CETutorialx32.step01.01.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step01.01.png)

我们需要找到 health，这出是一个整型。

所以，**设置内存扫描器来查找一个 确切的，整型值，然后把要查找的值设置为我们看到的是 100。**，大多数的整型都会存储在一个 4 字节的变量中，所以我们就从这里开始。

> 整型可以存在在 1 字节的变量(byte)，2 字节的变量（int16/short)，4 字节的变量（int32/int），或者 8 字节的变量（int64/long）。

\*\* **点击 first scan 按钮**.

[![Tutorials.CETutorialx32.step01.02.png](https://wiki.cheatengine.org/images/c/c3/Tutorials.CETutorialx32.step01.02.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step01.02.png)

在左边的列表中我们会看到很多的地址：

[![Tutorials.CETutorialx32.step02.03.png](https://wiki.cheatengine.org/images/e/e7/Tutorials.CETutorialx32.step02.03.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step02.03.png)

现在我们回到示例程序 **点击 hit me 按钮**, 然后 **在 CE 重新键入 改变后的值，接着点击 next scan 按钮**.

注意那些红色的值，表示有了改变。

[![Tutorials.CETutorialx32.step02.04.png](https://wiki.cheatengine.org/images/9/9f/Tutorials.CETutorialx32.step02.04.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step02.04.png)

在点击了 **next scan** 后，我们可以会需要多次的在示例程序上 **点击 hit me 按钮**，然后重新搜索改变后的值，以让我们的搜索结果数量最小化。

[![Tutorials.CETutorialx32.step02.05.png](https://wiki.cheatengine.org/images/b/bd/Tutorials.CETutorialx32.step02.05.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step02.05.png)

只需要 **结果列表中的 Address 列就可以 将这地址添加到 cheat table**. 接着 **改变这个值，同时进行冻结此地址操作**。双击 value 列进行编辑（改变值），点击最前面的复选框可以设置是否冻结。

[![Tutorials.CETutorialx32.step02.06.png](https://wiki.cheatengine.org/images/1/1f/Tutorials.CETutorialx32.step02.06.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step02.06.png)

现 **示例程序上的 next 按钮就可以点击了，我们点击后进入下一步**。如果 next 按钮无法点击，再次点击一下 hit me 按钮试试。

## Step 3 未初始化的值

步骤 3 启动的时候看起来是下面这样的。我们不知道开始的时候我们要查找的值是什么样的，所以得用另外一种方式来进行查找。

[![Tutorials.CETutorialx32.step03.01.png](https://wiki.cheatengine.org/images/3/35/Tutorials.CETutorialx32.step03.01.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step03.01.png)

就像帮助文档所说的一样，在我们开始一次新的搜索时，我们需要 **点击 new scan 按钮**

[![Tutorials.CETutorialx32.step03.02.png](https://wiki.cheatengine.org/images/0/08/Tutorials.CETutorialx32.step03.02.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step03.02.png)

这会清空搜索的结果列表，同时开始一个新的内存扫描。

在此之前，我建议先在示例程序中 **点击一下 Hit me 按钮**，这样可以看看这个值是如何减少的，以此来决定我们要查找什么类型的值。

[![Tutorials.CETutorialx32.step03.03.png](https://wiki.cheatengine.org/images/5/56/Tutorials.CETutorialx32.step03.03.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step03.03.png)

**可以看到，值是按整型减少的**，它不是一个小数。

所以我们设置 **扫描器扫描 4 bytes 且 unknown initial value**.。然后 **点击 first scan 按钮**.

[![Tutorials.CETutorialx32.step03.04.png](https://wiki.cheatengine.org/images/a/a6/Tutorials.CETutorialx32.step03.04.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step03.04.png)

在 示例程序中 **点击 the hit 按钮**.

然后 在扫描器中设置**扫描类型为 decreased value** 接着 **点击 nest scan 按钮**.

[![Tutorials.CETutorialx32.step03.05.png](https://wiki.cheatengine.org/images/7/79/Tutorials.CETutorialx32.step03.05.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step03.05.png)

注意搜索结果的数量，这个看起来不多，但是对于当今的游戏来说，可能会出现上百万个结果。

现在 **通过在示例程序中多次 hit me 按钮, 扫描器持续扫描一个 decreased value**, 直到我们的扫描结果数量足够小。

[![Tutorials.CETutorialx32.step03.06.png](https://wiki.cheatengine.org/images/c/c2/Tutorials.CETutorialx32.step03.06.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step03.06.png)

如果扫描的结果够少了，那么就 **选择一个地址， 改变它的值， 看看是不是和我们所期待行为一致**,

**这里，我建议，在修改值前都进行一下复制，如果我们发现那个地址不是我们需要的，我们可以恢复它的值。**在搞游戏的时候，以此来避免同时很多个未知地址的值和损坏我们保存的文件。

这个时候，**示例程序中的 next 按钮在我们把正确地址的值设置为 5000 可用**.

在我们把值改变为 5000 和点击 hit me 按钮的时候这个进度条就会被拉满，但是这个并不需要。

[![Tutorials.CETutorialx32.step03.07.png](../res/Tutorials.CETutorialx32.step03.07-1571500115958.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step03.07.png)

现在 **next 按钮应该可用了，点击进入下一步**。如果 next 依然不可用的情况下，那么就再次点击 hit me 按钮。

## Step 4 浮点数

示例程序中在步骤 4 如同下面这样。

[![Tutorials.CETutorialx32.step04.01.png](https://wiki.cheatengine.org/images/d/de/Tutorials.CETutorialx32.step04.01.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step04.01.png)

所以很简单，设置扫描器**扫描 float，确定值，然后输入当前的 Health 值**。

设置好后就 **点击 First Scan 按钮**。

[[![Tutorials.CETutorialx32.step04.02.png](../res/Tutorials.CETutorialx32.step04.02-1571500441240.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step04.02.png)

和先前 **Step 2** 步骤所做的一样，找到内存地址，添加到 **Cheat Table**。

现在设置一个新的扫描，扫描器设置为 **double，确切值，输入 ammo 的值**。

设置好后就 **点击 First Scan** 按钮。

[![Tutorials.CETutorialx32.step04.03.png](../res/Tutorials.CETutorialx32.step04.03-1571500524602.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step04.03.png)

两个地址都找到后，就把他们的值都改成 5000,然后示例程序中的 next 按钮就可以点击了。我们点击进入下一步。

## Step 5 指令查找

步骤看起来如下。

[![Tutorials.CETutorialx32.step05.01.png](https://wiki.cheatengine.org/images/0/0d/Tutorials.CETutorialx32.step05.01.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step05.01.png)

我们先 **找到值，然后放到 Cheat Table，也就是地址列表中**。

我们在这里先保存一下这个地址列表，还有密码。

在地址列表中 **右键点击一个地址，选择 find out what accesses this address**。

[![Tutorials.CETutorialx32.step05.02.png](https://wiki.cheatengine.org/images/a/a6/Tutorials.CETutorialx32.step05.02.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step05.02.png)

在弹出来的提示框上点击 YES。

[![Tutorials.CETutorialx32.step05.03.png](https://wiki.cheatengine.org/images/5/53/Tutorials.CETutorialx32.step05.03.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step05.03.png)

然后我们就会弹出一个调试器窗口，把它拖到一边，回到我们的 CE 主窗口，然后改变选定的那个地址的值。在调试器窗口内就会出现汇编代码了。

我们需要的是一个写入指令。所以我们会寻找就像 这样的代码

```asm
mov [**],**
add [**],**
sub [**],**
*** [**],**
```

在 {% post_link X86汇编操作数寻址 X86汇编操作数寻址 %} 对这些指令有一些解释和说明。

X86 的汇编代码，一般是一 `指令 targert, source ` 形式执行的，比如 `mov ax, 09h` 就是将立即属 09h 移到 ax 寄存器的意思。

选择调试器内，汇编代码是写入指令的行，然后我们可以点击 **show disassembler 按钮** 来看内存中的代码，然后我们点击 **replace 按钮**。

接着不要忘记点击 **Stop ** 按钮。

[![Tutorials.CETutorialx32.step05.04.png](https://wiki.cheatengine.org/images/9/95/Tutorials.CETutorialx32.step05.04.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step05.04.png)

**Replace** 按钮会将那行指令以 [NOPs](https://wiki.cheatengine.org/index.php?title=Assembler:Commands:NOP) 替换

同时 CE 会提示我们将这个替换用一个名字来记录在 adavaced options 表中。

**输入一个名称后点击 OK 按钮**.

[![Tutorials.CETutorialx32.step05.05.png](https://wiki.cheatengine.org/images/2/20/Tutorials.CETutorialx32.step05.05.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step05.05.png)

回到事例程序，**点击 Change Value**按钮。

**Next** 按钮现在可用，我们直接点击进入下一步。

、

我们可以在 CE 主界面的左下角 **Advanced Opions** 打开替换代码的窗口列表。

[![Tutorials.CETutorialx32.step05.06.png](https://wiki.cheatengine.org/images/f/f9/Tutorials.CETutorialx32.step05.06.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step05.06.png)

如果我们想要取消掉我们替换的指令，右键一项，然后选择 **restore with original code** 项。

[![Tutorials.CETutorialx32.step05.07.png](https://wiki.cheatengine.org/images/3/3d/Tutorials.CETutorialx32.step05.07.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step05.07.png)

颜色变黑。

[![Tutorials.CETutorialx32.step05.08.png](https://wiki.cheatengine.org/images/2/2d/Tutorials.CETutorialx32.step05.08.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step05.08.png)

## Step 6 指针

步骤 6 是下面这个样子。

[![Tutorials.CETutorialx32.step06.01.png](https://wiki.cheatengine.org/images/c/ca/Tutorials.CETutorialx32.step06.01.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step06.01.png)

首先查找到值，然后把地址添加到 **Address List**。

在地址列表中右键点击地址，然后选择 **Find out what accesses this address **。

[![Tutorials.CETutorialx32.step06.05.png](https://wiki.cheatengine.org/images/c/c7/Tutorials.CETutorialx32.step06.05.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step06.05.png)

示例程序中点击 **Change Value**，调试器会获取到修改代码。

**当我们选择要用来寻找指针的基址的指令代码时，请尝试选择一条不会与基址写入同一寄存器的指令。**

通过前后文的比较，我认为这里所说的 **基址**，指的是值，在内存中的地址。止不过，有的时候，这个内存中的地址不是直接访问，而是通过一个地址+偏移进行访问的。比如说一个值是在结构体中，或者类中的时候，就需要拿到这个结构体的地址，或者类的地址来进行偏移获取。这个概念要区别与寄存器寻址的时候所使用的段地址。

这里我们感兴趣的是用 `[ ]` 包起来的值，所以在我们的例子中我们需要的是 EDX 的值。

[![Tutorials.CETutorialx32.step06.02.png](https://wiki.cheatengine.org/images/9/91/Tutorials.CETutorialx32.step06.02.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step06.02.png)

在这里我们看出偏移量是 0,如果指令的格式如下：

```asm
mov [edx+12C],eax
```

偏移量将会是 12C，16 进制的值。

现在我们来设置扫描器，**搜索 4 字节，确切值，勾选 Hex，然后把从 EDX 处拿到的值输入进去**。

点击 **first scan 按钮**.

**在结果列表中选择绿色的地址，这些都是静态地址**。

[![Tutorials.CETutorialx32.step06.03.png](https://wiki.cheatengine.org/images/3/3e/Tutorials.CETutorialx32.step06.03.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step06.03.png)

很多时候，我们需要的都可能是最小的那个地址。

我们现在添加指针基址。

1. 双击结果列表中的一个绿色地址，添加到地址列表中。
2. 双击地址列表中的 Address 列。
3. 在弹窗中，将 Address 进行复制。格式如下：`["Tutorial-i386.exe"+XXXXXX]+0`
4. 勾选 Pointer ，将复制后的地址粘贴到最后一个框内。这出我们的偏移量是 0 就不用进行设置 offset 了。

[![Tutorials.CETutorialx32.step06.04.png](https://wiki.cheatengine.org/images/0/02/Tutorials.CETutorialx32.step06.04.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step06.04.png)

点击 OK，回到 CE 主界面。

现在我们把这个值改为 5000,并冻结，然后回到示例程序中，点击 **Change Pointer\***按钮， Next 按钮就会可用了。

如果这样做了以后 Next 按钮还是不可用，那说明可能我选择的绿色地址错误，重新选择一个进行操作试试

## Step 7 代码注入

步骤 7 看起来是下面这样的

[![Tutorials.CETutorialx32.step07.01.png](https://wiki.cheatengine.org/images/f/fc/Tutorials.CETutorialx32.step07.01.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step07.01.png)

这里我们按照步骤 5 进行操作，不过在 调试的时候，我们不再选择 replace 按钮，而是点击 **show disassembler 按钮**。

[![Tutorials.CETutorialx32.step07.02.png](https://wiki.cheatengine.org/images/b/b4/Tutorials.CETutorialx32.step07.02.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step07.02.png)

这会从指令地址处打开反汇遍窗口。

[![Tutorials.CETutorialx32.step07.03.png](https://wiki.cheatengine.org/images/2/23/Tutorials.CETutorialx32.step07.03.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step07.03.png)

**在选中的指令上执行 Ctrl +A** 来打开一个自动汇编窗口。

在自动汇编窗口菜单上选择 Template-> Full injection

[![Tutorials.CETutorialx32.step07.04.png](https://wiki.cheatengine.org/images/b/b8/Tutorials.CETutorialx32.step07.04.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step07.04.png)

这会生成一个自动的脚本。

[![Tutorials.CETutorialx32.step07.05.png](https://wiki.cheatengine.org/images/7/78/Tutorials.CETutorialx32.step07.05.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step07.05.png)

现在我们需要 **添加一些将值增加 2 的代码**，然后移除以前那些减少值的代码。

想要增加值我们使用 `INC, ADD` 指令。

现在让我们来试试 这段代码这样的内容：

```asm
...
newmem:
  add [ebx+478],2 //// Here Cheat Engine will assume that the value size is 4 bytes (dword)

code:
  //sub dword ptr [ebx+00000478],01
  jmp return

address:
  jmp newmem
  nop
  nop
return:
...
```

现在我们将这个脚本添加到我们的 cheatTable。在自动汇编窗口选择 File->Assign to current cheat table。

然后启用这个脚本，再在示例程序中点击 Hit Me。

Next 按钮可用，点击进行下一步。

## Step 8 多级指针（指针的指针的指针等等）

步骤 8 如下

[![Tutorials.CETutorialx32.step08.01.png](https://wiki.cheatengine.org/images/3/3a/Tutorials.CETutorialx32.step08.01.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step08.01.png)

同步骤 6 一样，我们找到值的地址后，我们继续查找是哪些在访问这个地址，然后再继续往上查找，直到找到一个静态地址。

在第一次进行地址扫描中确实找到了一个静态的基地址，但是我记得这是一个假的基地址。所以在这里我需要的是一个类似 `'process.exe+offset` 这样的基地址，你可以尝试类似 `module.dll+offset` 这样的静态基地址，但是我想说的是，他们也是假的指针。很多新的游戏都会有很多这种假的指针和值。

### 值的地址 01870AD0

找到值的地址为：**01870AD0**。我们接下来要做的事情就是：看这个地址处的值是如何被改变的。F6 调试器输出。

```asm
00428144 - B8 A00F0000 - mov eax,00000FA0
00428149 - E8 826BFEFF - call Tutorial-i386.exe+ECD0
0042814E - 89 46 18  - mov [esi+18],eax <<
00428151 - 8D 45 D4  - lea eax,[ebp-2C]
00428154 - E8 E7B7FDFF - call Tutorial-i386.exe+3940

EAX=0000087F
EBX=018BCBC8
ECX=00000000
EDX=0000087F
ESI=01870AB8
EDI=00617D78
ESP=0165F5C4
EBP=0165F700
EIP=00428151
```

```asm
mov [esi+18],eax <<
```

这一句可以看出来，是通过 ESI=01870AB8 进行地址偏移后，才得到了我们的值的地址，进行了修改操作。现在我们就需要找到 ESI=01870AB8 这个地址又是什么东西呢，可以认为，这个地址一般也是一个对象地址，而不是代码地址。

因为，在我们进行调试改变 **01870AD0** 地址处的值的时候，我们是无法知道 ESI 是否会被改变的，所以我们只能看看，其被哪些代码进行了访问。

### lv1base = 01870AB8

lv1base 通过偏移 18 即可得到我们最终的值的地址，我们需要知道 lv1base 又是存在在什么地方的呢？

我们通过扫描器来拿到其地址：018790D0

### lv2base

现在我们就需要知道 lv1base 是被怎么样访问的。

##

With that static address as the base my pointer will look like this.

```
[[[["Tutorial-i386.exe"+XXXXXX]+C]+14]+0]+18
```

[![Tutorials.CETutorialx32.step08.02.png](https://wiki.cheatengine.org/images/7/7b/Tutorials.CETutorialx32.step08.02.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step08.02.png)

**After you have found the pointer, freeze it at 5000, then click the change pointer button**. If you found the right base the next button should become enabled after about 2 seconds. So **click the next button** to go to the next step.

## Step 9[[edit](https://wiki.cheatengine.org/index.php?title=Tutorials:Cheat_Engine_Tutorial_Guide_x32&action=edit&section=9)]

When you start step 9 you should see the form looking like this.

[![Tutorials.CETutorialx32.step09.01.png](https://wiki.cheatengine.org/images/d/da/Tutorials.CETutorialx32.step09.01.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step09.01.png)

So here like the help text says there is far more then one solution.

First we need to **find one of the addresses and add it to the table**.

If you are having trouble finding an address, remember to try different value types, and don't forget to start *new scan*s.

Then like in step 7 we want to **see what accesses the address**, to find the function that writes to the actor's health.

Go ahead and **save the password** if you want to try different ways, this is the last step in the tutorial.

So here it's good to understand what we're actually looking for to tell allies and combatants apart.

When the game or engine is written, actors and players mite be written like this.

//// Actor, base for all actors class Actor(object){ string Name = 'Actor'; Coord Coords = new Coord(0, 0, 0); float Health = 100.0; ... } //// Player class Player(Actor){ //// Player inherits form Actor string Name = 'Player'; int Team = 1; ... }

The team it self could be a structure, say if it's declared as an object class like the 'Coords' variable, which we would want to look for a pointer to the actor's team structure.

So one way we could do this is to find the team id or team structure in the player structure.

### Find the team id in the player structure[[edit](https://wiki.cheatengine.org/index.php?title=Tutorials:Cheat_Engine_Tutorial_Guide_x32&action=edit&section=10)]

After you have found the function that decreases health.

**Right click the instruction in the disassembler view form, and select find out what addresses this instruction accesses**.

[![Tutorials.CETutorialx32.step09.02.png](https://wiki.cheatengine.org/images/3/3c/Tutorials.CETutorialx32.step09.02.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step09.02.png)

Then **click the attack button for all 4 values**.

You should have all 4 addresses in the debugger list.

[![Tutorials.CETutorialx32.step09.03.png](https://wiki.cheatengine.org/images/9/9f/Tutorials.CETutorialx32.step09.03.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step09.03.png)

So go ahead and **add them to the address list**.

[![Tutorials.CETutorialx32.step09.04.png](https://wiki.cheatengine.org/images/5/5a/Tutorials.CETutorialx32.step09.04.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step09.04.png)

Then let's open the dissect data structure form.

[![Tutorials.CETutorialx32.step09.05.png](https://wiki.cheatengine.org/images/d/da/Tutorials.CETutorialx32.step09.05.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step09.05.png)

You'll get some pop ups, after going thought them you should see a form like this. Note that I had to expand the width of the form to be able to move the columns.

[![Tutorials.CETutorialx32.step09.06.png](https://wiki.cheatengine.org/images/1/14/Tutorials.CETutorialx32.step09.06.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step09.06.png)

So here we can see that the team variable is at offset 0x10 of the structure.

Now we need to **add some injection code to a script**, then **add some code that checks the team variable of the structure**, to determine which actors are allies and which are combatants.

So we want some this like this.

[![Tutorials.CETutorialx32.step09.07.png](https://wiki.cheatengine.org/images/7/74/Tutorials.CETutorialx32.step09.07.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step09.07.png)

So with this script enabled, when the game writes to an actors health here is what will happen after the jump to the hook code:

1. Save ([PUSH](https://wiki.cheatengine.org/index.php?title=Assembler:Commands:PUSH)) the EFLAGS register, not completely needed but still a good habit when comparing.
2. Check if actor is on team 1.
   1. If actor is on team 1, then we set the new value to 5000 in a floating point format.
3. Check if actor is on team 2.
   1. If actor is on team 2, then we set the new value to 0 in hex format. (float 0 == int 0 == hex 0)
4. Restore ([POP](https://wiki.cheatengine.org/index.php?title=Assembler:Commands:POP)) the EFLAGS register, this is completely needed if the register was [PUSHed](https://wiki.cheatengine.org/index.php?title=Assembler:Commands:PUSH).

**With this script enabled, click the restart game and autoplay button**, then you should see the form change and look like this.

[![Tutorials.CETutorialx32.step09.08.png](https://wiki.cheatengine.org/images/4/44/Tutorials.CETutorialx32.step09.08.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step09.08.png)

So **click the next button** to complete the tutorial.

Then you should see a form telling you that you have completed the tutorial.

### Find a difference in the registers[[edit](https://wiki.cheatengine.org/index.php?title=Tutorials:Cheat_Engine_Tutorial_Guide_x32&action=edit&section=11)]

After you have found the function that decreases health.

**Right click the instruction in the disassembler view form, and select find out what addresses this instruction accesses**.

[![Tutorials.CETutorialx32.step09.02.png](https://wiki.cheatengine.org/images/3/3c/Tutorials.CETutorialx32.step09.02.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step09.02.png)

Then **click the attack button for all 4 values**.

You should have all 4 addresses in the debugger list.

[![Tutorials.CETutorialx32.step09.03.png](https://wiki.cheatengine.org/images/9/9f/Tutorials.CETutorialx32.step09.03.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step09.03.png)

Now let's look at the registers to see if we can find a difference in the allies and combatants.

**Select each address individually and press Ctrl+R**.

Arrange the forms to make it easier to compare.

[![Tutorials.CETutorialx32.step09.b.01.png](https://wiki.cheatengine.org/images/9/95/Tutorials.CETutorialx32.step09.b.01.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step09.b.01.png)

So here we can see that ESI is 1 for the combatants.

So a script like this should work.

[![Tutorials.CETutorialx32.step09.b.02.png](https://wiki.cheatengine.org/images/6/61/Tutorials.CETutorialx32.step09.b.02.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step09.b.02.png)

So with this script enabled, when the game writes to an actors health here is what will happen after the jump to the hook code:

1. Save ([PUSH](https://wiki.cheatengine.org/index.php?title=Assembler:Commands:PUSH)) the EFLAGS register, not completely needed but still a good habit when comparing.
2. Check if ESI register is 1.
   1. If ESI register is 1, then we set the new value to 0 in hex format. (float 0 == int 0 == hex 0)
   2. If ESI register is not 1, then we assume the actor is an ally so we set the new value to 5000 in a floating point format.
3. Restore ([POP](https://wiki.cheatengine.org/index.php?title=Assembler:Commands:POP)) the EFLAGS register, this is completely needed if the register was [PUSHed](https://wiki.cheatengine.org/index.php?title=Assembler:Commands:PUSH).

**With this script enabled, click the restart game and autoplay button**, then you should see the form change and look like this.

[![Tutorials.CETutorialx32.step09.08.png](https://wiki.cheatengine.org/images/4/44/Tutorials.CETutorialx32.step09.08.png)](https://wiki.cheatengine.org/index.php?title=File:Tutorials.CETutorialx32.step09.08.png)

So **click the next button** to complete the tutorial.

Then you should see a form telling you that you have completed the tutorial.
