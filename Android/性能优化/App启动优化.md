> APP启动优化优化的部分就是缩小Application的oCreate()  ->	启动页onResume()方法之间的时间,目的就是使用户在点击桌面的一瞬间在视觉上已经进入APP中.首先在点击桌面图标->执行makeApplication()方法,这个时间我们是无法优化的,所以只能在oCreate()  ->	启动页onResume()想办法

#### 检测启动APP方法

通过Android studio的CPU Profiler可以观察CPU 使用率和线程 Activity,并记录函数跟踪,可以观察到每个函数的调用顺序,以及每个函数的执行时间,可以通过该方法确定APP启动的时间,以及启动过程中每个方法所占用的时间.

#### 优化方式

1. 给启动的Activity的windown设置一个background,当Acitivity结束之后,再取消这个背景,这个方法本质上是没有改变根本问题
2. 再Application中延时加载、异步加载
3. 还有当Activity启动完成之后,在继续加载Application中的,需要根据实际的业务场景分析