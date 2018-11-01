#### AndoridMainfest.xml

1. 标明使用了哪些权限?
2. 入口组件是什么?
3. 有哪些组件?以及组件的Action
4. 可以通过meta-data做一些配置信息

这些都是给系统看的,使系统了解你的应用有哪些组件和权限

#### Context

上下文,环境 我的理解就是 “钱”

继承自Activity、Service、Application的组件统称为银行,用来印钞的

- 这个钱可以用来再开银行 
  startActicity() startService
- 拿钱可以干很多事,我们在日常生活中只要和外界接触,方方面面都需要钱
  类比于Android环境,各个组件之间交互context是必不可少的,但是在组件内部的逻辑有一部分是不需要context的,就好比我们一家人帮忙做点事是不需要报酬的