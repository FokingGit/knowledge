### JS数据类型

js有五种基本数据类型和一种复杂数据类型Object

- undefine
- Null
- Boolean
- Number
- String

检测某个变量的类型可以使用`typeof`操作符进行判断(**返回值均为小写**)

| 返回值     | 含义                   |
| ---------- | ---------------------- |
| "undefine" | 如果这个值未定义       |
| "boolean"  | 如果这个值是布尔值     |
| "string"   | 如果这个值是字符串     |
| "number"   | 如果这个值是数值       |
| “object”   | 如果这个值是对象或null |
| "function" | 如果这个值是函数       |

#### undefined

undefined类型只有一个值就是undefined,变量声明之后未对其初始化时，这个变量的值就是undefined

对于未声明的变量不能使用的，只能进行一项操作就是typeof，这时候返回的值也是undefine，所以针对未声明的变量和声明了但是没有定义的变量进行typeof操作结果都为undefine

#### null

null值表示一个空对象指针，如果定义的变量准备在将来用于保存对象，那么最好将变量的初始化为null而不是其他值

#### Boolean

true和false,这两个值和数字值不是一回事，因此true不一定等于1，而false也不一定等于0，其他类型转换为Boolean的结果

| 数据类型 | 转换为true的值 | 转换为false的值 |
| -------- | -------------- | --------------- |
| Boolean  | ture           | False           |
| String   | 任何非空字符串 | “”(空字符串)    |
| Number   | 任何非0数值    | 0和NaN          |
| Object   | 任何对象       | null            |
| Undefine | n/a            | undefined       |

#### Number类型

- 使用IEEE754格式来表示整数和浮点数值
- 在进行算术计算时，所有以8进制和16进制表示的数值最终都将被转换成10进制数值

##### 1. 浮点数值

- 保存浮点数值需要的内存空间是保存整数值的两倍，如果小数点之后没有数值或者浮点数值本身就是一个整数，那么该值会被转换为整数来存储
- todo

#### Object类型