---
layout: post
title: MPAndroidChart使用
description: "MPAndroidChart使用心得"
modified: 2017-03-15
type: dev
categories: [Android]
---

MPAndroidChart包含多种图表空控件。通用的属性方法包括：
* setData() 给图表设置数据
* clear() 清除图表控件中数据
* drawMarker(Canvans canvans) 基于高亮点绘制标签视图
* setTouchEnabled(boolean enabled) 控制响应touch事件
* setMarker(IMarker marker) 设置响应点击的标签视图，需要实现IMarker
* setDescription(Description desc) 图表描述
* getXAxis() 获取X轴实例
* disableScroll()/enableScroll() 禁用／启用拦截touch事件
* getData() 返回图表数据

##### 折线图  

gradle引入项目后，xml中直接通过<com.github.mikephil.charting.charts.LineChart />  引用，
默认的图标会绘制出x、y轴线，如不需要可通过getXAxis()、getAxisLeft()、getAxisRight()获取x、y轴，并setEnabled(false)禁止其绘制，实现MarkerView后需要点击方绘制在对应的position，如果需要默认显示某一MarkerView，可通过实例化Highlight对象实现：

    Hightlight hightlight = new Hightlight(x, int dataSetIndex)
    chart.highlightValue(highlight, false)
其中x为x轴position值，dataSetIndex为数据索引。

##### 饼图

