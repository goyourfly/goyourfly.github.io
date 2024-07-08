---
title: CircleImageView - 工作原理
published: 2017-03-30
description: 'CircleImageView - 工作原理'
image: ''
tags: [Android,Widgets]
category: 'Android'
draft: false 
---

[项目地址：CircleImageView](https://github.com/hdodenhof/CircleImageView)

在分析CircleImageView源码之前，先学习一些知识点
#### 知识点1：BitmapShader

BitmapShader 继承自Shader，而Shader有5个子类：

- BitmapShader
- ComposeShader
- LinearGradient
- RadialGradient
- SweepGradient

首先来认识一下Shader是什么？

直接翻译过来叫着色器，没了解过的人看了会感觉很蒙，就跟把Socket翻译成套接字似的，现实生活在跟本没有这玩意啊，怎么理解呢？

经过我细致的揣摩，我认为Shader就像是一把刷子或者说是印章，它决定写出来的东西是什么颜色的，或者画出到图案是什么样子到。

比如说我想要制作一台车，在所有都组装完毕之后，加了油就可以开了，但是看起来怎么这么不狂拽炫吊炸天呢?   哦对了，忘了给车喷漆了，这时候我们就拿几喷漆罐对着车喷一些文字或图案，那喷图案的过程就是Shader的过程．

我们上面提到有5种Shader：

- BitmapShader  ------  图案喷漆
- ComposeShader  ------  将其他喷漆合并
- LinearGradient  ------  线性渐变色的喷漆
- RadialGradient  ------  环形渐变色的喷漆
- SweepGradient  ------  扫描喷漆（类似于从某一个点射出很多不同颜色的射线）

那么我们重点关注一下BitmapShader，它是一个可以将图案绘制出来的喷漆，比如说我要把小狗的图像作为一个喷漆，那么我拿着这个喷漆对着车喷一下，车马上就会有一只可爱的小狗的图画．

#### 知识点2：ColorFilter

这个通过直译就比较好理解，颜色过滤器，就像是一个筛子，比如说我有３种类型的物体：三角形，矩形，圆形，让他们通过管道从一个地方传递到另外一个地方，如果管道是完全通透的，那么，这些物体从起始点到终点都是一样的，那么如果我中间加一个三角形过滤器，只有三角形能通过，那么剩下矩形和圆形就过不去了，颜色过滤器类似，但是它能设置量，既通过的多少．

ColorFilter在Android中是一个５* 4的矩阵：

````Java 
        {
            1, 0, 0, 0, 0    // 颜色种的红色Ｒ
            0, 1, 0, 0, 0    // 颜色种的绿色G
            0, 0, 1, 0, 0    // 颜色种的蓝色B
            0, 0, 0, 1, 0    // 颜色种的透明度Ａ
        }
````

如上所示，其中1的位置就是每个颜色的过滤器，范围0 ~ 2，其中1表示正常颜色，0表示完全隔断，2表示FF，所以我们如果要调整一张图片的颜色，只要调整ColorFilter种的值就可以做到了．


#### 知识点3：Matrix

> 这里讨论的是Android中`android.graphics.Matrix`这个类

Android种的Martix是用来做图像调整的，如平移，缩放，旋转，还有错切，它是一个3 * 3的矩阵．

````Java
{
缩放X, 错切X, 平移X,
错切Y, 缩放Y,  平移Y,
透视1, 透视2,  透视3,
}
````

其中透视基本不用，咱主要用到的是平移和缩放。

----
#### 正式开始

````Java
public class CircleImageView extends ImageView {
	//首先强制设置缩放类型为CENTER_CROP
    private static final ScaleType SCALE_TYPE = ScaleType.CENTER_CROP;
	
	//其次，Bitmap类型为ARGB_8888
    private static final Bitmap.Config BITMAP_CONFIG = Bitmap.Config.ARGB_8888;
    private static final int COLORDRAWABLE_DIMENSION = 2;

    private static final int DEFAULT_BORDER_WIDTH = 0;
    private static final int DEFAULT_BORDER_COLOR = Color.BLACK;
    private static final int DEFAULT_FILL_COLOR = Color.TRANSPARENT;
    private static final boolean DEFAULT_BORDER_OVERLAY = false;
````

然后在UI在创建时会执行init();

````Java
    private void init() {
    	//设置ScaleType
        super.setScaleType(SCALE_TYPE);
        mReady = true;

        if (mSetupPending) {
        	//重点看一下setup
            setup();
            mSetupPending = false;
        }
    }
````
我们重点关注一下setup()

````Java
    private void setup() {
        ...

		//初始化一个BitmapShader，就是上面讲的会喷图的喷漆
        mBitmapShader = new BitmapShader(mBitmap, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP);
		//将喷图赋予mBitmapPaint这个画笔，感觉上面的比喻有点不对，BitmapShader应该只是一个图案，喷漆应该是Paint
        mBitmapPaint.setAntiAlias(true);
        mBitmapPaint.setShader(mBitmapShader);

		//圆环形的边框用的画笔
        mBorderPaint.setStyle(Paint.Style.STROKE);
        mBorderPaint.setAntiAlias(true);
        mBorderPaint.setColor(mBorderColor);
        mBorderPaint.setStrokeWidth(mBorderWidth);

		//填充用到的画笔
        mFillPaint.setStyle(Paint.Style.FILL);
        mFillPaint.setAntiAlias(true);
        mFillPaint.setColor(mFillColor);

		//取到bitmap的宽高信息
        mBitmapHeight = mBitmap.getHeight();
        mBitmapWidth = mBitmap.getWidth();

		//计算圆形头像的范围，如果宽 > 高，则用高作为正方形边长
        mBorderRect.set(calculateBounds());
        //半径为宽或者高的一半
        mBorderRadius = Math.min((mBorderRect.height() - mBorderWidth) / 2.0f, (mBorderRect.width() - mBorderWidth) / 2.0f);
		//DrawableRect为里面图案的范围，而BorderRect则是边框的范围
        mDrawableRect.set(mBorderRect);
        if (!mBorderOverlay && mBorderWidth > 0) {
        	//缩小mBorderWidth - 1 个像素
            mDrawableRect.inset(mBorderWidth - 1.0f, mBorderWidth - 1.0f);
        }
        mDrawableRadius = Math.min(mDrawableRect.height() / 2.0f, mDrawableRect.width() / 2.0f);

		//设置颜色过滤器
        applyColorFilter();
        //更新Matrix
        updateShaderMatrix();
        //刷新UI
        invalidate();
    }
````

接下来再看看updateShaderMatrix

````Java
    private void updateShaderMatrix() {
        float scale;
        float dx = 0;
        float dy = 0;
		//reset the matrix
        mShaderMatrix.set(null);
		//计算从原始的bitmap到UI需要显示的bitmap，需要的缩放和位移，其中，如果原图宽度大于高度，则只位移X，否则位移Y
        if (mBitmapWidth * mDrawableRect.height() > mDrawableRect.width() * mBitmapHeight) {
            scale = mDrawableRect.height() / (float) mBitmapHeight;
            dx = (mDrawableRect.width() - mBitmapWidth * scale) * 0.5f;
        } else {
            scale = mDrawableRect.width() / (float) mBitmapWidth;
            dy = (mDrawableRect.height() - mBitmapHeight * scale) * 0.5f;
        }

		//传入缩放设置
        mShaderMatrix.setScale(scale, scale);
        //传入位移设置
        mShaderMatrix.postTranslate((int) (dx + 0.5f) + mDrawableRect.left, (int) (dy + 0.5f) + mDrawableRect.top);
		//将Martix设置给BitmapShader
        mBitmapShader.setLocalMatrix(mShaderMatrix);
    }
````

所以，通过查看源码我们就能了解到，CircleImageView的工作原理是将ImageView的Bitmap捕获到，然后将这个Bitmap设置给BitmapShader，其中利用Matrix调整Bitmap在BitmapShader中的大小和位置居中，然后把BitmapShader利用Paint.setShader赋值给画笔Paint，最后再利用这个画笔绘制一个圆形即可画出我们设置的图片，而且是圆形哦。