---
title: TensorFlow的环境安装
categories:
  - Python
date: 2018-09-06 14:36:51
updated: 2018-09-06 14:36:51
tags: 
  - TensorFlow
  - Python
---
也来学习一下机器学习的 TensorFlow 是什么。第一步当然是搭建环境了。据说大多数用的都是 Python 来使用这个库。所以我也这样想了。安装官方的教程，在 Mac 上进行安装。[官方教程](https://www.tensorflow.org/install/install_mac)

<!--more-->
有四种安装方式：

* Virtualenv
* “原生”pip
* Docker
* 从源代码安装（详情请参阅这篇[单独的指南](https://www.tensorflow.org/install/install_sources)）。

**官方建议采用 Virtualenv 安装方式**

这就设置到要安装 Virtualenv 环境了。而这我看了一下教程，又是需要用 pip 来安装的。所以我们一步步来吧。

# 保证机器上有 Python 环境 
我的 Mac 已经有 2.7.10 的 Python 就不需要自己安装了。

# 安装 pip 包管理器

```shell
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python get-pip.py

Collecting pip
  Downloading https://files.pythonhosted.org/packages/5f/25/e52d3f31441505a5f3af41213346e5b6c221c9e086a166f3703d2ddaf940/pip-18.0-py2.py3-none-any.whl (1.3MB)
    100% |████████████████████████████████| 1.3MB 483kB/s
Collecting wheel
  Downloading https://files.pythonhosted.org/packages/81/30/e935244ca6165187ae8be876b6316ae201b71485538ffac1d718843025a9/wheel-0.31.1-py2.py3-none-any.whl (41kB)
    100% |████████████████████████████████| 51kB 729kB/s
Installing collected packages: pip, wheel
Successfully installed pip-18.0 wheel-0.31.1
```

更详细的解释可以看 [这里](https://pip.pypa.io/en/stable/installing/)

# 安装 virtualenv

官方的安装教程在 [这里](https://virtualenv.pypa.io/en/stable/installation/)

简单的使用 pip 安装就好了。当然你也可以选择其他的方式。

```shell
sudo pip install virtualenv

Collecting virtualenv
  Downloading https://files.pythonhosted.org/packages/b6/30/96a02b2287098b23b875bc8c2f58071c35d2efe84f747b64d523721dc2b5/virtualenv-16.0.0-py2.py3-none-any.whl (1.9MB)
    100% |████████████████████████████████| 1.9MB 286kB/s
Installing collected packages: virtualenv
Successfully installed virtualenv-16.0.0
```

# 安装 TensorFlow

这个就可以完全按照官方教程来了：[Virtualenv安装TensorFlow-Mac](https://www.tensorflow.org/install/install_mac)


## 建立 Virtualenv 环境

```shell
virtualenv --system-site-packages tensorflow
```

## 激活 Virtualenv 环境：

```shell
$ cd tensorflow
$ source ./bin/activate      # If using bash, sh, ksh, or zsh
$ source ./bin/activate.csh  # If using csh or tcsh 
```
## 确保安装 pip 8.1 或更高版本

```shell
(targetDirectory)$ easy_install -U pip
```

## 将 TensorFlow 及其所需的所有软件包安装到活动 Virtualenv 环境

```shell
 (targetDirectory)$ pip install --upgrade tensorflow      # for Python 2.7
 (targetDirectory)$ pip3 install --upgrade tensorflow     # for Python 3.n
```

## 可选。如果第 6 步失败了（通常是因为您所调用的 pip 版本低于 8.1），请通过发出以下格式的命令在活动 Virtualenv 环境中安装 TensorFlow

```shell
 $ pip install --upgrade tfBinaryURL   # Python 2.7
 $ pip3 install --upgrade tfBinaryURL  # Python 3.n 
```


# 注意
请注意，每次在新的 shell 中使用 TensorFlow 时，您都必须激活 Virtualenv 环境。如果 Virtualenv 环境当前未处于活动状态（即提示符不是 (*targetDirectory*)），请调用以下某个命令：

```shell
$ cd targetDirectory
$ source ./bin/activate      # If using bash, sh, ksh, or zsh
$ source ./bin/activate.csh  # If using csh or tcsh 
```

您的提示符将变成如下所示，这表示您的 tensorflow 环境已处于活动状态：

```
 (targetDirectory)$ 
```

当 Virtualenv 环境处于活动状态时，您就可以从该 shell 运行 TensorFlow 程序了。

用完 TensorFlow 后，可以通过发出以下命令来停用此环境：

```
 (targetDirectory)$ deactivate 
```

提示符将恢复为您的默认提示符（由 PS1 所定义）。

# 验证安装 OK

 从 shell 中调用 Python，如下所示：

 ```shell
 $python
 ```

 在 Python 交互式 shell 中输入以下几行简短的程序代码：

 ```python
 # Python
import tensorflow as tf
hello = tf.constant('Hello, TensorFlow!')
sess = tf.Session()
print(sess.run(hello))
 ```

 如果系统输出以下内容，则说明您可以开始编写 TensorFlow 程序了：

 ```
 Hello, TensorFlow!
 ```