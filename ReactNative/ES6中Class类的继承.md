### ES6中Class类的继承

- 子类如果想实现不同的方法，可通过在子类中使用默认的属性去实现，然后这时候在调用`super(props)`的过程中，就间接的调用了子类自己的函数
- 如果子类中没有实现`component`的生命周期函数，RN会去调用父类的生命周期函数，这点和`Java`中类似