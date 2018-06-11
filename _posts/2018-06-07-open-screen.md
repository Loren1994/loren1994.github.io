---
layout: post
current: post
cover:  assets/images/welcome.jpg
navigation: True
title: Android自定义控件系列之：锁屏页
date: 2017-10-27 12:47:00
tags: [blog]
class: post-template
subclass: 'post tag-getting-started'
author: loren1994
picture: assets/images/ghost.png
---

## Android自定义控件系列之：锁屏页

<!-- more -->
#### 介绍

* 翻阅以前写过的demo，发现在1.52225年前撸过一个仿锁屏功能的自定义View，特此记录。

#### 基本思路

* 分析一下，android锁屏一般为九宫格，将屏幕九等分，以每部分的中点为圆心画圆，每个圆有默认状态和连接状态以及错误状态，根据触摸区域判断触摸的点，在onDraw里画出点和线，在需要的地方设置Paint的颜色等属性后postInvalidate一下，这样来实现点的不同状态的显示。每连接一个点都将此点的下标存入数组，可在activity设置开锁密码，连接完后判断是否正确。

#### 代码实现

给这个view起个低调且奢华的名字，继承View，然后开撸。

> 特别注意：名字不屌，View写不好。:happy:

* 计算9个点的坐标

~~~~Java
  int width = MeasureSpec.getSize(widthMeasureSpec);
  int height = MeasureSpec.getSize(heightMeasureSpec);
  //锁屏3x3点矩阵
  widthArea = width / 3;
  heightArea = height / 3;
  CIRCLE_RADIUS = widthArea / 4;//设置半径
  //根据屏幕设置9个坐标点
  a[0][0] = widthArea / 2;
  a[1][0] = widthArea * 2 * 0.75f;
  a[2][0] = widthArea * 3 * 5 / 6;
  a[3][0] = widthArea / 2;
  a[4][0] = widthArea * 2 * 0.75f;
  a[5][0] = widthArea * 3 * 5 / 6;
  a[6][0] = widthArea / 2;
  a[7][0] = widthArea * 2 * 0.75f;
  a[8][0] = widthArea * 3 * 5 / 6;
  a[0][1] = heightArea / 2;
  a[1][1] = heightArea / 2;
  a[2][1] = heightArea / 2;
  a[3][1] = heightArea * 2 * 0.75f;
  a[4][1] = heightArea * 2 * 0.75f;
  a[5][1] = heightArea * 2 * 0.75f;
  a[6][1] = heightArea * 3 * 5 / 6;
  a[7][1] = heightArea * 3 * 5 / 6;
  a[8][1] = heightArea * 3 * 5 / 6;
~~~~

* onDraw方法

~~~~java
protected void onDraw(Canvas canvas) {
        //画出已连接的点
        for (int i = 0; i < positionArr.size(); i++) {
            if (curCircle.contains(i)) {
                canvas.drawCircle(positionArr.get(i).x, positionArr.get(i).y, CIRCLE_RADIUS, mClickPaint);
            } else {
                canvas.drawCircle(positionArr.get(i).x, positionArr.get(i).y, CIRCLE_RADIUS, mPaint);
            }
        }
        //已连接的点之间的线
        for (int i = 0; i < curCircle.size(); i++) {
            if (i + 1 < curCircle.size()) {
                canvas.drawLine(positionArr.get(curCircle.get(i)).x, positionArr.get(curCircle.get(i)).y, positionArr.get(curCircle.get(i + 1)).x, positionArr.get(curCircle.get(i + 1)).y, mLinePaint);
            }
        }
        //最后的点与触摸位置之间的线
        if (moveX != 0 && moveY != 0 && curCircle.size() > 0 && curCircle.size() < 9 && isTouch) {
            canvas.drawLine(positionArr.get(curCircle.get(curCircle.size() - 1)).x, positionArr.get(curCircle.get(curCircle.size() - 1)).y, moveX, moveY, mLinePaint);
        }
}
~~~~

* onTouchEvent方法

~~~java
public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                break;
            case MotionEvent.ACTION_UP:
                checkPassword();
                break;
            case MotionEvent.ACTION_MOVE:
                moveX = event.getX();
                moveY = event.getY();
                //判断触摸点下标
                int pos = whichCircle(event.getX(), event.getY());
                //存入数组
                if (pos != -1 && !curCircle.contains(pos)) {
                    curCircle.add(pos);
                    System.out.println(pos);
                }
                postInvalidate();
                break;
        }
        return true;
}
~~~

- 判断触摸点

```java
private int whichCircle(float x, float y) {
        int position = -1;
        for (int i = 0; i < positionArr.size(); i++) {
            if (Math.abs(x - positionArr.get(i).x) 
                < CIRCLE_RADIUS && Math.abs(y - positionArr.get(i).y) <CIRCLE_RADIUS) {
                position = i;
            }
        }
        return position;
}
```

* 根据已连接的数组去判断密码是否正确，然后回调

~~~~java
private void checkPassword() {
        StringBuilder pwd = new StringBuilder();
        for (Integer pos : curCircle) {
            pwd.append(pos).append("");
        }
        //连接点数小于等于2则忽略此次连接
        if (curCircle.size() > 2) {
            listener.checkListener(pwd.toString());
            if (!password.equals(pwd.toString())) {
                mLinePaint.setColor(Color.RED);
                mClickPaint.setColor(Color.RED);
                postInvalidate();
                isTouch = false;
                postDelayed(new Runnable() {
                    @Override
                    public void run() {
                        reset();
                    }
                }, 1000);
            }
        } else {
          	//重置View
            reset();
        }
}
~~~~

#### 源码地址

控件:<http://t.cn/Rp4UZHK>  源码附有注释

用法:<http://t.cn/Rp4b4OV>

#### 最终效果

![1903537-5e53408ab2f0cd3d](assets/images/1903537-5e53408ab2f0cd3d.png)

> 最后祝看过的人进步～

