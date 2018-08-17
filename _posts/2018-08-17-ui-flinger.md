---
layout: post
title: Android流畅度分析
date: 2018-08-17
tags: [android,flinger]
author: loren1994
category: blog
---

#### Android流畅度分析

<!-- more -->
#### 背景

最近一个项目用的低性能android开发板开发，经常会在logcat中看到以下log，故查了一下，在此记录。

~~~~
I/Choreographer: Skipped 60 frames!  The application may be doing too much work on its main thread.
~~~~

#### 分析

首先说明一下android渲染机制。

Android系统每隔16ms就发出VSYNC (Vertical Synchronization垂直同步)信号，触发渲染，重新绘制一次Activity，也就是说，我们的应用必须在16ms内完成屏幕刷新的全部逻辑操作，即每一帧只能停留16ms。

为什么是16ms呢? 16ms相当于60fps。这是因为人眼与大脑之间的协作无法感知超过60fps的画面更新。12fps大概类似手动快速翻动书籍的帧率， 这明显是可以感知到不够顺滑的。24fps使得人眼感知的是连续线性的运动，这其实是归功于运动模糊的效果。 24fps是电影胶圈通常使用的帧率，因为这个帧率已经足够支撑大部分电影画面需要表达的内容，同时能够最大的减少费用支出。 但是低于30fps是 无法顺畅表现绚丽的画面内容的，此时就需要用到60fps来达到想要的效果，超过60fps就没有必要了。如果我们的应用没有在16ms内完成屏幕刷新的全部逻辑操作，页面就会发生卡顿。

渲染操作取决于两个核心组件:CPU和GPU。CPU负责包括Measure，Layout，Record，Execute的计算操作，GPU 负责Rasterization(栅格化)操作。当他俩因为各种原因处理时间大于16ms时，用户就会感知到页面卡顿。

![grid]({{site.baseurl}}/assets/images/grid.png)

上图为栅格化操作，就是将View拆分为不同的像素点以显示到屏幕上。也就是说要将xml转化为用户能看到的图像，首先要通过CPU转化为多边形和纹理，然后传递给GPU进行栅格化。栅格化和OpenGL有关，且是一个十分耗时的操作。所以这16ms主要被两件事所占用：1.xml转为多边形和纹理。2.CPU上传数据给GPU以进行栅格化。

![cgpu]({{site.baseurl}}/assets/images/cgpu.png)

要想缩短这两件事的耗时，就需要从CPU和GPU两方面考虑，即减少CPU转化的次数和上传给GPU的次数。

CPU：减少不必要的布局

GPU：避免过度绘制

> 优化的具体方式不再赘述。

#### Choreographer

> （英[ˌkɒrɪ'ɒɡrəfə(r)] 美[ˌkɒrɪ'ɒɡrəfə(r)]）

该log出自Choreographer的doFrame方法

~~~~java
private static final int SKIPPED_FRAME_WARNING_LIMIT = SystemProperties.getInt(
            "debug.choreographer.skipwarning", 30);
//绘制一帧
void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        synchronized (mLock) {
            //省略部分代码
            long intendedFrameTimeNanos = frameTimeNanos;
            startNanos = System.nanoTime();
            final long jitterNanos = startNanos - frameTimeNanos;
            if (jitterNanos >= mFrameIntervalNanos) {
                final long skippedFrames = jitterNanos / mFrameIntervalNanos;
                if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                    Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                            + "The application may be doing too much work on its main thread.");
                }
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
                if (DEBUG_JANK) {
                    Log.d(TAG, "Missed vsync by " + (jitterNanos * 0.000001f) + " ms "
                            + "which is more than the frame interval of "
                            + (mFrameIntervalNanos * 0.000001f) + " ms!  "
                            + "Skipping " + skippedFrames + " frames and setting frame "
                            + "time to " + (lastFrameOffset * 0.000001f) + " ms in the past.");
                }
                frameTimeNanos = startNanos - lastFrameOffset;
            }
        }
        //省略部分代码
    }
~~~~

SKIPPED_FRAME_WARNING_LIMIT默认30帧，所以当丢失超过30帧则会打印此log。

那么利用反射将此变量改为1，则可以实现只要一丢帧就打印出log。

也可实现Choreographer.FrameCallback接口实现更多功能。(两次绘制时间间隔等)

#### 大体流程

![vsync]({{site.baseurl}}/assets/images/vsync.png)



> 主要参考: https://www.jianshu.com/p/d126640eccb1

