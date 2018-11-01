帧动画,类gif图

##### 文件存放的地址

`res/drawable/filename.xml`

##### 示例代码

```xml
<?xml version="1.0" encoding="utf-8"?>
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot=["true" | "false"] >
    <item
        android:drawable="@[package:]drawable/drawable_resource_name"
        android:duration="integer" />
</animation-list>

location: res/drawable/rocket.xml
<?xml version="1.0" encoding="utf-8"?>
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="false">
    <item android:drawable="@drawable/rocket_thrust1" android:duration="200" />
    <item android:drawable="@drawable/rocket_thrust2" android:duration="200" />
    <item android:drawable="@drawable/rocket_thrust3" android:duration="200" />
</animation-list>
```

##### 相关说明

`<animation-list>`

**Required**, This must be the root element. Contains one or more `<item>`elements.

attributes:

- `android:oneshot`

  *Boolean*. "true" if you want to perform the animation once; "false" to loop the animation.

`<item>`

A single frame of animation. Must be a child of a`<animation-list>`element.

attributes:

- `android:drawable`

  *Drawable resource*. The drawable to use for this frame.

- `android:duration`

  *Integer*. The duration to show this frame, in milliseconds.

##### Java代码使用

```java
ImageView rocketImage = (ImageView) findViewById(R.id.rocket_image);
rocketImage.setBackgroundResource(R.drawable.rocket_thrust);

rocketAnimation = rocketImage.getBackground();
if (rocketAnimation instanceof Animatable) {
    ((Animatable)rocketAnimation).start();
}
```