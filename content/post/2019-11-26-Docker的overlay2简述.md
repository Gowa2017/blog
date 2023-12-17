---
title: Docker的overlay2简述
categories:
  - Docker
date: 2019-11-26 22:49:46
updated: 2019-11-26 22:49:46
tags: 
  - Docker
---
话说 Docker 使用了一种 union FS 的分层文件系统，理解这个，是制作镜像和存储的关键，所以就一下官方文档的说明。 Docker 新的使用的是 overlay2 这个了。需要在一定的 Linux 内核上才能运行。

<!--more-->

# overlay2 驱动工作原理
先来区分几个概念。

- OverlayFS 指的是 Linux 的内核驱动
- overlay/overlay2 指的是 Docker 的存储驱动。

OverlayFS 将单个 Linux 主机上的两个目录进行分层，然后将它们表示为一个目录。这些目录就叫做 **layers**，这个过程就被叫做 **union mount**。OverlayFS 将靠下一层的目录叫做 **lowerdir**，将靠上一层的叫做 **upperdir**。然后经过处理后我们看到的那个目录叫做 **merged**。

# Images 磁盘分层

在我们下载一个 4 层的镜像后 `docker pull ubuntu`，我们会在 **/var/lib/docker/overlay2** 下面看到 5个目录

```sh
drwx------    4 root     root          4096 Nov 26 14:36 88894c3938bf75095725df436b51ccf84fe860a06789847849390273c2c656ca
drwx------    3 root     root          4096 Nov 26 14:36 a4b654fa44aa1e2bbafa5d953afef7e4f0b069ceb1798d1657050e8b1d795d8a
drwx------    4 root     root          4096 Nov 26 14:40 a5ac07700a22ac2e30b90b547e5e967944d987d9c424c45282840466b5fe131d
drwx------    4 root     root          4096 Nov 26 14:36 efe208a758e893eda3a464fb338bbfc9628a03a9434f1e59dd32c2e79f87ea49
drwx------    2 root     root          4096 Nov 27 01:24 l
```

小写的 `l` 中是对这 4 个层的符号链接，其只是为了减少 mount 参数可能达到的限制而已。

```sh
lrwxrwxrwx    1 root     root            72 Nov 26 14:36 6HI34VCGTCFDV4JTCYDTOCC3DG -> ../efe208a758e893eda3a464fb338bbfc9628a03a9434f1e59dd32c2e79f87ea49/diff
lrwxrwxrwx    1 root     root            72 Nov 26 14:36 7FXGKUQN5DS26G42GP66EFRA4Y -> ../88894c3938bf75095725df436b51ccf84fe860a06789847849390273c2c656ca/diff
lrwxrwxrwx    1 root     root            72 Nov 26 14:36 7H77VL7XYJJYW22JCKZWBANAKN -> ../a5ac07700a22ac2e30b90b547e5e967944d987d9c424c45282840466b5fe131d/diff
lrwxrwxrwx    1 root     root            72 Nov 26 14:36 S2EFJV2TWAYK3YIRFVGA33NJQ3 -> ../a4b654fa44aa1e2bbafa5d953afef7e4f0b069ceb1798d1657050e8b1d795d8a/diff
```

最底下的一层，有一个叫做 **link** 的文件，其内是一个短名字。

```sh
88894c3938bf75095725df436b51ccf84fe860a06789847849390273c2c656ca:
committed  diff       link       lower      work

a4b654fa44aa1e2bbafa5d953afef7e4f0b069ceb1798d1657050e8b1d795d8a:
committed  diff       link

a5ac07700a22ac2e30b90b547e5e967944d987d9c424c45282840466b5fe131d:
committed  diff       link       lower      work

efe208a758e893eda3a464fb338bbfc9628a03a9434f1e59dd32c2e79f87ea49:
committed  diff       link       lower      work

l:
6HI34VCGTCFDV4JTCYDTOCC3DG  7H77VL7XYJJYW22JCKZWBANAKN
7FXGKUQN5DS26G42GP66EFRA4Y  S2EFJV2TWAYK3YIRFVGA33NJQ3
```

 文件 **diff** 就包含了层的内容。

# Container

现在我们来启动一个容器。

```sh
docker run -it ubuntu bash
```

我们现在来观察现在 docker 的存储层情况。发觉其多了两个层：

```sh
drwx------    5 root     root          4096 Nov 27 01:30 151f20b2b354325bc23bb37891c67faea9106dfc94e1835b901743a5d8997e0d
drwx------    4 root     root          4096 Nov 27 01:30 151f20b2b354325bc23bb37891c67faea9106dfc94e1835b901743a5d8997e0d-init
```

这是为 contaner 建立的层。其内就会含有一个  merged 的目录，这就包含了与其父层内容整合后的内容。

当我们启动一个container 的时候，我们看一下挂载的内容：

```sh
overlay on /var/lib/docker/overlay2/151f20b2b354325bc23bb37891c67faea9106dfc94e1835b901743a5d8997e0d/merged type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/4E4WSPA2J6FBHHUAWTUCBRD7BF:/var/lib/docker/overlay2/l/7H77VL7XYJJYW22JCKZWBANAKN:/var/lib/docker/overlay2/l/6HI34VCGTCFDV4JTCYDTOCC3DG:/var/lib/docker/overlay2/l/7FXGKUQN5DS26G42GP66EFRA4Y:/var/lib/docker/overlay2/l/S2EFJV2TWAYK3YIRFVGA33NJQ3,upperdir=/var/lib/docker/overlay2/151f20b2b354325bc23bb37891c67faea9106dfc94e1835b901743a5d8997e0d/diff,workdir=/var/lib/docker/overlay2/151f20b2b354325bc23bb37891c67faea9106dfc94e1835b901743a5d8997e0d/work)
```

可以看到顶层就是 container layer ，其会有多个 lowerdir ，而只有一个 upperdir ，将整合后的 merged 给挂载起来了

我们知道 diff 是层的内容，我们可以验证，这个时候  container layer 是空的：

```sh
ls  /var/lib/docker/overlay2/151f20b2b354325bc23bb37891c67faea9106dfc94e1835b901743a5d8997e0d/diff/

```



# Containers 对 overlay上文件的读写

## Read

考虑 Containers 需要通过 overlay 来打开一个文件的情况：

1. 在 container layer 不存在，那么就会从 image layer 去读取。
2. 只存在于 container layer，直接读写。
3. 既存在与 image layer 也存在于 container layers，读取 container layers 内的。

## Write

考虑下面几种情况

- 第一次写出一个文件。如果像一个已经存在的文件进行写入，但此文件并不存在于 container layer。那么 overlay 就会进行一个*copy_up* 操作，把文件从 image layer(lowerdir) 复制到 container layer(upperdir)。然后就都是在这个 container layer 上的文件进行操作。**然而**，OverlayFS 是工作在文件级别，而不是块级别。这就意味着所有的 copy_up 操作会将整个文件进行拷贝，即使这个文件非常大但是我们只修改其中的一小个地方。这在 container 的写性能还是有影响的。有两个值得注意的事情：
- 删除文件和目录。
  - 我们删除文件的时候，会在 container 层里面建立一个 **without** 文件（upperdir)。而 image layer (lowdir) 则不会被删除，因为其是 **read-only** 的。 **without** 文件会阻止对此文件的访问。
  - 当我们删除目录的时候，会建立一个 **opaque** 目录。这个文件的工作方式一致。
- 重命名目录。只有源文件和目标文路径都是在最顶层的时候才行，否则就会返回一个 **EXDEV** 错误。



## 验证

读取这个没有什么问题，很明显的查找逻辑，因为在 merge 里面都能看到。主要是写。

现在我们准备来对 /etc/host.conf 这个文件进行修改。看一下修改了以后我们的 container layer 是不是会有新文件，已经原来层的文件是否会改变。



## 修改前

 container layer 是空的：

```sh
ls /var/lib/docker/overlay2/151f20b2b354325bc23bb37891c67faea9106dfc94e1835b901743a5d8997e0d/diff/
```

hostname 存在与镜像层：

```sh
cat /var/lib/docker/overlay2/a4b654fa44aa1e2bbafa5d953afef7e4f0b069ceb1798d1657050e8b1d795d8a/diff/etc/host.conf
# The "order" line is only used by old versions of the C library.
order hosts,bind
multi on
```

我们在 container 内进行修改，将第一行的注释干掉

```sh
sed -i '1d' /etc/host.conf
```

## 修改后

```sh
cat /var/lib/docker/overlay2/151f20b2b354325bc23bb37891c67faea9106dfc94e1835b901743a5d8997e0d/diff/etc/host.conf
order hosts,bind
multi on
```

```sh
cat /var/lib/docker/overlay2/a4b654fa44aa1e2bbafa5d953afef7e4f0b069ceb1798d1657050e8b1d795d8a/diff/etc/host.conf
```

镜像层内的内容并没有改变。

# 总结

OK，那么明白了。我们可以把所谓的层，都看做是一个有文件的目录，然后多个目录间有自下至上的依赖关系，然后最终合并成是一个视图，再将这个合并后的目录，挂载到 docker 进程内去。

我们的写都是在 Container layer 上进行的，不会影响镜像层。另外，我们在里面删除 Image layer 文件并不是真正的删除文件，而是建立一个对应的文件来阻止对相关文件的访问。

