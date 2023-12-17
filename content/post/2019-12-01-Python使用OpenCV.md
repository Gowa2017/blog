---
title: Python使用OpenCV
categories:
  - Python
date: 2019-12-01 21:37:53
updated: 2019-12-01 21:37:53
tags: 
  - Python
  - OpenCV
---
想要进行图片的识别，或者配合上深度学习来进行一些实验，所以就看上了 OpenCv 这个库，Python 的使用非常的简单，一句话就能安装。

<!--more-->

我的环境是 macOS 10.15

```sh
pip install opencv-python
```

然后在我的 venv 环境就能看到已经安装 了：

```sh
ls -1 venv/lib/python3.7/site-packages/cv2
LICENSE-3RD-PARTY.txt
LICENSE.txt
__init__.py
__pycache__
cv2.cpython-37m-darwin.so
data
```

那个 so 就是 opencv 编译出来的供我们进行调用的 C 库了。

官方的 Demo:

```py
import cv2
print(cv2.__version__)

img = cv2.imread('clouds.jpg')
gray = cv2.cvtColor(img, cv2.COLOR_RGB2GRAY)

cv2.imshow("Over the Clouds", img)
cv2.imshow("Over the Clouds - gray", gray)
cv2.waitKey(0)
cv2.destroyAllWindows()

```

其中， **clouds.jpg** 是一张图片，你可以自己弄一张放到自己的脚本目录下。

执行后你就能看到两个窗口，一个窗口是原图，一个窗口是灰色的图。
