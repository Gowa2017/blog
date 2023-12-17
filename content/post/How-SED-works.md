---
title: How SED works
toc: true
date: 2016-11-24 15:18:22
tags: [SED, Linux, 运维, Shell]
categories: 
- [Linux/Unix]
---
Linux下三大神器之一的`sed`，为你处理字符，批量替换，提供了超级效率的操作方法。
`sed`是面向`流(stream)`，更确切的说是`字符流`，以行为单位进行操作。循环直至结束。
<!--more-->
# 工作机制
sed 使用了两个数据缓存器  活动的 __pattern space__, 辅助的 __hold space__ 。其刚开始命令的时候都是空的。  
sed 按以下循环进行工作:  

1. sed读取输入文件的一行，移除尾部的换行符，然后放入__pattern_space__。  
2. 执行命令。每个命令都可以指定执行的位置。  
3. 执行完毕后，如果没有打开`-n`选项就会将 __pattern space__ 内容进行输出，并加上删除掉的换行符。  
4. 回到 **[1.]**   

处理完一行，通常(某些情况我们可以使它保留)是将 __pattern_space__ 清空，但并不清空 __hold_space__。  
对于每一个命令，我们都可以给它指定执行的范围（某一行，或，某一范围）。
> 理解 sed 每次只会读取输入的一行到缓冲区，对与理解 `n` 和`N`命令非常有用。

# 命令格式
通常情况下我们是这样使用sed的：

	sed OPTIONS... [SCRIPT] [INPUTFILE...]

## Options
OPTIONS可以说如下值[多个]

`--help` 帮助  
`--version` 版本信息  
`-n --quiet --silent` 默认情况下，完成一行的处理后，sed默认打印pattern space的内容，此选项将关闭默认打印。  
`-e SCRIPT --expression=SCRIPT` 指定执行的命令，可以多次使用。当然，也可以用{}进行分组指定。  
`-f SCRIPT-file --file=SCRIPT-file` 在文件中指定执行命令。  
`-i` 直接修改输出到文件，而不是到标准输出  
`-l N | --line-length=N` 指定`l`命令换行的长度。长度为0表明从不换行，如果没有指定的话，值就是70。  
`-b --binary` 当操作系统区别文本文件和二进制文件的时候有用。  
`--follow-symlinks` 指定`-i`选项的时候，如果输入文件是一个符号链接则会跟随链接到文件。默认情况是不跟随。  
`-E | -r | --regexp-extend` 扩展正则表达式。  
`-u | --unbufferd` 尽量少的缓存输入和输入。(当用`tail -f`作为输入的时候，这个选项可以让你尽快的看到结果)
## 退出状态
- 0 success  
- 1 Invalid command  
- 2 One or more input file specified could not be opend 
- 4 An I/O error。

# 范围的指定
我们可以指定动作执行的范围（文件中的哪些行），及需要执行的命令（ __pattern_space__ )上执行。  
`number` 一个确切的数字，指定某行  
`$` 最后一行  
`first~step` first=起始 step=步长，想取奇数行就用 1~2,每五行就用 1~5。 __GNU SED扩展__  
`/regexp/` 满足正则表达式的行。如果正则表达式内包含`/`，需要使用`\`进行转义。  
`\%regexp%` 为了避免上面那样的反引，你可以用任何一个符号来代替`%`。  
`/regexp/I` `\%regexp%I` 以大小写不敏感的方式来匹配正则式。 __GNU 扩展__
`/regexp/M` `\%regexp%M` 多行模式匹配正则式。这个时候`^`表示其前有一个换行，`$`表示其后跟随一个换行。 __GNU扩展__   
`ADDR1,+N` `ADDR1`及其后的N行。 __GNU 扩展__  
`ADDR1,~N` 找到ADDR1后的，继续匹配直到是N的倍数的行。 _注意与first~step区别,这条是匹配到了就不继续匹配了,而first-step是匹配到结束。_ __GNU 扩展__  
`!` 跟随在一个地址范围后，其意义是就是不匹配地址范围表达式的才会被选择。 __GNU扩展__  

# 常用的命令
`#` 注释  
`q` [EXIT-CODE] 只接受一个地址范围参数，打印当前 __pattern_ space__ 后退出，并返回退出码。 __GNU扩展__  
`d` 删除当前 __pattern_ space__，立即开始处理下一行，不会继续执行之后的命令。 

	seq 3 | sed 2d 
	seq 3 | sed 
`D` 如果 __pattern_ space__ _没有_ 换行符，等同于`d`命令。否则，将 __pattern_ space__ 内容删除到第一个换行符为止，然后重新在 __pattern_ space__ 上执行命令,并不会读入下一行到 __pattern_ space__。  
`p` 打印当前 __pattern_ space__ 到标准输出，一般跟 `-n`选项配合使用  

	seq 3 | sed -n 2p
`P` 打印 __pattern_ space__ 到出现第一个换行符的位置  
`n` 如果自动打印没有被`-n`关闭，此命令会打印当前 __pattern_ space__， 然后把内容替换为下一行，如果没有输入了，就停止执行命令。  
这个命令对于跳过某些行非常有用，比如处理每 `Nth`行。  

	seq 6 | sed 'n;n;s/./x/'
`N` 给 __pattern_ space__ 添加一个换行符 同时将下一行的输入读进来。如果没有输入，不执行命令即退出。  
`{commands}` 一连串命令集合，以';'分割开的多条命令，在 __pattern_ space__ 上执行。 __GNU特性__  

	seq 3 | sed -n '2{s/2/X/ ; p} 

---
## __`s` 替换命令__
命令原型:

	s/REGEXP/REPLACEMENT/FLAGS 
`/`号同样可以用 其他符号代替，如:# % 等等避免`/`需要用转义
__工作流程是__
用REGEXP匹配 __pattern_ space__ 如果成功，就用REPLACEMENT替换。
REGEXP 可以用 '\(' '\)'进行分组，
在REPLACEMENT，可以用 \1..9 进行引用，&代表整个匹配的内容

在 __GNU扩展__ 内 你还用一组'\'跟'L','l','U','u','E' 组成的序列。但这些并不常用。
`\L` 将REPLACEMENT变换为小写，直到 `\U` or `\E`出现。
`\l` 将REPLACEMENT中的下一个字符转变为小写.
`\U` 将REPLACEMENT变换为大写，直到 `\L` or `\E`出现。
`\u` 将REPLACEMENT中的下一个字符转变为大写.
`\E` 在REPLACEMENT中结束`\L`,`\U`的作用。

__FLAGS：__
`g` 替换 __pattern_ space__ 内所有匹配的位置。不加此标志，只替换第一次匹配的位置
`NUMBER` 替换 __pattern_ space__ 内第`NUMBER`匹配位置。（POSIX内并 __没有定义__ NUMBER 与g一起使用会是什么情况，__GNU内，则表示从第NUMBER到最后一个匹配的位置__)
`w FILENAME` 如果s命令成功执行，将输出写到文件内。__GNU扩展 支持使用`/dev/stdout /dev/stderr`__
`e` 如果s命令匹配成功，在 __pattern_ space__ 内的内容将会被当作命令执行，同时 __pattern_ space__被替换为命令的输出。 __GNU 扩展__
`p` 打印出匹配后的 __pattern_ space__。同时指定`ep` 或`pe` 效果是不一样的。pe将会打印找到的命令，然后打印e输出，ep只是打印e的输出
`I,i` 以大小写不敏感的方式来匹配正则式。__GNU 扩展__ 
`M,m` 多行模式匹配正则式。这个时候`^`表示其前有一个换行，`$`表示其后跟随一个换行。__GNU扩展__ 

## `y` 命令

	y/SOURCE_CHARS/DEST-CHARS/
逐个替换为对应位置的字符比如 y/abc/ABC/ 凡 a被替换为A b替换为B
## 其他命令
**a\  
TEXT** (append)在当前循环输出后面加上TEXT。如果你用`-n`命令关闭了打印的话，则只打印`TEXT`。  
	
	seq 3 | sed '2a\hello'
**i\  
TEXT** (insert)在当前输出前加上`TEXT`。 

	seq 3 | sed '2i\hello' 
**c\  
TEXT** 删除 __pattern_ space__ 打印`TEXT`  

	seq 3 | sed '2c\hello'
`=` 打印当前输入的是第几行，后面跟随一个换行符
`l N` 清晰的模式打印 __pattern_ space__ ，`N`是行宽，中文全变成\232 \245这样的的了。行尾加上$
`r FILENAME` 读取文件到当前循环的输出流后。文件名不存在也没事，就当读了个空。 __GNU扩展支持 /dev/stdin__  
	
	seq 3 | sed '2r/etc/passwd'
`w FILENAME` 将 __pattern_ space__ 写到文件。 __GNU扩展支持/dev/stderr /dev/stdout__ 文件将被创建（当不存在）或被截短（当已存在），在没有读入行之前。所有w指令(包括s成功执行的w标志)不会关闭或者重新打开文件（提高效率）  

	seq 3 | sed '2woutfile'
`h` 用 __pattern_ space__ 的内容替换 __hold_ space__  
`H` 在 __hold_ space__ 后加一个换行符 同时把 __pattern_ space__ 的内容copy 过来  
`g` 用 __hold_ space __的内容替换 __pattern_ space__的内容  
`G` 在 __pattern_ space__ 后加一个换行符 同时把 __hold_ space__ 的内容copy 过来  
`x` 交换 __pattern_ space__ 和 __hold_ space__的内容  
`:` LABEL 设置个标签  
`b` LABEL 无条件跳往 LABEL标签。下一循环开始的时候，将会忽略这个标签  
`t` LABEL s成功执行后跳转到LABEL标签  

# 标签使用
<<<<<<< HEAD
标签主要使用 `:, t, b`三个命令实现的，分别是 **设置标签，替换成功后跳转，无条件跳转**三个意思。

=======
>>>>>>> 7fa33ff02a6d163b73353e437388e527d9d7fefe
使用标签可以让我们进入大师的行列。我们要明白的是，我们设置的标签，只是在当前的buffer内执行动作。比如，我们要给 123456789 加上千分号，变成 `123,456,789`这样：

```bash
sed -e :a -e 's/\(.*[0-9]\)\([0-9]\{3\}\)/\1,\2/;ta' file
```
这先执行一个标签，然后匹配 **pattern_space**中的内容并进行替换，如果替换成功了，就跳转到标签，继续进行替换。有点类似 while循环的意思。

**居中**  

我们可以以79个字符为宽度把文本居中：

```bash
sed -e :a -e 's/^.\{1,77\}/ & /' 
```
