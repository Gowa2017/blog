---
title: TensorFlow中的张量的stack与unstack
categories:
  - Python
date: 2018-11-02 22:12:25
updated: 2018-11-02 22:12:25
tags: 
  - TensorFlow
  - Python
---
学习一个简单的例子，关于神经网络的，总是会看到关于 stack unstack 函数的使用。但这涉及到的东西实在是多，比如最基本的一个前提概念就是，Tensor 张量，非常难以理解这个东西。


<!--more-->

# 张量

[张量 TensorFlow 定义](https://www.tensorflow.org/programmers_guide/tensors?hl=zh-cn)

TensorFlow 框架涉及到两个重要的元素：**张量** 和 **操作** 。

按照定义，**张量** 是对矢量和矩阵向潜在更高维度的泛化。

我们在使用 TensorFlow 的时候，我们操作和传递的主要对象都是 **张量，tf.Tensor**。**tf.Tensor** 对象表示一个部分定义的计算，最终会产生一个值。TensorFlow 程序首先会构建一个 tf.Tensor 对象图，详细说明如何基于其他可用张量计算每个张量，然后运行该图的某些部分以获得期望的结果。

张量具有以下两个属性：

* 数据类型
* 形状 shape

张量中的每个元素具有相同的数据类型，且该数据类型一定是已知的。形状，即是张量的维数和每个维度的大小，可能只是部分已知。如果其输入的形状也完全已知，则大多数指令会生成形状完全已知的张量，但在某些情况下，只能在图的执行时间找到张量的形状。

以下是一些特殊的张量：

* tf.Variable
* tf.constant
* tf.placeholder
* tf.SparseTensor


除了 tf.Variable ，张量的值不变。因此，执行一个任务的时候，张量只有一个值。但重复评估同一张量可能会有不同的值。

## 阶 
**张量** 对象的 **阶** 是其本身的维数。其同义词包括：**秩、等级或 n 维**。 TensorFlow 中的阶与数学矩阵中的阶并不是同一概念。 TensorFlow中的每个阶都对应一个不同的数学实例。

张量的 **形状** 中元素数量与阶（维数）相等。

|阶|数学实例|
|---|-----|
|0|标量（只有大小）|
|1|矢量（大小和方向）|
|2|矩阵（数据表）|
|3|3阶张量（数据立体）|
|n|n阶张量（自行想象）|

### 0 阶

```py
mammal = tf.Variable("Elephant", tf.string)
ignition = tf.Variable(451, tf.int16)
floating = tf.Variable(3.14159265359, tf.float64)
its_complicated = tf.Variable(12.3 - 4.85j, tf.complex64)
```

>注意：字符串在 TensorFlow 中被视为单一项，而不是一连串字符串。TensorFlow 可以有标量字符串，字符串矢量，等等。


### 1 阶

要建立 1 阶的 tf.Tensor 对象，可传递一个项目列表作为初值：

```py
mystr = tf.Variable(["Hello"], tf.string)
cool_numbers  = tf.Variable([3.14159, 2.71828], tf.float32)
first_primes = tf.Variable([2, 3, 5, 7, 11], tf.int32)
its_very_complicated = tf.Variable([12.3 - 4.85j, 7.5 - 6.23j], tf.complex64)
```

### 更高阶 
2 阶 tf.Tensor 对象至少包含一行和一列：


```py
mymat = tf.Variable([[7],[11]], tf.int16)
myxor = tf.Variable([[False, True],[True, False]], tf.bool)
linear_squares = tf.Variable([[4], [9], [16], [25]], tf.int32)
squarish_squares = tf.Variable([ [4, 9], [16, 25] ], tf.int32)
rank_of_squares = tf.rank(squarish_squares)
mymatC = tf.Variable([[7],[11]], tf.int32)
```

同样，更高阶的张量由一个 n 维数组组成。例如，在图像处理过程中，会使用许多 4 阶张量，维度对应批次大小、图像宽度、图像高度和颜色通道。

```py
my_image = tf.zeros([10, 299, 299, 3])  # batch x height x width x color
```

## 获取 tf.Tensor 对象的阶

要确定 tf.Tensor 对象的阶，需调用 tf.rank 方法。例如，以下方法以编程方式确定上一章节中所定义的 tf.Tensor 的阶：


```py
r = tf.rank(my_image)
# After the graph runs, r will hold the value 4.
```

## 引用 tf.Tensor 切片
由于 tf.Tensor 是 n 维单元数组，要访问 tf.Tensor 中的某一单元，需要指定 n 个索引。

0 阶张量（标量）不需要索引，因为其本身便是单一数字。

对于 1 阶张量（矢量）来说，通过传递单一索引可以访问某个数字：

```py
my_scalar = my_vector[2]
```

请注意，如果想从矢量中动态地选择元素，那么在 [] 内传递的索引本身可以是一个标量 tf.Tensor。

对于 2 阶及以上的张量来说，情况更为有趣。对于 2 阶 tf.Tensor 来说，传递两个数字会如预期般返回一个标量：

```py
my_scalar = my_matrix[1, 2]
```

而传递一个数字则会返回一个矩阵子矢量，如下所示：

```py
my_row_vector = my_matrix[2]
my_column_vector = my_matrix[:, 3]
```

符号 **:** 是 Python 切片语法，意味“不要触碰该维度”。这对更高阶的张量来说很有用，可以帮助访问其子矢量，子矩阵，甚至其他子张量。

## 形状

张量的**形状**是每个维度中元素的数量。TensorFlow 在图的构建过程中自动推理形状。这些推理的形状可能具有已知或未知的阶。如果阶已知，则每个维度的大小可能已知或未知。

TensorFlow 文件编制中通过三种符号约定来描述张量维度：阶，形状和维数。下表阐述了三者如何相互关联：

|阶|形状|维度|示例|
|---|---|---|---|
|0|[]|0-D|0维张量。标量|
|1|[D0]|1-D|形状为[5]的1维张量|
|2|[D0,D1]|2-D|形状为[3,4]的2维张量|
|3|[D0, D1, D2]|3-D|形状为[1,4,3]的3维张量|
|n|[D0, D1,  .... Dn-1]|n维|形状为 [D0,D1,...Dn-1]的张量|

形状包含了两个属性，*dims, ndim*

### tf.TensorShape
此是形状在内部的实现。[https://www.tensorflow.org/api_docs/python/tf/TensorShape](https://www.tensorflow.org/api_docs/python/tf/TensorShape)

### tf.shape()

```py
tf.shape(
    input,
    name=None,
    out_type=tf.int32
)
```

返回张量的形状。

这个操作会返回一个 1-D 整数张量，代表了 *input* 的形状。

例如：

```py
t = tf.constant([[[1, 1, 1], [2, 2, 2]], [[3, 3, 3], [4, 4, 4]]])
tf.shape(t)  # [2, 2, 3]
```

# tf.concat()
要了解 `tf.stack()` 就必须要了解 `tf.concat()`。这函数会沿某一维度拼接张量。

```py
tf.concat(
    values,
    axis,
    name='concat'
)
```
> 沿 *axis* 维度连接 *values* 中的张量列表。如果 *values[i].shape=[D0, D1, ... Daxis(i), ... Dn]* 那么结果就是：  
> [D0, D1, ... Raxis, ...Dn]  
> 其中   
> Raxis = sum(Daxis(i))

这也就是说，输入张量内的数据沿 *axis* 维度连接了。

输入张量的维数必匹配，除了 *axis* 外的维度必须相等。

例如：

```py
t1 = [[1, 2, 3], [4, 5, 6]]
t2 = [[7, 8, 9], [10, 11, 12]]
tf.concat([t1, t2], 0)  # [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10, 11, 12]]
tf.concat([t1, t2], 1)  # [[1, 2, 3, 7, 8, 9], [4, 5, 6, 10, 11, 12]]

# tensor t3 with shape [2, 3]
# tensor t4 with shape [2, 3]
tf.shape(tf.concat([t3, t4], 0))  # [4, 3]
tf.shape(tf.concat([t3, t4], 1))  # [2, 6]
```

而：

```py
tf.concat([tf.expand_dims(t, axis) for t in tensors], axis)
tf.stack(tensors, axis=axis)
```
这两者是相等的。


# tf.stack()
[官方定义](https://www.tensorflow.org/api_docs/python/tf/stack)

```py
tf.stack(
    values,
    axis=0,
    name='stack'
)
```

把一个 R 阶的张量列表堆成一个 R+1 阶的张量。

通过沿 *axis* 方向，将 *values* 中的张量列表打包到一个比 *values* 中的任何一个张量高一阶的张量中。现在给定一个长度为 *N* 的列表，其中的张量形状为 *(A, B, C)*。

* 如果 *axis == 0* ，输出张量的形状是 *(N, A, B, C)*
* 如果 *axis == 1* ，输出张量的形状是 *(A, N, B, C)*  
.......     
.......

例如：

```py
x = tf.constant([1, 4])
y = tf.constant([2, 5])
z = tf.constant([3, 6])
tf.stack([x, y, z])  # [[1, 4], [2, 5], [3, 6]] (Pack along first dim.)
tf.stack([x, y, z], axis=1)  # [[1, 2, 3], [4, 5, 6]]

```

理解的时候，我把 *[x, y, z]* 理解为一个张量了，而其实上，我不应该混淆，**张量** 与其在 Python 中的阵列实现混为一谈。这个时候，其只是作为一个 张量列表 传递给 stack() 函数的。

列表中的张量其形状都是 *[2]*， 列表长度为 *3*，那么，stack() 后的张量形状应该为 *[3,2]*。而当 *axis = 1*，的时候，就应该是 *[2,3]*


这与 unstack 是相反的操作。这与 ：

```py
tf.stack([x, y, z]) = np.stack([x, y, z])
```
一致。


参数：

* **values** 具有相同形状和类型的**张量**对象列表
* **axis** *int* 值。要压缩的轴方向。默认就是第一个。负值回绕，所以有效值就是 *[-(R+1), R+1)*。
* **name** 操作名称

返回值：

* **output** 与 *values* 类型相同的压缩后的 **张量** 。

抛出：

* **ValueError** 如果 *axis* 超过了范围 *[-(R+1), R+1)*


# tf.unstack()

```py
tf.unstack(
    value,
    num=None,
    axis=0,
    name='unstack'
)
```

把 R 阶的张量解压为 R-1 阶。

通过沿 *axis* 方向解开，从 *value* 张量内解压出 *num* 个张量。如果没有指定 *num（默认情况）*，这会从 *value* 的形状去推测出来。但如果 *value.shape[axis]* 是未知的，就会抛出 *ValueError* 错误。

比如给出一个形状为 **(A, B, C, D)**的张量：

如果 **axis==0**，那么 **输出** 的第 *i* 个张量就是切片 **value[i, :, :, :]**，且输出中的每个张量形状都是 **(B, C, D)**。**注意，解压的轴消失了。这和 `split` 不一样**。

如果 **axis==1**，那么 **输出** 的第 *i* 个张量就是切片 **value[:, i, :, :]**，且输出中的每个张量形状都是 **(A, C, D)**。**注意，解压的轴消失了。这和 `split` 不一样**。

参数：

* **value**: 一个 R > 0 阶张量
* **num**: 轴的长度。
* **axis**: 解压的轴方向。
* **name**: 操作名称（可选）

返回值：

一个从 **value** 解压出来的 张量列表。