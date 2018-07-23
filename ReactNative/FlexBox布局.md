

## FlexBox布局

[TOC]

## 父容器中使用

### flexDirection
相当于LinearLayout中的oritation方向
 * row: 从左向右依次排列
 * row-reverse: 从右向左依次排列
 * column(default): 默认的排列方式，从上向下排列
 * column-reverse: 从下向上排列

### alignItems
alignItems是用来定义子View在父容器的交叉轴/垂直轴上的对其方式
*   baseline：子View对其基准线
*   flex-start 元素向侧轴起点对齐。
*   flex-end 元素向侧轴终点对齐。
*   center 元素在侧轴居中。如果元素在侧轴上的高度高于其容器，那么在两个方向上溢出距离相同。
*   stretch 弹性元素被在侧轴方向被拉伸到与容器相同的高度或宽度。

### flexWrap
flexWrap属性定义了子元素在父视图内是否允许多行排列，默认为nowrap。
- nowrap（默认值）：即使空间不足，伸缩容器也不允许换行
- wrap：伸缩容器在空间不足的情况下允许换行。如果主轴为水平轴，换行的方向为从上到下

### justifyContent
用来定义伸缩项目沿主轴线的对齐方式
flex-start（默认值）：伸缩项目向主轴线的起始位置靠齐
flex-end：伸缩项目向主轴线的结束位置靠齐
center：伸缩项目向主轴线的中间位置靠齐
space-between：伸缩项目回平均分布在主轴线上，第一个伸缩项目在主轴线的起始位置，最后一个伸缩项目在主轴线的结束位置
space-around：伸缩项目会平均地分布在主轴线上，两端保留一半的空间

justifyContent和alignSelf是一对,前者代表在伸缩item在主轴上的对其方式,后者代表在伸缩item在垂直轴上的对其方式

## 子容器中使用
### flex
相当于Android中的weight权重
flex键的类型是数值，取值有0和1..，默认值是0，如果值为1的话，子组件将自动缩放以适应父组件剩下的空白空间，我们在第一个子元素的样式属性中添加flex属性.
### alignSelf
alignSelf用来设置子View在交叉轴上的对齐方式，总共有6个属性,
auto：子View按照自身设置的宽高显示，如果没有设置，则按stretch来计算其值
flex-start：子View向交叉轴的开始位置靠齐
flex-end：子View向交叉轴的结束位置靠齐
center：子View向交叉轴的中心位置靠齐
baseline：子View按基线对齐
stretch：子View在交叉轴方向占满伸缩容器

- [React-Native中的flexbox布局的使用](http://blog.csdn.net/hai_qing_xu_kong/article/details/72672404)
- [React Native布局详细指南](http://www.devio.org/2016/08/01/Reac-Native布局详细指南/)