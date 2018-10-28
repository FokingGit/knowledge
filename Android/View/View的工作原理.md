#### 自定义View的流程，自定义View需要注意的问题

- 配置warp_content

- 处理padding、margin值

- 有线程或者动画,需要及时停止

自定义View需要重写onMeasure方法来处理wrap_content,不需要重写onLayout方法,注意在onDraw方法中处理padding、margin

### invalidate、postInvalidate、requestLayout的区别

### View的绘制流程

- View的measure
  1. setMeasureDimension() 设置View宽/高测量值
  2. getDefaultSize() AT_MOST和EXACTLY 这两种测量模式设置的宽高分别是就是测量的宽高,而UNSPECIFIED 则根据getSuggestedMinimunXXX()来确定,会在minWidth和背景Drawable中做选择进行取值
  3. 从getDefaultSize中可以了解到,View的宽高由specSize决定,直接继承View的自定义控件需要重写onMeasure方法**并设置wrap_content时自身大小(根据需要的大小在onMeasure中自己确定)**,否则在布局中使用wrap_content就相当于match_parent

- ViewGroup的mesure过程
  1. 通过measureChild的方法来完成子控件的测量,根据LayoutParams
  2. ViewGroup中没有给提供具体的onMeasure的过程,这个是因为不同的容器控件有自己的布局特性,FrameLayout、LinearLayout都有不同的特性 eg:LinearLayout中竖直方向布局,循环调用子控件的measure方法完成 调用getMeasureXXX来获取,之后进行累加

- layout过程
  1. 计算好padding、margin 调用给子控件onLayout,最中确定View的宽高

- draw过程
  1. canvas知识

### View的性能优化，页面卡顿的原因

### 常用到的回调方法以及含义

- onSizeChanged:当View的宽高发生变化的时候

  调用的路径ViewGroup#layout->child.layout->setFrame->sizeChange->onSizeChanged

做过Android开发的同志们都知道,一提View的绘制肯定就会想到那三大方法:onMeasure(),onLayout(),onDraw(),分别负责对控件的测量,排版.还有显示,那这篇文章对这三个方法做进一步的分析,主要解决以下2个问题:
### 1.这三个方法什么时候调用?
### 2.各个方法中的参数主要干什么?怎么来的?

>问题一:这三个方法什么时候调用?

我们知道Activity是一个用户界面,而他的界面性是通过Window来展示的,那么相应的关于View的绘制肯定与这个window有关,如下图所示是这样的一个层级结构.

![Android视图层级结构](http://upload-images.jianshu.io/upload_images/3747483-76c852d27bee757b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
当Activity启动之后会通过WindowMangerGlobal间接调用ViewRootImpl类的requestLayout()方法,那么在requestLayout()的方法中又会做什么呢?

```java
public void requestLayout() {
         //如果不是正在刷新界面
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }

void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            //通过Handler将Runnable发送过去
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }

final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
```
局势马上就要明朗了
```java
void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }

            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }
```
我们可以看到在doTraversal()中调用了performTraversals(),我们最后再看看,这个方法里面到底做了什么?
差点吐血...这个方法800行代码,但是我在其中惊喜的发现了几行关键代码
```java
private void performTraversals() {
         ...
         //根据窗口的尺寸获得DecorVIew测量规格
         int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
         int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
         ...
         //执行测量流程
         performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
         ...
         //执行排版流程
         performLayout(lp, mWidth, mHeight);
         ...
        //指定展示流程
        performDraw();
}
```
通过以上分析,我们就知道我们重写的那三个方法的调用起点就从这开始的,然后某一个地方执行回调,就调用了我们自己代码.
>问题二:各个方法中的参数主要干什么?怎么来的?

```
1.onMeasure(int widthMeasureSpec, int heightMeasureSpec)
2.onLayout(boolean changed, int left, int top, int right, int bottom)
3.onDraw(Canvas canvas)
```
### MeasureSpec
对于onMeasure中的两个Int值的参数,每一个都表示一个int值的整数,高两位代表测量规格,低30位代表某种测量规格下的大小既SpecSize,这个MeasureSpec是用来表示如何测量这个View的
```java
protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```
我们可以看到View的MeasureSpec是根据父视图的MeasureSpec和自身的LayoutParams得来的.
三种测量模式:

UNSPECIFIED:不指定测量模式,父视图没有限制子视图显示的大小,子视图可以拿到任意的大小,这种测量方式
​                         一般会用到系统级别

EXACTLY:精确测量,当该视图的layout_width或者layout_height设置成固定大小,或者是match_parent的时候,该视图的测量模式就是精确测量,这个时候测量出来的宽高就是SpecSize.

AT_MOST:最大值模式,就是当该视图设置的layout_height或者layouy_width设置文wrap_content的时候,该视图的测量模式就是最大值模式,指示该视图的尺寸是不超过父视图的大小的任意尺寸

```java
sUseZeroUnspecifiedMeasureSpec = targetSdkVersion < Build.VERSION_CODES.M;

case MeasureSpec.UNSPECIFIED:
    if (childDimension >= 0) {
        // Child wants a specific size... let him have it
        resultSize = childDimension;
        resultMode = MeasureSpec.EXACTLY;
    } else if (childDimension == LayoutParams.MATCH_PARENT) {
        // Child wants to be our size... find out how big it should
        // be
        resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
        resultMode = MeasureSpec.UNSPECIFIED;
    } else if (childDimension == LayoutParams.WRAP_CONTENT) {
        // Child wants to determine its own size.... find out how
        // big it should be
        resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
        resultMode = MeasureSpec.UNSPECIFIED;
    }
    break;
```

| Child-size/Parent | EXACTLY                    | AT_MOST                    | UNSPECIFIED  |
| ----------------- | -------------------------- | -------------------------- | ------------ |
| size>0            | Size/EXACTLY               | Size/EXACTLY               | Size/EXACTLY |
| much_parent       | parentSize-padding/EXACTLY | parentSize-padding/AT_most | 分版本 见上  |
| warp_content      | parentSize-padding/AT_most | parentSize-padding/AT_most | 分版本 见上  |

### onLayout参数
后四个参数分别代表左上右下,分别是针对父控件的位置,那么chenged代表什么呢?
在View.java中我们可以看到:
```java
public void layout(int l, int t, int r, int b) {
...
boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

 if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);

```
这个change就是代表是否改变过是否需要重绘
### Canvas
顾名思义,就是画布的意思,我们想要这个View呈现什么样的形状,就是在这张画布上进行展示.
可以想到有画布,必须得有画笔,有画笔了你想画什么就得有路线,也就是path.下面这篇文章写的特别详细,这里我就不赘述了
[Canvas详解](http://www.jianshu.com/p/f69873371763)