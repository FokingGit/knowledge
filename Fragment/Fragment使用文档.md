# Fragment使用文档

平时在开发中我们对切换`Fragment`的方式有两类

- `replace()`
- `add()`+`remove()`+`hide()`+`show()`

提交的方式有：

- `int commit()`
- `int commitAllowingStateLoss()`
- `void commitNow()`
- `void commitNowAllowingStateLoss()`

有时我们还会通过`setCustomAnimations()`来设置动画，还会通过`addToBackStack() `将添加到返回栈

添加到返回栈我们还有可能进行弹栈Pop，弹栈的方式有：

- `void popBackStack()`
- `boolean popBackStackImmediate()`
- `void popBackStack(String name, int flags)`
- `boolean popBackStackImmediate(String name, int flags)`
- `void popBackStack(int id, int flags)`
- ` boolean popBackStackImmediate(int id, int flags)`

> 本片文章主要阐述的就是：
>
> 1. 两种切换方式有什么不同？
> 2. 四种提交方式有什么不同？
> 3. 六种弹栈方式有什么不同？

