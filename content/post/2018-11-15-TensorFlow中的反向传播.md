---
title: TensorFlow中的反向传播
categories:
  - Python
date: 2018-11-15 00:59:24
updated: 2018-11-15 00:59:24
tags: 
  - TensorFlow
---
反向传播算法对于快速训练大型神经网络来说至关重要。本文将介绍该算法的工作原理。来源于谷歌对神经网络的一个简单展示。原文地址：[反向传播算法](https://google-developers.appspot.com/machine-learning/crash-course/backprop-scroll/)

<!--more-->

# 简单的神经网络

![](../res/backprop_01.png)  
在上边，您会看到一个神经网络，其中包含一个输入节点、一个输出节点，以及两个隐藏层（分别有两个节点）。 

相邻的层中的节点通过权重 $$𝑤_{𝑖𝑗}$$ 相关联，这些权重是网络参数。

# 激活函数
![](../res/backprop_02.png)  
每个节点都有一个总输入 𝑥、一个激活函数 𝑓(𝑥) 以及一个输出 𝑦=𝑓(𝑥)。 𝑓(𝑥) 必须是非线性函数，否则神经网络就只能学习线性模型。 

常用的激活函数是 S 型函数：$$f(\color{input}x\color{black}) = \frac{1}{1+e^{-\color{input}x}}$$

# 误差函数
![](../res/backprop_03.png) 

目标是根据数据自动学习网络的权重，以便让所有输入 $$x_{input}$$ 的预测输出 $$y_{output}$$ 接近目标 $$y_{target}$$。

为了衡量与该目标的差距，我们使用了一个误差函数 𝐸。 常用的误差函数是 $$E(\color{output}y_{output}\color{black},\color{output}y_{target}\color{black}) = \frac{1}{2}(\color{output}y_{output}\color{black} - \color{output}y_{target}\color{black})^2 $$

# 正向传播
![](../res/backprop_04.png)
首先，我们取一个输入样本 $$(\color{input}x_{input}\color{black},\color{output}y_{target}\color{black})$$，并更新网络的输入层。<br><br>
        为了保持一致性，我们将输入视为与其他任何节点相同，但不具有激活函数，以便让其输出与输入相等，即$$ \color{output}y_1 \color{black} = \color{input} x_{input} $$。
        
## 更新隐藏层
![](../res/backprop_05.png)

现在，我们更新第一个隐藏层。我们取上一层节点的输出 $$\color{output}y$$，并使用权重来计算下一层节点的输入 $$\color{input}x$$。
$$ \color{input} x_j \color{black} = \sum_{i\in in(j)} w_{ij}\color{output} y_i\color{black} +b_j$$

![](../res/backprop_06.png)

然后，我们更新第一个隐藏层中节点的输出。
为此，我们使用激活函数 $$f(x)$$。
$$ \color{output} y \color{black} = f(\color{input} x \color{black})$$

![](../res/backprop_07.png)

使用这两个公式，我们可以传播到网络的其余内容，并获得网络的最终输出。

$$ \color{output} y \color{black} = f(\color{input} x \color{black})$$

$$ \color{input} x_j \color{black} =  \sum_{i\in in(j)} w_{ij}\color{output} y_i \color{black} + b_j$$

# 误差导数

![](../res/backprop_08.png)

反向传播算法会对特定样本的预测输出和理想输出进行比较，然后确定网络的每个权重的更新幅度。

为此，我们需要计算误差相对于每个权重$$\frac{dE}{dw_{ij}}$$ 的变化情况。

获得误差导数后，我们可以使用一种简单的更新法则来更新权重：

$$w_{ij} = w_{ij} - \alpha \frac{dE}{dw_{ij}}$$

其中，𝛼 是一个正常量，称为“学习速率”，我们需要根据经验对该常量进行微调。

[注意] 该更新法则非常简单：如果在权重提高后误差降低了$$(\frac{dE}{dw_{ij}} < 0)$$，则提高权重；否则，如果在权重提高后误差也提高了 $$(\frac{dE}{dw_{ij}} > 0)$$，则降低权重。

## 其他导数

![](../res/backprop_09.png)

为了帮助计算 $$\frac{dE}{dw_{ij}}$$，我们还为每个节点分别存储了另外两个导数，即误差随以下两项的变化情况：

节点 $$\frac{dE}{dx}$$ 的总输入，以及 $$\frac{dE}{dy}$$ 的输出

# 反向传播
![](../res/backprop_10.png)

我们开始反向传播误差导数。

由于我们拥有此特定输入样本的预测输出，因此我们可以计算误差随该输出的变化情况。

$$
E = \frac{1}{2}(\color{output}y_{output}\color{black} - \color{output}y_{target}\color{black})^2
$$

我们可以得出：

$$
\frac{\partial E}{\partial y_{output}}  = y_{output} - y_{target}
$$

![](../res/backprop_11.png)


现在我们获得了$$\frac{dE}{dy}$$，接下来便可以根据链式法则得出 $$\frac{dE}{dx}$$。

$$ \frac{\partial E}{\partial x}  = \frac{dy}{dx}\frac{\partial E}{\partial y}  = \frac{d}{dx}f(x)\frac{\partial E}{\partial y}$$

其中，当 $$f(\color{input}x\color{black})$$是 S 型激活函数时，$$\frac{d}{dx}f(\color{input}x\color{black}) = f(\color{input}x\color{black})(1 - f(\color{input}x\color{black}))$$

![](../res/backprop_12.png)

一旦得出相对于某节点的总输入的误差导数，我们便可以得出相对于进入该节点的权重的误差导数。

$$
\frac{\partial E}{\partial w_{ij}}  = \frac{\partial x_j}{\partial w_{ij}} \frac{\partial E}{\partial x_j}  = y_i \frac{\partial E}{\partial x_j}
$$

![](../res/backprop_13.png)

根据链式法则，我们还可以根据上一层得出 $$\frac{dE}{dy}$$。此时，我们形成了一个完整的循环。

$$ \frac{\partial E}{\partial y_i}  = \sum_{j\in out(i)} \frac{\partial x_j}{\partial y_i} \frac{\partial E}{\partial x_j} = \sum_{j\in out(i)} w_{ij}  \frac{\partial E}{\partial x_j}$$

接下来，只需重复前面的 3 个公式，直到计算出所有误差导数即可。
# BPTT 
BPTT 是 Backpropagation through time 的缩写。在 RNN 中这是对传统 反向传播BP的一个扩展。 因为在 RNN 中，我们无法直接应用 反向传播算法，因为在计算图中 RNN 网络是循环的。所以我们就会将 RNN 进行展开。

![](../res/RNN-unrolled.png)

这样，RNN 就可以看作是一个前馈网络，我们就可以使用 BP 了。

但是，因为有 梯度消失/梯度爆炸的情况，想要将梯度传播到多个层后非常的困难。此外，展开RNN并为一个非常长的序列传播梯度的计算要求非常大。

所以才出现了 BPTT，他背后的基本思想是：每次处理一个时间步长的序列，每处理 K1 步长，然后运行 BTPP K2 个步长。
# 反向传播的截短

通过设计，递归神经网络（RNN）的输出取决于任意远距离的输入。不幸的是，这使得反向传播计算变得困难。为了使学习过程易于处理，通常的做法是创建网络的“展开”版本，其中包含固定数量（num_steps）的LSTM输入和输出。然后在RNN的这种有限近似上训练该模型。这可以通过一次馈送长度输入num_steps并在每个这样的输入块之后执行反向传递来实现。

这是一个简化的代码块，用于创建执行截断反向传播的图形：

```py
# Placeholder for the inputs in a given iteration.
words = tf.placeholder(tf.int32, [batch_size, num_steps])

lstm = tf.contrib.rnn.BasicLSTMCell(lstm_size)
# Initial state of the LSTM memory.
initial_state = state = lstm.zero_state(batch_size, dtype=tf.float32)

for i in range(num_steps):
    # The value of state is updated after processing each batch of words.
    output, state = lstm(words[:, i], state)

    # The rest of the code.
    # ...

final_state = state
```

在所有数据集上实现迭代：

```py
# A numpy array holding the state of LSTM after each batch of words.
numpy_state = initial_state.eval()
total_loss = 0.0
for current_batch_of_words in words_in_dataset:
    numpy_state, current_loss = session.run([final_state, loss],
        # Initialize the LSTM state from the previous iteration.
        feed_dict={initial_state: numpy_state, words: current_batch_of_words})
    total_loss += current_loss
```