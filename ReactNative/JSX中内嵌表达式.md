### JSX中内嵌表达式
可以用花括号`{}`把任意的JavaScript表达式嵌入到JSX中。

### 表达式(Expression)与语句(Statement)的区别

#### 表达式(Expression)

> An expressionis a phrase of JavaScript that a JavaScript interpreter can evaluate to produce a value.

**表达式一定有返回值！**

除了**带运算符**的表达式外，还有以下几类：

*   `直接量`表达式: `1.7`,`Javscript`
*   `对象和数组初始化`表达式：`[]`,`[1,2,3]`,`{}`
*   `函数定义`表达式(**注意如何区分函数定义与函数表达式**)：`var foo = function () {}`;
*   `属性访问`表达式：`expression.identifier`，`expression[ expression]`
*   `函数调用`表达式：`Math.max(x,y,z)`
*   `创建对象`表达式: `new Object()`, `new Object`

#### 语句(Statement)

> Expressions are evaluatedto produce a value, but statements are executed to make something happen.

但有一些语句是由有副作用(side-effect)的表达式构成的，比如

*   赋值和函数调用表达式(expression statement)
*   声明新的变量和定义函数时(declaration statements)