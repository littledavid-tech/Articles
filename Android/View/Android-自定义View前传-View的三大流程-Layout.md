## Android-自定义View前传-View的三大流程-Layout

### 参考
* 《Android开发艺术探索》
* https://github.com/hongyangAndroid/FlowLayout

### 写在前头

[在之前的文章中](http://shycoder.cn/index.php/archives/39/) ， 我们学习了Android View的 Measure的流程， 本篇文章来学习一下View的 `Layout` 的过程。 学完了这一篇文章后，我们可以尝试自己去自定义一个自己的Layout。

### Overview

> 我对于Layout过程的理解：Layout的过程就是给Child安家的过程

Layout的过程主要是放在 `ViewGroup` 中的，ViewGroup不仅需要定位自己，还需要定位Child。

### View和ViewGroup

Layout流程的起点也是在 ViewRootImpl 中的 `performTraversals` 方法中。

```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
        int desiredWindowHeight) {
    mLayoutRequested = false;
    mScrollMayChange = true;
    mInLayout = true;
    final View host = mView;
    if (host == null) {
        return;
    }

    try {
        //首先调用了host的layout方法 host = mView = DecorView
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
        //....
    }
```

我们接着来看View的layout方法

```java
public void layout(int l, int t, int r, int b) {
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
        onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }

    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;
	//调用 setOpticalFrame或者 setFrame 来确定自己的位置
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {、
		//调用onLayout
        onLayout(changed, l, t, r, b);
        //.....
}



protected boolean setFrame(int left, int top, int right, int bottom) {
    boolean changed = false;

    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
        changed = true;

        // Remember our drawn bit
        int drawn = mPrivateFlags & PFLAG_DRAWN;

        int oldWidth = mRight - mLeft;
        int oldHeight = mBottom - mTop;
        int newWidth = right - left;
        int newHeight = bottom - top;
        boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

        // Invalidate our old position
        invalidate(sizeChanged);
		//确定4个点
		//这四个点一旦确定了那么View在ViewGroup中的位置也就确定了
        mLeft = left;
        mTop = top;
        mRight = right;
        mBottom = bottom;
		
        mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);
        //......
```
View的 `layout` 方法主要是做了:
1. 通过 `setFrame` 确定了自己的位置，一篇Left,Top,Right,Bottom这几个值确定了,那么View的位置也就确定了。
2. 紧接着调用了`onLayout` 方法。

### LinearLayout的onLayout

View是不需要实现onLayout方法的，只用ViewGroup才需要实现。由于各种ViewGroup的布局方式的不同，无法统一，所以ViewGroup也并没有实现`onLayout`. 而是将onLayout的过程放到了子类中。我们还是通过 `LinearLayout` 来学习。

```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    if (mOrientation == VERTICAL) {
        layoutVertical(l, t, r, b);
    } else {
        layoutHorizontal(l, t, r, b);
    }
}
```
onLayout根据 Orientation 属性来调用 `layoutVertical` 或者 `layoutHorizontal` 

```java
void layoutVertical(int left, int top, int right, int bottom) {
    final int paddingLeft = mPaddingLeft;

    int childTop;
    int childLeft;

    // Where right end of child should go
    final int width = right - left;
    int childRight = width - mPaddingRight;

    // Space available for child
	//获取Child可用的空间
    int childSpace = width - paddingLeft - mPaddingRight;
	//Child的Group
    final int count = getVirtualChildCount();

    final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
    final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;
	//根据Gravity确定初始的childTop
    switch (majorGravity) {
       case Gravity.BOTTOM:
           // mTotalLength contains the padding already
           childTop = mPaddingTop + bottom - top - mTotalLength;
           break;

           // mTotalLength contains the padding already
       case Gravity.CENTER_VERTICAL:
           childTop = mPaddingTop + (bottom - top - mTotalLength) / 2;
           break;

       case Gravity.TOP:
       default:
           childTop = mPaddingTop;
           break;
    }
	//LayoutView
	//遍历
    for (int i = 0; i < count; i++) {
        final View child = getVirtualChildAt(i);
        if (child == null) {
            childTop += measureNullChild(i);
        } else if (child.getVisibility() != GONE) {//过滤Child的Visibility是GONE的情况
			//获取Child的大小
            final int childWidth = child.getMeasuredWidth();
            final int childHeight = child.getMeasuredHeight();
			//获取LP
            final LinearLayout.LayoutParams lp =
                    (LinearLayout.LayoutParams) child.getLayoutParams();

            int gravity = lp.gravity;
            if (gravity < 0) {
                gravity = minorGravity;
            }
            final int layoutDirection = getLayoutDirection();
            final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
			//处理Child的Gravity
            switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
				//水平居中
                case Gravity.CENTER_HORIZONTAL:
                    childLeft = paddingLeft + ((childSpace - childWidth) / 2)
                            + lp.leftMargin - lp.rightMargin;
                    break;
				//居右
                case Gravity.RIGHT:
                    childLeft = childRight - childWidth - lp.rightMargin;
                    break;

                case Gravity.LEFT:
                default:
                    childLeft = paddingLeft + lp.leftMargin;
                    break;
            }
			//childTop 加上 Divider
            if (hasDividerBeforeChildAt(i)) {
                childTop += mDividerHeight;
            }
			//加上Margin
            childTop += lp.topMargin;
			
			//调用Child的layout方法
            setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                    childWidth, childHeight);
            childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

            i += getChildrenSkipCount(child, i);
        }
    }
}

private void setChildFrame(View child, int left, int top, int width, int height) {
    child.layout(left, top, left + width, top + height);
}
```
上面的代码还是比较清晰的，首先是有一个childTop变量，来确定child与ViewGroup顶部的距离，通过不断的遍历Child然后不断增加childTop的值，这样就实现了LinearLayout的垂直布局的效果。当然其中也有处理LinearLayout和Child的Gravity的过程。

### FlowLayout

通过学习LinearLayout的Layout的过程，发现其实Layout的过程就是确定View的Left,Top,Right,Bottom 4个值的过程，学习了 `Measure` 和 `Layout` 的过程以后，我们就已经可以着手做一个自己的Layout了，这里我选的是模仿 hongyang大神的[FlowLayout](https://github.com/hongyangAndroid/FlowLayout)。效果图如下:

![Snag_f7f609ee](http://cdn.shycoder.cn/Snag_f7f609ee.png)

```kotlin
package layout

import android.annotation.SuppressLint
import android.content.Context
import android.util.AttributeSet
import android.view.View
import android.view.ViewGroup

/**
 * Created by ShyCoder on 2019/1/16.
 */
class MyFlowLayout(context: Context?, attrs: AttributeSet?, defStyleAttr: Int)
    : ViewGroup(context, attrs, defStyleAttr) {
    constructor(context: Context?, attributeSet: AttributeSet?) : this(context, attributeSet, 0)

    constructor(context: Context?) : this(context, null)

    init {

    }

    /**
     * 存贮所有的View根据行来存贮
     * */
    private val mAllViews = mutableListOf<List<View>>()

    /**
     * 存贮每一行View的高度
     * */
    private val mHeightList = mutableListOf<Int>()


    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        //从MeasureSpec获取Mode和Size
        val widthMode = MeasureSpec.getMode(widthMeasureSpec)
        val widthSize = MeasureSpec.getMode(widthMeasureSpec)
        val heightMode = MeasureSpec.getMode(heightMeasureSpec)
        val heightSize = MeasureSpec.getMode(heightMeasureSpec)

        //计算Wrap_content的情况
        var totalHeight = 0//总高度
        var totalWidth = 0//总宽度

        var lineWidth = 0//当前行的宽度
        var lineHeight = 0//当前行的高度

        val viewCount = childCount

        for (i in 0.until(viewCount)) {
            val child = getChildAt(i)
            //如果是GONE状态不用测量
            if (child.visibility == View.GONE) {
                continue
            }
            //测量Child
            measureChild(child, widthMeasureSpec, heightMeasureSpec)
            val lp = child.layoutParams as MarginLayoutParams

            //计算child的width = 测量后宽度+左右的两个Margin
            val childWidth = child.measuredWidth + lp.leftMargin + lp.rightMargin
            //计算child的Height = 测量后高度 + 上下两个Margin
            val childHeight = child.measuredHeight + lp.bottomMargin + lp.topMargin

            //需要换行时候的处理方式
            //如果已有的行宽+当前child的宽度> FlowLayout的宽度(减去左右的Padding)
            if (childWidth + lineWidth > widthSize - this.paddingLeft + this.paddingRight) {
                totalWidth = Math.max(totalWidth, lineWidth)
                lineWidth = childWidth
                totalHeight += lineHeight
                lineHeight = childHeight
            } else {//如果不需要换行增加行宽
                lineWidth += childWidth
                //获取最大高度
                lineHeight = Math.max(lineHeight, childHeight)
            }
            //最后一个View
            if (i == viewCount - 1) {
                totalWidth = Math.max(totalWidth, lineWidth)
                totalHeight += lineHeight
            }
        }
        this.setMeasuredDimension(
                //如果MeasureSpec的Mode是EXACTLY 的话，测量后大小等会传进来的测量大小，
                //否则则是我们自己计算的大小
                if (widthMode == MeasureSpec.EXACTLY) widthSize else totalWidth,
                if (heightMode == MeasureSpec.EXACTLY) heightSize else totalHeight
        )
    }

    override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
        this.mAllViews.clear()
        this.mHeightList.clear()

        val viewCount = this.childCount

        var lineHeight = 0
        var lineWidth = 0

        var lineViews = mutableListOf<View>()

        for (i in 0.until(viewCount)) {
            val child = this.getChildAt(i)
            if (child.visibility == View.GONE) {
                continue
            }
            val lp = child.layoutParams as MarginLayoutParams

            val childWidth = child.measuredWidth + lp.leftMargin + lp.rightMargin
            val childHeight = child.measuredHeight + lp.topMargin + lp.bottomMargin

            //行上无法继续放置View
            if (childWidth + lineWidth > this.width - this.paddingLeft - this.paddingRight) {
                //添加line height
                mHeightList.add(lineHeight)
                lineWidth = 0
                //添加正行的View到集合中
                this.mAllViews.add(lineViews)
                lineViews = mutableListOf()

            }
            //区当前行的View的最大高度
            lineHeight = StrictMath.max(lineHeight, childHeight)
            lineWidth += childWidth
            //向行上添加View
            lineViews.add(child)
        }

        //进行Layout
        var left = this.paddingLeft
        var top = this.paddingTop

        for (i in 0.until(this.mAllViews.size)) {
            val lineViews = mAllViews[i]
            val lineHeight = mHeightList[i]
            left = this.paddingLeft

            for (j in 0.until(lineViews.size)) {
                val child = lineViews[j]
                val lp = child.layoutParams as MarginLayoutParams

                //view的四个边
                val l = left + lp.leftMargin
                val t = top + lp.topMargin
                val r = l + child.measuredWidth
                val b = t + child.measuredHeight

                //调用Child的Layout方法
                child.layout(l, t, r, b)

                left += child.measuredWidth + lp.leftMargin + lp.rightMargin
            }
            top += lineHeight
        }
    }
}
```

在onMeasure对wrap_content的情况进行了处理，计算出所需要的大小。

在onLayout方法中，首先计算出每一行的高度并存储和获取每一行的View的List进行存储，当这些都计算完之后，进行layout操作。

在layout操作的时候，首先是在同一行上的View的top是统一的，每当这一行的View处理完成之后，就执行换行的操作，即增加view的大小。


### 写在最后

本篇文章就到此结束了，Layout的过程还是相对简单的，在下一篇文章呢，我们将会学习View的最后一个流程 `Draw` 流程。