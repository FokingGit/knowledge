### Promise
Promise就是一个对象，用来传递异步消息的结果；

**异步消息有三种状态**：
- Pending(进行中)
- Resolve（已成功）
- Reject（已拒绝）

**状态的改变只涉及两种**
- Pending->Resolve
- Pending->Reject

> 与原生交互的Promise的语法

```JavaScript
(picturePat,pictureId) => {
    AliCloudUpload.upLoadPic(picturePat, pictureId)
                  .then((res) => {
                    Alert.alert(res)
                  });
```
原生的定义
```java
void upLoadPic(String picturePath, String pictureId, final Promise promise)
```
> 基本用法

Promise的构造方法必须接收一个函数
```JavaScript
new Promise((resolve,reject)=>{
     //异步代码
     if(isSucccess){
        resolve(res);
     }else{
        reject(error)
     }
}).then((res)=>{
     console.log(res);
});
```
PS：resolve不仅仅只返回一个参数，还可以返回一个Promise对象

> then() & catch()

还是已上述上传图片为例，upLoadPic的返回值就是一个`Promise`我们这个时候，就可以通过`.then`这种方式,接收返回结果.`then()`调用的时机就是当`Promise`的状态发生改变的时候。`.then()`返回的也可以是一个`Promise`，类似于`Rxjava`的`andthen`;

`catch`就是一个捕获异常的方法
