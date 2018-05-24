### NDK开发

> **生成头文件**

```java
javah -d jni -classpath library\build\intermediates\classes\debug com.accloud.device
service.DeviceService
```

需要使用本地代码类中调用静态代码块

```
static {
    System.loadLibrary("Device-Service");
}
```

> **编译JNI文件**

安装NDK之后，会在本地有ndk-build.cmd，可以在对应的jni目录下执行，ndk-build.cmd编译jni代码，编译的结果会在jnit统计目录下，生成libs文件夹，会生成对应架构的so文件。同时生成obj文件夹，存放编译过程中的代码生成的文件

> **编译NDK的方式**

- Android Studio 2.2 之前通过ndk-build方式
  - 需要创建两个文件`Android.mk`和`Application.md`文件
- 2.2之后推荐使用CMake的方式
  - 需要创建CMakeList.txt配置文件

> **调用方式**

- C也可以调用Java的方式，分为调用静态方法和实例方法，类似反射

> **手机架构**

| CPU架构 | ABI         | 附加           |
| ------- | ----------- | -------------- |
| ARMv5   | armeabi     | 32位，从2010年 |
| ARMv7   | armeabi-v7a | 32位，从2010年 |
| x86     | x86         | 32位，从2011年 |
| MIPS    | mips        | 32位，从2012年 |
| ARMv8   | arm64-v8a   | 64位，从2014年 |
| MIPS64  | mips64      | 64位，从2014年 |
| x86_64  | x86_64      | 64位，从2014年 |

