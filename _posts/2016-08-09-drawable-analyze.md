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