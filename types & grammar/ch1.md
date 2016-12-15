# 类型

一个类型是一组固有的，内建的性质，对于引擎和开发者来说，它独一无二地标识了一个特定的值的行为，并将它与其他值区分开。

## 内建类型

7种内建类型，除了`object`都被称为“基本类型（primitives）”。

### `typeof`操作符

检测给定值的类型，而且总是返回7种字符串值中的一种：

```js
typeof undefined     === "undefined"; // true
typeof true          === "boolean";   // true
typeof 42            === "number";    // true
typeof "42"          === "string";    // true
typeof { life: 42 }  === "object";    // true

// 在ES6中被加入的！
typeof Symbol()      === "symbol";    // true
```

`null`是特殊的，它与`typeof`操作符组合时是有bug的：

```js
typeof null === "object"; // true
```

如果你想要使用`null`类型来测试`null`值，你需要一个复合条件：`null`是唯一一个“falsy”，但是在`typeof`检查中返回`"object"`的基本类型。

```js
var a = null;

(!a && typeof a === "object"); // true
```

可以返回的第7种字符串值：

```js
typeof function a(){ /* .. */ } === "function"; // true
```

很容易认为在JS中`function`是一种顶层的内建类型。它实际上是对象（object）的“子类型”。一个函数被称为“可调用对象” —— 一个拥有`[[Call]]`内部属性，允许被调用的对象。函数对象拥有一个`length`属性，它被设置为函数被声明时的正式参数的数量。

数组也是对象的“子类型”，带有被数字索引的附加性质，并维护着一个自动更新的`.length`属性：

```js
typeof [1,2,3] === "object"; // true
```

## 值作为类型

在JavaScript中，变量没有类型 -- 值才有类型。变量可以在任何时候，持有任何值。如果你对一个变量使用`typeof`，它不会像表面上看起来那样询问“这个变量的类型是什么？”，因为JS变量是没有类型的。实际上它会询问“在这个变量里的值的类型是什么？”

### `undefined` vs "undeclared"

当前还不拥有值的变量，实际上拥有`undefined`值。对这样的变量调用`typeof`将会返回`"undefined"`：

```js
var a;

typeof a; // "undefined"

var b = 42;
var c;

// 稍后
b = c;

typeof b; // "undefined"
typeof c; // "undefined"
```

一个“undefined”变量是在可访问的作用域中已经被声明过的，但是在这个时刻它里面没有任何值。相比之下，一个“undeclared”变量是在可访问的作用域中还没有被正式声明的。

```js
var a;

a; // undefined
b; // ReferenceError: b is not defined
```

一个恼人的困惑是浏览器给这种情形分配的错误消息。这个消息是“b is not defined”，这很容易而且很合理地使人将它与“b is undefined.”搞混。需要重申的是，“undefined”和“is not defined”是非常不同的东西。

`typeof`操作符甚至为“undeclared”变量返回`"undefined"`。当我们执行`typeof b`时，即使`b`是一个未声明变量，也不会有错误被抛出。这是`typeof`的一种特殊的安全防卫行为。

```js
var a;

typeof a; // "undefined"

typeof b; // "undefined"
```

`typeof` Undeclared在浏览器中处理JavaScript时这种安全防卫是一种有用的特性：

检查变量是否被声明过：

```js
// 噢，这将抛出一个错误！
if (DEBUG) {
	console.log( "Debugging is starting" );
}

// 这是一个安全的存在性检查
if (typeof DEBUG !== "undefined") {
	console.log( "Debugging is starting" );
}
```

为一个内建的API做特性检查，不带抛出错误的检查很有帮助：

```js
if (typeof atob === "undefined") {
	atob = function() { /*..*/ };
}
```

如果你在`if`语句内部声明`var atob`，即使这个`if`条件没有通过（因为全局的`atob`已经存在），这个声明也会被提升到作用域的顶端。忽略`var`可以防止这种提升声明。

另一种不带有`typeof`的安全防卫特性，而对全局变量进行这些检查的方法是，将所有的全局变量作为全局对象的属性来观察：

```js
if (window.DEBUG) {
	// ..
}

if (!window.atob) {
	// ..
}
```

和引用未声明变量不同的是，在你试着访问一个不存在的对象属性时（即便是在全局的`window`对象上），不会有`ReferenceError`被抛出。

检查包含它的程序是否已经定义了一个特定的变量：

```js
function doSomethingCool() {
	var helper =
		(typeof FeatureXYZ !== "undefined") ?
		FeatureXYZ :
		function() { /*.. 默认的特性 ..*/ };

	var val = helper();
	// ..
}
```

```js
// 一个IIFE（参见本系列的 *作用域与闭包* 中的“立即被调用的函数表达式”）
(function(){
	function FeatureXYZ() { /*.. my XYZ feature ..*/ }

	// 引入 `doSomethingCool(..)`
	function doSomethingCool() {
		var helper =
			(typeof FeatureXYZ !== "undefined") ?
			FeatureXYZ :
			function() { /*.. 默认的特性 ..*/ };

		var val = helper();
		// ..
	}

	doSomethingCool();
})();
```

这里，`FeatureXYZ`根本不是一个全局变量，但我们仍然使用`typeof`的安全防卫来使检查变得安全。而且重要的是，我们在这里没有可以用于检查的对象（就像我们使用`window.___`对全局变量做的那样），所以`typeof`十分有帮助。
