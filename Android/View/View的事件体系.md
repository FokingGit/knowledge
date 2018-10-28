

[TOC]

#### MotionEvent

- ACTION_CANCEL: 父控件收回事件,子空间回调到这个事件

> 那什么情况下会回调这个事件呢?
>
> 在一个序列事件中,父控件没有拦截DOWN事件,子控件拦截了处理了DOWN事件,并且onTouch()或者onTouchEvent()方法返回了true,在此之后父控件拦截了该序列的其他事件,那么,子控件就会回调这个事件

在ViewGroup的dispatchTouchEvent中

```java
			//该种情况下显然mFirstTouchTarget是不为null的
            if (mFirstTouchTarget == null) {
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        //判断是够取消给子控件,显然这个时候的intercepted为true
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }

		//在dispatchTransformedTouchEvent中如果是cancel
 		final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            //人工封装CANCEL事件
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                //分发给子事件
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }
//有同学就问了,那么这个事件也没有给父控件消费调啊,确实如此,说到底这个事件是给子控件消费掉了,只不过在发给子控件的过程中,MotionEvent的Action被人工转化为ACTION_CANCEL事件

//在这个分发结束之后,mFirstTouchTarget会被设置为null,那么这个序列接下来的时间,都会分发给父控件进行处理
```

- ACTION_OUTSIDE: 一般VIew无论如何不管怎么划出都不会回调这个事件,Dialog中回适用到这个action,当我们想监听这个事件的时候,必须设置对应的WIndowManager的flag为FLAG_WATCH_OUTSIDE_TOUCH,才能收到这个事件

#### View的位置参数有哪些，left，x，translationX的含义以及三者的关系

基本参数有left、top、right、bottom

Android 3.0 之后新添加了 x、y、translateX、translateY

translateX相对于父控件偏移的位置.

关系

```java
x = left + translateX
y = top + translateY
```

- View的绘制流程，onMeasure，onLayout的计算流程，MeasureSpec的几种模式

#### 处理滑动的几种方式

- 通过View的scrollTo/scrollBy

​    1. **只能将View的内容进行滑动,并不能将View本身进行滑动**

​    2. View的左边缘在View内容左边缘的右边时,mSCrollX为正值,也就是说,当右->左滑动时,mSrollX为正值

- 通过View的平移动画

- 设置View的LayoutParams的位置,使得 View重新布局

#### 滑动的方式以及Scroller滑动的原理

- 使用Scroller:将要滑动的距离根据滑动事件分解为小片段,利用invalidate()+重写computeScroll()方法+scrollTo来实现缓慢滑动,其实Scroller内部只是封装了互动相关的信息

- 通过属性动画,ValueAnimatot.ofInt(0.1).setDuration(100),根据每一帧执行时获取动画完成的比例,来计算当前View移动的距离

- 使用延时策略:Handler

#### Touch事件的分发流程

```java
事件在分发的过程中是一个责任链模式,各级的ViewGroup就是责任人,通过dispatchTouchEvent方法将请求分发,onTouchEvent将结果返回
```

**第一步 : 事件到达Activity**

在Activity中的dispatchTouchEvent中调用了getWindow().superDispatchTouchEvent()

**第二步 : 事件到达PhoneWindow**

调用getDecorView的superDispatchTouchEvent

**第三步:事件到达ViewGroup**

由于DecorView调用了super.dispatchTouchEvent(),且其父控件为Framelayout,单Framelayout没有对应的实现,所以事件就到了ViewGroup中

1. 如果是Down事件
    ​    1.1  清除TouchTarget链表
    ​    1.2. 清除状态,将设置的FLAG_DISALLIWOW_INTERCEPT复位
2. 如果是Down事件或者mFirstTouchTarget !=null 
    ​    2.1 判断子控件是否要求父控件不要拦截事件
    ​            2.1.1 子控件没有要求,判断自身的onInterceptTouchEvent是否拦截
    ​            2.1.2 子控件需要,不拦截事件,前提是不拦截该序列事件的DOWN事件
    ​    2.2 检测事件是否取消
    ​    2.3. 如果事件未被取消和拦截
    ​            2.3.1 如果该事件是单点的DOWN事件,或者多点的DOWN事件,或者是鼠标划过的MOVE事件
    ​               2.3.1.1 获取该Viewgroup下的子节点,进行遍历,如果该事件满足要求则**分发到对应的View**
    ​               2.3.1.2 在addTouchTarget方法中设置mFirstTouchTarget
3. 判断mFirstTouchTarget是否未null
    ​    3.1 为空,有可能是这个ViewGroup没有子控件,也可能是父View拦截了事件,那么将调用View中的dispatchTouchEvent,最后嗲用自己的onTouch()方法,或者onTouchEvent()方法
    ​    3.2 不为空,说明成功完成了2.3.1中对应的流程,那么接下来的事件直接走到这里负责除DOWN事件的分发

第四步:事件到达View

通过第三步的事件分发,事件会到View的dispatchTouchEvent

1. 是否是DOWN事件,如果是的话,调用stopNestedScroll停止滑动
2. 如果设置了onTouchEventListener则回调onTouch()方法
3. 如果没有设置onTouchEventListener()或者onTouch()返回false,那会调用View的onTouchEvent()方法,消费事件

#### 长按事件

```
通过Handler的delay操作来完成,在Down事件中,发起这个message,在Move和up和cancel取消这个是事件,如果在延时结束之前还没有移除这个Message,那就出发这个longClick事件
```

#### 处理滑动冲突

外部拦截法:父控件重写onInterceptorTouchEvent,子控件需要的事件不拦截

内部拦截法:父容器默认拦截除ACTION_DOWN之外的其他事件,子控件可调用requestDisallowInterceptorTouchEvent()

