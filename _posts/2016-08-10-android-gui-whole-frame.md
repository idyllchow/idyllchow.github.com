---
layout: post
title: Android GUI系统分析
description: "Android GUI系统分析之整体架构"
modified: 2016-06-13
categories: [android]
type: dev
---

Android的GUI系统由C语言框架和Java语言框架组成，C语言层通过调用输入设备和输出设备驱动将Android的软件系统和底层硬件联系起来，Java框架层提供各种绘图接口供上层应用调用。  
![android_gui]({{ site.url }}/images/android_post/android_gui.png)  
C语言部分包括:  
* PixelFlinger(下层工具库);  
* libui(GUI框架库)；  
* SurfaceFlinger(Surface的管理和处理);
* Skia图形图像引擎;  
* OpenGL 3D引擎;  
* JNI(向JAVA提供接口);  
Java框架层主要包括  
* android.graphics类(对应Skia底层库，提供绘图接口);  
* android.view.Surface(构建显示界面);  
* android.view.view(各种UI元素的基类);  
* javax.microedition.khronos.opengles(标准的OpenGL接口);  
* android.opengl(Android系统和OpenGL的联系层)    
<!-- more -->  
pixelfinger是Android中一个下层的用C语言实现的工具类库，负责像素级别的基本处理:提供像素格式定义、画点、画线、画多边线、纹理颜色填充以及多层处理等操作接口。生成目标动态库libpixelflinger.so，它只连接了Android的C语言基础库libcutils。  

libui库提供GUI系统本地部分框架。此库提供了一些接口，由其它的库通过类继承方式来实现，libui提供了包括：  
1. format(颜色格式，此部分本身定义颜色空间的枚举类型和数据结构)；  2. Egl窗口(用于显示，实现一个本地显示的接口)；  
3. Key/Event(按键及事件处理，系统输入的基础，定义按键的映射，通过操作Event事件设备来实现获取系统的输入)；
4. Surface(显示界面，定义显示界面较高层次的接口，包含部分显示界面的管理功能)；  
5. Overlay(显示叠加层接口，覆盖在主显示层之上，通常用于视频输出)；  
6. Camera(照相机接口，主要在CameraService部分实现)等多个方面的定义。      

##### SurfaceFlinger
SurfaceFlinger是Surface部分的本地实现，实现了Surface的建立，控制，管理等功能。SurfaceFlinger可以支持图形层的创建、叠加、混合等功能，Surface系统通过JNI向Java框架层提供接口，在Java框架层根据Surface系统提供的接口构建各个UI元素。它主要提供android.view.SurfaceSession和android.view.Surface两个Java类，后者表示一个可以绘制图形的界面，前者是客户端请求建立Surface时与SurfaceFlinger建立的Session。 
##### Skia和2D图形系统  
![Skia Image]({{ site.url }}/images/android_post/android_2d_skia.png)
{: .image-right}  

Skia是Google一个底层的图形、图像、动画、SVG、文本、等多方面的图形库，它是Android中图形系统的引擎。 
Android的图形包(graphics):Android系统的2D图形API  
Android 图形系统为Java层提供了绘制基本图形的功能，是GUI系统的基础。通过它也完成Java程序的基本图形、图片、文字等2D的绘制。  
Canvas.java定义了Android图形系统中最为重要且基础的一个类：android.graphics.Canvas。该类处理“draw”，绘制内容时需要4个基本组件：保持像素的Bitmap，处理绘制调用的Canvas(写入Bitmap)，绘制的内容(可能是Point、Rect、Path、text、Bitmap等)和paint(用来描述颜色和样式)，Bitmap.java文件实现了类android.graphics.Bitmap，它表示内存中的一个位图。Android中的UI元素也是通过调用Canvas类来构建的在View.java中，实现的是类android.view.View，通过建立这个Canvas类来构建绘画的基础。  

android.view.View类呈现最基本的UI构造块。一个视图占据屏幕上的一个方形区域，并且负责绘制和事件处理。View的直接扩展结构如右：  
![View Image]({{ site.url }}/images/android_post/android_view.png)
{: .image-right}

Android中的UI元素常常在Layout中进行描述，android.view.View的其中一个重要扩展者是android.view.ViewGroup，它表示一个视图的集合，其中可以包含众多子视图，本身也是一个视图。基本的UI元素图如下：  

除此之外，继承android.view.View的自定义视图也是基于2D图形API的实现OnDraw()的过程，例如：  

##### OpenGL 3D图形系统
![Skia Image]({{ site.url }}/images/android_post/android_3d_opengl.png)
{: .image-right} 
在Android中，可以直接支持3D的绘制，主要使用OpenGL标准的类javax.microedition.khronos.egl，实现过程如下：  
1. 扩展实现android.view.GLSurfaceView类。  
2. 实现前一个类的Renderer。  
3. 实现GLSurfaceView::Renderer中的onDrawFrame()函数  

