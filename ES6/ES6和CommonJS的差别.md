[TOC]

## ES6和CommonJS的差别
最开始是没有模块化的，只要全局变量，后来慢慢演变出现CommonJS和ES6,两种解决方案；

### 写法不同
#### CommonJS
CommonJS 规范是为了解决 JavaScript 的作用域问题而定义的模块形式，可以使每个模块它自身的命名空间中执行。该规范的主要内容是，模块必须通过 `module.exports` 导出对外的变量或接口，通过 `require()` 来导入其他模块的输出到当前模块作用域中

CommonJS规范规定，每个模块内部，`module`变量代表当前模块。这个变量是一个对象，它的`exports`属性（即`module.exports`）是对外的接口。加载某个模块，其实是加载该模块的`module.exports`属性。

**特点**
*   所有代码都运行在模块作用域，不会污染全局作用域。
*   模块可以多次加载，但是只会在第一次加载时运行一次，然后运行结果就被缓存了，以后再加载，就直接读取缓存结果。要想让模块再次运行，必须清除缓存。
*   模块加载的顺序，按照其在代码中出现的顺序。

**导出**
```JavaScript
    var x = 5;
    var addX = function (value) {
      return value + x;
    };
    module.exports.x = x;
    module.exports.addX = addX;
```
**导入**
```JavaScript
var example = require('./example.js');
console.log(example.x); // 5
console.log(example.addX(1)); // 6
```
**在module中有个默认exports属性，来指向module.exports**

等同于
```javaScript
var exports = module.exports;
```
造成的结果是，在对外输出模块接口时，可以向exports对象添加方法。
```
exports.area = function (r) {
  return Math.PI * r * r;
};
exports.circumference = function (r) {
  return 2 * Math.PI * r;
};
```
注意，不能直接将exports变量指向一个值，因为这样等于切断了`exports`与`module.exports`的联系。
```
exports = function(x) {console.log(x)};
```
上面这样的写法是无效的，因为`exports`不再指向`module.exports`了。
下面的写法也是无效的。
```
exports.hello = function() {
  return 'hello';
};
module.exports = 'Hello world';
```
上面代码中，`hello`函数是无法对外输出的，因为`module.exports`被重新赋值了。

这意味着，如果一个模块的对外接口，就是一个单一的值，不能使用`exports`输出，只能使用`module.exports`输出。
```
module.exports = function (x){ console.log(x);};
```

#### ES6
import+export写法
```JavaScript
export { name1, name2, …, nameN };
export { variable1 as name1, variable2 as name2, …, nameN };
export let name1, name2, …, nameN; // also var, const
export let name1 = …, name2 = …, …, nameN; // also var, const
export function FunctionName(){...}
export class ClassName {...}

export default expression;
export default function (…) { … } // also class, function*
export default function name1(…) { … } // also class, function*
export { name1 as default, … };

export * from …;
export { name1, name2, …, nameN } from …;
export { import1 as name1, import2 as name2, …, nameN } from …;
export { default } from …;
```
`export default` 默认导出，类、函数、变量，一个module中只有一个默认导出

### 加载机制不同
CommonJS模块输出的是一个值的拷贝，而ES6模块输出的是一个值的引用，也就是说CommonJs一旦输出一个值，模块内部的变化就影响不到模块外部了；

而对于ES6来说，加载import时不会去执行，只是建立一个引用,只有到真正用到时候才会去对应的模块下取值，或者执行对应的逻辑，这种关系就保证了，当模块内的值发生变化的时候，模块外引用的地方也会发生变化

### 循环加载的机制不同
CommonJS中遇到循环加载的时候，只会执行到循环加载之前的语句，另外一个模块也只能获取到循环加载之前的数据，循环加载部分则不会执行

而对于ES6来说，由于只是持有引用，所有遇到这种情况，会采用递归的方式来完成循环加载