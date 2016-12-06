# 原生类型

# 强制转换

## 转换值

将一个值从一个类型明确地转换到另一个类型通常称为“类型转换（type casting）”，当这个操作隐含地完成时称为“强制转换（coercion）”（根据一个值如何被使用的规则来强制它变换类型）。“类型转换（type casting/conversion）”发生在静态类型语言的编译时，而“类型强制转换（type coercion）”是动态类型语言的运行时转换。JavaScript强制转换总是得到基本标量值的一种，比如`string`，`number`，或`boolean`。没有强制转换可以得到像`object`和`function`这样的复杂值。

## 抽象值操作

### `ToString`

当任何一个非`string`值被强制转换为一个`string`表现形式时，这个转换的过程是由`ToString`抽象操作处理的。

1. 内建的基本类型值拥有自然的字符串化形式：`null`变为`"null"`，`undefined`变为`"undefined"`，`true`变为`"true"`。`number`一般会以你期望的自然方式表达，非常小或非常大的`number`将会以指数形式表达。

2. 如果一个对象上拥有它自己的`toString()`方法，而你又以一种类似`string`的方式使用这个对象，那么它的`toString()`将会被自动调用，而且这个调用的`string`结果将被使用。

3. 对于普通的对象，除非你指定你自己的，默认的`toString()`（可以在`Object.prototype.toString()`找到）将返回内部`[[Class]]`，例如`"[object Object]"`。

4. 数组拥有一个覆盖版本的默认`toString()`，将数组字符串化为它所有的值（每个都字符串化）的（字符串）连接，并用`","`分割每个值

`toString()`可以明确地被调用，也可以通过在一个需要`string`的上下文环境中使用一个非`string`来自动地被调用。

#### JSON字符串化

任何JSON安全的值都可以被`JSON.stringify(..)`字符串化。JSON安全的值就是任何可以用JSON表现形式合法表达的值。考虑JSON不安全的值可能更容易一些。一些例子是：`undefined`，`function`，（ES6+）`symbol`，和带有循环引用的`object`。对于标准的JSON结构来说这些都是非法的值，主要是因为他们不能移植到消费JSON值的其他语言中。`JSON.stringify(..)`工具在遇到`undefined`，`function`，和`symbol`时将会自动地忽略它们。如果在一个`array`中遇到这样的值，它会被替换为`null`（这样数组的位置信息就不会改变）。如果在一个`object`的属性中遇到这样的值，这个属性会被简单地剔除掉。如果你试着`JSON.stringify(..)`一个带有循环引用的`object`，就会抛出一个错误。

```js
JSON.stringify( undefined );					// undefined
JSON.stringify( function(){} );					// undefined

JSON.stringify( [1,undefined,function(){},4] );	// "[1,null,null,4]"
JSON.stringify( { a:2, b:function(){} } );		// "{"a":2}"
```

JSON字符串化有一个特殊行为，如果一个`object`值定义了一个`toJSON()`方法，这个方法将会被首先调用，以取得用于序列化的值。一个很常见的误解是，`toJSON()`应当返回一个JSON字符串化的表现形式。`toJSON()`应当返回合适的实际普通值（无论什么类型），而`JSON.stringify(..)`自己会处理字符串化。

```js
var a = {
	val: [1,2,3],

	// 可能正确！
	toJSON: function(){
		return this.val.slice( 1 );
	}
};

var b = {
	val: [1,2,3],

	// 可能不正确！
	toJSON: function(){
		return "[" +
			this.val.slice( 1 ).join() +
		"]";
	}
};

JSON.stringify( a ); // "[2,3]"

JSON.stringify( b ); // ""[2,3]""
```

`JSON.stringify(..)`的第二个参数值是可选的，它称为替换器（replacer）。这个参数值既可以是一个`array`也可以是一个`function`。与`toJSON()`为序列化准备一个值的方式类似，它提供一种过滤机制，指出一个`object`的哪一个属性应该或不应该被包含在序列化形式中，来自定义这个`object`的递归序列化行为

```js
var a = {
	b: 42,
	c: "42",
	d: [1,2,3]
};

JSON.stringify( a, ["b","c"] ); // "{"b":42,"c":"42"}"

JSON.stringify( a, function(k,v){
	if (k !== "c") return v;
} );
// "{"b":42,"d":[1,2,3]}"
```

1. 如果替换器是一个`array`，那么它应当是一个`string`的`array`，它的每一个元素指定了允许被包含在这个`object`的序列化形式中的属性名称。如果一个属性不存在于这个列表中，那么它就会被跳过。

2. 如果替换器是一个`function`，那么它会为`object`本身而被调用一次，并且为这个`object`中的每个属性都被调用一次，而且每次都被传入两个参数值，key 和 value。要在序列化中跳过一个 key，可以返回`undefined`。否则，就返回被提供的 value。

3. 在`function`替换器的情况下，第一次调用时key参数`k`是`undefined`（而对象`a`本身会被传入）。`if`语句会过滤掉名称为`c`的属性。字符串化是递归的，所以数组`[1,2,3]`会将它的每一个值（`1`，`2`，和`3`）都作为`v`传递给替换器，并将索引值（`0`，`1`，和`2`）作为`k`。

`JSON.stringify(..)`还可以接收第三个可选参数值，称为填充符（space），在对人类友好的输出中它被用做缩进。填充符可以是一个正整数，用来指示每一级缩进中应当使用多少个空格字符。或者，填充符可以是一个`string`，这时每一级缩进将会使用它的前十个字符

```js
var a = {
	b: 42,
	c: "42",
	d: [1,2,3]
};

JSON.stringify( a, null, 3 );
// "{
//    "b": 42,
//    "c": "42",
//    "d": [
//       1,
//       2,
//       3
//    ]
// }"

JSON.stringify( a, null, "-----" );
// "{
// -----"b": 42,
// -----"c": "42",
// -----"d": [
// ----------1,
// ----------2,
// ----------3
// -----]
// }"
```

`string`，`number`，`boolean`，和`null`值在JSON字符串化时，与它们通过`ToString`抽象操作的规则强制转换为`string`值的方式基本上是相同的。如果传递一个`object`值给`JSON.stringify(..)`，而这个`object`上拥有一个`toJSON()`方法，那么在字符串化之前，`toJSON()`就会被自动调用来将这个值（某种意义上）“强制转换”为JSON安全的。

### `ToNumber`

如果任何非`number`值，以一种要求它是`number`的方式被使用，比如数学操作，就会发生`ToNumber`抽象操作。

1. `true`变为`1`而`false`变为`0`。`undefined`变为`NaN`，而（奇怪的是）`null`变为`0`。

2. 对于一个`string`值来说，`ToNumber`工作起来很大程度上与数字字面量的规则/语法很相似。如果它失败了，结果将是`NaN`（而不是`number`字面量中会出现的语法错误）。一个不同之处的例子是，在这个操作中`0`前缀的八进制数不会被作为八进制数来处理（而仅作为普通的十进制小数），虽然这样的八进制数作为`number`字面量是合法的。

3. 对象（以及数组）将会首先被转换为它们的基本类型值的等价物，而后这个结果值（如果它还不是一个`number`基本类型）会根据刚才提到的`ToNumber`规则被强制转换为一个`number`。为了转换为基本类型值的等价物，`ToPrimitive`抽象操作将会查询这个值，看它有没有`valueOf()`方法。如果`valueOf()`可用并且它返回一个基本类型值，那么这个值就将用于强制转换。如果不是这样，但`toString()`可用，那么就由它来提供用于强制转换的值。如果这两种操作都没提供一个基本类型值，就会抛出一个`TypeError`。

```js
var a = {
	valueOf: function(){
		return "42";
	}
};

var b = {
	toString: function(){
		return "42";
	}
};

var c = [4,2];
c.toString = function(){
	return this.join( "" );	// "42"
};

Number( a );			// 42
Number( b );			// 42
Number( c );			// 42
Number( "" );			// 0
Number( [] );			// 0
Number( [ "abc" ] );	// NaN
```

在ES5中，你可以创建这样一个不可强制转换的对象 —— 没有`valueOf()`和`toString()` —— 如果他的`[[Prototype]]`的值为`null`，这通常是通过`Object.create(null)`来创建的。

### `ToBoolean`

“falsy”值列表，如果一个值在这个列表中，它就是一个“falsy”值，而且当你在它上面进行`boolean`强制转换时它会转换为`false`。任何没有明确地存在于falsy列表中的东西，都是truthy。

* `undefined`
* `null`
* `false`
* `+0`, `-0`, and `NaN`
* `""`

#### Falsy对象

不是包装着falsy值的对象。看起来和动起来都像一个普通对象（属性，等等）的值，但是当你强制转换它为一个`boolean`时，它会变为一个`false`值。

`document.all`本身从来就不是“标准的”，而且从很早以前就被废弃/抛弃了。因为从`document.all`到`boolean`的强制转换（比如在`if`语句中）几乎总是用来检测老的，非标准的IE。老旧的`if (document.all) { /* it's IE */ }`代码依然留在世面上，而且大多数可能永远都不会消失。所以不能完全移除`document.all`，但是IE不再想让`if (document.all) { .. }`代码继续工作了，这样现代IE的用户就能得到新的，符合标准的代码逻辑。于是黑进JS的类型系统并假装`document.all`是falsy！”

## 明确的强制转换

### 明确地：Strings <--> Numbers

为了在`string`和`number`之间进行强制转换，我们使用内建的`String(..)`和`Number(..)`函数，但非常重要的是，我们不在它们前面使用`new`关键字。这样，我们就不是在创建对象包装器。取而代之的是，我们实际上在两种类型之间进行明确地强制转换。

```js
var a = 42;
var b = String( a );

var c = "3.14";
var d = Number( c );

b; // "42"
d; // 3.14
```

`String(..)`使用早先讨论的`ToString`操作的规则，将任意其它的值强制转换为一个基本类型的`string`值。`Number(..)`使用早先讨论过的`ToNumber`操作的规则，将任意其他的值强制转换为一个基本类型的`number`值。

除了`String(..)`和`Number(..)`，还有其他的方法可以把这些值在`string`和`number`之间进行“明确地”转换。

```js
var a = 42;
var b = a.toString();

var c = "3.14";
var d = +c;

b; // "42"
d; // 3.14
```

1. 调用`a.toString()`在表面上是明确的，但是这里有一些藏起来的隐含性。`toString()`不能在像`42`这样的基本类型值上调用。所以JS会自动地将`42`“封箱”在一个对象包装器中，这样`toString()`就可以针对这个对象调用。

2. 这里的`+c`是`+`操作符的一元操作符（操作符只有一个操作数）形式。取代进行数学加法的是，一元的`+`明确地将它的操作数（`c`）强制转换为一个`number`值。

3. 一元`-`操作符也像`+`一样进行强制转换，但它还会翻转数字的符号。但是你不能放两个减号`--`来使符号翻转回来，因为那将被解释为递减操作符。取代它的是，你需要这么做：`- -"3.14"`，在两个减号之间加入空格，这将会使强制转换的结果为`3.14`。

#### 从`Date`到`number`

另一个一元`+`操作符的常见用法是将一个`Date`对象强制转换为一个`number`，其结果是这个日期/时间值的unix时间戳（从世界协调时间的1970年1月1日0点开始计算，经过的毫秒数）表现形式。这种习惯性用法经常用于取得当前的现在时刻的时间戳。

```js
var timestamp = +new Date();

var d = new Date( "Mon, 18 Aug 2014 08:53:06 CDT" );

+d; // 1408369986000
```

一些开发者知道一个JavaScript中的特别的语法“技巧”，就是在构造器调用（一个带有`new`的函数调用）中如果没有参数值要传递的话，`()`是可选的。所以你可能遇到`var timestamp = +new Date;`形式。然而，不是所有的开发者都同意忽略`()`可以增强可读性，因为它是一种不寻常的语法特例，只能适用于`new fn()`调用形式，而不能用于普通的`fn()`调用形式。

一个不使用强制转换的方式可能更好，因为它更加明确。

```js
var timestamp = new Date().getTime();
// var timestamp = (new Date()).getTime();
// var timestamp = (new Date).getTime();
```

一个更更好的不使用强制转换的选择是使用ES5加入的`Date.now()`静态函数。

#### 奇异的`~`

JS中位操作符是如何仅为32位操作定义的，这意味着我们强制它们的操作数遵循32位值的表现形式。这个规则如何发生是由`ToInt32`抽象操作控制的。

`ToInt32`首先进行`ToNumber`强制转换，这就是说如果值是`"123"`，它在`ToInt32`规则实施之前会首先变成`123`。虽然它本身没有技术上进行强制转换（因为类型没有改变），但对一些特定的特殊`number`值使用位操作符（比如`|`或`~`）会产生一种强制转换效果，这种效果的结果是一个不同的`number`值。

```js
0 | -0;			// 0
0 | NaN;		// 0
0 | Infinity;	// 0
0 | -Infinity;	// 0
```

这些特殊的数字是不可用32位表现的，所以`ToInt32`将这些值的结果指定位`0`。

` ~`操作符首先将值“强制转换”为一个32位`number`值，然后实施按位取反（翻转每一个比特位），`~x`大致与`-(x+1)`相同。

通过进行这个操作，能够产生结果`0`（或者从技术上说`-0`！）的唯一的值是什么？`-1`。换句话说，`~`用于一个范围的`number`值时，将会为输入值`-1`产生一个falsy（很容易强制转换为`false`）的`0`，而为任意其他的输入产生truthy的`number`。

JavaScript在定义`string`的`indexOf(..)`搜索一个子字符串，如果找到就返回它从0开始计算的索引位置，没有找到的话就返回`-1`。

```js
var a = "Hello World";

if (a.indexOf( "lo" ) >= 0) {	// true
	// 找到了！
}
if (a.indexOf( "lo" ) != -1) {	// true
	// 找到了
}

if (a.indexOf( "ol" ) < 0) {	// true
	// 没找到！
}
if (a.indexOf( "ol" ) == -1) {	// true
	// 没找到！
}
```

它基本上是一种“抽象泄漏”，这里它将底层的实现行为 —— 使用哨兵值`-1`表示“失败” —— 泄漏到我的代码中。将`~`和`indexOf()`一起使用可以将值“强制转换”（实际上只是变形）为可以适当地强制转换为`boolean`的值。

```js
var a = "Hello World";

~a.indexOf( "lo" );			// -4   <-- truthy!

if (~a.indexOf( "lo" )) {	// true
	// 找到了！
}

~a.indexOf( "ol" );			// 0    <-- falsy!
!~a.indexOf( "ol" );		// true

if (!~a.indexOf( "ol" )) {	// true
	// 没找到！
}
```

##### 截断比特位

使用双波浪线`~~`来截断一个`number`的小数部分（也就是，将它“强制转换”为一个“整数”）。`~~`的工作方式是，第一个`~`实施`ToInt32`“强制转换”并进行按位取反，然后第二个`~`进行另一次按位取反，将每一个比特位都翻转回原来的状态。于是最终的结果就是`ToInt32`“强制转换”（也叫截断）。它仅在32位值上可以可靠地工作。但更重要的是，它在负数上工作的方式与`Math.floor(..)`不同！

```js
Math.floor( -49.6 );	// -50
~~-49.6;				// -49
```

` ~~x`可以将值截断为一个（32位）整数。但是`x | 0`也可以，而且看起来还（稍微）省事儿一些。那么，为什么你可能会选择`~~x`而不是`x | 0`？操作符优先权。

```js
~~1E20 / 10;		// 166199296

1E20 | 0 / 10;		// 1661992960
(1E20 | 0) / 10;	// 166199296
```

### 明确地：解析数字字符串

从`string`的字符内容中解析（parsing）出一个`number`：

```js
var a = "42";
var b = "42px";

Number( a );	// 42
parseInt( a );	// 42

Number( b );	// NaN
parseInt( b );	// 42
```

从一个字符串中解析出一个数字是容忍非数字字符的 —— 从左到右，如果遇到非数字字符就停止解析 —— 而强制转换是不容忍并且会失败而得出值`NaN`。

解析不应当被视为强制转换的替代品。这两种任务虽然相似，但是有着不同的目的。当你不知道/不关心右手边可能有什么其他的非数字字符时，你可以将一个`string`作为`number`解析。当只有数字才是可接受的值，而且像`"42px"`这样的东西作为数字应当被排除时，就强制转换一个`string`（变为一个`number`）。

不要忘了`parseInt(..)`工作在`string`值上。向`parseInt(..)`传递一个`number`绝对没有任何意义。传递其他任何类型也都没有意义，如果你传入一个非`string`，你所传入的值首先将自动地被强制转换为一个`string`（见早先的“`ToString`”），这很明显是一种隐藏的隐含强制转换。在你的程序中依赖这样的行为真的是一个坏主意，所以永远也不要将`parseInt(..)`与非`string`值一起使用。

在ES5之前，`parseInt(..)`还存在另外一个坑，这曾是许多JS程序的bug的根源。如果你不传递第二个参数来指定使用哪种进制（也叫基数）来翻译数字的`string`内容，`parseInt(..)`将会根据开头的字符进行猜测。如果开头的两个字符是`"0x"`或`"0X"`，那么猜测（根据惯例）将是你想要将这个`string`翻译为一个16进制`number`。否则，如果第一个字符是`"0"`，那么猜测（也是根据惯例）将是你想要将这个`string`翻译成8进制`number`。

```js
var hour = parseInt( selectedHour.value );
var minute = parseInt( selectedMinute.value );

console.log( "The time you selected was: " + hour + ":" + minute);
```

试着在小时上选择`08`在分钟上选择`09`。你会得到`0:0`。为什么？因为`8`和`9`都不是合法的8进制数。ES5之前的修改很简单，但是很容易忘：总是在第二个参数值上传递`10`。在ES5中，`parseInt(..)`不再猜测八进制数了。除非你指定，否则它会假定为10进制（或者为`"0x"`前缀猜测16进制数）。

解析非字符串

```js
parseInt( 1/0, 19 ); // 18
```

回到我们的`parseInt( 1/0, 19 )`例子。它实质上是`parseInt( "Infinity", 19 )`。它如何解析？第一个字符是`"I"`，在愚蠢的19进制中是值`18`。第二个字符`"n"`不再合法的数字字符集内，所以这样的解析就礼貌地停止了，就像它在`"42px"`中遇到`"p"`那样。

```js
parseInt( 0.000008 );		// 0   ("0" from "0.000008")
parseInt( 0.0000008 );		// 8   ("8" from "8e-7")
parseInt( false, 16 );		// 250 ("fa" from "false")
parseInt( parseInt, 16 );	// 15  ("f" from "function..")

parseInt( "0x10" );			// 16
parseInt( "103", 2 );		// 2
```

### 明确地：* --> Boolean

`Boolean(..)`（当然，不带`new`！）是强制进行`ToBoolean`转换的明确方法，但是它并不常见也不为人所惯用。

```js
var a = "0";
var b = [];
var c = {};

var d = "";
var e = 0;
var f = null;
var g;

Boolean( a ); // true
Boolean( b ); // true
Boolean( c ); // true

Boolean( d ); // false
Boolean( e ); // false

Boolean( f ); // false
Boolean( g ); // false
```

正如一元`+`操作符将一个值强制转换为一个`number`，一元的`!`否定操作符可以将一个值明确地强制转换为一个`boolean`。问题是它还将值从truthy翻转为falsy，或反之。所以，大多数JS开发者使用`!!`双否定操作符进行`boolean`强制转换，因为第二个`!`将会把它翻转回原本的true或false。

```js
var a = "0";
var b = [];
var c = {};

var d = "";
var e = 0;
var f = null;
var g;

!!a;	// true
!!b;	// true
!!c;	// true

!!d;	// false
!!e;	// false
!!f;	// false
!!g;	// false
```

一个`ToBoolean`强制转换的用例是，如果你想在数据结构的JSON序列化中强制转换一个`true`/`false`。

```js
var a = [
	1,
	function(){ /*..*/ },
	2,
	function(){ /*..*/ }
];

JSON.stringify( a ); // "[1,null,2,null]"

JSON.stringify( a, function(key,val){
	if (typeof val == "function") {
		// 强制函数进行 `ToBoolean` 转换
		return !!val;
	}
	else {
		return val;
	}
} );
// "[1,true,2,true]"
```

```js
var a = 42;

var b = a ? true : false;
```

这里有一个隐藏的隐含强制转换，就是表达式`a`不得不首先被强制转换为`boolean`来进行真假测试。我称这种惯用法为“明确地隐含”。另外，我建议你在JavaScript中完全避免这种惯用法。它不会提供真正的好处，而且会让事情变得更糟。
