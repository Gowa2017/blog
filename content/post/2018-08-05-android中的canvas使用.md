---
title: android中的canvas使用
categories:
  - Android
date: 2018-08-05 09:20:16
updated: 2018-08-05 09:20:16
tags: 
  - Android
---

canvas类中拥有 `draw` 调用。联系要绘制某些内容的话，需要四个组件：**用来存储像素的bitmap，用来包含绘制调用的Canvas，绘制区域（如矩行，路径，文字，bitmap），以及一个画刷（用来定义绘制的颜色和风格）**。
<!--more-->

本章参考原文：[https://guides.codepath.com/android/Basic-Painting-with-Views](https://guides.codepath.com/android/Basic-Painting-with-Views)。
# 自定义一个View
```java
public class SimpleDrawingView extends View {
    public SimpleDrawingView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }
}
```
在我们的布局文件内加入这个自定义的View：

```xml
    <hello.gowa.com.myapplication.customview.SimpleDrawingView
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />
```

# 定义绘制

## 定义画刷

```java
public class SimpleDrawingView extends View {
  // setup initial color
  private final int paintColor = Color.BLACK;
  // defines paint and canvas
  private Paint drawPaint;

  public SimpleDrawingView(Context context, AttributeSet attrs) {
    super(context, attrs);
    setFocusable(true);
    setFocusableInTouchMode(true);
    setupPaint();
  }
  
  // Setup paint with color and stroke styles
  private void setupPaint() {
    drawPaint = new Paint();
    drawPaint.setColor(paintColor);
    drawPaint.setAntiAlias(true);
    drawPaint.setStrokeWidth(5);
    drawPaint.setStyle(Paint.Style.STROKE);
    drawPaint.setStrokeJoin(Paint.Join.ROUND);
    drawPaint.setStrokeCap(Paint.Cap.ROUND);
  }
}
```

## 重写onDraw函数

```java
    @Override
    protected void onDraw(Canvas canvas) {
      canvas.drawCircle(50, 50, 20, drawPaint);
      drawPaint.setColor(Color.GREEN);
      canvas.drawCircle(50, 150, 20, drawPaint);
      drawPaint.setColor(Color.BLUE);
      canvas.drawCircle(50, 250, 20, drawPaint);
    }
```

上面的代码，绘制了三个不同颜色的圆。形状的绘制，通过 Canvas里面的各种调用来实现的。


# 处理触摸事件
如果现在我们想在每次用户触摸一下屏幕的时候，就在触摸的地方绘制一个圈，那我们就必须跟踪用户所触摸的每个点。在`onTouch`事件中我们可以获得这个坐标。：


```java
public class SimpleDrawingView extends View {
  // setup initial color
  private final int paintColor = Color.BLACK;
  // defines paint and canvas
  private Paint drawPaint;
  // Store circles to draw each time the user touches down
  private List<Point> circlePoints;

  public SimpleDrawingView(Context context, AttributeSet attrs) {
    super(context, attrs);
    setupPaint(); // same as before
    circlePoints = new ArrayList<Point>();
  }

  // Draw each circle onto the view
  @Override
  protected void onDraw(Canvas canvas) {
    for (Point p : circlePoints) {
      canvas.drawCircle(p.x, p.y, 5, drawPaint);
    }
  }

  // Append new circle each time user presses on screen
  @Override
  public boolean onTouchEvent(MotionEvent event) {
    float touchX = event.getX();
    float touchY = event.getY();
    circlePoints.add(new Point(Math.round(touchX), Math.round(touchY)));
    // indicate view should be redrawn
    postInvalidate();
    return true;
  }

  private void setupPaint() {
    // same as before
    drawPaint.setStyle(Paint.Style.FILL); // change to fill
    // ...
  }
}
```

# 根据路径绘制
*Path*类可以包含多个线条，轮廓甚至其他形状。我们添加一个*Path* 变量来跟踪我们的绘制：

```java
public class SimpleDrawingView extends View {
    private Path path = new Path();

    // Get x and y and append them to the path
    public boolean onTouchEvent(MotionEvent event) {
        float pointX = event.getX();
        float pointY = event.getY();
        // Checks for the event that occurs
        switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            // Starts a new line in the path
            path.moveTo(pointX, pointY);
            break;
        case MotionEvent.ACTION_MOVE:
            // Draws line between last point and this point
            path.lineTo(pointX, pointY);
            break;
        default:
            return false;
       }

       postInvalidate(); // Indicate view should be redrawn
       return true; // Indicate we've consumed the touch
    }

   // ...
}
```

在`onDraw`中进行绘制路径：

```java
public class SimpleDrawingView extends View {
  // ... onTouchEvent ...

  // Draws the path created during the touch events
  @Override
  protected void onDraw(Canvas canvas) {
      canvas.drawPath(path, drawPaint);
  }

  private void setupPaint() {
    // same as before
    drawPaint.setStyle(Paint.Style.STROKE); // change back to stroke
    // ...
  }
}
```

后面的效果类似于手写字一样。

# 用 bitmap缓存来提高效率
当在 canvas上绘制的时候，可以将图片缓存到 bitmap 来大大的提高效率。

