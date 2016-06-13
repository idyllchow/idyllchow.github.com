---
layout: post
title: NavigationView实现抽屉效果及视图定制
description: "NavigationView添加底部视图"
modified: 2016-06-13
tags: [dev]
notes: [dev]
type: dev
---
Android design包提供了NavigationView可配合DrawerLayout方便地实现灵活的抽屉效果，NavigationView免除了自定义视图的繁琐。

![Smithsonian Image]({{ site.url }}/images/patterns_navdrawer_settings1.png)
{: .image-right}

要实现[Material Design](https://material.google.com/patterns/navigation-drawer.html#)中类似这样的效果，只需在NavigationView xml文件中加入app:headerLayout和app:menu即可，headerLayout的定义如同普通的布局文件，menu由对应的item构成，每个item可以添加子item实现分类，item也可以被group包含实现分组。xml文件如下：

act_drawer.xml

    <android.support.v4.widget.DrawerLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/drawer_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:openDrawer="start">

    <FrameLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

    <!-- Navigation Drawer -->
    <android.support.design.widget.NavigationView
        android:id="@+id/nav_view"
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:layout_gravity="start"
        android:fitsSystemWindows="true"
        app:headerLayout="@layout/header_drawer"
        app:menu="@menu/menu_drawer" />

    </android.support.v4.widget.DrawerLayout>    

header_drawer.xml

	<?xml version="1.0" encoding="utf-8"?>
	<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="142dp"
    android:gravity="bottom"
    android:orientation="vertical"
    android:theme="@style/ThemeOverlay.AppCompat.Dark">

    <LinearLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:orientation="vertical">

        <ImageView
            android:id="@+id/img_avatar"
            android:layout_width="60dp"
            android:layout_height="60dp"
            android:layout_gravity="center_horizontal"
            android:background="@mipmap/bg_head_portrait"/>

        <TextView
            android:id="@+id/tv_name"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="bottom|center_horizontal"
            android:textColor="#fffffe"
            android:layout_marginTop="12dp"
            android:textSize="15sp" />

    </LinearLayout>

    <!--header与menu分割线-->
    <View
        android:layout_width="match_parent"
        android:layout_height="0.5dp"
        android:layout_gravity="bottom"
        android:background="#5c5c5c" />
    
	</FrameLayout>
	
menu_drawer.xml

	<?xml version="1.0" encoding="utf-8"?>
	<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <group
        android:id="@+id/group_history"
        android:checkableBehavior="single">
        <item
            android:id="@+id/menu_drawer_history"
            android:icon="@mipmap/ic_drawer_history"
            android:title="@string/history_stats"
            app:actionLayout="@layout/item_drawer_menu_right" />
    </group>

    <group
        android:id="@+id/group_cruyff"
        android:checkableBehavior="single">
        <item
            android:id="@+id/menu_drawer_cruyff"
            android:icon="@mipmap/ic_drawer_cruyff"
            android:title="@string/training"
            app:actionLayout="@layout/item_drawer_menu_right" />
    </group>

    <group
        android:id="@+id/group_setting"
        android:checkableBehavior="single">
        <item
            android:id="@+id/menu_drawer_setting"
            android:icon="@mipmap/ic_drawer_setting"
            android:title="@string/settings" />
    </group>

	</menu>
	
代码中对其操作如下:

	NavigationView navigationView = (NavigationView) findViewById(R.id.nav_view);
        View headerLayout = navigationView.getHeaderView(0);
        ImageView imgHeader = (ImageView) headerLayout.findViewById(R.id.img_avatar);
        TextView tvName = (TextView) headerLayout.findViewById(R.id.tv_name);
        Menu menu = navigationView.getMenu();
        MenuItem menuItem = menu.findItem(R.id.menu_drawer_history);
        View actionView = MenuItemCompat.getActionView(menuItem);
        //其它详细的控件通过类似方法find
        if (navigationView != null) {
            setupDrawerContent(navigationView);
        }
        
    /**
     * navigationView item选中监听
     * @param navigationView
     */
    private void setupDrawerContent(NavigationView navigationView) { 
        navigationView.setNavigationItemSelectedListener(new NavigationView.OnNavigationItemSelectedListener() {
            @Override
            public boolean onNavigationItemSelected(MenuItem item) {
                switch (item.getItemId()) {
                       case R.id.menu_drawer_history: 
                        //处理跳转 
                         break; 
                      case R.id.menu_drawer_cruyff:
                          break; 
                      case R.id.menu_drawer_setting:
                          break; 
                      default:
                        break;
                     }
                 mDrawerLayout.closeDrawers(); 
                return true; 
            }
        });
    }
           	
	     
	     
但有时需要在NavigationView中显示其它信息(比如在底部横向居中出显示app版本号)怎么实现呢，可以用两个(原则上可以更多)NavigationView来实现，也就是抽屉界面顶部为header，下部由两个NavigationView构成，一个NavigationView完成item布局，一个NavigationView实现底部其余视图的布局，在底部的NavigationView中我们可以用headerLayout实现我们想要的视图，xml如下：

	<android.support.design.widget.NavigationView
        android:id="@+id/nav_view_wrapper"
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:layout_gravity="start">

        <android.support.design.widget.NavigationView
            android:id="@+id/nav_view"
            android:layout_width="wrap_content"
            android:layout_height="match_parent"
            android:layout_gravity="start|top"
            android:background="@color/black"
            android:fitsSystemWindows="true"
            app:headerLayout="@layout/header_drawer"
            app:itemIconTint="#fffffe"
            app:itemTextColor="#fffffe"
            app:menu="@menu/menu_drawer" />

        <android.support.design.widget.NavigationView
            android:id="@+id/nav_view_footer"
            android:layout_width="wrap_content"
            android:layout_height="60dp"
            android:layout_gravity="bottom"
            android:background="@color/black"
            android:fitsSystemWindows="true"
            app:headerLayout="@layout/footer_drawer" />

    </android.support.design.widget.NavigationView>
    
    
效果如下：

![Smithsonian Image]({{ site.url }}/images/device-2016-06-13-161319.png)

	
   
     





