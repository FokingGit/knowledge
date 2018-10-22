# 冯剑

## 个人信息

- 求职意向:   Android开发
- 基本信息：1993.3.7/男/汉/党员
- 教育经历：(2011-2015) 沈阳理工大学/信息科学与工程学院/英语六级（工作之后参与过《Android插件化开发指南》翻译）
- 联系方式：18302424842 / mister_fengjian@163.com
- **Github**不定期更新：
  - 地址：https://github.com/FokingGit/knowledge
  - 读书笔记
  - 每周工作、知识总结

## 工作经历

- **(2017.07-至今) 北京智云奇点科技有限公司(AbleCloud)** 

  主要负责智云优估APP、智能家居APP、智能家居Android SDK开发,18年初开始担任移动端负责人,负责团队技术方案的确定、任务分配以及一些技术文档和规范的撰写.

- **(2015.04-2017.06) 沈阳众智网络科技有限公司**

  参与统一教育APP的开发,完成新功能开发、APP优化、视频离线下载、直播间、版本重构等

## 项目经历

#### 1. 智云优估

​	是一款汽车评估工具应用,用户通过APP上传车辆图片及基本信息,后台会估算出车辆的价值、车况反馈给客户,该项目采用了ReactNative+原生模式开发.

> 负责的技术点

- 自定义相机拍照功能,图片优化,结合阿里云完成图片的上传和下载及本地缓存
- 集成Realm数据库,设计数据库表结构,封装数据库操作
- 车辆评估单编辑过程相关信息实时保存到本地数据库
- 通过Gradle配置优化打包流程,使用shell脚本完成热更新bundle包的一键上传
- APP架构的搭建和重构,后台多接口返回数据重新整合,评估步骤、评估基本信息的动态配置(输入类型、单选类型总共8种)

> 遇到的难点及解决方法

- 数据实时保存:在评估过程中涉及的组件较多,保证每个组件在变化时能够监控,并且写到数据库中,最后通过重写组件的生命周期+添加数据库交互间隔的方式解决.
- 图片的下载:该项目因为没有搭建文件管理系统,使用了阿里云OSS,RN端需要展示图片,首先需要根据图片在阿里存储的id在原生端下载成功并转化为本地uri供RN展示,解决方式:RN端需要展示的id打包为一个数组,给到原生端,原生下载完成之后,回传给RN端,过程中涉及位置标示等

#### 2. AbleCloud Matrix SDK

​	AbleCloud Matrix SDK是AbleCloud推出的Android平台上用于快速进行物联网APP开发的软件开发工具包,内部实现了SmartConfig配网、AP配网、封装通用接口请求、数据订阅.

> 负责的技术点

- 负责SDK从1.X到2.X版本的重构,设计组件化的架构,将原本臃肿的SDK分拆分为5个模块
- 独立负责common模块的开发,协助matrix-app和matrix-local模块,在开发过程中加强了对设计模式的理解

- 使用Socket基于UDP和TCP完成与设备局域网通信
- 增强SDK的安全性,参与和后端同学设计请求鉴权方式,添加SSL pining和混淆
- 基于WebSocket协议,完成设备数据的订阅和推送

> 遇到的难点及解决方案

- 如何保证重构之后的功能完善:通过两个方向来确定,第一基于1.6版本的单元测试,完成2.X版本的单元测试,第二开发演示Demo既可以验证2.X SDK功能,也相当于一个2.XSDK的使用文档

#### 3. 智能家居APP

​	基于Matrix SDK针对不同硬件厂商定制化开发APP,使APP具有配网、设备控制、相关数据展示等功能,主要负责了亚都、苏珀尔、莱克、零微、菲斯曼、SEB、millHeat等客户.

> 项目内容

- 独立维护/开发各个客户的定制APP,参与硬件协议的确定
- 7.0&8.0系统适配、屏幕尺寸适配、凹凸屏幕适配、优化工作
- Java与JS交互,js中多语言适配
- 自定义控件开发,柱状图、曲线图,通用组合控件封装
- 使用DataBinding框架,保证数据展示页面实时更新
- 配合硬件开发的同学,联调设备

> 遇到的难点和解决方案

- 在每个项目末期需要到前场通过APP进行实地测试,出现最多的问题就是配网失败(很多是因为测试覆盖不到),解决的方式:基于平台的推送框架,开发了一款专门收集APP配网过程中日志的APP,一旦前场在使用过程中出现了问题,会推送到这款APP中,方便开发人员快速定位并解决问题

#### 4. 统一教育

​	统一教育是一款在线教育的应用,采用okHttp+RxJava+MVP开发模式,项目主要涉及有视频的点播、直播、流媒体视频离线缓存下载,引入360智能摄像头实时监控,原生与JS的大量交互.

> 负责的技术点

- 网络请求的二次封装以及请求接口和参数的对称加密。
- Android和JS进行交互与WebView的优化
- M3U8流媒体文件离线缓存及创建本地服务器进行播放
- 视频的点播、直播拉流与直播间在线试卷的发放和回收
- UI层框架的搭建以及继承式自定义控件的开发
- 集成360智能摄像头，使用极光推送实现推送,使用支付宝、微信、二维码 、完成支付

## 技能

- 掌握语言 Java,熟悉语言 Kotlin,JavaScript
- 掌握Android基础,熟悉NDK开发流程,参与过Java和C混合开发
- 熟悉单例、代理、builder、责任链等设计模式和代码规范
- 了解http&https、TCP&UDP以及Socket编程,熟悉okhttp源码
- 了解ReactNative启动过程、通信过程
- 熟练使用git、charles、understand等其他开发工具

## 业余

读书：

- 《ES6标准入门》
- 《Gradle For Android》
- 《编写可读代码的艺术》
- 《Android开发艺术探索》

运动：篮球、游泳

## 致谢

感谢您花时间阅读我的简历，期待有机会与您一起共事。

简历地址：https://github.com/FokingGit/knowledge/blob/master/README.md

