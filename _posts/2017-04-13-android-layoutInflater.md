---
layout: post
title: Android LayoutInflater探究
description: "Android LayoutInflater探究"
modified: 2017-04-13
type: dev
categories: [Android]
---

LayoutInflater承载的功能是将一个布局文件实例化为对应的View对象，Inflater本身也有“充气”之意。对应的类LayoutInflater位于view包下，为抽象类。
获取标准的LayoutInflater实例可以通过Activity中的getLayoutInflater()或者Context中getSystemService()方法:
{% highlight java %}
 LayoutInflater inflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE)
{% endhighlight java %}

可以通过Factory接口为你自己的view创建一个新的LayoutInflater对象，你可以使用clineInContext克隆一个已存在的ViewFactory，然后调用setFactory设置成自己的Factory。基于性能的原因，view inflation(填充)主要依赖于在build时完成的预处理XML文件。它只适用于从编译的资源返回的XmlPullParser，而非运行时。
在进行view填充时常用到如下方法：
{% highlight java %}
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    if (DEBUG) {
        Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                + Integer.toHexString(resource) + ")");
    }
    
    final XmlResourceParser parser = res.getLayout(resource);
    try {
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
{% endhighlight java %}
其中参数resource为资源id，会以递归的方式完成装载，在解析资源文件的时候（android系统多采用pull方式解析）如果遇到merge标签【layout去掉外层多余root节点】，而此时root为空或者attachToRoot是false则会抛出InflateException异常。

参数root依赖于第三个参数attachToRoot，为生成层级的父view（attachToRoot为true 时），或者为返回的根层级提供一组LayoutParams值（attachToRoot为false时）。如果root为null，方法返回inflate的view，如果root不为null且attachToRoot为true，方法返回通过addview把inflate的view添加到root后的root。

参数attachToRoot用来确定被inflated的层级是否添加到root参数（第二个参数）， 如果为false，root仅用于为xml中的root view创建正确的LayoutParams子类。代码体现如下：

{% highlight java %}
if (root != null) {
    if (DEBUG) {
        System.out.println("Creating params from root: " +
                root);
    }
    // Create layout params that match root, if supplied
    params = root.generateLayoutParams(attrs);
    if (!attachToRoot) {
        // Set the layout params for temp if we are not
        // attaching. (If we are, we use addView, below)
        temp.setLayoutParams(params);
    }
}
{% endhighlight java %}

无论是在listview的getView()还是通过inflate填充view的时候，需要注意attachToRoot的使用。

