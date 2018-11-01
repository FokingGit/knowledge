#### 属性动画

属性动画可以真实改变对象的属性

##### 1. Evaluator 

估值器,估值器也可以自定义

```java
public interface TypeEvaluator<T> {
    // fraction 比例 
    public T evaluate(float fraction, T startValue, T endValue);
}
```

##### 2. AnimatorSet

用来组合多个Animator,并制定这些动画是顺序播放还是同时播放

##### 3. ValueAnimator

ValueAnimator是属性动画中最重要的一个类,继承在Animator.它定义了属性动画大部分的核心功能,包括计算各个帧的属性值、处理更新事件、按照属性值的类型控制计算规则

一个完成的属性动画有两部分构成:

- 计算各个动画各个帧相关的属性值
- 将这些属性值设置给制定的对象

ValueAnimator和估值器,为我们提供了第一部分的功能,那么第二部分需要我们自己实现,ValueAnimation的构造函数是空实现,需要我们用静态工厂的方式去实现然后结合

Animate.addUpdateListener()的方式,将对应的目标对性进行设置

##### 4. ObjectAnimator

ObjectAnimator是系统为我们提供的实现了第二部分功能的类



动画最终还是走到了**Choreographer**