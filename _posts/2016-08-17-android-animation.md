---
layout: post
title: Android 动画详解
description: "Android GUI分析之Animation"
modified: 2016-08-17
categories: [Android]
type: dev
---

Android动画一共有三类，分别是View Animation(视图动画)、Drawable Animation(帧动画)、Property Animation(属性动画)，下文将分别加以分析说明。

- #### View Animation

  View动画，也叫Tween（补间）动画，是指作用在View上的包括透明度、缩放、旋转，平移在内的4种效果，分别对应包android.view.animation下的AlphaAnimation、RotateAnimation、ScaleAnimation、TranslateAnimation。可以用代码和xml两种方式定义，Android提供AnimationSet类来管理动画，相应的xml中动画对应的标签为`<set>`：  

  <!-- more -->   

  - AlphaAnimation，对应标签`</alpha>`
    - android:fromAlpha: 透明度的起始值，如0.1
    - android:toAlpha: 透明度的结束值，如1
  - RotateAnimation，对应标签`<rotate>`，常用属性：
    - android:fromDegress: 旋转开始的角度
    - android:toDegress: 旋转结束的角度
    - android:pivotX: 旋转轴点（View围绕该点旋转）x坐标
    - android:pivotY: 旋转轴点（同上）y坐标
  - ScaleAnimation，对应标签`<scale>`，常用属性：
    - android:fromXScale: 水平方向缩放起始值，如0.1
    - android:toXScale: 水平方向缩放结束值，如1.5
    - android:fromYScale: 垂直方向缩放起始值
    - android:toYScale: 垂直方向缩放结束值
    - android:pivotX: 缩放轴点的x坐标
    - android:pivotY: 缩放轴点的y坐标
  - TranslateAnimation，对应标签`<translate>`，常用属性：
    - android:fromXDelta: x方向起始值，如0
    - android:toXDelta: x方向结束值，如100
    - android:fromYDelta: y方向起始值
    - android:toYDelta: y方向结束值

  上述属性也可用代码完成，通常对应相应的构造函数，如：

    

  ```
  public AlphaAnimation(float fromAlpha, float toAlpha) {
      mFromAlpha = fromAlpha;
      mToAlpha = toAlpha;
  }
  ```

    

  此外，Animation及其子类AnimationSet还提供大量的属性可用于动画的设置，见下表：

  | XML属性                | JAVA代码实现                          | 说明                                 |
  | -------------------- | --------------------------------- | ---------------------------------- |
  | android:fillAfter    | setFillAfter(boolean fillAfter)   | 动画结束时是否保持动画结束状态                    |
  | android:fillBefore   | setFillBefore(boolean fillBefore) | 动画结束时是否保持动画开始状态                    |
  | android:repeatMode   | setRepeatMode(int repeatMode)     | 动画重复方式，reverse表示倒序回放，restart表示从头播放 |
  | android:startOffset  | setStartOffset(long startOffset)  | 动画延时开始时间值，默认为0                     |
  | android:duration     | setDuration(long durationMillis)  | 动画持续时间                             |
  | android:repeatCount  | setRepeatCount(int repeatCount)   | 动画重复次数                             |
  | android:interpolator | setInterpolator(Interpolator i)   | 动画插值器，用来控制动画速度                     |

  View动画除了使用系统提供的上述4种动画外，也可以继承Animation，通过矩阵变换实现自定义动画，矩阵变换是自定义View动画的基础和重点，通常放在Animation的空方法applyTransformation中实现。

- #### Drawable Animation

  帧动画是顺序播放一组图片形成的动画效果，类似幻灯片，对应的xml标签为`</animation-list>`，其用法相对简单，如下例：

  ```
  <?xml version="1.0" encoding="utf-8"?>
  <animation-list xmlns:android="http://schemas.android.com/apk/res/android"
      android:oneshot="false">
  <!-- >
  因其本质是Drawable,动画的定义需要放到res/drawable/下,android:oneshot表示是否只展示一遍,false即不断循环
  </-->

      <item android:drawable="@drawable/img1" android:duration="500" />
      <item android:drawable="@drawable/img2" android:duration="500" />
      <item android:drawable="@drawable/img3" android:duration="500" />
      <item android:drawable="@drawable/img4" android:duration="500" />
      
  </animation-list>
  ```

  ```
  TextView mTextView = (TextView) findViewById(R.id.tv);
  mTextView.setBackgroundResource(R.drawable.drawable_animation);
  AnimationDrawable animationDrawable = (AnimationDrawable) mTextView.getBackground();
  animationDrawable.start();
  ```

- #### Property Animation

  Android 3.0版本中引入了属性动画（Property Animation），和视图动画（View Animation）不同的是，它通过动画作用的对象提供set的方法，在一段时间内不断地作用到指定对象的制定属性上，直到set最终值完成动画，而不仅仅针对View的4种动画操作。可以这样理解，视图动画是对象的外观改变而引起的动画效果，所以改变的是外观而非内部属性，属性动画是对象的内部属性改变而到达的动画效果，属性改变引起外观改变。属性动画抽象主要依赖其基类Animator和子类AnimatorSet、ValueAnimator、ObjectAnimator、TimeAnimator实现。

  AnimatorSet：对应的视图动画中的AnimationSet类，用于按特定顺序播放一组动画集合，可以是同时、顺序、延时播放。对应的XML标签为`</set>`，有两种不同的添加动画方式：

  - 调用playTogether(Animator... items)或者playSequentially(List`<Animator>` items)一次性完成动画的添加。
  - 使用play(Animator)和AnimatorSet.Builder逐个添加。

  ValueAnimator：为动画对象提供时间引擎，该动画引擎计算动画值并把它设置到目标对象上，所有动画使用单个时间脉冲，它运行在一个定制的handler中以确保属性的改变发生在UI线程。对应的XML标签为</animator>，属性动画中有两个重要的概念**TimeInterpolator**和**TypeEvaluator**。

  - TimeInterpolator 插值器

    用来确定动画的变化率，可以让动画非线性运动，系统提供的有加速和减速或匀速插值器。

  - TypeEvaluator 估值器

    用来根据当前当前属性改变的百分比计算改变后的属性值。

  比如在某个View的属性动画采用了LinearInterpolator（线性插值器）和IntEvaluator（整型估值器），那么在动画的中间时间点该View的坐标改变就为0.5（因为此时的时间流逝百分比刚好时0.5，对线性插值器来说，它的坐标变换伴随时间是匀速的），此时它坐标的具体值变为多少了呢？就要通过估值器得出了，整型估值器源码如下：

  ```
  public class IntEvaluator implements TypeEvaluator<Integer> {
      public Integer evaluate(float fraction, Integer startValue, Integer endValue) {
          int startInt = startValue;
          return (int)(startInt + fraction * (endValue - startInt));
      }
  }
  ```

  可见其左边改变刚好到0.5的位置。此外，属性动画还提供AnimatorUpdateListener、AnimatorListener两个监听器用于监听动画的播放过程。

  ```
  public static interface AnimatorListener {
      void onAnimationStart(Animator animation);
      void onAnimationEnd(Animator animation);
      void onAnimationCancel(Animator animation);
      void onAnimationRepeat(Animator animation);
  }

  public static interface AnimatorUpdateListener {
      void onAnimationUpdate(ValueAnimator animation);
  }
  ```

  利用这两个监听器，可以在动画过程中做其它处理。

  ObjectAnimator：ValueAnimator的子类，让动画属性作用在目标对象上，在实际中使用更多：

  ```
  ObjectAnimator anim = ObjectAnimator.ofFloat(mObject, "alpha", 0f, 1f);
  anim.setDuration(1000);
  anim.start();
  ```

  TimeAnimator：为动画提供回调机制监听每一帧的动画对象，总运行时间，已运行时间。