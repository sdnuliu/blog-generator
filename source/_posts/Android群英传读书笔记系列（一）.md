---
title: Android群英传读书笔记系列（一）
comments: true
toc: true
categories: Android
tags:
  - 读书
  - 笔记
abbrlink: fdcd72ec
date: 2017-01-15 09:33:41
---

Android群英传读书笔记记录，备忘，关于一些知识点以及重要的地方。



## 1.Android体系与系统架构

### 1.1
Android体系大致分为四层：Linux内核层、库和运行时、Framework层和应用层。  
Android的体系架构鼓励系统组件重用，共享组件间的数据，并且定义组件间的访问权限控制。可以说这些层次结构既是相互独立，又是相互关联的。

#### 1.1.1 Linux层
Linux层包含系统的核心服务，包括硬件驱动，进程管理，系统安全等。

#### 1.1.2 Dalvik和ART
5.0之前 每个APP都会分配Dalvik，运行时编译；  
5.0之后，ART取代了Dalvik,安装时编译，运行时就不用编译了。

### 1.2组件架构
关键词：协同工作，信使Intent

#### 1.2.1 应用运行上下文对象

创建Context的时间点： 
 
- 创建Application  
- 创建Activity  
- 创建service

### 1.3 Android系统源代码目录与系统目录

#### 1.3.1 系统源代码目录

可查看源码的网站：<http://www.androidxref.com>

#### 1.3.2 Android系统目录

- /system/app  系统app
- /system/bin  Linux自带组件
- /system/build.prop 系统属性信息
- /system/fonts/  系统字体
- /system/framework  系统核心文件、框架层
- /system/lib/  共享库（.so）
- /system/media/  系统提示音，系统铃声
- /system/usr/  用户的配置文件
- /data/app  用户的大部分数据信息
- /data/data app数据信息，报名区分
- /data/system 手机的各项系统信息
- /data/misc  wifi、VPN信息


## 2.Android开发工具

关键词：Android studio  
镜像网站：<http://www.androiddevtools.cn/>
### 2.1 导入项目中的问题
导入项目的gradle版本与本地不一致时，会去下载新的gradle版本，导致界面卡住。  
解决办法：先用本地gradle创建一个新的项目，然后将项目中的gradle文件夹和build.gradle文件替换要导入项目中的对应文件夹和文件，重新编译。

### 2.2 adb常用命令

- adb install  安装应用
- adb push  推送文件 需指定目录
- adb pull  从手机获取文件
- 命令来源：\system\core\toolbox   和  \frameworks\base\cmds


## 3.Android控件架构和自定义控件

viewGroup和view 两大类  

### 3.1 Android界面架构图

#### 3.1.1 标准视图树的建立过程

每个activity都包含一个window对象，这个对象通常是由phonewindow来实现的。phonewindow将decorview作为整个应用窗口的根view。DecorView作为窗口界面的顶层视图，封装了一些窗口操作的通常方法。  
在显示上，它将窗口分为两部分，一个是TitleView,一个是ContentView,ContentView是一个id为content的framelayout ,activity_main.xml就是设置在这个framelayout里。

### 3.2 View的测量

Android通过提供MeasureSpec类，帮助我们测量view.MeasureSpec是一个32位的int值，高2位为测量的模式，低30位为测量的大小。

#### 3.2.1 测量模式

- EXACTLY  
  ---精确值模式，指定大小
- AT_MOST  
  ---最大值模式，wrap\_content
- UNSPECIFIED  
  ---不指定模式，想多大多大，一般用在自定义view
  
View默认onMeasure只支持EXACTLY模式，自定义view可以通过重写onMeasure()方法进行修改。
  
源码可以发现系统最终调用setMeasuredDimension()方法将测量后的宽高设置进去，所以在重写onMeasure方法时，需要将测量后的宽高值作为参数设置给setMeasureDimension()方法。  

``` Java  
	protected void onMeasure(int widthMeasureSpec,int heightMeasureSpec){
		setMeasuredDimension(measureWidth(widthMeasureSpec),measureHeight(heightMeasureSpec));
	}
```

关于measureWidth方法

``` Java
	private int measureWidth(int measureSpec){
		int result=0;
		int specMode=MeasureSpec.getMode(measureSpec);
		int specSize=MeasureSpec.getSize(measureSpec);
		if(specMode==MeasureSpec.EXACTLY){
			result=specSize;
		}else{
			result=200;
			if(specMode==MeasureSpec.AT_MOST){
				result=Math.min(result,specSize);
			}
		}
		return result;
	}
```

### 3.3 View的绘制

关键词：onDraw()

对Canvas的理解：new Canvas时一般会传入一个bitmap对象，view的绘制是在bitmap上进行的绘制。

### 3.4 自定义View

View中比较重要的一些回调方法：

- onFinishInflate()  xml加载组件后回调
- onSizeChanged()  组件大小改变时回调
- onMeasure()  回调该方法用来测量
- onLayout()  回调该方法确定显示位置
- onTouchEvent()  监听到触摸事件时回调

#### 3.4.1 自定义控件三种方式

- 对现有控件进行扩展
- 通过组合实现新的控件
- 重写view实现全新的控件

#### 3.4.2 为view添加自定义属性

在res资源目录的values目录下创建attrs.xml的属性定义文件（多属性的时候，在format里通过"|"区别，例:reference|color）

1.typeArray获取完值之后，要调用recycle()方法避免重新创建时的错误。

#### 3.4.3 引用ui模板

引入自定义的命名空间
xmlns:custom="http://schemes.android.com/apk/res_auto"

custom为命名空间的名字，可自定义。

#### 3.4.4 绘制圆弧

【基本语法】

``` Java
	public void drawArc (RectF oval, float startAngle, float sweepAngle, boolean useCenter, Paint paint)
```

参数说明

oval：圆弧所在的椭圆对象。

startAngle：圆弧的起始角度。

sweepAngle：圆弧的角度。

useCenter：是否显示半径连线，true表示显示圆弧与圆心的半径连线，false表示不显示。

paint：绘制时所使用的画笔。

#### 3.4.5 自定义ViewGroup

通常重写onMeasure()方法对子view进行测量，重写onLayout()方法确定子view的位置，重写onTouchEvent()方法增加响应事件。	
#### 3.4.6 事件分发

关键三个方法：

- dispatchTouchEvent(MotionEvent ev)
- onInterceptTouchEvent(MotionEvent ev)  true拦截false不拦截
- onTouchEvent(MotionEvent ev)  true处理(不再往上传递)false不处理

## 4.ListView的使用技巧

备注:目前recycleview用的比较多

### 4.1 使用ViewHolder模式提高效率

可以提高50%以上

### 4.2 ListView的分割线

android:divider

android:dividerHeight

android:divider="@null"  可以将分割线设置成透明

### 4.3 隐藏listview的滚动条

android:scrollbars="none"

### 4.4 取消listview的item的点击效果

android:listSelector="#00000000"

或者 android:listSeletor="@android:color/transparent"

### 4.5 设置listview要显示在第几项

listView.setSelection(N); 瞬间移动完成
listView.smoothScrollBy(distance,duration)  
listView.smoothScrollByOffset(offset)  
listView.smoothScrollToPosation(index)  下面三个为平滑滚动

### 4.6 处理空listview

ListView提供了一个方法setEmptyView(),通过这个方法可以给listview设置一个空数据下显示的默认提示

### 4.7 listview的滑动监听

- onTouchListener
- onScrollListener




