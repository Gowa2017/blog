---
title: TensorFlow模型建立过程
categories:
  - Python
date: 2018-11-14 21:50:38
updated: 2018-11-14 21:50:38
tags: 
  - TensorFlow
---
简单的总结一下 TensorFlow 建立模型进行使用的过程。嗯，到现在为止，一直想要研究一下怎么样整一个神经网络来预测二进制的序列，一直感觉看得如天书一般 ，不知道是如何进行的，所以还是从头开始来。

<!--more-->

# 概括
简单来说，我们建立一个模型。然后给出一定的带标签的样本数据，传递给模型。模型在训练过程中会计算出每次训练的损失，然后逐渐的改变我们模型中的权重参数，最终达到让我们的损失到达一个期望值。就可以说模型收敛了。

# 神经网络中的几个概念

*  **epoch** 对所有训练数据的**一次** 前向传递 和 反向传递。
*  **batch size** 在一次 前向/反向 传递中训练样本数。此值越大，需要的内存就越大。
*  **iterations** 数量 = 传递次数。每次传递使用  [batch size] 个样本。更清晰一点说明 一次传递 = 一次反向传递 + 一次前向传递。

比如： 如果有 1000 个样本， batch size 是500， 那么就会需要 2 次遍历来完成 1 次 epoch。

# 例子
以一个线性回归的例子来进行解析说明。

```py
'''
A linear regression learning algorithm example using TensorFlow library.

Author: Aymeric Damien
Project: https://github.com/aymericdamien/TensorFlow-Examples/
'''

from __future__ import print_function

import tensorflow as tf
import numpy
import matplotlib.pyplot as plt
rng = numpy.random

# 训练参数
learning_rate = 0.01 # 学习速率，梯度下降是用来选择下一个点
training_epochs = 1000 # 训练次数
display_step = 50  # 训练中显示的步长

# Training Data
train_X = numpy.asarray([3.3,4.4,5.5,6.71,6.93,4.168,9.779,6.182,7.59,2.167,
                         7.042,10.791,5.313,7.997,5.654,9.27,3.1]) # 样本
train_Y = numpy.asarray([1.7,2.76,2.09,3.19,1.694,1.573,3.366,2.596,2.53,1.221,
                         2.827,3.465,1.65,2.904,2.42,2.94,1.3]) # 标签
n_samples = train_X.shape[0] # 样本数量

# tf Graph Input
X = tf.placeholder("float")
Y = tf.placeholder("float")

# Set model weights
W = tf.Variable(rng.randn(), name="weight")
b = tf.Variable(rng.randn(), name="bias")

# Construct a linear model
# 类似于输出为 y = wx + b
pred = tf.add(tf.multiply(X, W), b)

# Mean squared error 均方差
cost = tf.reduce_sum(tf.pow(pred-Y, 2))/(2*n_samples)
# Gradient descent 梯度下降
# 这里 minimize() 知道需要修改 W and b ，因为默认情况下变量是可训练的。
optimizer = tf.train.GradientDescentOptimizer(learning_rate).minimize(cost)

# Initialize the variables (i.e. assign their default value)
init = tf.global_variables_initializer()

# Start training
with tf.Session() as sess:

    # Run the initializer
    sess.run(init)

    # Fit all training data
    for epoch in range(training_epochs):
        for (x, y) in zip(train_X, train_Y):
            sess.run(optimizer, feed_dict={X: x, Y: y})

        # Display logs per epoch step
        if (epoch+1) % display_step == 0:
            c = sess.run(cost, feed_dict={X: train_X, Y:train_Y})
            print("Epoch:", '%04d' % (epoch+1), "cost=", "{:.9f}".format(c), \
                "W=", sess.run(W), "b=", sess.run(b))

    print("Optimization Finished!")
    training_cost = sess.run(cost, feed_dict={X: train_X, Y: train_Y})
    print("Training cost=", training_cost, "W=", sess.run(W), "b=", sess.run(b), '\n')

    # Graphic display
    plt.plot(train_X, train_Y, 'ro', label='Original data')
    plt.plot(train_X, sess.run(W) * train_X + sess.run(b), label='Fitted line')
    plt.legend()
    plt.show()

    # Testing example, as requested (Issue #2)
    test_X = numpy.asarray([6.83, 4.668, 8.9, 7.91, 5.7, 8.7, 3.1, 2.1])
    test_Y = numpy.asarray([1.84, 2.273, 3.2, 2.831, 2.92, 3.24, 1.35, 1.03])

    print("Testing... (Mean square loss Comparison)")
    testing_cost = sess.run(
        tf.reduce_sum(tf.pow(pred - Y, 2)) / (2 * test_X.shape[0]),
        feed_dict={X: test_X, Y: test_Y})  # same function as cost above
    print("Testing cost=", testing_cost)
    print("Absolute mean square loss difference:", abs(
        training_cost - testing_cost))

    plt.plot(test_X, test_Y, 'bo', label='Testing data')
    plt.plot(train_X, sess.run(W) * train_X + sess.run(b), label='Fitted line')
    plt.legend()
    plt.show()
```

# 总结
基本过程如下：

1. 准备我们的样本数据（带标签的）
2. 建立模型
3. 选择损失函数
4. 训练

