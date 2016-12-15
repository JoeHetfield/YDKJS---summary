# 值

## Array

在一个`array`值上使用`delete`将会从这个`array`上移除一个值槽，但就算你移除了最后一个元素，它也不会更新`length`属性。留下或创建空的/丢失的值槽称为“稀散”的`array`，虽然这样的值槽看起来拥有`undefined`值，但是它不会像被明确设置（`a[1] = undefined`）的值槽那样动作：

```js
var a = [ ];

a[0] = 1;
// 这里没有设置值槽`a[1]`
a[2] = [ 3 ];

a[1];		// undefined

a.length;	// 3
```

`array`是被数字索引的，但它们也是对象，可以在它们上面添加`string`键/属性（但是这些属性不会计算在`array`的`length`中）。然而，如果一个可以被强制转换为10进制`number`的`string`值被用作键的话，它会认为你想使用`number`索引而不是一个`string`键！

```js
var a = [ ];

a["13"] = 42;

a.length; // 14
```

### 类Array

各种DOM查询操作会返回一个DOM元素的列表，这些列表不是真正的`array`但是也足够类似`array`。函数为了像列表一样访问它的参数值，而暴露了`arugumens`对象（类`array`，在ES6中被废弃了）。将一个类`array`值转换为一个真正的`array`，很常见的方法是对这个值借用`slice(..)`工具：

```js
function foo() {
	var arr = Array.prototype.slice.call( arguments );
	arr.push( "bam" );
	console.log( arr );
}

foo( "bar", "baz" ); // ["bar","baz","bam"]
```

在ES6中，还有一种称为`Array.from(..)`的内建工具可以执行相同的任务：

```js
...
var arr = Array.from( arguments );
...
```

## String

String确实与`array`有很肤浅的相似性 -- 也就是上面说的，类`array` -- 它们都有一个`length`属性，一个`indexOf(..)`方法（在ES5中仅有`array`版本），和一个`concat(..)`方法：

```js
var a = "foo";
var b = ["f","o","o"];

a.length;							// 3
b.length;							// 3

a.indexOf( "o" );					// 1
b.indexOf( "o" );					// 1

var c = a.concat( "bar" );			// "foobar"
var d = b.concat( ["b","a","r"] );	// ["f","o","o","b","a","r"]

a === c;							// false
b === d;							// false

a;									// "foo"
b;									// ["f","o","o"]
```

JavaScript的`string`是不可变的，而`array`是相当可变的。在JavaScript中用位置访问字符的`a[1]`形式不总是广泛合法的。老版本的IE就不允许这种语法（但是它们现在允许了）。相反正确的方式是`a.charAt(1)`。`string`不可变性的进一步的后果是，`string`上没有一个方法是可以原地修改它的内容的，而是创建并返回一个新的`string`。与之相对的是，许多改变`array`内容的方法实际上是原地修改的：

```js
c = a.toUpperCase();
a === c;	// false
a;			// "foo"
c;			// "FOO"

b.push( "!" );
b;			// ["f","O","o","!"]
```

许多`array`方法在处理`string`时非常有用，虽然这些方法不属于`string`，但我们可以对我们的`string`“借用”非变化的`array`方法：

```js
a.join;			// undefined
a.map;			// undefined

var c = Array.prototype.join.call( a, "-" );
var d = Array.prototype.map.call( a, function(v){
	return v.toUpperCase() + ".";
} ).join( "" );

c;				// "f-o-o"
d;				// "F.O.O."
```

### 翻转一个`string`

`array`拥有一个原地的`reverse()`修改器方法，但是`string`没有：

```js
a.reverse;		// undefined

b.reverse();	// ["!","o","O","f"]
b;				// ["!","o","O","f"]
```

“借用”`array`修改器不起作用，因为`string`是不可变的，因此它不能被原地修改：

```js
Array.prototype.reverse.call( a );
// 仍然返回一个“foo”的String对象包装器（见第三章） :(
```

将`string`转换为一个`array`，实施我们想做的操作，然后将它转回`string`：

```js
var c = a
	// 将`a`切分成一个字符的数组
	.split( "" )
	// 翻转字符的数组
	.reverse()
	// 将字符的数组连接回一个字符串
	.join( "" );

c; // "oof"
```

这种方法对含有复杂（unicode）字符（星号，多字节字符等）的`string`不起作用。你需要支持unicode的更精巧的工具库来准确地处理这种操作。

## Number

在JS中，一个“整数”只是一个没有小数部分的小数值。也就是说，`42.0`和`42`一样是“整数”。JavaScript的`number`的实现基于“IEEE 754”标准，通常被称为“浮点”。JavaScript明确地使用了这个标准的“双精度”（也就是“64位二进制”）格式。

### 语法

小数的整数部分如果是`0`，是可选的：

```js
var a = 0.42;
var b = .42;
```

小数在`.`之后的小数部分如果是`0`，是可选的：

```js
var a = 42.0;
var b = 42.;
```

默认情况下，大多数`number`将会以10进制小数的形式输出，并去掉末尾小数部分的`0`：

```js
var a = 42.300;
var b = 42.0;

a; // 42.3
b; // 42
```

非常大或非常小的`number`将默认以指数形式输出，与`toExponential()`方法的输出一样。

因为`number`值可以用`Number`对象包装器封装，`number`值可以访问内建在`Number.prototype`上的方法。`toFixed(..)`方法允许你指定一个值在被表示时，带有多少位小数。它的输出实际上是一个`number`的`string`表现形式，而且如果你指定的位数多于值持有的小数位数时，会在右侧补`0`：

```js
var a = 42.59;

a.toFixed( 0 ); // "43"
a.toFixed( 1 ); // "42.6"
a.toFixed( 2 ); // "42.59"
a.toFixed( 3 ); // "42.590"
a.toFixed( 4 ); // "42.5900"
```

`toPrecision(..)`很相似，但它指定的是有多少有效数字用来表示这个值：

```js
var a = 42.59;

a.toPrecision( 1 ); // "4e+1"
a.toPrecision( 2 ); // "43"
a.toPrecision( 3 ); // "42.6"
a.toPrecision( 4 ); // "42.59"
a.toPrecision( 5 ); // "42.590"
a.toPrecision( 6 ); // "42.5900"
```

但你不得不小心`.`操作符。因为`.`是一个合法数字字符，如果可能的话，它会首先被翻译为`number`字面的一部分，而不是被翻译为属性访问操作符。

`number`还可以使用科学计数法的形式指定，这在表示很大的`number`时很常见：

```js
var onethousand = 1E3;						// 代表 1 * 10^3
var onemilliononehundredthousand = 1.1E6;	// 代表 1.1 * 10^6
```

`number`字面量还可以使用其他进制表达，比如二进制，八进制，和十六进制：

```js
0xf3; // 十六进制的: 243
0Xf3; // 同上

0363; // 八进制的: 243
```

从ES6 + `strict`模式开始，不再允许`0363`这样的的八进制形式。至于ES6，下面的新形式也是合法的：

```js
0o363;		// 八进制的: 243
0O363;		// 同上

0b11110011;	// 二进制的: 243
0B11110011; // 同上
```

### 小数值

使用二进制浮点数的最出名（臭名昭著）的副作用是：

```js
0.1 + 0.2 === 0.3; // false
```

`0.1`和`0.2`的二进制表示形式是不精确的，所以它们相加时，结果不是精确地`0.3`。而是非常接近的值：`0.30000000000000004`。最常见的做法是使用一个很小的“错误舍入”值作为比较的容差。这个很小的值经常被称为“机械极小值（machine epsilon）”，对于JavaScript来说这种`number`通常为`2^-52`。在ES6中，使用这个容差值预定义了`Number.EPSILON`，你也可以在前ES6中安全地填补这个定义：

```js
if (!Number.EPSILON) {
	Number.EPSILON = Math.pow(2,-52);
}
```

我们可以使用这个`Number.EPSILON`来比较两个`number`的“等价性”：

```js
function numbersCloseEnoughToEqual(n1,n2) {
	return Math.abs( n1 - n2 ) < Number.EPSILON;
}

var a = 0.1 + 0.2;
var b = 0.3;

numbersCloseEnoughToEqual( a, b );					// true
numbersCloseEnoughToEqual( 0.0000001, 0.0000002 );	// false
```

可以被表示的最大的浮点值大概是`1.798e+308`，它预定义为`Number.MAX_VALUE`。在极小的一端，`Number.MIN_VALUE`大概是`5e-324`，他不是负数但是非常接近于0。

### 安全整数范围

可以“安全地”被表示的最大整数是`2^53 - 1`。在ES6中这个值预定义为`Number.MAX_SAFE_INTEGER`。还有一个最小值，它在ES6中定义为`Number.MIN_SAFE_INTEGER`。

### 测试整数

测试一个值是否是整数，你可以使用ES6定义的`Number.isInteger(..)`：

```js
Number.isInteger( 42 );		// true
Number.isInteger( 42.000 );	// true
Number.isInteger( 42.3 );	// false
```

可以为前ES6填补`Number.isInteger(..)`：

```js
if (!Number.isInteger) {
	Number.isInteger = function(num) {
		return typeof num == "number" && num % 1 == 0;
	};
}
```

要测试一个值是否是安全整数，使用ES6定义的`Number.isSafeInteger(..)`：

```js
Number.isSafeInteger( Number.MAX_SAFE_INTEGER );	// true
Number.isSafeInteger( Math.pow( 2, 53 ) );			// false
Number.isSafeInteger( Math.pow( 2, 53 ) - 1 );		// true
```

可以为前ES6浏览器填补`Number.isSafeInteger(..)`：

```js
if (!Number.isSafeInteger) {
	Number.isSafeInteger = function(num) {
		return Number.isInteger( num ) &&
			Math.abs( num ) <= Number.MAX_SAFE_INTEGER;
	};
}
```

### 32位（有符号）整数

虽然整数可以安全地最大达到约9万亿（53比特），但有一些数字操作（比如位操作符）是仅仅为32位`number`定义的，所以对于被这样使用的`number`来说，“安全范围”一定会小得多。这个范围是从`Math.pow(-2,31)`到`Math.pow(2,31)-1`（正负21亿）。要强制`a`中的`number`值是32位有符号整数，使用`a | 0`。这可以工作是因为`|`位操作符仅仅对32位值起作用（意味着它可以只关注32位，而其他的位将被丢掉）。而且，和0进行“或”的位操作实质上是什么也不做。

## 特殊值

### 不是值的值

`null`是一个特殊的关键字，不是一个标识符，因此你不能将它作为一个变量对待来给它赋值。然而，`undefined`是一个标识符。

### Undefined

在非`strict`模式下，给在全局上提供的`undefined`标识符赋一个值实际上是可能的（虽然这是一个非常不好的做法！）。在非`strict`模式和`strict`模式下，你可以创建一个名叫`undefined`局部变量。但这又是一个很差劲儿的主意。

#### `void`操作符

表达式`void __`会“躲开”任何值，所以这个表达式的结果总是值`undefined`。它不会修改任何已经存在的值；只是确保不会有值从操作符表达式中返回来：

```js
var a = 42;

console.log( void a, a ); // undefined 42
```

从惯例上讲（大约是从C语言编程中发展而来），要通过使用`void`来独立表现值`undefined`，你可以使用`void 0`。在`void 0`，`void 1`和`undefined`之间没有实际上的区别。如果你需要确保一个表达式没有结果值（即便它有副作用）`void`操作符可以十分有用：

```js
function doSomething() {
	// 注意：`APP.ready`是由我们的应用程序提供的
	if (!APP.ready) {
		// 稍后再试一次
		return void setTimeout( doSomething, 100 );
	}

	var result;

	// 做其他一些事情
	return result;
}

// 我们能立即执行吗？
if (doSomething()) {
	// 马上处理其他任务
}
```

### 特殊的数字

#### 不是数字的数字

如果你不使用同为`number`（或者可以被翻译为10进制或16进制的普通`number`的值）的两个操作数进行任何算数操作，那么操作的结果将失败而产生一个不合法的`number`，在这种情况下你将得到`NaN`值。不是一个数字’的类型是‘数字’：

```js
var a = 2 / "foo";		// NaN

typeof a === "number";	// true
```

`NaN`是一种“哨兵值”（一个被赋予了特殊意义的普通的值），它代表`number`集合内的一种特殊的错误情况：“我试着进行数学操作但是失败了，而这就是失败的`number`结果。”`NaN`是一个非常特殊的值，它从来不会等于另一个`NaN`值（也就是，它从来不等于它自己）。实际上，它是唯一一个不具有反射性的值（没有恒等性`x === x`）。所以，`NaN !== NaN`。

使用称为`isNaN(..)`的内建全局工具，它告诉我们这个值是否是`NaN`：

```js
var a = 2 / "foo";

isNaN( a ); // true
```

`isNaN(..)`工具有一个重大缺陷：

```js
var a = 2 / "foo";
var b = "foo";

a; // NaN
b; // "foo"

window.isNaN( a ); // true
window.isNaN( b ); // true -- 噢!
```

`"foo"`根本不是一个`number`，但它也绝不是一个`NaN`值！这个bug从最开始的时候就存在于JS中了。在ES6中，终于提供了一个替代它的工具：`Number.isNaN(..)`。有一个简单的填补，可以让你即使是在前ES6的浏览器中安全地检查`NaN`值：

```js
if (!Number.isNaN) {
	Number.isNaN = function(n) {
		return (
			typeof n === "number" &&
			window.isNaN( n )
		);
	};
}

var a = 2 / "foo";
var b = "foo";

Number.isNaN( a ); // true
Number.isNaN( b ); // false -- 咻!
```

通过利用`NaN`与它自己不相等这个特殊的事实，我们可以更简单地实现`Number.isNaN(..)`的填补。在整个语言中`NaN`是唯一一个这样的值；其他的值都总是等于它自己：

```js
if (!Number.isNaN) {
	Number.isNaN = function(n) {
		return n !== n;
	};
}
```

#### 无穷

在JS中，“除数为0”是明确定义的，而且它的结果是值`Infinity`（也就是`Number.POSITIVE_INFINITY`），`-Infinity`（也就是`Number.NEGATIVE_INFINITY`）。`Infinity / Infinity`不是一个有定义的操作，它的结果为`NaN`。一个有限的正`number`除以`Infinity`是`0`。

#### 零

JavaScript拥有普通的零`0`（也称为正零`+0`）和一个负零`-0`。除了使用字面量`-0`指定，负的零还可以从特定的数学操作中得出：

```js
var a = 0 / -3; // -0
var b = 0 * -3; // -0
```

加法和减法无法得出负零。

但是根据语言规范，如果你试着将一个负零转换为字符串，它将总会被报告为`"0"`：

```js
var a = 0 / -3;

// 至少（有些浏览器）控制台是对的
a;							// -0

// 但是语言规范坚持要向你撒谎！
a.toString();				// "0"
a + "";						// "0"
String( a );				// "0"

// 奇怪的是，就连JSON也加入了骗局之中
JSON.stringify( a );		// "0"
```

有趣的是，反向操作（从`string`到`number`）不会撒谎：

```js
+"-0";				// -0
Number( "-0" );		// -0
JSON.parse( "-0" );	// -0
```

除了一个负零的字符串化会欺骗性地隐藏它实际的值外，比较操作符也被设定为（有意地）要说谎：

```js
var a = 0;
var b = 0 / -3;

a == b;		// true
-0 == 0;	// true

a === b;	// true
-0 === 0;	// true

0 > -0;		// false
a > b;		// false
```

区分`-0`和`0`：

```js
function isNegZero(n) {
	n = Number( n );
	return (n === 0) && (1 / n === -Infinity);
}

isNegZero( -0 );		// true
isNegZero( 0 / -3 );	// true
isNegZero( 0 );			// false
```

#### 特殊等价

在ES6中，有一个新工具可以用于测试两个值的绝对等价性，而没有任何这些例外。它称为`Object.is(..)`：

```js
var a = 2 / "foo";
var b = -3 * 0;

Object.is( a, NaN );	// true
Object.is( b, -0 );		// true

Object.is( b, 0 );		// false
```

对于前ES6环境，这是一个相当简单的`Object.is(..)`填补：

```js
if (!Object.is) {
	Object.is = function(v1, v2) {
		// 测试 `-0`
		if (v1 === 0 && v2 === 0) {
			return 1 / v1 === 1 / v2;
		}
		// 测试 `NaN`
		if (v1 !== v1) {
			return v2 !== v2;
		}
		// 其他情况
		return v1 === v2;
	};
}
```

`Object.is(..)`可能不应当用于那些`==`或`===`已知安全的情况，因为这些操作符可能高效得多，并且更惯用/常见。`Object.is(..)`很大程度上是为这些特殊的等价情况准备的。
