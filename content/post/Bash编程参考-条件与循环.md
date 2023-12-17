---
title: Bash编程参考-条件与循环
toc: true
date: 2016-12-06 15:42:58
tags: [Linux, Shell, 运维]
categories: 
  - [Linux/Unix]
---
一直以来都有在用sh进行自动化的任务部署，运维管理监控，但是一直没有系统的去看一下bash的参考手册。有的时候，写的脚本并不能达到的以为的结果，所以才萌生了从头到尾看一下参考手册并进行总结归纳的念头。
条件与循环是每个编程语言、脚本语言，都不会缺少的语句，通过这些简单的语句，来实现复杂的逻辑，bash也不例外。
<!--more-->
# 条件判断
条件表达式（一元、二元）一般是通过**[[**、**test**、**[**命令使用的。一元表达式常用来测试文件状态，字符串操作符和数字比较操作符也是有的。对于符号链接，一般是其目标文件而不是链接本身。
## 逻辑判断
- **!** 非，取反的意思
- **( expr )** 用来改变优先级。
- **expr1 -a expr2** 与
- **expr1 -o expr2** 或
## 文件类型
- **-a file** 文件是否存在？
- **-e file** 文件是否存在(exist)，与**-a**等价）？
- **-b file** 文件是否为块设备(block)？
- **-c file** 文件是否为字符设备(char)？
- **-d file** 文件是否目录(directory)？
- **-f file** 文件是否常规文件(regular file)？
- **-h file** 文件是否是符号链接？
- **-L file** 文件是否是符号链接？
- **-p file** 文件是否是管道(PIPO)？
- **-t fd** 文件描述符fd是一个终端？
- **-S file** 是套接字文件？

## 文件状态
- **-r file** 文件可读？
- **-w file** 文件可写？
- **-x file** 文件可执行？
- **-O file** 当前用户的有效用户ID和文件的所有者ID是否一致？
- **-G file** 当前用户的有效组ID和文件的组ID是否一致？
- **-N file** 自上次访问后已被修改（文件访问时间小于修改时间）？
- **-s file** 文件大小大于0？
- **-u file** is setuid?
- **-g file** is setgid?
- **-k file** is sticky bit set?

## 字符
- **[-n] string**	string长度不为0？
- **-z string**	string长度为0？
- **string1 != string2** string1与string2不相等？
- **string1 < string2** string1<string2？（字典序）
- **string1 > string2** string1>string2？（字典序）
- **string1 == string2** **string1 = string2**	string1与string2相等？当与**[[**命令使用的时候，进行模式匹配。**=**应该与**test**命令一起用来兼容**POSIX**。
```sh
    [[ "good" == g* ]] && echo true || echo false  
    [ "good" == g* ] && echo true || echo false
```

## 变量检查
- **-o arg**	shell选项arg enable？
- **-v arg**	变量arg已设置（赋了一个值）？
- **-R arg**	变量arg已设置（赋了一个值）且是一个**name reference**？

## 算术比较
- *arg1 OP arg2*	OP={**-eq, -ne, -lt, -le, -gt, -ge**}，分别代表{**等于，不等于，小于，小于等于，大于，大于等于**}

# 算术运算
shell通过**((**命令，内建命令**let、declare -i** 进行求值。
以**0**开始的值被当做**八进制** 解释，**0x、0X**开头的当做**十六进制** 解释。一般的值表示是**[base#]n**，**base** 从2到64间的一个，作为算数进制的**基**；如果省略了**[base#]** ，那么，就是10进制的。
运算符的优先级、结合性、值和C语言一致。下面是一个由高至低的优先级排列，可以用**( )**来改变优先级。

| 运算符        | 说明  |
| ---------- | :-----:|
|id++ id-- |使用后自增/减 |
|++id --id |使用前自增/减 |
|- + | 正/负号|
|! ~ | 非 取反|
|** | 指数|
|* / % | 乘 除 取余 |
|+ - | 加 减|
|<< >> | 按位左移/右移|
|<= >= < > | 比较|
|== != | 等于 不等于|
|& | 按位与|
|^ | 按位异或|
|&#124; | 按位或|
|&& | 逻辑与|
|&#124;&#124; |逻辑或 |
|cond ? expr1 : expr2 | 条件运算|
|= *= /= %= += -= <<= >>= &= ^= &#124;=| 赋值|
|expr1, expr2|逗号运算|

# 条件与循环语句
## 循环语句
bash的循环跟C语言类似，有**until、while、for**三种，其基本格式为：
```sh
until condition; do cmd ...; done
while condition; do cmd ...; done
for name [ [in [words ...] ] ; ] do cmd ...; done
for i; do echo $i; done	#将会逐个输出位置参数 $1 $2 ...
for (( expr1; expr2; expr3 )) ; do cmd ...; done
```
## 条件语句
bash的条件语句有**if、case、select、(( ))、[[ ]]**。最简单的，当然是**if**。
### if
```sh
if condition1; then
    cmd1 ...;
[elif condition2; then
    cmd2 ...;]
[else cmd3 ...;]
fi
```
>对 **condition** 的不同形式请关注一下 [Shell编程与C的一些不同](/2016/11/13/Shell编程与C的一些不同/)

### case
```sh
case word in 
    [ [(] pattern [| pattern ...]) 
        cmd
	;; ] 
esac
```
> 通常用**\***来在最后表示默认动作。
> **;;（只执行匹配后语句）**可以用**;&（继续执行后面的语句）**或**;;&（继续匹配后面的条件）**结束[bash v4 才具有 ;& ;;&]。

### select
**select**你方便的生成菜单。
```sh
select NAME [in WORDS ...]; do COMMANDS; done
select sel in pwd date "ifconfig -a" "ping www.baidu.com"; do $sel; break; done
```
### (( ... ))
对算数表达式`expr`求值（bash特有，sh没有）。
`(( expr ))` 与 `let "expr"`完全等价。
### [[ ... ]]
对条件表达式求值（1 or 0）。


# 正则表达式
任何出现在一个模式内的字符，除了下面提到的特殊字符，匹配其自身。**NUL**字符不应该出现在一个模式中。一个`\`号会将下面的字符反引来匹配其自身。要匹配特殊模式字符，必须要进行引用。

* `*` 匹配任何字符，包括 null 字符。
* `?` 匹配任意单个字符
* `[...]` 匹配被包含字符中的一个。被连字符 `-` 连接起来的两个字符来就表示一个 RANGE EXPRESSION；这两个怎间的被排序，使用本地的排序和字符集进行匹配。如果在 `[` 后的第一个字符是 `!` 或 `^`，表示不匹配其中字符。当连字符是第一个或最后一个字符的时候，其也会被匹配。`]` 在作为第一个字符的时候也会被匹配。
比如，在默认的C语言环境中中，`[a-dx-z]` 等于 `[abcdxyz]`。很多语言以字典顺序排序字符，那么在这样的环境中 `[a-dx-z]` 并不等于 `[abcdxyz]`；有可能等于的是 `[aBbCcDdxXyYz]`。为了获得传统的范围解释，可以强制设置 `LC_COLLATE, LC_ALL` 环境变量为 `C` 来解决。
在 `[` 和 `]` 中，**字符类** 可以用 `[:CLASS:]` 来指定，**CLASS** 是POSIX标准中指定的一种：

```
alnum alpha ascii blank cntrl digit graph
lower print punct space upper word xdigit
```

`word` 类会匹配 字母，数字，以及 `_`。

如果 `extglob` shell 选项被 `shopt` 内建命令启用，几个扩展的模式匹配会被识别。在下面的描述中，一个 **PATTERN-LIST** 是一个表达式，或是多个以`|` 分隔的表达式。组合的模式，可能是被以下一个或多个子模式组成的：

`?(PATTERN-LIST)` 匹配0或1次给定模式
`*(PATTERN-LIST`  匹配0或多次给定模式
`+(PATTERN-LIST)` 匹配一次或多次给定模式
`@(PATTERN-LIST)` 匹配一次给定模式
`!(PATTERN-LIST)` 匹配给定模式之外的内容。

# ERE
shell默认使用的是ERE，比如我的博客的文件名是用日期加上标题命名的，我想要匹配的话，就可以这样写：

```bash
for f in `ls`; do
	if [[ $f =~ [0-9]{4}(-[0-9]{2}){2}-* ]]; then
		echo $f
	fi
done
```

**另外，在[[ ]] 中，已经不用把表达式进行 "" 引用了**