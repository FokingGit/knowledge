#### 对Window的理解

1. Window就是每一个页面的对象,以万达房地产的例子来说明

   万达相比于一个应用,万达有很多块地皮,对应于一个应用有许多window,给window设置View是由WindowManager来完成的;

   - PhoneWindow:地基

   - Activity:总设计师

   - Dialog对应的window:商品房

   - Toast对应的window:普通住宅房

   最后通过这个几个角色,将这个建筑完成,没有总设计师是没有办法建地基的,也就是没有Activity是没有PhoneWindow的

2. Window是分级的,根据WindowManager.LayoutParams.type来确定

   - 应用的Window:1-99
   - 子Window:1000-1999
   - 系统Window:2000-2999

   采用系统window层级需要额外声明权限

#### 创建和内部机制

1. Dialog创建需要的Context,因为要添加window需要一个token,而这个token只有Activity有
2. Toast所依附的是系统的window,针对系统的window是不需要token的

