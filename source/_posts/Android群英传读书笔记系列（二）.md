---
title: Android群英传读书笔记系列（二）
comments: true
toc: true
categories: Android
tags:
  - 读书
  - 笔记
abbrlink: cddb9e34
date: 2017-01-18 15:29:23
---

Android群英传读书笔记记录，备忘，关于一些知识点以及重要的地方。


## 1. Android Scroll分析

关键词：绝对坐标与视图坐标的区别

### 1.1. Android坐标系

在Android中，是以屏幕的左上角作为坐标系的原点，向右是x轴，向下是Y轴。

系统提供了getLocationOnScreen(int locaiton[]) 获取坐标系位置，另外触控事件中getRawX()  ,getRawY()获取的同样是Android坐标系中的坐标。

### 1.2 视图坐标系

关于方向与1.1坐标系是一致的，只不过视图坐标系中，一个view的原点是以父视图坐上角为坐标原点。触控事件中getX()  getY() 得到的就是视图坐标系中的坐标。

### 1.3 触控事件——MotionEvent

``` Java
	//单点触摸按下动作
	public static final int ACTION_DOWN=0;
	//单点触摸离开动作
	public static final int ACTION_UP=1;
	//触摸点移动动作
	public static final int ACTION_MOVE=2;
	//触摸点取消动作
	public static final int ACTION_CACEL=3;
	//触摸动作超出边界
	public static final int ACTION_OUTSIDE=4;
	//多点触摸按下动作
	public static final int ACTION_POINTER_DOWN=5;
	//多点离开动作
	public static final int ACTION_POINTER_UP=6;
```

### 1.4 实现滑动的七种方法

#### 1.4.1 layout()方法

onTouchEvent()计算滑动距离，不断layout()，让view跟随手指滑动。

#### 1.4.2 offsetLeftAndrRight()和offsetTopAndrBottom()

系统提供的对上下、左右移动的封装，参数为偏移量

``` Java
	//同时对left和right进行偏移
	offsetLeftAndrRight(offsetX)
	//同时对top和bottom进行偏移
	offsetTopAndrBotton(offsetY)
```

#### 1.4.3 LayoutParams

LayoutParams保存了一个view的参数，通过getLayoutParams()获取，然后进行修改

``` Java
	LinearLayout.LayoutParams layoutParams=(LinearLayout.LayoutParams)getLayoutParams();
	layoutParams.leftMargin=getLeft()+offsetX;
	layoutParams.topMargin=getTop()+offsetY;
	setLayoutParams(layoutParams);
```

或者

``` Java
	ViewGroup.MarginLayoutParms layoutParams=(ViewGroup.MarginLayoutParams)getLayoutParms();
	layoutParms.leftMargin=getLeft()+offsetX();
	layoutParms.topMargin=getTop()+offsetY();
	setLayoutParams(layoutParms);
```

#### 1.4.4 scrollTo与scrollBy

scrollTo移动到某个位置，scrollBy移动某个增量。

关键点：viewgroup中 移动的是子view,view中，移动的是view中的内容。

移动方向问题，反向移动，可以理解为：view未动，移动的是屏幕

#### 1.4.5 Scroller

实现平滑滚动

三个步骤：

- 初始化Scroller

``` Java
	//初始化Scroller
	mScroller=new Scroller(context);
```

- 重写computeScroll()方法，实现模拟滑动

``` Java
	@Override
	public void computeScroll(){
		super.computeScroll();
		//判断Scroller是否执行完毕
		if(mScroller.computeScrollOffset()){
			((View)getParent()).scrollTo(mScroller.getCurrX(),mScroller.getCurrY());
			//通过重绘不断调用computeScroll
			invalidate();
		}
	}
```

- startScroll 开启模拟过程

#### 1.4.6 属性动画

#### 1.4.7 ViewDragHelper


- 初始化ViewDragHelper

``` Java
	mViewDragHelper=ViewDragHelper.create(this,callback);
```

- 拦截事件

``` Java
	@Override
	public boolean onInterceptTouchEvent(MotionEvent ev)	{
		return mViewDragHelper.shouldInterceptTouchEvent(ev);
	}
	
	@Override
	public boolean onTouchEvent(MotionEvent event){
		//将触摸事件传递给ViewDragHelper,此操作必不可少
		mViewDragHelper.processTouchEvent(event);
	}
```

- 处理computeScroll()

模板写法

``` Java
	@Override
	public void computeScroll(){
		if(mViewDragHelper.continueSetting(true)){
			ViewCompat.postInvalidateOnAnimation(this);
		}
	}
```

- 处理回调Callback

``` Java
	private ViewDragHelper.Callback callback=new ViewDragHelper.Callback(){
	@Overrride
	public boolean tryCaptureView(View child,int pointerId){
		return false;//根据具体需求修改
	}
	}
```

滑动方法

clampViewPositionVertical()

clampViewPositionHorizontal()


## 2.Android 绘图机制与处理技巧

### 2.1屏幕的尺寸信息

#### 2.1.1 屏幕参数

- 屏幕大小

对角线长度

- 分辨率

手机屏幕像素点个数

- PPI 

Android使用mdpi(密度值为160)的屏幕作为标准，在这个屏幕上1px=1dp.其他屏幕按比例换算。

mdpi 1dp=1px

hdpi 1dp=1.5px

xhdpi 1dp=2px

xxhdpi 1dp=3px

### 2.2 2D绘图基础

paint属性

- setAntiAlias()  //设置画笔的锯齿效果
- setColor()   //设置画笔的颜色
- setARGB()  //设置画笔的ARGB值
- setAlpha()  //设置画笔的Alpha值
- setTextSize()  //设置字体的尺寸
- setStyle()   //设置画笔的风格
- setStrokeWidth()  //设置空心边框的宽度

#### 2.2.1 canvas 绘图API

- DrawPonit 绘制点

canvas.drawPoint(x,y,paint)

- DrawLine 绘制直线

canvas.drawLine(startX,startY,endX,endY,paint)

- DrawLines 绘制多条直线

``` Java
	floast[] pts={startX1,startY1,endX1,endY1,...}
	canvas.drawLines(pts,ponit)
```

- DrawRect 绘制矩形

canvas.drawRect(left,top,right,bottom,paint)

- DrawRoundRect 绘制圆角矩形

canvas.drawRoundRect(left,top,bottom,raidusX,radiuxY,paint)

- DrawCircle 绘制圆

canvas.drawCircle(circleX,circleY,radius,paint)

- DrawArc 绘制弧形，扇形

``` Java
	paint.setStyle(Paint.Style.STROKE);
	canvas.drawArc(left,top,right,bottom,startAngle,sweepAngle,userCenter,paint)
	
```

- DarwOval 绘制椭圆

canvas.drawOval(left,top,right,bottom,paint)

- DrawText 绘制文本

canvas.drawText(text,startX,startY,paint)

- DrawPosText 在指定位置绘制文本

canvas.drawPosText(text,new float[]{X1,Y1,X2,Y2,...Xn,Yn})

- DrawPath 绘制路径

``` Java
	Path path=new Path();
	path.moveTo(50,50);
	path.lineTo(100,100);
	path.lineTo(100,300);
	path.lineTo(300,50);
	canvas.drawPath(path,paint);
```

### 2.3 Android Xml绘图

#### 2.3.1 bitmap 

src引用

#### 2.3.2 Shape

注意：shape支持的参数

#### 2.3.3 Layer

图层示例：

``` xml

	<?xml version="1.0" encoding="utf-8">
	<layer-list xmlns:android="http://schemas.android.com/apk/res/android">
		<item android:drawable="@drawable/ic_launcher"/>
		<item android:drawable="@drawable/ic_launcher"
				android:left="10.0dip"
				android:top="10.0dip"
				android:right="10.0dip"
				android:bottom="10.0dip"/>
	</layer-list>

```

#### 2.3.4 selector

主要实现不同效果

selector中可以组合使用shape

### 2.4 Android 绘图技巧

#### 2.4.1 canvas

Canvas，几个非常有用的方法

- Canvas.save()
- Canvas.restore()
- Canvas.translate()
- Canvas.rotate()

save 保存画布，让后续操作就像在一个新的图层上操作一样

restore  合并图层，save之后的所有图像与save之前的图像合并

translate  坐标系的平移

rotate  也可以理解为坐标系的翻转


可以利用rotate画布的旋转和translate坐标系的平移实现一些复杂的效果

#### 2.4.2 Layer图层

通过saveLayer()方法  saveLayerAlpha()方法将一个图层入栈，使用restore()方法  restoreToCount()方法将一个图层出栈。

出栈的时候，会把图像绘制到上层Canvas上。

### 2.5 Android 图像处理之色彩特效处理

#### 2.5.1 色彩矩阵分析

- 色调--物体传播的颜色
- 饱和度--颜色的纯度  从0到100%
- 亮度--颜色的相对明暗程度

颜色矩阵  ColorMatrix

这块比较复杂，专门写一篇文章，这里跳过

### 2.6 Android图像处理之画笔特效处理

#### 2.6.1 PorterDuffXfermode

控制两个图像的混合显示模式

dst 是先画的图形   src是后画的图形

最常用的是通过DST_IN  SRC_IN实现将一个矩形图片变成圆角图片或者圆形图片的效果

``` Java
	mBitmap=BitmapFactory.decodeResouce(getResources(),R.drawable.test1);
	mOut=Bitmap.createBitamp(mBitmap.getWidth(),mBitmap.getHeight(),Bitmap.Config.ARGB_8888);
	Canvas canvas=new Canvas(mOut);
	mPaint =new Paint();
	mPaint.setAntiAlias(true);
canvas.drawRoundRect(0,0,mBitmap.getWidth(),mBitmap.getHeight(),80,80,mPaint);
mPaint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC_IN));

canvas.drawBitmap(mBitmap,0,0,mPaint);
```

### 2.7 SurfaceView

#### 2.7.1 surfaceView与view

- view主要用于主动更新的情况下，surfaceView主要适用于被动更新，例如频繁的刷新
- view在主线程中对画面进行刷新，surfaceView通过一个子线程进行页面的刷新
- view在绘图时没有使用双缓冲机制，surfaceView在底层实现机制中就已经实现了双缓冲机制

#### 2.7.2 创建一个surfaceview的模板

- 创建SurfaceView

创建自定义的SurfaceView继承自SurfaceView,并实现两个接口-SurfaceHolder.Callback和Runnable

``` Java
	public class SurfaceViewTemplate extends SurfaceView implements SurfaceHolder.Callback,Runnable
```

- 初始化SurfaceView

在自定义surfaceview中，通常需要定义三个成员变量

``` Java
	//SurfaceHolder
	private SurfaceHolder mHolder;
	//用于绘图的Canvas
	private Canvas mCanvas;
	//子线程标志位
	private boolean mIsDrawing;
```

初始化方法就是对surfaceholder进行初始化

``` Java
	mHolder=getHolder();
	mHolder.addCallback(this);
```

- 使用surfaceview

在surfaceCreated()方法中开启子线程进行绘制，在子线程使用一个while（mIsDrawing）的循环不停进行绘制，通过lockCanvas()方法获得的Canvas对象进行绘制，通过unlockCanvasAndPost(mCanvas)方法对画布内容进行提交。


模板代码

``` Java
	public class SurfaceViewTemplate extends SurfaceView implements SurfaceHolder.Callback,Runnable{
	private SurfaceHolder mHolder;
	private Canvas mCanvas;
	private boolean mIsDrawing;
	
	public SurfaceViewTemplate(Context context){
		super(context);
		initView();
	}
	public SurfaceViewTemplate(Context context,AttributeSet attrs){
		super(context,attrs);
		initView();
	}
	public SurfaceViewTemplate(Context context,AttributeSet attrs,int defStyle){
		super(context,attrs,defStyle);
		initView();
	}
	private void initView(){
		mHolder=getHolder();
		mHolder.setCallback(this);
		setFocusable(true);
		setFocusableInTouchMode(true);
		this.setKeepScreenOn(true);
	}
	
	@Override
	public void surfaceCreate(SurfaceHolder holder){
		mIdDrawing=true;
		new Thread(this).start();
	}
	
	@Override
	public void surfaceChanged(SurfaceHolder holder,int format, int width,int height){
	}
	
	@Override
	public void surfaceDestroyed(SurfaceHolder holder){
		mIdDrawing=false;
	}
	
	@Override
	public void run(){
		while(mIdDrawing){
			draw();
		}
	}
	
	private void draw(){
		try{
			mCanvas=mHolder.lockCanvas();
			//draw something
		} catch(Exception e){
		} finally{
			if(mCanvas!=null){
				mHolder.unlockCanvasAndPost(mCanvas);
			}
		}
	}
	
	
	}
```



