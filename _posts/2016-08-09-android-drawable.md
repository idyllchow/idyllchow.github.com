---
layout: post
title: Android GUI系统之Drawable
description: "Android GUI分析之Drawable"
modified: 2016-08-13
type: dev
categories: [Android]
---

Android中Drawable可以用来作为图像显示或者作为View的背景，包路径为android.graphics.drawable，Drawable是可绘制对象的图像的抽象，它不接受事件，无法与用户交互，一般通过XML定义，占用空间小，在实际开发中作用不可小觑。  

##### Drawable的种类  

* Bitmap，简单常见的drawable,PNG或JPEG图片，对应xml标签<bitmap>
* Nine Patch，基于png格式的可伸缩区域 ，对应标签</nine-patch>
* Shape，允许size、颜色、渐变等可变的几何图形，对应xml标签，对应xml标签</shape>  
* Layer，层次化的drawable组合，对应xml标签</layer-list>  
* State List，Drawable集合，集合中每个Drawable对应View的一种状态，对应标签</selector> 
* Level List，Drawable集合，集合中每个Drawable有level的概念，对应标签</level-list>
* Scale，可根据level将指定Drawable缩放到一定比例，对应标签</scale>
* Transition，用于实现Drawable之间的淡入淡出效果，对应标签</transition>
* Inset，可接受其它Drawable内嵌到其中，对应标签</inset>
* Clip，可根据当前level裁剪另一个Drawable，对应标签</clip> 



##### 自定义Drawable

在需要圆形，圆角等图片时，除了可以自定义ImageView外，更简单高效的是自定义Drawable
使用getIntrinsicHeight()，getIntrinsicWidth()可以返回drawable的固有高宽，setBounds(Rect)用来确定Drawable被绘制的大小，常见用法：

{% highlight java %}
public class RoundDrawable extends Drawable {

    private Paint mPaint;
    private Bitmap mBitmap;

    private RectF rectF;

    public RoundDrawable(Bitmap bitmap) {
        mBitmap = bitmap;
        BitmapShader bitmapShader = new BitmapShader(bitmap, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP);
        mPaint = new Paint();
        mPaint.setShader(bitmapShader);
    }

    @Override
    public void draw(Canvas canvas) {
        canvas.drawRoundRect(rectF, 20, 20, mPaint);
    }

    @Override
    public void setAlpha(int alpha) {
        mPaint.setAlpha(alpha);
        invalidateSelf();
    }

    @Override
    public void setColorFilter(ColorFilter colorFilter) {
        mPaint.setColorFilter(colorFilter);
        invalidateSelf();
    }

    @Override
    public int getOpacity() {
        return PixelFormat.TRANSLUCENT;
    }

    @Override
    public void setBounds(int left, int top, int right, int bottom) {
        super.setBounds(left, top, right, bottom);
        rectF = new RectF(left, top, right, bottom);
    }
}
{% endhighlight %}


其中的PorterDuff提供了16种两图相交的模式，效果如下![Porterduff_mode Image]({{ site.url }}/images/android_image/porterduff_mode.png)
