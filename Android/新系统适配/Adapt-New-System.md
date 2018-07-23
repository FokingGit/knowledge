> 何为前台应用：
>
> 1.该应用的Activity处于可见状态，无论这个Activity是处于创建还是暂停状态；
>
> 2.拥有前台服务；
>
> 3.有前台应用绑定了该应用的服务或者使用了该应用的ContentProvider与该应用相连；
>
> 如果一个应用没有上述特征，则视为后台应用；



### 适配7.0

* 针对应用间共享文件
  * FileProvider的使用

* 移除了三种广播：后台的App将不能监听到这三种广播，当App处于前台，并且通过context.registerBroadcast的方式是可以监听到这些广播的

  1. `CONNECTIVITY_ACTION` ：监听网络变化
  2. `ACTION_NEW_PICTURE` ：
  3. `ACTION_NEW_VIDEO`：
     ​

  `NEW_PICTRE`和`NEW_VIDEO`可通过`JobScheduler`来完成

* 系统将阻止应用动态链接非公开 NDK 库

* 语言区域国际化

  - Configration.local(7.0以上过期)
  - Configration.getLocalList().get(0) 7.0 以上使用

### 适配8.0

* java.lang.IllegalStateException: Only fullscreen opaque activities can request orientation 

  ```xml
  <style name="MyDialogStyleBottom" parent="android:Theme.Dialog">
      <item name="android:windowAnimationStyle">@style/AnimBottom</item>
      <item name="android:windowFrame">@null</item>
      !-- A border -->
      <item name="android:windowIsFloating">true</item>
      !-- Whether on the activity -->
      <item name="android:windowIsTranslucent">true</item>
      !-- translucent -->
      <item name="android:windowNoTitle">true</item>
      !-- No title -->
      <item name="android:windowBackground">@android:color/transparent</item>
      !-- The background transparent -->
      <item name="android:backgroundDimEnabled">true</item>
      !-- The fuzzy -->
      </style>
  ```

  ```xml
  <activity
   	android:name="com.rihuisoft.loginmode.view.SelectPicPopupWindow"
   	android:screenOrientation="portrait"
   	android:theme="@style/MyDialogStyleBottom" />
  ```

当在andorid 8.0上 如果将Activity设置成窗口模式，那就不能设置固定的横竖屏；
目前处理的方式是将设置成竖屏的方式去掉
奇怪的是，去掉之后，就算开了自动旋转，这个窗口Activity也不会旋转成横屏 

* 服务
  * 后台服务，当App处于后台的时候，一段时间之后，这个后台服务就像调用了stopService()一样；
  * 启动前台服务的方式；


* 后台位置获取

  * LocationManager；
  * WifiManager getScanResults 的返回结果 

  ```
  当应用处于后台的时候，位置更新不会已原来的频率更新位置；同样情况下，就算startScan()系统也只会返回上一次Scan的结果；
  ```

  ​

* 语言区域国际化

  * 使用DISPLAYl类别语言区域的时候，函数使用 Locale.getDefault(Category.DISPLAY) 来代替 Locale.getDefault()
  * 调用 Currency.getDisplayName(null) 会引发 NullPointerException （不知道是干啥的？）

### 适配Android P

* 针对所有API级别
  * 对于非SDK接口的调用限制：现已禁止访问特定的非 SDK 接口，无论是直接访问，还是通过 JNI 或反射进行间接访问。尝试访问受限制的接口将会生成 NoSuchFieldException 和 NoSuchMethodException 之类的错误
  * 当应用处于后台时，禁止访问麦克风、摄像头和传感器
  * 移除加密提供程序：从 Android P 开始，Crypto JCA 提供程序现已被移除。调用 SecureRandom.getInstance("SHA1PRNG", "Crypto") 将会引发 NoSuchProviderException。
  * 更严格的 UTF-8 解码器：在 Android P 中，针对 Java 语言的 UTF-8 解码器比以往更严格，并且遵循 Unicode 标准。
  * 强制执行 FLAG_ACTIVITY_NEW_TASK 要求：在 Android P 中，您不能从非 Activity 环境中启动 Activity，除非您传递 Intent 标志 FLAG_ACTIVITY_NEW_TASK。 如果您尝试在不传递此标志的情况下启动 Activity，则该 Activity 不会启动，系统会在日志中输出一则消息。
* 针对API级别为P的设备
  * 使用前台服务必须申请权限：想要使用前台服务的应用必须首先请求 FOREGROUND_SERVICE 权限。这是普通权限，因此，系统会自动为请求权限的应用授予此权限。在未获得此权限的情况下启动前台服务将会引发 SecurityException。 
  * 移除对Build.serial的直接访问：现在，需要 Build.serial 标识符的应用必须请求 READ_PHONE_STATE 权限，然后使用 Android P 中新增的新 Build.getSerial() 函数。
  * 不允许共享 WebView 数据目录 ：现在，不允许应用在不同进程之间共享一个 WebView 数据目录。如果您的应用有多个进程使用 WebView、CookieManager 或 android.webkit 软件包中的任何其他 API，则在第二个进程调用 WebView 函数时，您的应用将会崩溃。 
  * 弃用 Bouncy Castle 加密 : Android P 弃用了几个来自 Bouncy Castle 提供程序中的加密技术，代之以由 Conscrypt 提供程序提供的加密技术。调用请求 Bouncy Castle 提供程序的 getInstance() 将会生成 NoSuchAlgorithmException 错误。要解决这些错误，请不要在 getInstance() 中指定提供程序（也就是请求默认实现）。
