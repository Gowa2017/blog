---
title: GNU-Parallel并行计算提高大文件处理效率
categories:
  - Linux/Unix
date: 2020-09-01 17:00:11
updated: 2020-09-01 17:00:11
tags:
  - Linux/Unix
  - GNU Parallel
---

之所以遇到这个问题，是因为，要对一个比较大的文件进行分析，统计。5 亿行数据，20G 左右。采用 awk 进行分列，归集，效率低得发指，用 `top` 命令看了一下情况，发现只用了 100% 的 CPU，也就是说在一个核上跑，这是万万不能接受的。

<!--more-->

# 项目地址

[GNU Parallel](https://www.gnu.org/software/parallel/)

**GNU parallel** 是一个用来在单个或多个计算机上并行执行任务的 shell 工具。一个任务可以一单个的命令或者一个小小的脚本，他们都必须在输入的每一行上进行执行。典型的输入是 _一个文件列表，一个主机的列表，用户的列表，等等_。一个任务也可以是一个从管道读取输入的命令。 **GNU Parallel** 可以将输入进行分块然后并行的通过管道传递给命令。

# 文件格式

我 的文件格式是这样的：

```
TASK_TYPE,USER_NAME,RECEIVE_DATE
512,"17772044354","2019-01-01 00:00:01"
512,"13317675924","2019-01-01 00:00:01"
1102,"19142780897","2019-01-01 00:00:02"
512,"18978149422","2019-01-01 00:00:03"
512,"18176135815","2019-01-01 00:00:03"
512,"19977181430","2019-01-01 00:00:04"
512,"17707888660","2019-01-01 00:00:07"
512,"13377499065","2019-01-01 00:00:07"
512,"19978869062","2019-01-01 00:00:07"
```

我需要过滤出 User_NAME 列包含特定的字符的数据，然后再统计 task_type 类型的数量。

# awk 统计

```sh
awk -F',' '(toupper($2) ~ /.*@IPTV.*/){a[$1]++} END{for (it in a) print it ":" a[it]}' file.csv
```

很慢，半天不出效果，我居然过了 40 分都没有出结果？难道是我哪里错了。

# python 实现

我用 python 来逐行读取进行处理

```python
def reduce():
    r = dict()
    with open(FILE, 'r') as f:
        i = 0
        for line in f:
            t, u, _ = line.strip().split(',')
            if 'TASK_TYPE' in t:
                continue
            i = i+1
            if i % 1000000 == 1:
                print('%s' % datetime.now(), i)
            if '@IPTV.GX' not in u.upper():
                continue
            if not r.has_key(t):
                r[t] = 0
            r[t] = r[t] + 1
        print(r)
```

执行大概用了：20 分钟，也不慢了。

# parallel + awk

现在我们就来看奇迹的一刻：

```sh
time cat worksheet20200901.csv | bin/parallel -q -m -u -j 80 --pipe  awk -F',' '(toupper($2) ~ /.*@IPTV.*/){a[$1]++} END{for (it in a) print it ":" a[it]}'  | awk -F':' '{a[$1]+=$2} END{for (it in a) print it ":" a[it]}'
```

结果：

```
real    2m55.864s
user    52m2.883s
sys     6m28.013s
```

实际执行时间，在 3 分钟左右，效率还是大大的提高了。不过，奇怪的是，我这有 48 个逻辑 CPU 的设备上，只给了 16 个进程起来，为何？

# parallel 命令说明

```sh
bin/parallel -h
Usage:

parallel [options] [command [arguments]] < list_of_arguments
parallel [options] [command [arguments]] (::: arguments|:::: argfile(s))...
cat ... | parallel --pipe [options] [command [arguments]]

-j n            Run n jobs in parallel
-k              Keep same order
-X              Multiple arguments with context replace
--colsep regexp Split input on regexp for positional replacements
{} {.} {/} {/.} {#} {%} {= perl code =} Replacement strings
{3} {3.} {3/} {3/.} {=3 perl code =}    Positional replacement strings
With --plus:    {} = {+/}/{/} = {.}.{+.} = {+/}/{/.}.{+.} = {..}.{+..} =
                {+/}/{/..}.{+..} = {...}.{+...} = {+/}/{/...}.{+...}

-S sshlogin     Example: foo@server.example.com
--slf ..        Use ~/.parallel/sshloginfile as the list of sshlogins
--trc {}.bar    Shorthand for --transfer --return {}.bar --cleanup
--onall         Run the given command with argument on all sshlogins
--nonall        Run the given command with no arguments on all sshlogins

--pipe          Split stdin (standard input) to multiple jobs.
--recend str    Record end separator for --pipe.
--recstart str  Record start separator for --pipe.

See 'man parallel' for details

Academic tradition requires you to cite works you base your article on.
If you use programs that use GNU Parallel to process data for an article in a
scientific publication, please cite:

  Tange, O. (2020, August 22). GNU Parallel 20200822 ('Beirut').
  Zenodo. https://doi.org/10.5281/zenodo.3996295

This helps funding further development; AND IT WON'T COST YOU A CENT.
If you pay 10000 EUR you should feel free to use GNU Parallel without citing.
```

[详细文档看这里](https://www.gnu.org/software/parallel/man.html#NAME)

具体自己读，注意的是 `-q` 是为了让我们在写命令的时候不用加上恼人的反引号。
