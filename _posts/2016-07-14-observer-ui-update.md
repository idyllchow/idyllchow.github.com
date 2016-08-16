---
layout: post
title: 观察者模式实现TextView实时更新
description: "观察者模式实现TextView实时更新"
modified: 2016-07-14
categories: [Android]
type: dev
---

### 背景

完成此功能之时，RxJava并没有大行其道，现在把之前的实现方式记录下来，当是回溯厘清自己思考问题的路子。项目APP中需要根据用户操作实时生成不同「事件」，并把事件的简短描述显示在某些View上。因为在不同的地方(类)会涉及到「事件」的变更，比如不同的操作会添加类型不同的一个「事件」，对应的有删除一个「事件」。「事件」通过池来维护，简单的在每个事件变更的地方调用textview.setText(CharSequence text)的方式显然不合适，于是考虑用观察者模式实现，JDK和Android源码中都有了观察者模式的底层实现。这里我们从头实现。

<!-- more -->

观察者模式提及的两个核心概念是主题「Subject」和观察者「Observer」,如同订阅报纸，主题「出版者」的任务是发行报纸，观察者「订阅者」需要的是报纸有更新时第一时间收到新的报纸，当观察者「订阅者」退订报纸后，就不再接收到新的报纸。主题「出版者」只有一个，而观察者「订阅者」可有多个。

### 具体实现

观察者接口

    public interface Observer {
        /**
         * 比赛状态改变时,主题会把当前进程状态值当作方法参数,传送给观察者
         */
        void update(Event event);
	}

被观察者接口，这里我们用Subject命名

    public interface Subject {
        /**
         * 注册观察者
         */
        void registerObserver(Observer o);
        /**
         * 移除观察者
         */
        void removeObserver(Observer o);
        /**
         * 主题状态改变时,通知所有观察者
         */
        void notifyObserver();
    }

显示元素接口

    public interface IDisplayElement {

        /**
         * 需要展示时,调用此方法
         */
    	void display();
	}

具体被观察者类，其中用ArrayList存放观察者，event即具体的事件对象，每一次产生事件，通过set方法传进来

    public class EventDataImpl implements Subject {
        //存放观察者对象
        private ArrayList observers;
        private Event event;

        public EventData() {
            observers = new ArrayList();
        }

       	@Override
        public void registerObserver(Observer o) {
            if (o == null) {
                throw new IllegalArgumentException("The observer is null.");
            }
            //需要注册时添加
            observers.add(o);
        }

        @Override
    	public void removeObserver(Observer o) {
        	if (o == null) {
            	throw new IllegalArgumentException("The observer is null.");
        	}
        	//取消注册时,删除
        	int i = observers.indexOf(o);
        	if (i > 0) {
            	observers.remove(i);
        	}
    	}

    	@Override
    	public void notifyObserver() {
        	//告诉每一个观察者
        	int size = observers.size();
        	for (int i = 0; i < size; i++) {
            	Observer o = (Observer) observers.get(i);
            	o.update(event);
        	}
    	}

    	/**
     	* 从源处得到更新的比赛进程时,通知观察者
     	*/
    	public void eventChanged() {
        	notifyObserver();
    	}

    	public void setEvent(Event event) {
        	this.event = event;
        	eventChanged();
    	}
	}

接下来就是事件显示面板了

	public class CurrentEventDisplayImpl implements Observer, IDisplayElement {
    	private Event event;
    	private String title;
    	private TextView tv;

    	public CurrentEventDisplay(Context context, Subject eventData, TextView tv) {
        	this.context = context;
        	this.tv = tv;
        	eventData.registerObserver(this);
    }

    	@Override
    	public void display() {
        	if (event != null) {
        	    title = EventHelper.getEventDes(event);
        	} else {
           		title = "";
        	}
        	tv.setText(title);
    	}

    	@Override
    	public void update(Event event) {
        	this.event = event;
        	display();
    	}
	}

在需要有频繁更新textview的地方，完成CurrentEventDisplayImpl的初始化即可
