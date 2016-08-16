---
layout: post
title: Android Drawable 分析
description: "Android Drawable 分析"
modified: 2016-06-13
categories: [android]
type: dev
---

Drawable包路径:android.graphics.drawable
Drawable是可绘制对象的抽象,drawable不接受事件，无法与用户交互。
drawable有以下类型:  
* Bitmap 简单常见的drawable,PNG或JPEG图片  
* Nine Patch 基于png格式的可伸缩区域  
* Shape 允许size，颜色渐变等可变的几何图形  
* Layers 多个drawable组合  
* States  
* Levels  
* Scale  
#### 自定义drawables
在需要圆形，圆角等图片时，除了可以自定义ImageView外，更简单高效的是自定义Drawable
使用getIntrinsicHeight(),getIntrinsicWidth()可以返回drawable的固有高宽，setBounds(Rect)用来确定Drawable被绘制的大小等
PorterDuff提供了16种两图相交的模式，效果如下![Smithsonian Image]({{ site.url }}/images/android_image/porterduff_mode.png)





| 返回类型 | 方法名                                      | 说明                                       |
| :--- | ---------------------------------------- | ---------------------------------------- |
| void | drawRGB(int r, int g, int b)             | RGB色值在porterduff srcover模式下填充整个位图        |
| void | drawARGB(int a, int r, int g, int b)     | ARGB色值在porterduff srcover模式下填充整个位图       |
| void | drawColor(@ColorInt int color)           | 用指定颜色通过porterduff srcover模式填充整个位图，调用native_drawColor(long nativeCanvas, int color, int mode)，三个参数分别传mNativeCanvasWrapper, color, PorterDuff.Mode.SRC_OVER.nativeInt |
| void | drawColor(@ColorInt int color, @NonNull PorterDuff.Mode mode | 指定的颜色和porter-duff xfermode填充位图           |
| void | drawPaint(@NonNull Paint paint)          | 指定的paint填充位图。 相当于通过指定的paint绘制一个无限大小的矩形(更快速) |
| void | drawPoints(@Size(multiple=2) float[] pts, int offset, int count, @NonNull Paint paint) | 绘制一系列的点，点的形状通过paint的Cap指定                |
| void | drawLine(float startX, float startY, float stopX, float stopY,@NonNull Paint paint) | 绘制指定两点间的一条线                              |
| void | drawRect(@NonNull RectF rect, @NonNull Paint paint) | 绘制矩形                                     |
| void | drawOval(@NonNull RectF oval, @NonNull Paint paint) | 绘制椭圆                                     |
| void | drawCircle(float cx, float cy, float radius, @NonNull Paint paint) | 绘制圆                                      |
| void | drawArc(@NonNull RectF oval, float startAngle, float sweepAngle, boolean useCenter, @NonNull Paint paint) | 通过指定圆弧所使用的矩形区域大小，起始角度，扫过的角度，是否使用中心，画笔绘制圆弧，userCenter为false时，圆弧是的起始点和终止点连接的弧形，userCenter为true时，圆弧时包括圆心的扇形。 |
| void | drawRoundRect(@NonNull RectF rect, float rx, float ry, @NonNull Paint paint) | 绘制圆角矩形                                   |
| void | drawPath(@NonNull Path path, @NonNull Paint paint) | 绘制路径                                     |





