---
layout: post
title: Android GUI 系统之View
description: "Android GUI系统之View"
modified: 2016-08-16
type: dev
dev: [Android]
---

android.view.View类呈现最基本的UI构造块。一个View（视图）占据屏幕上的一个方形区域，是与用户交互的直接所在，负责绘制和事件处理。 

![View Image]({{ site.url }}/images/android_post/android_view.png)
{: .image-right}  

Android中的UI元素常常在Layout中进行描述，android.view.View的其中一个重要扩展者是android.view.ViewGroup，它表示一个View的集合，其中可以包含众多子View，其本身也是一个View。View用@UiThread来注解，表明其工作在UI线程(app主线程)中，调用所有与之相关的方法应在主线程中。(android中所有的应用组件(Activities, Services, ContentProviders, BroadcastReceivers)在UI线程中被创建)，下文分别从View的位置，滑动，绘制，事件机制，自定义等几个方面加以分析，希望能有一个直观清楚的梳理。    

<!-- more -->  

#### View 的位置  

![View Image]({{ site.url }}/images/android_post/android_view_position.png)
{: .image-right}  

如图所示，View的位置由四个顶点决定，分别对应View的四个属性：left(左横坐标)、top(上纵坐标)、right(右横坐标)、bottom(下纵坐标)，其值分别由getLeft()、getTop()、getRight()、getBottom()获取，宽高和坐标的关系：

{% highlight java %}
width = right - left  
height = bottom - top
{% endhighlight %}

除此之外，View还有相对于父容器的几个值x(左上角横坐标), y(左上角纵坐标), translationX(View左上角相对于父容器的X方向偏移量), translationY(View左上角相对于父容器的Y方向偏移量)，对应关系为：  

{% highlight java %}
x = left + translationX  
y = top + translationY
{% endhighlight %}

#### View 的滑动  

滑动操作对Android应用的重要性不言而喻，通常可以通过以下3种方式实现View的滑动：

*    使用scrollTo/scrollBy
     这两个方法的源码如下： 

{% highlight java %}
/**
 * Set the scrolled position of your view. This will cause a call to
 * {@link #onScrollChanged(int, int, int, int)} and the view will be
 * invalidated.
 * @param x the x position to scroll to
 * @param y the y position to scroll to
 */
public void scrollTo(int x, int y) {
    if (mScrollX != x || mScrollY != y) {
        int oldX = mScrollX;
        int oldY = mScrollY;
        mScrollX = x;
        mScrollY = y;
        invalidateParentCaches();
        onScrollChanged(mScrollX, mScrollY, oldX, oldY);
        if (!awakenScrollBars()) {
            postInvalidateOnAnimation();
        }
     }
}
    
/**
 * Move the scrolled position of your view. This will cause a call to
 * {@link #onScrollChanged(int, int, int, int)} and the view will be
 * invalidated.
 * @param x the amount of pixels to scroll by horizontally
 * @param y the amount of pixels to scroll by vertically
 */
public void scrollBy(int x, int y) {
    scrollTo(mScrollX + x, mScrollY + y);
}
{% endhighlight %}

其中，mScrollX为View内容左边缘与View左边缘在水平方向的距离，mScrollY为View内容上边缘与View上边缘在竖直方向上的距离，其取值正负如图所示。scrollBy也是调用scrollTo方法，它改变的是View内容位置而不是View本身位置。

  ![View Image]({{ site.url }}/images/android_post/android_view_coordinate.png)
  {: .image-right}

* 使用动画

  [Android 动画详解](https://zhoushibo.com/android/android-animation/)。

* 改变布局参数

  改变LayoutParams，可以有两种方式实现，一是改变View的margin值，二是在View旁边增加空白View，通过改变View的宽高让View实现滑动。改变margin值实现滑动例：  

{% highlight java %}
ImageView imgTest = (ImageView) findViewById(R.id.img_test);
ViewGroup.MarginLayoutParams params =(ViewGroup.MarginLayoutParams)imgTest.getLayoutParams();
params.width += 150;
params.leftMargin += 150;
imgTest.requestLayout();
{% endhighlight %}
上文所述的View滑动都是在瞬时完成，要提升用户体验，需要动画是渐进式的，也就是把一次大的滑动拆分成若干次小的滑动，在一段时间内完成。通常可以有使用Scroller、使用动画、使用延时策略三种方式，下面分别说明。

1. 使用Scroller

   Scroller即弹性滑动对象，通常Scroller的用法如下：  

{% highlight java %}
Scroller scroller = new Scroller(mContext);
private void smoothScrollTo(int destX, int destY) {
    int scrollX = getScrollX();
    int delta = destX - scrollX;
    scroller.startScroll(scrollX, 0, delta, 0, 1000);
    invalidate();
}

@Override
public void computeScroll() {
    if (scroller.computeScrollOffset()) {
   	scrollTo(scroller.getCurrX(), scroller.getCurrY());
    postInvalidate();
    }
}
{% endhighlight %}
从中发现Scroller对象调用startScroll方法，该方法实现如下：

{% highlight java %}
public void startScroll(int startX, int startY, int dx, int dy, int duration) {
	mMode = SCROLL_MODE;
	mFinished = false;
	mDuration = duration;
	mStartTime = AnimationUtils.currentAnimationTimeMillis();
	mStartX = startX;
	mStartY = startY;
	mFinalX = startX + dx;
	mFinalY = startY + dy;
	mDeltaX = dx;
	mDeltaY = dy;
	mDurationReciprocal = 1.0f / (float) mDuration;
}
{% endhighlight %}

可以看到最终的位置是在开始的位置上加滑动的距离，即startX和startY是滑动的起点坐标，dx和dy表示滑动的距离，duration表示滑动的时间，Scroller本身并不能实现那View的滑动，需要配合View的computeScroll使用。**使用时调用invalidate方法导致View重绘，在View的draw方法中会调用computeScroll方法，computeScroll方法在View中是一个空实现，我们需要如上面那样去实现此方法获取当前的scrollX和scrollY，然后通过scrollTo实现滑动，接着又调用postInvalidate方法进行第二次重绘，如此重复直到滑动结束**。  

2. 使用动画

   其思想类似Scroller，在动画对象的监听里按百分比实现scrollTo完成View的滑动。

3. 使用延时策略

   延时策略通过使用Handler或View的postDelayed方法，以及线程的sleep方法，不断发送延时消息配合scrollTo完成弹性滑动。

#### View 的绘制

有机的展示视图定然离不开视图管理和呈现，这两部分工作分别对应于WindowManager和DecorView，两者又是如何关联的呢？答案是通过ViewRoot。Activity本身并不和视图控制相关，只控制生命周期和事件的处理，真正控制的视图控制者是Window，在ActivityThread中，Activity对象被创建后，会将DecorView添加到Window中，并创建ViewRootImpl对象，将ViewRootImpl对象和DecorView对象建立关联，过程实现在WindowManagerGlobal类的addView方法中：  

{% highlight java %}
root = new ViewRootImpl(view.getContext(), display);
view.setLayoutParams(wparams);
mViews.add(view);
mRoots.add(root);
mParams.add(wparams);
root.setView(view, wparams, panelParentView);
{% endhighlight %}
View的绘制从ViewRootImpl的performTraversals方法开始，经过measure（测量View的宽高）、layout（确定View在父容器中的位置）、draw（绘制）三个阶段，首先依次调用performMeasure、performLayout、performDraw方法完成顶层View的measure、layout、draw。在performMeasure中调用measure方法，其中onMeasure会对所有子元素进行measure，接着子元素会重复父容器的measure过程以完成View树的遍历，其它两个过程与之类似，performLayout过程通过layout实现，performDraw的传递过程在draw方法中的dispatchOnDraw实现。

![View Image]({{ site.url }}/images/android_post/android_decorview.png)
{: .image-right}

DecorView是顶级的FrameLayout View，它是一个竖直摆放的LinearLayout，里面包含了title和content两部分，在Activity中setContentView方法所设置的View就是content部分。View的顶层事件都是先经过DecorView，再传给我们可见的其它子View。

###### View measure

在View的measure中，有个至关重要的概念是MeasureSpec，它决定了View的尺寸规格，当然View的尺寸规格还会受到父容器的影响，系统会将View的LayoutParams根据父容器所施加的规则转换成对应的MeasureSpec，测量出View的宽、高。MeasureSpec的核心方法如下：  

{% highlight java %}
private static final int MODE_SHIFT = 30;
private static final int MODE_MASK  = 0x3 << MODE_SHIFT;
public static final int UNSPECIFIED = 0 << MODE_SHIFT;
public static final int EXACTLY     = 1 << MODE_SHIFT;
public static final int AT_MOST     = 2 << MODE_SHIFT;
        
public static int makeMeasureSpec(int size, int mode) {
    if (sUseBrokenMakeMeasureSpec) {
        return size + mode;
    } else {
        return (size & ~MODE_MASK) | (mode & MODE_MASK);
    }
}

public static int makeSafeMeasureSpec(int size, int mode) {
    if (sUseZeroUnspecifiedMeasureSpec && mode == UNSPECIFIED) {
        return 0;
    }
        return makeMeasureSpec(size, mode);
    }

public static int getMode(int measureSpec) {
    return (measureSpec & MODE_MASK);
}

public static int getSize(int measureSpec) {
    return (measureSpec & ~MODE_MASK);
}
{% endhighlight %}
MeasureSpec代表一个32位的int值，高2位代表测量模式SpecMode，低30位代表规格大小SpecSize，SpecMode有3类，分别是：
* UNSPECIFIED
  父容器不对View做任何限制，此模式一般用于系统内部，表示一种测量状态
* EXACTLY
  父容器检测出了View所需要的精确大小，此模式下View的最终大小就是SpecSize所指定的值。对应于LayoutParams中的match_parent和具体的数值这两种模式。
* AT_MOST
  父容器指定了一个可用大小即SpecSize，View的大小不能大于这个值，具体是什么值要看不同View的具体实现。对应于LayoutParams中的wrap_content。  
  View测量的时候，系统会将LayoutParams在父容器的约束下转换成对应的MeasureSpec，然后再根据这个MeasureSpec来确定View测量的宽和高。这一过程作用在顶级View（DecorView）和普通View各有不同，对于DecorView，其MeasureSpec由窗口的尺寸和其自身的LayoutParams共同确定，对于普通View，其MeasureSpec由父容器的MeasureSpec和自身的LayoutParams来共同决定，MeasureSpec一旦确定，onMeasure就可以确定View的宽高（对于DecorView和普通View 的不同其实也不难理解，因为DecorView直接加在窗口中，没有父View，当然受窗口尺寸影响）。

DecorView：  
ViewRootImpl中的measureHierarchy方法确定了DecorView的MeasureSpec创建过程，它会调用getRootMeasureSpec方法如下：  

{% highlight java %}
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {
    case ViewGroup.LayoutParams.MATCH_PARENT:
        // Window can't resize. Force root view to be windowSize.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // Window can resize. Set max size for root view.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        // Window wants to be an exact size. Force root view to be that size.
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
{% endhighlight %}

由以上代码可见，如果是LayoutParams.MATCH_PARENT模式，DecorView的大小就是窗口的大小，如果是LayoutParams.WRAP_CONTENT模式，DecorView采用最大模式，最大值不能超过窗口大小，默认情况下Decor大小位固定大小，其值为LayoutParams中指定的大小（如100dp）。

布局中的普通View：

View的measure过程由ViewGroup完成，其对应的方法为measureChildWithMargins，如下：  

{% highlight java %}
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
    
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);
    
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
{% endhighlight %}
不难发现，在调用子元素的measure方法之前，先调用了getChildMeasureSpec方法得到子元素的MeasureSpec，并且这一值的得出和父容器的MeasureSpec、View的margin、View的padding相关，getChildMeasureSpec确定了View的大小，源码如下：  

{% highlight java %}
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);
    int size = Math.max(0, specSize - padding);
    int resultSize = 0;
    int resultMode = 0;
    
    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
{% endhighlight %}
当View采用固定宽高时，View的MeasureSpec不受父容器的MeasureSpec影响，始终是EXACTLY（精确模式）；当View的宽高是MATCH_PARENT时，如果父容器的模式是EXACTLY，View也是EXACTLY，并且大小是父容器的剩余空间，如果父容器的模式是AL_AMOST（最大模式），那么View也是最大模式且不超过父容器剩余空间；当View的宽高采用wrap_content时，无论父容器的模式是EXACTLY（精确模式）还是ATMOST（最大模式），View的模式都是AL_MOST（最大模式）且大小不会超过父容器的剩余空间。  
View的measure过程由measure方法调用onMeasure方法完成，onMeasure方法如下：  

{% highlight java %}
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
{% endhighlight %}
其中调用的getDefaultSize如下：  

{% highlight java %}
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);
    
    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
{% endhighlight %}

可见View的宽高由specSize决定。//TODO自定义View的onMeasure方法特殊处理

上述过程是View的measure过程，ViewGroup除了上述过程外，还需要遍历完成所有子元素的measure，与View中的measure对应的方法是measureChildren：  

{% highlight java %}
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
    final int size = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < size; ++i) {
        final View child = children[i];
        if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
        }
    }
}
{% endhighlight %}

ViewGroup是abstract类，它不同的子类有不同的布局特性，所以它并没有定义测量的具体过程，其测量交由子类具体实现，如LinearLayout，RelativeLayout等。measure完成后，可以通过get方法获取mMeasuredWidth，mMeasuredHeight。

###### View 布局

ViewGroup通过Layout来确定子元素的位置，ViewGroup中通过onLayout方法确定子元素的位置，子元素通过layout方法确定本身的位置，layout方法如下：  

{% highlight java %}
@SuppressWarnings({"unchecked"})
public void layout(int l, int t, int r, int b) {
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
        onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }
    
    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;
    
    boolean changed = isLayoutModeOptical(mParent) ?
          setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
        mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLayoutChangeListeners != null) {
            ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
            }
        }
    }
    
    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
    mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
}
{% endhighlight %}

其中setFrame确定四个顶点的位置，onLayout方法确定子元素的位置，onLayout方法具体实现由子类完成 ，View的宽高有测量宽高和最终宽高，通常来讲，二者是一致的，get方法如下：

{% highlight java %}
@ViewDebug.ExportedProperty(category = "layout")
public final int getWidth() {
    return mRight - mLeft;
}

@ViewDebug.ExportedProperty(category = "layout")
public final int getHeight() {
    return mBottom - mTop;
}

public final int getMeasuredWidth() {
   return mMeasuredWidth & MEASURED_SIZE_MASK;
}

public final int getMeasuredWidthAndState() {
    return mMeasuredWidth;
}
{% endhighlight %}

###### View 的绘制  

View的绘制分为6步（步骤2&5非必须），源码见View类的draw方法

1. drawBackground，如果有Background，绘制Background
2. onDraw，绘制本身
3. dispatchDraw，绘制子View
4. onDrawForeground，绘制前景

#### View 事件机制

首先是相关概念的介绍：MotionEvent、TouchSlop、VelocityTracker、GestureDetector
MotionEvent是View的事件对象，继承自InputEvent，用户在于设备接触 过程中产生的事件有3种：  
- ACTION_DOWN，接触屏幕的按下事件
- ACTION_MOVE，在按下事件基础上的移动事件
- ACTION_UP，从屏幕上松开的事件

TouchSlop是用来界定是否滑动的最小距离，可以通过ViewConfiguration.get(context).getScaledTouchSlop()获取，VelocityTracker，速度追踪，用来追踪滑动过程中移动速度，包括水平和垂直方向的速度，参见类VelocityTracker。GestureDetector，手势监测，监测用户手势行为，使用时创建GestureDetector对象并实现相应接口：

{% highlight java %}
GestureDetector mGestureDetector = new GestureDetector(this, this);
mGestureDetector.setIsLongpressEnabled(false);
mGestureDetector.setOnDoubleTapListener(this);
{% endhighlight %}

这两个接口分别对应如下方法：

| 返回类型    | 方法名                  | 说明                                       | 接口名称                |
| ------- | -------------------- | ---------------------------------------- | ------------------- |
| boolean | onDown               | 触摸屏幕瞬间产生的事件，由ACTION_DOWN触发               | OnGestureListener   |
| void    | onShowPress          | 触摸屏幕时ACTION_DOWN触发，此时没有松开也没有移动           | OnGestureListener   |
| boolean | onSingleTapUp        | 单击行为中的松开事件，由ACTION_UP触发                  | OnGestureListener   |
| boolean | onScroll             | 按下并滑动的事件，由1个ACTION_DOWN和多个ACTION_MOVE触发  | OnGestureListener   |
| void    | onLongPress          | 长按事件                                     | OnGestureListener   |
| boolean | onFling              | 按下，快速滑动后松开，包括了ACTION_DOWN，ACTION_MOVE，ACTION_UP | OnGestureListener   |
| boolean | onSingleTapConfirmed | 单击事件，比onSingleTapUp更为严格，不同之处在于其后不能跟另一个单击行为 | OnDoubleTapListener |
| boolean | onDoubleTap          | 双击事件，由两次onSingleTapUp单击事件组成              | OnDoubleTapListener |
| boolean | onDoubleTapEvent     | 双击行为事件，按下，滑动，抬起均会触发                      | OnDoubleTapListener |

View事件分发的对象就是MotionEvent，过程由3个方法完成：dispatchTouchEvent、onInterceptTouchEvent、onTouchEvent。

- dispatchTouchEvent 用于事件分发，所有事件必须经过此方法分发，然后决定当前事件是自身消费还是继续往下分发给子控件。返回true表示不继续分发，事件没有被消费。返回false表示继续分发，如果是ViewGroup则分发给onInterceptTouchEvent进行判断是否拦截事件。
- onInterceptTouchEvent ViewGroup独有的方法，负责事件的拦截，返回true表示拦截当前事件不继续往下分发，交给自身onTouchEvent处理。返回false表示不拦截，继续往下传。
- onTouchEvent 在dispatchTouchEvent中调用来进行事件的处理，返回true表示消费当前事件，返回false不处理，交给子控件继续进行分发。

用自定义View测试View的事件机制，在自定EA义ImageView中复写方法diapatchTouchEvent和onTouchEvent方法如下：

{% highlight java %}
@Override
public boolean dispatchTouchEvent(MotionEvent event) {
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            System.out.println("EAImageView dispatchTouchEvent DOWN");
            break;
        case MotionEvent.ACTION_MOVE:
            System.out.println("EAImageView dispatchTouchEvent MOVE");
            break;
        case MotionEvent.ACTION_UP:
            System.out.println("EAImageView dispatchTouchEvent UP");
            break;
        default:
            break;
    }
    return super.dispatchTouchEvent(event);
}

@Override
public boolean onTouchEvent(MotionEvent event) {
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            System.out.println("EAImageView onTouchEvent DOWN");
            break;
        case MotionEvent.ACTION_MOVE:
            System.out.println("EAImageView onTouchEvent MOVE");
            break;
        case MotionEvent.ACTION_UP:
            System.out.println("EAImageView onTouchEvent UP");
            break;
        default:
            break;
    }
    return super.onTouchEvent(event);
}
{% endhighlight %}

在Activity布局文件中加入此自定义控件，设置其OnClickListener,OnTouchListener，并复写Activity的onTouch,

dispatchTouchEvent,onTouchEvent方法：

{% highlight java %}
@Override
public boolean onTouch(View v, MotionEvent event) {
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            System.out.println("EAImageView onTouch DOWN");
            break;
        case MotionEvent.ACTION_MOVE:
            System.out.println("EAImageView onTouch MOVE");
            break;
        case MotionEvent.ACTION_UP:
            System.out.println("EAImageView onTouch UP");
            break;
        default:
            break;
    }
    return false;
}

@Override
public boolean dispatchTouchEvent(MotionEvent event) {
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            System.out.println("Activity dispatchTouchEvent DOWN");
            break;
        case MotionEvent.ACTION_MOVE:
            System.out.println("Activity dispatchTouchEvent MOVE");
            break;
        case MotionEvent.ACTION_UP:
            System.out.println("Activity dispatchTouchEvent UP");
            break;
        default:
            break;
    }
    return super.dispatchTouchEvent(event);
}

@Override
public boolean onTouchEvent(MotionEvent event) {
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            System.out.println("Activity onTouchEvent DOWN");
            break;
        case MotionEvent.ACTION_MOVE:
            System.out.println("Activity onTouchEvent MOVE");
            break;
        case MotionEvent.ACTION_UP:
            System.out.println("Activity onTouchEvent UP");
            break;
        default:
            break;
    }
    return super.onTouchEvent(event);
}
{% endhighlight %}

点击ImageView，日志输入顺序如下：

{% highlight java %}
I/System.out: Activity dispatchTouchEvent DOWN
I/System.out: EAImageView dispatchTouchEvent DOWN
I/System.out: EAImageView onTouch DOWN
I/System.out: EAImageView onTouchEvent DOWN
I/System.out: Activity dispatchTouchEvent MOVE
I/System.out: EAImageView dispatchTouchEvent MOVE
I/System.out: EAImageView onTouch MOVE
I/System.out: EAImageView onTouchEvent MOVE
I/System.out: Activity dispatchTouchEvent MOVE
I/System.out: EAImageView dispatchTouchEvent MOVE
I/System.out: EAImageView onTouch MOVE
I/System.out: EAImageView onTouchEvent MOVE
I/System.out: Activity dispatchTouchEvent MOVE
I/System.out: EAImageView dispatchTouchEvent MOVE
I/System.out: EAImageView onTouch MOVE
I/System.out: EAImageView onTouchEvent MOVE
I/System.out: Activity dispatchTouchEvent MOVE
I/System.out: EAImageView dispatchTouchEvent MOVE
I/System.out: EAImageView onTouch MOVE
I/System.out: EAImageView onTouchEvent MOVE
I/System.out: Activity dispatchTouchEvent UP
I/System.out: EAImageView dispatchTouchEvent UP
I/System.out: EAImageView onTouch UP
I/System.out: EAImageView onTouchEvent UP
I/System.out: EAImageView clicked!
{% endhighlight %}

可见事件的传递总是从Activity开始，Activity传给Window，Window再传给DecorView，DecorView收到事件后，就会按照事件分发机制分发事件，并且View的OnTouchListener优先级 > onTouchEvent > OnClickListener。下面按照日志逐一分析：

首先执行TestActivity的dispatchTouchEvent进行事件分发，dispatchTouchEvent返回super.dispatchTouchEvent(event)，调用基类Activity的dispatchTouchEvent进行事件分发，基类方法为：

{% highlight java %}
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
{% endhighlight %}

因为事件的前提是有按下事件，所以这里只需要对ACTION_DOWN进行处理，如按下事件成立，调用onUserInteraction方法，这是一个空方法，需要在自己的Activity中实现它，可以用它在事件分发前对事件进行处理，不对事件传递结果产生影响，接着判断getWindow().superDispatchTouchEvent(ev)，源码如下：

{% highlight java %}
/**
* Used by custom windows, such as Dialog, to pass the touch screen event
* further down the view hierarchy. Application developers should
* not need to implement or call this.
    *
    */
   public abstract boolean superDispatchTouchEvent(MotionEvent event);
{% endhighlight %}

这是一个应用于自定义Window的抽象方法，例如Dialog传递触屏事件，我们不需要实现此方法，在我们的TestActivity中如果将dispatchTouchEvent返回设置为true，那么这里执行结果就是true，因为并没有这么设置，将调用下面的onTouchEvent方法，事件从Activity分发到了EAImageView，但通过日志发现接下来首先调用的却是onTouch，怎么回事呢， 我们来到View的dispatchTouchEvent方法查看源码：

{% highlight java %}
public boolean dispatchTouchEvent(MotionEvent event) {
    // If the event should be handled by accessibility focus first.
    if (event.isTargetAccessibilityFocus()) {
    // We don't have focus or no virtual descendant has it, do not handle the event.
        if (!isAccessibilityFocusedViewOrHost()) {
             return false;
        }
        // We have focus and got the event, then use normal event dispatch.
        event.setTargetAccessibilityFocus(false);
    }
    
    boolean result = false;
    
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(event, 0);
    }
    
    final int actionMasked = event.getActionMasked();
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        // Defensive cleanup for new gesture
        stopNestedScroll();
    }
    
    if (onFilterTouchEventForSecurity(event)) {
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }
    
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }
    
    if (!result && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
    }
    
    // Clean up after nested scrolls if this is the end of a gesture;
    // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
    // of the gesture.
    if (actionMasked == MotionEvent.ACTION_UP ||
            actionMasked == MotionEvent.ACTION_CANCEL ||
            (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
        stopNestedScroll();
    }
    
    return result;
}
{% endhighlight %}

其中当li.mOnTouchListener != null和当前View是ENABLE条件满足时返回true，而mOnTouchListener是通过如下方法设置：

{% highlight java %}
public void setOnTouchListener(OnTouchListener l) {
    getListenerInfo().mOnTouchListener = l;
}
{% endhighlight %}

在TestActivity中，我们设置了View的OnTouchListener，就是调用此方法，如果在TestActivity的onTouch方法返回true，则整个条件都满足，因此dispatchTouchEvent返回true，表示事件不继续往下分发，被onTouch消费了，如果onTouch方法返回false，则判断条件不成立，接着执行onTouchEvent(event)进行判断，如果该方法返回true，表示事件被onTouchEvent处理了，dispatchTouchEvent返回true。由此可见此处的执行顺序是先onTouch后onTouchEvent。接着追踪源码发现在onTouchEvent方法中，MotionEvent.ACTION_UP抬起事件中会调用performClick，该方法会调用li.mOnClickListener.onClick(this)，这也解释了为什么onClick为什么最后调用：

{% highlight java %}
public boolean performClick() {
    final boolean result;
    final ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        li.mOnClickListener.onClick(this);
        result = true;
    } else {
        result = false;
    }
    
    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
    return result;
}
{% endhighlight %}

上面分析的是View的事件分发机制，对于ViewGroup来说是怎样的呢，我们先定义一个EALayout，复写dispatchTouchEvent、onInterceptTouchEvent、onTouchEvent方法，在xml中为刚才的EA添加该层EALayout，并在TestActivity中为EALayout设置onTouch和onClick事件监听，点击按钮日志输出如下：

{% highlight java %}
I/System.out: Activity dispatchTouchEvent DOWN
I/System.out: EALayout dispatchTouchEvent DOWN
I/System.out: EALayout onInterceptTouchEvent DOWN
I/System.out: EAImageView dispatchTouchEvent DOWN
I/System.out: EAImageView onTouch DOWN
I/System.out: EAImageView onTouchEvent DOWN
I/System.out: Activity dispatchTouchEvent MOVE
I/System.out: EALayout dispatchTouchEvent MOVE
I/System.out: EALayout onInterceptTouchEvent MOVE
I/System.out: EAImageView dispatchTouchEvent MOVE
I/System.out: EAImageView onTouch MOVE
I/System.out: EAImageView onTouchEvent MOVE
I/System.out: Activity dispatchTouchEvent MOVE
I/System.out: EALayout dispatchTouchEvent MOVE
I/System.out: EALayout onInterceptTouchEvent MOVE
I/System.out: EAImageView dispatchTouchEvent MOVE
I/System.out: EAImageView onTouch MOVE
I/System.out: EAImageView onTouchEvent MOVE
I/System.out: Activity dispatchTouchEvent MOVE
I/System.out: EALayout dispatchTouchEvent MOVE
I/System.out: EALayout onInterceptTouchEvent MOVE
I/System.out: EAImageView dispatchTouchEvent MOVE
I/System.out: EAImageView onTouch MOVE
I/System.out: EAImageView onTouchEvent MOVE
I/System.out: Activity dispatchTouchEvent MOVE
I/System.out: EALayout dispatchTouchEvent MOVE
I/System.out: EALayout onInterceptTouchEvent MOVE
I/System.out: EAImageView dispatchTouchEvent MOVE
I/System.out: EAImageView onTouch MOVE
I/System.out: EAImageView onTouchEvent MOVE
I/System.out: Activity dispatchTouchEvent UP
I/System.out: EALayout dispatchTouchEvent UP
I/System.out: EALayout onInterceptTouchEvent UP
I/System.out: EAImageView dispatchTouchEvent UP
I/System.out: EAImageView onTouch UP
I/System.out: EAImageView onTouchEvent UP
I/System.out: EAImageView clicked!
{% endhighlight %}

从日志顺序得出ViewGroup的事件传递顺序为Activity－>自定义Layout－>自定义View，可见事件的传递是从ViewGroup到View，接着调用了EALayout的dispatchTouchEvent，在其中作了什么处理呢？ViewGroup的相关源码如下：

{% highlight java %}
// Check for interception.
final boolean intercepted;
if (actionMasked == MotionEvent.ACTION_DOWN
        || mFirstTouchTarget != null) {
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {
        intercepted = onInterceptTouchEvent(ev);
        ev.setAction(action); // restore action in case it was changed
    } else {
        intercepted = false;
    }
} else {
	// There are no touch targets and this action is not an initial down
	// so this view group continues to intercept touches.
	intercepted = true;
	}
{% endhighlight %}

其中调用了onInterceptTouchEvent方法，如果返回false则不拦截，如果返回true则拦截，其默认返回为false，

{% highlight java %}
public boolean onInterceptTouchEvent(MotionEvent ev) {
    return false;
}
{% endhighlight %}

如果我们将EALayout中的onInterceptTouchEvent返回改为true，重新调试得到日志如下：

{% highlight java %}
I/System.out: Activity dispatchTouchEvent DOWN
I/System.out: EALayout dispatchTouchEvent DOWN
I/System.out: EALayout onInterceptTouchEvent DOWN
I/System.out: EALayout onTouch DOWN
I/System.out: EALayout onTouchEvent DOWN
I/System.out: Activity dispatchTouchEvent MOVE
I/System.out: EALayout dispatchTouchEvent MOVE
I/System.out: EALayout onTouch MOVE
I/System.out: EALayout onTouchEvent MOVE
I/System.out: Activity dispatchTouchEvent MOVE
I/System.out: EALayout dispatchTouchEvent MOVE
I/System.out: EALayout onTouch MOVE
I/System.out: EALayout onTouchEvent MOVE
I/System.out: Activity dispatchTouchEvent MOVE
I/System.out: EALayout dispatchTouchEvent MOVE
I/System.out: EALayout onTouch MOVE
I/System.out: EALayout onTouchEvent MOVE
I/System.out: Activity dispatchTouchEvent MOVE
I/System.out: EALayout dispatchTouchEvent MOVE
I/System.out: EALayout onTouch MOVE
I/System.out: EALayout onTouchEvent MOVE
I/System.out: Activity dispatchTouchEvent UP
I/System.out: EALayout dispatchTouchEvent UP
I/System.out: EALayout onTouch UP
I/System.out: EALayout onTouchEvent UP
I/System.out: EALayout clicked!
{% endhighlight %}

此次输出日志中，完全没了EAImageView的踪影，正是由于在EALayout的onInterceptTouchEvent中返回true，拦截了事件，并交给EALayout的onTouchEvent并在最后执行其点击事件，导致事件不再分发给EAImageView的缘故。

另外从下面源码中发现setOnClickListener和setOnLongClickListener会自动将View的CLICKABLE和LONG_CLICKABLE设置为true：

{% highlight java %}
public void setOnClickListener(@Nullable OnClickListener l) {
    if (!isClickable()) {
        setClickable(true);
    }
    getListenerInfo().mOnClickListener = l;
}

public void setOnLongClickListener(@Nullable OnLongClickListener l) {
    if (!isLongClickable()) {
        setLongClickable(true);
    }
    getListenerInfo().mOnLongClickListener = l;
}
{% endhighlight %}

#### View 自定义  

自定义View按照基类的不同可分为两种不同类型，一类是继承自View及其子类（如ImageView、TextView等）的自定义View，另一类是继承自ViewGroup及其子类（如FrameLayout、LinearLayout）的布局Layout。通常需要注意的问题包括但不限于wrap_content、padding不生效；滑动冲突；内存泄漏等。下面按照基类的不同对两种自定义View分别说明。

##### 继承自View  

继承自View的情况一般用在需要实现一些系统没有的形状效果，或是在系统已有View的基础上增加其它属性，此时一般需要复写onDraw方法，如果直接集成View，需要对wrap_content、padding做处理，如果继承自系统派生的如ImageView之类的View，则不需要对wrap_content、padding做处理（通过前面的分析，这也很好理解，查看ImageView的源码，也能发现其onMeasure和onDraw方法已对相应情况作了处理）。要想在布局中使用自定义View时wrap_content生效，需要在onMeasure中对MeasureSpec和MeasureMode处理，再调用setMeasuredDimension方法，要想使用时padding生效，则需要在onDraw中对paddingLeft、paddingTop、paddingRight、paddingBottm做处理。如下例：  

{% highlight java %}
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec , heightMeasureSpec);
    int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
    int widthSpceSize = MeasureSpec.getSize(widthMeasureSpec);
    int heightSpecMode=MeasureSpec.getMode(heightMeasureSpec);
    int heightSpceSize=MeasureSpec.getSize(heightMeasureSpec);
    
    if(widthSpecMode==MeasureSpec.AT_MOST&&heightSpecMode==MeasureSpec.AT_MOST){
        setMeasuredDimension(mWidth, mHeight);
    }else if(widthSpecMode==MeasureSpec.AT_MOST){
        setMeasuredDimension(mWidth, heightSpceSize);
    }else if(heightSpecMode==MeasureSpec.AT_MOST){
        setMeasuredDimension(widthSpceSize, mHeight);
    }
}
    
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    int width = getWidth();
    int height = getHeight();
    int radius = Math.min((width - getPaddingLeft() - getPaddingRight()) / 2,
        (height - getPaddingTop() - getPaddingBottom()) / 2);
    canvas.drawColor(Color.GRAY);
    canvas.drawCircle(width / 2, height / 2, radius, mPaint);
}
{% endhighlight %}


##### 继承自ViewGroup  

继承自ViewGroup需要处理ViewGroup的测量、布局，同时也需要考虑子元素的布局，可能会涉及到的方法有：onInterceptTouchEvent、onTouchEvent、onMeasure、onLayout等，不再详细说明。  

对于解决滑动冲突来说，把握的思想是根据场景或者滑动手势、角度确定当前需要滑动的方向，然后对事件拦截处理，父容器需要此事件就拦截，不需要此事件就不拦截（也可以不通过父容器而通过子元素判断，即父容器不拦截事件，所有事件交由子元素处理，子元素可以选择直接消费事件或者交给父容器处理，此方法需要配合getParent().requestDisallowInterceptTouchEvent方法使用），内存泄漏需要注意的是如果在自定义View中尽量用post而不是Handler来处理消息，如果有新启动的线程或者动画，最好是在onAttachedToWindow中启动，在onDetachedFromWindow中停止。
