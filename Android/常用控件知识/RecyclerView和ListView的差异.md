### RecyclerView和ListView的差异

[TOC]

####  API使用

##### 1. 写法 

- ListView需要适配器继承BaseAdapter,需要自定义ViewHolder+convertView+setTag来完成复用

- RecyclerVIew需要适配器继承RecyclerVIew.Adatpter,需要强制开发者实现继承自Recycler.ViewHolder(),同时需要额外创建一个LayoutManager,来负责布局.

##### 2. HeaderView和FootView

- ListView提供了封装好的方法addHeader()和addFooterVIew()
- RecyclerView没有对应的方法,需要我们同时`getItemViewType(int pistion)`来实现

##### 3. 空数据

- ListView有`setEmptyView()`
- RecyclerView 需要自己添加逻辑

##### 4. 局部刷新

- 在ListView中通常刷新数据是用notifyDataSetChanged() ，但是这种刷新数据是全局刷新的（每个item的数据都会重新加载一遍），这样一来就会非常消耗资源；但是如果要在ListView实现局部刷新，依然是可以实现的，当一个item数据刷新时，我们可以在Adapter中，实现一个onItemChanged()方法，在方法里面获取到这个item的position（可以通过getFirstVisiblePosition()），然后调用getView()方法来刷新这个item的数据；
- RecyclerView中可以实现局部刷新，例如：notifyItemChanged()；

#### 功能

##### 1. 动画

- RecyclerView封装了Item添加或者移除的动画 SimpleItemAnimator和DefaultItemAnimator,也可以自定义
- ListView需要我们自己去实现这部分功能

##### 2. 瀑布流(弹幕效果)

https://www.jianshu.com/p/6649f5239aef

#### 缓存

##### 1. 缓存的对象不同

- ListView缓存的是View
- RecyclerView缓存的是View+ViewHolder

##### 2. 缓存的层级不同

- ListView缓存两级
- RecyclerView缓存4级,添加了RecyclerViewPool