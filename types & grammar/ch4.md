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

## 隐含的强制转换

### 隐含地：Strings <--> Numbers

为了服务于`number`的相加和`string`的连接两个目的，`+`操作符被重载了。

```js
var a = [1,2];
var b = [3,4];

a + b; // "1,23,4"
```

如果`+`的两个操作数之一是一个`string`（或在上面的步骤中成为一个`string`），那么操作就会是`string`连接。否则，它总是数字加法。这个操作现在和`ToNumber`抽象操作处理`object`的过程是一样的。在`array`上的`valueOf()`操作将会在产生一个简单基本类型时失败，于是它退回到一个`toString()`表现形式。两个`array`因此分别变成了`"1,2"`和`"3,4"`。现在，`+`就如你通常期望的那样连接这两个`string`：`"1,23,4"`。

可以简单地通过将`number`和空`string``""`“相加”来把一个`number`强制转换为一个`string`。

```js
var a = 42;
var b = a + "";

b; // "42"
```

使用一个`+ ""`操作将`number`（隐含地）强制转换为`string`是极其常见/惯用的。由于`ToPrimitive`抽象操作的工作方式，`a + ""`在值`a`上调用`valueOf()`，它的返回值再最终通过内部的`ToString`抽象操作转换为一个`string`。但是`String(a)`只直接调用`toString()`。两种方式的最终结果都是一个`string`，但如果你使用一个`object`而不是一个普通的基本类型`number`的值，你可能不一定得到相同的`string`值！

```js
var a = {
	valueOf: function() { return 42; },
	toString: function() { return 4; }
};

a + "";			// "42"

String( a );	// "4"
```

`-`操作符是仅为数字减法定义的，所以`a - 0`强制`a`的值被转换为一个`number`。虽然少见得多，`a * 1`或`a / 1`也会得到相同的结果，因为这些操作符也是仅为数字操作定义的。

```js
var a = "3.14";
var b = a - 0;

b; // 3.14
```

```js
var a = [3];
var b = [1];

a - b; // 2
```

两个`array`值都不得不变为`number`，但它们首先会被强制转换为`string`（使用意料之中的`toString()`序列化），然后再强制转换为`number`，以便`-`减法操作可以实施。

### 隐含地：Booleans --> Numbers

```js
function onlyOne() {
	var sum = 0;
	for (var i=0; i < arguments.length; i++) {
		// 跳过falsy值。与将它们视为0相同，但是避开NaN
		if (arguments[i]) {
			sum += arguments[i];
		}
	}
	return sum == 1;
}

var a = true;
var b = false;

onlyOne( b, a );				// true
onlyOne( b, a, b, b, b );		// true

onlyOne( b, b );				// false
onlyOne( b, a, b, b, b, a );	// false
```

```js
function onlyOne() {
	var sum = 0;
	for (var i=0; i < arguments.length; i++) {
		sum += Number( !!arguments[i] );
	}
	return sum === 1;
}
```

我们首先使用`!!arguments[i]`来将这个值强制转换为`true`或`false`。这样你就可以像`onlyOne( "42", 0 )`这样传入非`boolean`值了，而且它依然可以如意料的那样工作（要不然，你将会得到`string`连接，而且逻辑也不正确）。一旦我们确认它是一个`boolean`，我们就使用`Number(..)`进行另一个明确的强制转换来确保值是`0`或`1`。

### 隐含地：* --> Boolean

隐含的强制转换是当你以强制一个值被转换的方式使用这个值时才启动的。对于数字和`string`操作，很容易就能看出这种强制转换是如何发生的。

（隐含地）要求/强制一个`boolean`转换的表达式操作：

1. 在一个`if (..)`语句中的测试表达式。
2. 在一个`for ( .. ; .. ; .. )`头部的测试表达式（第二个子句）。
3. 在`while (..)`和`do..while(..)`循环中的测试表达式。
4. 在`? :`三元表达式中的测试表达式（第一个子句）。
5. `||`（“逻辑或”）和`&&`（“逻辑与”）操作符左手边的操作数（它用作测试表达式）。

在这些上下文环境中使用的，任何还不是`boolean`的值，将通过本章早先讲解的`ToBoolean`抽象操作的规则，被隐含地强制转换为一个`boolean`。

```js
var a = 42;
var b = "abc";
var c;
var d = null;

if (a) {
	console.log( "yep" );		// yep
}

while (c) {
	console.log( "nope, never runs" );
}

c = d ? a : b;
c;								// "abc"

if ((a && d) || c) {
	console.log( "yep" );		// yep
}
```

### `||`和`&&`操作符

在JavaScript中它们实际上不会得出一个逻辑值（也就是`boolean`），这与它们在其他的语言中的表现不同。它们得出两个操作数中的一个（而且仅有一个）。换句话说，它们在两个操作数的值中选择一个：

```js
var a = 42;
var b = "abc";
var c = null;

a || b;		// 42
a && b;		// "abc"

c || b;		// "abc"
c && b;		// null
```

`||`和`&&`操作符都在第一个操作数（`a`或`c`） 上进行`boolean`测试。如果这个操作数还不是`boolean`（就像在这里一样），就会发生一次普通的`ToBoolean`强制转换。

1. 对于`||`操作符，如果测试结果为`true`，`||`表达式就将第一个操作数的值作为结果。如果测试结果为`false`，`||`表达式就将第二个操作数的值作为结果。

2. 对于`&&`操作符，如果测试结果为`true`，`&&`表达式将第二个操作数的值作为结果。如果测试结果为`false`，那么`&&`表达式就将第一个操作数的值作为结果。

`||`或`&&`表达式的结果总是两个操作数之一的底层值，不是（可能是被强制转换来的）测试的结果。

```js
a || b;
// 大体上等价于：
a ? a : b;

a && b;
// 大体上等价于：
a ? b : a;
```

我说`a || b`“大体上等价”于`a ? a : b`，是因为虽然结果相同，但是这里有一个微妙的不同。在`a ? a : b`中，如果`a`是一个更复杂的表达式（例如像调用`function`那样可能带有副作用），那么这个表达式`a`将有可能被求值两次（如果第一次求值的结果为truthy）。相比之下，对于`a || b`，表达式`a`仅被求值一次，而且这个值将被同时用于强制转换测试和结果值（如果合适的话）。

一个极其常见，而且很有帮助的用法：

```js
function foo(a,b) {
	a = a || "hello";
	b = b || "world";

	console.log( a + " " + b );
}

foo();					// "hello world"
foo( "yeah", "yeah!" );	// "yeah yeah!"

foo( "That's it!", "" ); // "That's it! world" <-- Oops!
```

作为第二个参数的`""`是一个falsy值，所以`b = b || "world"`测试失败，而默认值`"world"`被替换上来，即便本来的意图可能是想让明确传入的`""`作为赋给`b`的值。你不得不只在所有的falsy值应当被跳过时使用它。不然，你就需要在你的测试中更加具体，而且可能应该使用一个`? :`三元操作符。

`&&`操作符会“选择”第二个操作数，当且仅当第一个操作数测试为truthy，这种用法有时被称为“守护操作符” —— 第一个表达式的测试“守护”着第二个表达式。

```js
function foo() {
	console.log( a );
}

var a = 42;

a && foo(); // 42
```

`foo()`仅在`a`测试为truthy时会被调用。如果这个测试失败，这个`a && foo()`表达式语句将会无声地停止 —— 这被称为“短接” —— 而且永远不会调用`foo()`。几乎很少有人手动编写这样的东西。通常，他们会写`if (a) { foo(); }`。但是JS压缩器选择`a && foo()`是因为它短的多。

`if`语句和`for`循环包含`a && (b || c)`这样的复合的逻辑表达式，它们到底都是怎么工作的：

```js
var a = 42;
var b = null;
var c = "foo";

if (a && (b || c)) {
	console.log( "yep" );
}
```

`a && (b || c)`的结果实际上是`"foo"`，不是`true`。所以，这之后`if`语句强制值`"foo"`转换为一个`boolean`，这理所当然地将是`true`。

### Symbol 强制转换

从一个`symbol`到一个`string`的明确强制转换是允许的，但是相同的隐含强制转换时不被允许的，而且会抛出一个错误。

```js
var s1 = Symbol( "cool" );
String( s1 );					// "Symbol(cool)"

var s2 = Symbol( "not cool" );
s2 + "";						// TypeError
```

## 宽松等价与严格等价

`==`允许在等价性比较中进行强制转换，而`===`不允许强制转换。

### 等价性的性能

`==`好像要比`===`慢一些。强制转换确实要花费一点点处理时间，但也就是仅仅几微秒。如果你比较同类型的两个值，`==`和`===`使用的是相同的算法，所以除了在引擎实现上的一些微小的区别，它们做的应当是相同的工作。`==`和`===`都会检查它们的操作数的类型。不同之处在于它们在类型不同时如何反应。

### 抽象等价性

`==`操作符的行为被定义为“抽象等价性比较算法”。

1. 如果两个被比较的值是同一类型，它们就像你期望的那样通过等价性简单自然地比较。比如，`42`只和`42`相等，而`"abc"`只和`"abc"`相等。在一般期望的结果中，有一些例外需要小心：

* `NaN`永远不等于它自己
* `+0`和`-0`是相等的

2. `object`（包括`function`和`array`）的`==`宽松相等性比较。这样的两个值仅在它们引用完全相同的值时相等。

`===`严格等价比较的定义一模一样，包括关于两个`object`的值的规定。很少有人知道，在两个`object`被比较的情况下，`==`和`===`的行为相同。

如果你使用`==`款所等价来比较两个不同类型的值，它们两者或其中之一将需要被隐含地强制转换。由于这个强制转换，两个值最终归于同一类型，可以使用简单的值的等价性来直接比较它们相等与否。

#### 比较：`string`与`number`

1. 如果Type(x)是Number而Type(y)是String，返回比较x == ToNumber(y)的结果。
2. 如果Type(x)是String而Type(y)是Number，返回比较ToNumber(x) == y的结果。

```js
var a = 42;
var b = "42";

a === b;	// false
a == b;		// true
```

#### 比较：任何东西与`boolean`

1. 如果Type(x)是Boolean，返回比较 ToNumber(x) == y 的结果。
2. 如果Type(y)是Boolean，返回比较 x == ToNumber(y) 的结果。

```js
var x = true;
var y = "42";

x == y; // false
```

`Type(x)`确实是`Boolean`，所以它会实施`ToNumber(x)`，将`true`强制转换为`1`。现在，`1 == "42"`会被求值。这里面的类型依然不同，所以（实质上是递归地）我们再次向早先讲解过的算法求解，它将`"42"`强制转换为`42`，而`1 == 42`明显是`false`。

```js
var x = "42";
var y = false;

x == y; // false
```

`Type(y)`是`Boolean`，所以`ToNumber(y)`给出`0`。`"42" == 0`递归地变为`42 == 0`，这当然是`false`。

值`"42"`既不`== true`也不`== false`：

`"42"`的确是truthy，但是`"42" == true`根本就不是在进行一个boolean测试/强制转换，不管你的大脑怎么说，`"42"` 没有被强制转换为一个`boolean`（`true`），而是`true`被强制转换为一个`1`，而后`"42"`被强制转换为`42`。`ToBoolean`甚至都没参与到这里，所以`"42"`的真假是与`==`操作无关的！

永远，不要在任何情况下，使用`== true`或`== false`：

```js
var a = "42";

// 不好（会失败的！）：
if (a == true) {
	// ..
}

// 也不该（会失败的！）：
if (a === true) {
	// ..
}

// 足够好（隐含地工作）：
if (a) {
	// ..
}

// 更好（明确地工作）：
if (!!a) {
	// ..
}

// 也很好（明确地工作）：
if (Boolean( a )) {
	// ..
}
```

`=== true`和`=== false`不允许强制转换，所以它们没有`ToNumber`强制转换，因而是安全的。

#### 比较：`null`与`undefined`

1. 如果x是null而y是undefined，返回true。
2. 如果x是undefined而y是null，返回true。

当使用`==`宽松等价比较`null`和`undefined`，它们是互相等价（也就是互相强制转换）的，而且在整个语言中不会等价于其他值了。这意味着`null`和`undefined`对于比较的目的来说，如果你使用`==`宽松等价操作符来允许它们互相隐含地强制转换的话，它们可以被认为是不可区分的。

```js
var a = null;
var b;

a == b;		// true
a == null;	// true
b == null;	// true

a == false;	// false
b == false;	// false
a == "";	// false
b == "";	// false
a == 0;		// false
b == 0;		// false
```

推荐使用这种强制转换来允许`null`和`undefined`是不可区分的，如此将它们作为相同的值对待。

```js
var a = doSomething();

if (a == null) {
	// ..
}
```

`a == null`检查仅在`doSomething()`返回`null`或者`undefined`时才会通过，而在任何其他值的情况下将会失败，即便是`0`，`false`，和`""`这样的falsy值。

明确形式没有必要地难看太多了（而且性能可能有点儿不好！）：

```js
var a = doSomething();

if (a === undefined || a === null) {
	// ..
}
```

#### 比较：`object`与非`object`

1. 如果Type(x)是一个String或者Number而Type(y)是一个Object，返回比较 x == ToPrimitive(y) 的结果。
2. 如果Type(x)是一个Object而Type(y)是String或者Number，返回比较 ToPrimitive(x) == y 的结果。

```js
var a = 42;
var b = [ 42 ];

a == b;	// true
```

值`[ 42 ]`的`ToPrimitive`抽象操作被调用，结果为值`"42"`。这里它就变为`42 == "42"`，我们已经讲解过这将变为`42 == 42`，所以`a`和`b`被认为是强制转换地等价。

这些条款仅提到了`String`和`Number`，而没有`Boolean`。这是因为，正如我们早先引述的，任何出现的`Boolean`操作数强制转换为一个`Number`。“拆箱”，就是一个基本类型值的`object`包装器（例如`new String("abc")`这样的形式）被展开，其底层的基本类型值（`"abc"`）被返回。这种行为与`==`算法中的`ToPrimitive`强制转换有关。

```js
var a = "abc";
var b = Object( a );	// 与`new String( a )`相同

a === b;				// false
a == b;					// true
```

`a == b`为`true`是因为`b`通过`ToPrimitive`强制转换为它的底层简单基本标量值`"abc"`。

然而由于`==`算法中的其他覆盖规则，有些值是例外：

```js
var a = null;
var b = Object( a );	// 与`Object()`相同
a == b;					// false

var c = undefined;
var d = Object( c );	// 与`Object()`相同
c == d;					// false

var e = NaN;
var f = Object( e );	// 与`new Number( e )`相同
e == f;					// false
```

值`null`和`undefined`不能被装箱 —— 它们没有等价的对象包装器 —— 所以`Object(null)`就像`Object()`一样，它们都仅仅产生一个普通对象。`NaN`可以被封箱到它等价的`Number`对象包装器中，当`==`导致拆箱时，比较`NaN == NaN`会失败，因为`NaN`永远不会它自己相等。

### 边界情况

#### 一个拥有其他值的数字

```js
Number.prototype.valueOf = function() {
	return 3;
};

new Number( 2 ) == 3;	// true
```

`2 == 3`不会调到这个陷阱中，这是由于`2`和`3`都不会调用内建的`Number.prototype.valueOf()`方法，因为它们已经是基本`number`值，可以直接比较。然而，`new Number(2)`必须通过`ToPrimitive`强制转换，因此调用`valueOf()`。

```js
var i = 2;

Number.prototype.valueOf = function() {
	return i++;
};

var a = new Number( 42 );

if (a == 2 && a == 3) {
	console.log( "Yep, this happened." );
}
```

这些都是邪恶的技巧。不要这么做。

#### False-y 比较

```js
"0" == null;			// false
"0" == undefined;		// false
"0" == false;			// true -- 噢！
"0" == NaN;				// false
"0" == 0;				// true
"0" == "";				// false

false == null;			// false
false == undefined;		// false
false == NaN;			// false
false == 0;				// true -- 噢！
false == "";			// true -- 噢！
false == [];			// true -- 噢！
false == {};			// false

"" == null;				// false
"" == undefined;		// false
"" == NaN;				// false
"" == 0;				// true -- 噢！
"" == [];				// true -- 噢！
"" == {};				// false

0 == null;				// false
0 == undefined;			// false
0 == NaN;				// false
0 == [];				// true -- 噢！
0 == {};				// false
```

#### 疯狂的情况

```js
[] == ![];		// true
2 == [2];		// true
"" == [null];	// true
0 == "\n";		// true
```

#### 7个麻烦的，坑人的强制转换

```js
"0" == false;			// true -- 噢！
false == 0;				// true -- 噢！
false == "";			// true -- 噢！
false == [];			// true -- 噢！
"" == 0;				// true -- 噢！
"" == [];				// true -- 噢！
0 == [];				// true -- 噢！
```

这个列表中7个项目的4个与`== false`比较有关，我们早先说过你应当 **总是，总是** 避免的。我不认为你在程序里有很大的可能要在一个`boolean`测试中使用`== []`。你可能会使用`== ""`或`== 0`：

```js
function doSomething(a) {
	if (a == "") {
		// ..
	}
}

function doSomething(a,b) {
	if (a == b) {
		// ..
	}
}
```

#### 安全地使用隐含强制转换

1. 如果比较的任意一边可能出现`true`或者`false`值，那么就永远，永远不要使用`==`。
2. 如果比较的任意一边可能出现`[]`，`""`，或`0`这些值，那么认真地考虑不使用`==`。
3. 另一个强制转换保证 *不会* 咬到你的地方是`typeof`操作符。`typeof`总是将返回给你7中字符串之一（见第一章），它们中没有一个是空`""`字符串。这样，检查某个值的类型时不会有任何情况与 *隐含* 强制转换相冲突。`typeof x == "function"`就像`typeof x === "function"`一样100%安全可靠

## 抽象关系比较

“抽象关系型比较”算法：

1. 首先在两个值上调用`ToPrimitive`强制转换，如果两个调用的返回值之一不是`string`，那么就使用`ToNumber`操作规则将这两个值强制转换为`number`值，并进行数字的比较。

```js
var a = [ 42 ];
var b = [ "43" ];

a < b;	// true
b < a;	// false
```

2. 如果`<`比较的两个值都是`string`的话，就会在字符上进行简单的字典顺序（自然的字母顺序）比较。

```js
var a = [ "42" ];
var b = [ "043" ];

a < b;	// false
```

```js
var a = [ 4, 2 ];
var b = [ 0, 4, 3 ];

a < b;	// false
```

```js
var a = { b: 42 };
var b = { b: 43 };

a < b;	// ??
```

```js
var a = { b: 42 };
var b = { b: 43 };

a < b;	// false
a == b;	// false
a > b;	// false

a <= b;	// true
a >= b;	// true
```

语言规范说，对于`a <= b`，它实际上首先对`b < a`求值，然后反转那个结果。因为`b < a`*也是*`false`，所以`a <= b`的结果为`true`。JS更准确地将`<=`考虑为“不大于”（`!(a > b)`，JS将它作为`(!b < a)`）。另外，`a >= b`被解释为它首先被考虑为`b <= a`，然后实施相同的推理。
