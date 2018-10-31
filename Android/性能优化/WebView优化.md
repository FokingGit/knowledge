#### 使用优化

问题:加载速度慢

1. 添加缓存
2. 预加载资源文件
3. 使用第三方 WebView 内核(腾讯X5内核)

#### 防止内存泄漏

引起内存泄漏的原因:

1. 持有Java代码的引用

2. 部分Android版本,如果webView destory之后不会移除callback

处理内存泄漏的方法:

1. 独立的web进程,在清单文件中使用process与主进程隔开,相互通信使用(AIDL进程间通信)
2. 将webView动态添加到Activity中,在Activity的onDestory()方法中,再从Activity中将WebView移除,并且调用webView.getParent()将webView移除,这样webView会回调onDetachToWindow()从而移除掉callback.最后调用webView.destory()