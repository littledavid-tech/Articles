## 自定义View前传-View的三大流程-Measure

### 参考
* 《Android开发艺术探索》
* https://developer.android.google.cn/reference/android/view/View.MeasureSpec
### 写在前面

View的 `measure` 、`layout` `draw` 的三大流程的重要性不用多说，只有学习这三大流程，清楚了View的工作方式，才能够在进行 `自定义View`  的时候更得心应手。在学习三大流程的时候，需要看很多的相关的源码，其中有很多困难，同时我是在不管的学习着View的三大流程，在这里想把自己的体会和理解分享给大家，希望能够给大家一些帮助。

### MeasureSpec

从字面上来看，不管是将 Measure Spec 翻译成什么，都是和View的测量时分不开的，在我看来View的 `measure` 的过程就是处理MeasureSpec和LayoutParams的过程。

MeasureSpec中压缩着来自父布局对子View的 `大小(size)` 和 `模式(mode)` 的要求。从表现上来看，可以将MeasureSpec看为一个int值，这个int值中，包含了size和mode，可以通过MeasureSpec类提供的方法，将size和mode从int值中解析出来，或者根据size和mode创建一个新的int值。

```java
/**
    MeasureSpec是View类的一个嵌套类
*/
public static class MeasureSpec {
    
    private static final int MODE_SHIFT = 30;
    private static final int MODE_MASK  = 0x3 << MODE_SHIFT;
    
    public static final int UNSPECIFIED = 0 << MODE_SHIFT;

    public static final int EXACTLY     = 1 << MODE_SHIFT;

    public static final int AT_MOST     = 2 << MODE_SHIFT;
    
    //根据size和mode创建一个MeasureSpec
    public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << 
    MeasureSpec.MODE_SHIFT) - 1) int size,
                                    @MeasureSpecMode int mode) {
        if (sUseBrokenMakeMeasureSpec) {
            return size + mode;
        } else {
            return (size & ~MODE_MASK) | (mode & MODE_MASK);
        }
    }

    //从代表MeasureSpec的int值中将Mode解析出来
    @MeasureSpecMode
    public static int getMode(int measureSpec) {
        //noinspection ResourceType
        return (measureSpec & MODE_MASK);
    }
    //从代表MeasureSpec的int值中将Size解析出来
    public static int getSize(int measureSpec) {
        return (measureSpec & ~MODE_MASK);
    }
}
```

上面的这些代码就是MeasureSpec类的主要的代码，看完上面的代码你可能会产生一些问题:

* 是如何将size和mode压缩成为一个int值的？
* size 和 mode 是如果解析出来的?

这些问题都可以暂时不用考虑，看了上面的代码我们只需对MeasureSpec有一个简洁而清晰的认识就好了

1. MeasureSpec表现为一个int值，这个int值中包含了size和mode
2. MeasureSpec提供了方法来创建MeasureSpec和解析size和mode
3. MeasureSpec有不同的模式

#### MeasureSpec的Mode

MeasureSpec的Mode有三种

* UNSPECIFIED 父节点对字节点没有任何的约束，想多大就多大
* EXACTLY 父节点为字节点指定了确切的大小，不敢这样 子View不能够超过这个边界，对应着 View大小的 `match_parent` 和 指定大小
* AT_MOST 子View想多大就多大，但是不能搞过指定大小，对应着的是 `warp_content`

#### MeasureSpec的创建
MeasureSpec的创建氛围两种情况
1. 非顶级View的MeasureSpec是根据父View的大小和自己的LayoutParams创建的
2. 顶级View的MeasureSpec是根据Window的大小和自己的LayoutPrams创建的

PS: 这里先了解一下就行，到时候看到源码的时候回详细解析

### 从ViewRootImpl开始

在ViewRootImpl中有一个 `performTraversals` 方法， View的三大流程就是从这里开始的。ViewRoomImpl中保存着 `DecorView` ，而我们所看所有的View都是DecorView的子View(这么说可能不太准确，但是可以先这么理解)，WindowManager通过ViewRootImpl来管理View。这里先不用纠结，只需要知道 View的三大流程是在ViewRoomImpl的  `performTraversals` 方法开始的。

下图是 performTraversals 方法内部的主要的执行顺序

![Snag_2812992c](http://cdn.shycoder.cn/Snag_2812992c.png)

本片文章主要聚焦于 `measure` 流程，下面是源码

```java
private void performTraversals() {
//...
    if (!mStopped || mReportNextDraw) {
        boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
        if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                || mHeight != host.getMeasuredHeight() || contentInsetsChanged ||
                updatedConfiguration) {
            
            //获取顶级View的MeasureSpec
            int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
            int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
            if (DEBUG_LAYOUT) Log.v(mTag, "Ooops, something changed!  mWidth="
                    + mWidth + " measuredWidth=" + host.getMeasuredWidth()
                    + " mHeight=" + mHeight
                    + " measuredHeight=" + host.getMeasuredHeight()
                    + " coveredInsetsChanged=" + contentInsetsChanged);
            // Ask host how big it wants to be
            //执行measure流程
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
//...
}
```

上面的代码中我们可以看到是显示获取了 顶级View的 `MeasureSpec` 的，然后执行了performMeasure方法。我们先来看一下顶级View的MeasureSpec是如何创建的。

```java
/**
根据Window的大小和LayoutPrams来创建顶级View的MeasureSpec
*/
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
```

上面的代码很简单，我们这里就不做过得讨论，直接看 `performMeasure` 方法

```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    if (mView == null) {
        return;
    }
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

上面的代码调用了 `mView` 的 measure方法，这里的 `mView` 就是我们说的顶级View，也就是我们之前简单提到的DecorView。接下来我们来看View的 `measure` 方法的源码。

### View 的measure的过程

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    //...
    if (cacheIndex < 0 || sIgnoreMeasureCache) {
    // measure ourselves, this should set the measured dimension flag back
    //调用onMeasure 测量自身
    onMeasure(widthMeasureSpec, heightMeasureSpec);
    mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    } else {
        long value = mMeasureCache.valueAt(cacheIndex);
        // Casting a long to int drops the high 32 bits, no mask needed
        setMeasuredDimensionRaw((int) (value >> 32), (int) value);
        mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }
    //..
}
```

view的measure方法中主要是调用了 onMeasure方法来测量自己，我们来看一下onMeasure方法的源码。

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

onMeasure主要是调用了 getSuggestedMinimumWidth 方法来确定了最小的高度和宽度。

```java
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
```

从上面的代码中可以看到，如果有背景的话，View的最小大小是在背景的大小和View本身的最小的最下大小之间取 `最大值`。

View的onMeasure就这么简单，因为不同的View大小的测量方式并不一样，不同的View会重写onMeasure 方法来来计算自己的大小。

### ViewGroup的Measure流程

ViewGroup跟View不同的是，View仅仅需要测量自己就好了，ViewGroup不仅得测量自己还得通过递归来测量子View。

ViewGroup本身并没有重写 `onMeasure` 方法（因为不同的Layout测量大小的方式都不相同），但是 ViewGroup提供了一系列的测量子View的方法。

```java
/**
遍历所有的Child，并调用 measureChild方法测量Child
*/
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
    final int size = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < size; ++i) {
        final View child = children[i];
        if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
            //
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
        }
    }
}

protected void measureChild(View child, int parentWidthMeasureSpec,
        int parentHeightMeasureSpec) {
    final LayoutParams lp = child.getLayoutParams();
    //获取Child的MeasureSpec
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom, lp.height);
    //调用Child的measure方法
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

`measureChildren` 主要是遍历了所有的子View，然后调用 `measureChild` 来测量子View。在测量子View的时候，先通过 `getChildMeasureSpec` 创建了对应的 MeasureSpec。

```java
/**
参数说明
spec ViewGroup的MeasureSpec
padding 内边距
childDimension 子View的大小
*/
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    //解析出来ViewGroup的size和mode
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);
    int size = Math.max(0, specSize - padding);
    //测量的结果
    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // Parent has imposed an exact size on us
    //处理大小是 martch_parent 或者 具体数值的情况
    case MeasureSpec.EXACTLY:
        //如果子view的大小>=0，即不是march_parent的情况(!=-1)
        if (childDimension >= 0) {
            //大小 是具体数值
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            //如果是 match_parent 的情况
            //大小等于ViewGrope的大小
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            //如果是wrap_content的情况，
            //更改mode
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    // Parent has imposed a maximum size on us
    //ViewGroup的MeasureSpec的Mode是AT_MOST的情况
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            //如果子View有具体的大小
            //更改Mode
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
            //更改模式
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            //如果sUseZeroUnspecifiedMeasureSpec=true则返回0
            //sUseZeroUnspecifiedMeasureSpec 默认为 false
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
    //noinspection ResourceType
    //最终创建 MeasureSpec
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

从上面的代码中，我们可以总结出来下面的表格(下面的表格来自Android开发艺术探索)

![Snag_2c312c0e](http://cdn.shycoder.cn/Snag_2c312c0e.png)

### LinearLayout的 measure的过程

上面了解了 view 和 ViewGroup 的measure的过程，下面我们来结合实例来分析一下，这里我选择的是 `LinearLayout`来分析，选择LinearLayout的原因是，它难度适中，并且我们还可以了解到 LinearLayout的 `权重(weight)` 是怎么计算的。

首先是LinearLayout的 `onMeasure` 方法

```java
//根据orientation属性执行不同的测量方法
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    if (mOrientation == VERTICAL) {
        measureVertical(widthMeasureSpec, heightMeasureSpec);
    } else {
        measureHorizontal(widthMeasureSpec, heightMeasureSpec);
    }
}
```

onMeasure方法根据 `orientation` 属性分别调用了测量垂直布局和水平布局的方法，这里我们以 `measureVertical` 方法来分析。

```java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    
	mTotalLength = 0;
    int maxWidth = 0;
    int childState = 0;
    int alternativeMaxWidth = 0;
    int weightedMaxWidth = 0;
    boolean allFillParent = true;
    float totalWeight = 0;
	
	
	
	//获取子View的数量(包含虚拟View)
	//因为LinearLayout并不包含虚拟View(TableLayout是包含虚拟View的)，所以这里就是全部的View
    final int count = getVirtualChildCount();
	//获取Mode
    final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    final int heightMode = MeasureSpec.getMode(heightMeasureSpec);

    boolean matchWidth = false;
    boolean skippedMeasure = false;

    final int baselineChildIndex = mBaselineAlignedChildIndex;
    final boolean useLargestChild = mUseLargestChild;

    int largestChildHeight = Integer.MIN_VALUE;
    int consumedExcessSpace = 0;

    int nonSkippedChildCount = 0;

    // See how tall everyone is. Also remember max width.
    for (int i = 0; i < count; ++i) {
        final View child = getVirtualChildAt(i);
        if (child == null) {
            mTotalLength += measureNullChild(i);
            continue;
        }
		//忽略Visibility是GONE的View的测量
        if (child.getVisibility() == View.GONE) {
           i += getChildrenSkipCount(child, i);
           continue;
        }

        nonSkippedChildCount++;
		//计算Divider
        if (hasDividerBeforeChildAt(i)) {
            mTotalLength += mDividerHeight;
        }
		//获取Child的LP
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
		//统计Weight(权重)
        totalWeight += lp.weight;

        final boolean useExcessSpace = lp.height == 0 && lp.weight > 0;
		//Mode是EXACTLY的时候的计算方式
        if (heightMode == MeasureSpec.EXACTLY && useExcessSpace) {
            // Optimization: don't bother measuring children who are only
            // laid out using excess space. These views will get measured
            // later if we have space to distribute.
            final int totalLength = mTotalLength;
            mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);
            skippedMeasure = true;
        } else {
            if (useExcessSpace) {
                // The heightMode is either UNSPECIFIED or AT_MOST, and
                // this child is only laid out using excess space. Measure
                // using WRAP_CONTENT so that we can find out the view's
                // optimal height. We'll restore the original height of 0
                // after measurement.
				//如果使用了权重，先给一个值，后面再重新计算
                lp.height = LayoutParams.WRAP_CONTENT;
            }

            // Determine how big this child would like to be. If this or
            // previous children have given a weight, then we allow it to
            // use all available space (and we will shrink things later
            // if needed).
			//决定Child想要多大，如果上一个Child使用了权重，我们允许
			//当前的View使用所有的可用大小
			
            final int usedHeight = totalWeight == 0 ? mTotalLength : 0;
			//测量Child
            measureChildBeforeLayout(child, i, widthMeasureSpec, 0,
                    heightMeasureSpec, usedHeight);

            final int childHeight = child.getMeasuredHeight();

            if (useExcessSpace) {
                // Restore the original height and record how much space
                // we've allocated to excess-only children so that we can
                // match the behavior of EXACTLY measurement.
				//恢复原始的高度并且记录它有多大(将会用于计算Weight的分配)
                lp.height = 0;
                consumedExcessSpace += childHeight;
            }

            final int totalLength = mTotalLength;
			//计算上下的Margin和Offset
            mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
                   lp.bottomMargin + getNextLocationOffset(child));

            if (useLargestChild) {
                largestChildHeight = Math.max(childHeight, largestChildHeight);
            }
        }

        /**
         * If applicable, compute the additional offset to the child's baseline
         * we'll need later when asked {@link #getBaseline}.
         */
        if ((baselineChildIndex >= 0) && (baselineChildIndex == i + 1)) {
           mBaselineChildTop = mTotalLength;
        }

        // if we are trying to use a child index for our baseline, the above
        // book keeping only works if there are no children above it with
        // weight.  fail fast to aid the developer.
        if (i < baselineChildIndex && lp.weight > 0) {
            throw new RuntimeException("A child of LinearLayout with index "
                    + "less than mBaselineAlignedChildIndex has weight > 0, which "
                    + "won't work.  Either remove the weight, or don't set "
                    + "mBaselineAlignedChildIndex.");
        }

        boolean matchWidthLocally = false;
        if (widthMode != MeasureSpec.EXACTLY && lp.width == LayoutParams.MATCH_PARENT) {
            // The width of the linear layout will scale, and at least one
            // child said it wanted to match our width. Set a flag
            // indicating that we need to remeasure at least that view when
            // we know our width.
            matchWidth = true;
            matchWidthLocally = true;
        }
		//计算宽度
		//所有的Margin
        final int margin = lp.leftMargin + lp.rightMargin;
        final int measuredWidth = child.getMeasuredWidth() + margin;
        maxWidth = Math.max(maxWidth, measuredWidth);
        childState = combineMeasuredStates(childState, child.getMeasuredState());

        allFillParent = allFillParent && lp.width == LayoutParams.MATCH_PARENT;
        if (lp.weight > 0) {
            /*
             * Widths of weighted Views are bogus if we end up
             * remeasuring, so keep them separate.
             */
            weightedMaxWidth = Math.max(weightedMaxWidth,
                    matchWidthLocally ? margin : measuredWidth);
        } else {
            alternativeMaxWidth = Math.max(alternativeMaxWidth,
                    matchWidthLocally ? margin : measuredWidth);
        }

        i += getChildrenSkipCount(child, i);
    }

    if (nonSkippedChildCount > 0 && hasDividerBeforeChildAt(count)) {
        mTotalLength += mDividerHeight;
    }
	//计算Mode 是 AT_MOST 和 UNSPECIFIED 的情况
    if (useLargestChild &&
            (heightMode == MeasureSpec.AT_MOST || heightMode == MeasureSpec.UNSPECIFIED)) {
        mTotalLength = 0;

        for (int i = 0; i < count; ++i) {
            final View child = getVirtualChildAt(i);
            if (child == null) {
                mTotalLength += measureNullChild(i);
                continue;
            }

            if (child.getVisibility() == GONE) {
                i += getChildrenSkipCount(child, i);
                continue;
            }

            final LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams)
                    child.getLayoutParams();
            // Account for negative margins
            final int totalLength = mTotalLength;
            mTotalLength = Math.max(totalLength, totalLength + largestChildHeight +
                    lp.topMargin + lp.bottomMargin + getNextLocationOffset(child));
        }
    }

	//总长度家加上Padding
    // Add in our padding
    mTotalLength += mPaddingTop + mPaddingBottom;

    int heightSize = mTotalLength;

    // Check against our minimum height
    heightSize = Math.max(heightSize, getSuggestedMinimumHeight());

    // Reconcile our calculated size with the heightMeasureSpec
    int heightSizeAndState = resolveSizeAndState(heightSize, heightMeasureSpec, 0);
    heightSize = heightSizeAndState & MEASURED_SIZE_MASK;
    // Either expand children with weight to take up available space or
    // shrink them if they extend beyond our current bounds. If we skipped
    // measurement on any children, we need to measure them now.
    int remainingExcess = heightSize - mTotalLength
            + (mAllowInconsistentMeasurement ? 0 : consumedExcessSpace);
	
	//==========计算权重==========
	//如果在LinearLayout中使用了权重，那么就需要重新测量，处理权重的问题
    if (skippedMeasure || remainingExcess != 0 && totalWeight > 0.0f) {
        float remainingWeightSum = mWeightSum > 0.0f ? mWeightSum : totalWeight;

        mTotalLength = 0;

        for (int i = 0; i < count; ++i) {
            final View child = getVirtualChildAt(i);
            if (child == null || child.getVisibility() == View.GONE) {
                continue;
            }
			//获取LP
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
			//获取Weight
            final float childWeight = lp.weight;
			//开始计算权重
            if (childWeight > 0) {
				//计算权重
                //share = child 的weight * 剩下的高度 / 剩下的Weight的总和
				final int share = (int) (childWeight * remainingExcess / remainingWeightSum);
				//减去
                remainingExcess -= share;
                remainingWeightSum -= childWeight;

                final int childHeight;
				//使用最大大小
                if (mUseLargestChild && heightMode != MeasureSpec.EXACTLY) {
                    childHeight = largestChildHeight;
                } else if (lp.height == 0 && (!mAllowInconsistentMeasurement
                        || heightMode == MeasureSpec.EXACTLY)) {
					//当Height是0的时候，并且Mode不是 EXACTLY的情况
					//大小就等于share的大小
                    // This child needs to be laid out from scratch using
                    // only its share of excess space.
                    childHeight = share;
                } else {
                    // This child had some intrinsic height to which we
                    // need to add its share of excess space.
					//如果height!=0 那么Child的大小= share的大小+ Child 原本的大小
                    childHeight = child.getMeasuredHeight() + share;
                }
				//获取MeasureSpec
                final int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                        Math.max(0, childHeight), MeasureSpec.EXACTLY);
                final int childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                        mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin,
                        lp.width);
				//重新测量Child
                child.measure(childWidthMeasureSpec, childHeightMeasureSpec);

                // Child may now not fit in vertical dimension.
                childState = combineMeasuredStates(childState, child.getMeasuredState()
                        & (MEASURED_STATE_MASK>>MEASURED_HEIGHT_STATE_SHIFT));
            }

            final int margin =  lp.leftMargin + lp.rightMargin;
            final int measuredWidth = child.getMeasuredWidth() + margin;
            maxWidth = Math.max(maxWidth, measuredWidth);

            boolean matchWidthLocally = widthMode != MeasureSpec.EXACTLY &&
                    lp.width == LayoutParams.MATCH_PARENT;

            alternativeMaxWidth = Math.max(alternativeMaxWidth,
                    matchWidthLocally ? margin : measuredWidth);

            allFillParent = allFillParent && lp.width == LayoutParams.MATCH_PARENT;

            final int totalLength = mTotalLength;
            mTotalLength = Math.max(totalLength, totalLength + child.getMeasuredHeight() +
                    lp.topMargin + lp.bottomMargin + getNextLocationOffset(child));
        }

        // Add in our padding
        mTotalLength += mPaddingTop + mPaddingBottom;
        // TODO: Should we recompute the heightSpec based on the new total length?
    } else {
        alternativeMaxWidth = Math.max(alternativeMaxWidth,
                                       weightedMaxWidth);


        // We have no limit, so make all weighted views as tall as the largest child.
        // Children will have already been measured once.
        if (useLargestChild && heightMode != MeasureSpec.EXACTLY) {
            for (int i = 0; i < count; i++) {
                final View child = getVirtualChildAt(i);
                if (child == null || child.getVisibility() == View.GONE) {
                    continue;
                }

                final LinearLayout.LayoutParams lp =
                        (LinearLayout.LayoutParams) child.getLayoutParams();

                float childExtra = lp.weight;
                if (childExtra > 0) {
                    child.measure(
                            MeasureSpec.makeMeasureSpec(child.getMeasuredWidth(),
                                    MeasureSpec.EXACTLY),
                            MeasureSpec.makeMeasureSpec(largestChildHeight,
                                    MeasureSpec.EXACTLY));
                }
            }
        }
    }

    if (!allFillParent && widthMode != MeasureSpec.EXACTLY) {
        maxWidth = alternativeMaxWidth;
    }

    maxWidth += mPaddingLeft + mPaddingRight;

    // Check against our minimum width
    maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
            heightSizeAndState);

    if (matchWidth) {
        forceUniformWidth(count, heightMeasureSpec);
    }
}
```
上面的代码大体的流程如下
1. 计算未采用权重布局的Child的大小
2. 计算宽度
3. 如果采用了权重
4. 计算Child的Weight
5. 重新测量Child

### 计算Weight

首先呢，可以对LinearLayout的权重的计算的公式进行一个总结:
**View所占的宽/高+在剩余的大小中所占的比例**

比如当height/width 设置为 0 的时候，weight表现的是符合我们的预期的，但是一旦出现了类似下面的这种布局的时候就可能会对大小造成一些困扰。

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <Button
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:text="Button1"
        android:layout_weight="2" />

    <Button
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:text="Button2"
        android:layout_weight="3" />
</LinearLayout>
```

![Snag_19c754c5](http://cdn.shycoder.cn/Snag_19c754c5.png)

在上面的布局中，Button2 的 Weight = 3 ，但是却占据的空间更小，其实这仍然是符合我们上面提出的公式的，我们可以来套入公式试验一下。

假设 屏幕的高度 = H
* Button1和Button2 的Height都是 `match_parent` 所以，各自大小都是H，所以总大小是 2H

$$
Button2的高度= H - (H-2H) * 3/5 = 2/5H
$$

通过上面的公式，我们得出了和布局的样式相同的结构，这样就可以证明我们的公式是正确的，如果大家还是有些疑问，可以自己看一下源码是怎么计算的，多试验几个例子就明白了。

### 写在最后

本篇文章就到此结束了，如果文中有任何问题或者纰漏，欢迎大家指正；在此躬谢。下一篇文章我们将会学习 Android View的Layout的流程。