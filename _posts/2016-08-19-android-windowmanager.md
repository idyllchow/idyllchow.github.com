---
layout: post
title: Android GUI 系统之Window,WindowManager
description: "Android GUI分析之Window,WindowManager"
modified: 2016-08-19
dev: [Android]
type: dev
---

Android中View提供用户可见的视图，它们是怎么组织到一起的呢？它们又是如何被管理的呢？答案就是Window和WindownManager。  

#### Window 创建

如同江湖是依赖于侠客的抽象存在一样，有View的地方就有Window，Window是一个抽象基类，PhoneWindow是其实现类，View依附于Window而存在，也就是说View的直接管理者是Window而非Activity、Dialog、Toast。那Window又是如何被创建的呢？拿Activity中Window的创建来说：



<!-- more -->    



Activity 的Window对象创建发生在attach方法

{% highlight java %}
mWindow = new PhoneWindow(this);
mWindow.setCallback(this);
{% endhighlight %}

此处设置的Callback是什么呢？

{% highlight java %}
public interface Callback {
        public boolean dispatchKeyEvent(KeyEvent event);
        public boolean dispatchKeyShortcutEvent(KeyEvent event);
        public boolean dispatchTouchEvent(MotionEvent event);
        public boolean dispatchTrackballEvent(MotionEvent event);
        public boolean dispatchGenericMotionEvent(MotionEvent event);
        public boolean dispatchPopulateAccessibilityEvent(AccessibilityEvent event);
        public View onCreatePanelView(int featureId);
        public boolean onCreatePanelMenu(int featureId, Menu menu);
        public boolean onPreparePanel(int featureId, View view, Menu menu);
        public boolean onMenuOpened(int featureId, Menu menu);
        public boolean onMenuItemSelected(int featureId, MenuItem item);
        public void onWindowAttributesChanged(WindowManager.LayoutParams attrs);
        public void onContentChanged();
        public void onWindowFocusChanged(boolean hasFocus);
        public void onAttachedToWindow();
        public void onDetachedFromWindow();
        public void onPanelClosed(int featureId, Menu menu);
        public boolean onSearchRequested();
        public boolean onSearchRequested(SearchEvent searchEvent);
        public ActionMode onWindowStartingActionMode(ActionMode.Callback callback);
        public ActionMode onWindowStartingActionMode(ActionMode.Callback callback, int type);
        public void onActionModeStarted(ActionMode mode);
        public void onActionModeFinished(ActionMode mode);
    }
{% endhighlight %}

不难发现我们平时对Activity的诸多操作都体现在这些回调之中。创建了Window之后，接下来看看View怎么附着在Window上的，在Activity的setContentView方法中：

{% highlight java %}
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar(); //处理了根View的ActionBar
}
{% endhighlight %}

这里的具体操作交给了Window的具体实现类PhoneWindow，此处重载了几个setContentView方法：

{% highlight java %}
@Override
public void setContentView(View view, ViewGroup.LayoutParams params) {
    // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
    // decor, when theme attributes and the like are crystalized. Do not check the feature
    // before this happens.
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        view.setLayoutParams(params);
        final Scene newScene = new Scene(mContentParent, view);
        transitionTo(newScene);
    } else {
        mContentParent.addView(view, params);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
    cb.onContentChanged();
    }
}
{% endhighlight %}

可以看到，在PhoneWindow的setContentView方法中，如果放置DecorView及其子View的mContentParent为空，则先创建DecorView和mContentParent，分别通过下列方法：

{% highlight java %}
protected DecorView generateDecor() {
    return new DecorView(getContext(), -1);
}

protected ViewGroup generateLayout(DecorView decor) {
    ......
    View in = mLayoutInflater.inflate(layoutResource, null);
    decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
    mContentRoot = (ViewGroup) in;

    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    ......
    return contentParent;
}      
{% endhighlight %}

完成了DecorView中mContentParent的View添加后，将会回调onContentChanged方法通知Activity视图已发生改变。然后在ActivityThread中通过如下调用顺序被用户可见：handleResumeActivity->performResumeActivity->callActivityOnResume->onResume，handleResumeActivity中调用Activity的makeVisible方法设置mDecor可见。

其它诸如Dialog等类型的Window创建过程大致相同，不再赘述。



  

​









