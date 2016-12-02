# 原生类型

## 原生类型

1. 种类 9 + 1 种： `String()`、`Number()`、`Boolean()`、`Array()`、`Object()`、`Function()`、`RegExp()`、`Date()`、`Error()`、`Symbol()`

2. 创建值的构造器形式（`new String("abc")`）的结果是一个基本类型值（`"abc"`）的包装器对象。`typeof`显示这些对象不是它们自己的特殊类型，而是`object`类型的子类型


## 内部`[[Class]]`

1. `typeof`的结果为`"object"`的值被额外地打上了一个内部的标签属性`[[Class]]`。这个属性不能直接地被访问，但通常可以间接地通过在这个值上借用默认的`Object.prototype.toString(..)`方法调用来展示。

```js
Object.prototype.toString.call( [1,2,3] );			// "[object Array]"
Object.prototype.toString.call( /regex-literal/i );	// "[object RegExp]"
```

2. 在大多数情况下，这个内部的`[[Class]]`值对应于关联这个值的内建的原生类型构造器，但事实却不总是这样。

```js
Object.prototype.toString.call( null );			// "[object Null]"
Object.prototype.toString.call( undefined );	// "[object Undefined]"
```

不存在`Null()`和`Undefined()`原生类型构造器，但不管怎样`"Null"`和`"Undefined"`是被暴露出来的内部`[[Class]]`值。

```js
Object.prototype.toString.call( "abc" );	// "[object String]"
Object.prototype.toString.call( 42 );		// "[object Number]"
Object.prototype.toString.call( true );		// "[object Boolean]"
```

每一个简单基本类型都自动地被它们分别对应的对象包装器封箱，这就是为什么`"String"`，`"Number"`，和`"Boolean"`分别被显示为内部`[[Class]]`值

## 封箱包装器

1. 基本类型值没有属性或方法，所以为了访问`.length`或`.toString()`你需要这个值的对象包装器。JS将会自动地封箱（也就是包装）基本类型值来满足这样的访问

2. 这么做看起来很有道理：一开始就得到一个这个值的对象形式，于是JS引擎就不需要隐含地为你创建一个。但事实证明这是一个坏主意。浏览器们长久以来就对`.length`这样的常见情况进行性能优化，这意味着如果你试着直接使用对象形式（它们没有被优化过）进行“提前优化”，那么实际上你的程序将会变慢。一般来说，基本上没有理由直接使用对象形式。让封箱在需要的地方隐含地发生会更好

3. 对象包装器的坑

```js
var a = new Boolean( false );

if (!a) {
	console.log( "Oops" ); // never runs
}
```

4. 如果你想手动封箱一个基本类型值，你可以使用`Object(..)`函数（没有`new`关键字）

```js
var a = "abc";
var b = new String( a );
var c = Object( a );

typeof a; // "string"
typeof b; // "object"
typeof c; // "object"

b instanceof String; // true
c instanceof String; // true

Object.prototype.toString.call( b ); // "[object String]"
Object.prototype.toString.call( c ); // "[object String]"
```

5. 如果你有一个包装器对象，而你想要取出底层的基本类型值，你可以使用`valueOf()`方法

```js
var a = new String( "abc" );
var b = new Number( 42 );
var c = new Boolean( true );

a.valueOf(); // "abc"
b.valueOf(); // 42
c.valueOf(); // true
```

## 原生类型作为构造器

### Array(..)

1. `Array(..)`构造器不要求在它前面使用`new`关键字。`Array(1,2,3)`和`new Array(1,2,3)`的结果是一样的

2. `Array`构造器有一种特殊形式，如果它仅仅被传入一个`number`参数，与将这个值作为数组的内容不同，它会被认为是用来“预定数组大小”用的长度。其实没有预定数组大小这样的东西。你所创建的是一个空数组，并将这个数组的`length`属性设置为那个指定的数字值。带有至少一个“空值槽”的数组经常被称为“稀散数组”

```js
var a = new Array( 3 );
var b = [ undefined, undefined, undefined ];
var c = [];
c.length = 3;
```

如`c`，数组中的空值槽可以在数组的创建之后发生。将数组的`length`改变为超过它实际定义的槽值的数目，你就隐含地引入了空值槽。你甚至可以在上面的代码段中调用`delete b[1]`，而这么做将会在`b`的中间引入一个空值槽。`b`的序列化表现为`[ undefined, undefined, undefined ]`，与之相对的是`a`和`c`的`[ undefined x 3 ]`

在ES5中，列表（数组值，属性列表等等）末尾的逗号是允许的（被砍掉并忽略）。所以如果你在你的程序或控制台中敲入`[ , , , ]`值，你实际上得到的是一个底层为`[ , , ]`的值（也就是，一个带有三个空值槽的数组）

```js
a.join( "-" ); // "--"
b.join( "-" ); // "--"

a.map(function(v,i){ return i; }); // [ undefined x 3 ]
b.map(function(v,i){ return i; }); // [ 0, 1, 2 ]
```

`a.map(..)`调用会失败是因为值槽根本就不实际存在，所以`map(..)`没有东西可以迭代。`join(..)`的工作方式不同，基本上我们可以认为它是像这样被实现的

```js
function fakeJoin(arr,connector) {
	var str = "";
	for (var i = 0; i < arr.length; i++) {
		if (i > 0) {
			str += connector;
		}
		if (arr[i] !== undefined) {
			str += arr[i];
		}
	}
	return str;
}

var a = new Array( 3 );
fakeJoin( a, "-" ); // "--"
```

`join(..)`好用是因为它认为值槽存在，并循环至`length`值。不管`map(..)`内部是在做什么，它（显然）没有做出这样的假设，所以源自于奇怪的“空值槽”数组的结果出人意料，而且好像是失败了

创建一个实际的`undefined`值的数组（不只是“空值槽”）

```js
var a = Array.apply( null, { length: 3 } );
a; // [ undefined, undefined, undefined ]
```

第一个参数是一个`this`对象绑定，在这里我们不关心它，所以我们将它设置为`null`。第二个参数应该是一个数组（或“类数组对象”）。这个“数组”（`{ length: 3 }`对象值）的内容作为这个函数的参数“分散”开来。在`apply(..)`内部，我们可以预见这里有另一个`for`循环（有些像上面的`join(..)`），它从`0`开始上升但不包含至`length`（这个例子中是`3`）。对于每一个索引，它从对象中取得相应的键。所以如果这个数组对象参数在`apply(..)`内部被命名为`arr`,那么这种属性访问实质上是`arr[0]`，`arr[1]`，和`arr[2]`。当然，没有一个属性是在`{ length: 3 }`对象值上存在的，所以这三个属性访问都将返回值`undefined`。换句话说，调用`Array(..)`的结局基本上是这样：`Array(undefined,undefined,undefined)`，这就是我们如何得到一个填满`undefined`值的数组的，而非仅仅是一些（疯狂的）空值槽

### Object(..)

几乎没有理由使用`new Object()`构造器形式，尤其因为它强迫你一个一个地添加属性，而不是像对象的字面形式那样一次添加许多

### Function(..)

`Function`构造器仅在最最罕见的情况下有用，也就是你需要动态地定义一个函数的参数和/或它的函数体。不要将`Function(..)`仅仅作为另一种形式的`eval(..)`。你几乎永远不会需要用这种方式动态定义一个函数

### RegExp(..)

用字面量形式（`/^a*b+/g`）定义正则表达式是被大力采用的，不仅因为语法简单，而且还有性能的原因——JS引擎会在代码执行前预编译并缓存它们。和我们迄今看到的其他构造器形式不同，`RegExp(..)`有一些合理的用途：用来动态定义一个正则表达式的模式

### Date(..)

1. 目前你构建一个日期对象的最常见的理由是要得到当前的时间戳（一个有符号整数，从1970年1月1日开始算起的毫秒数）。你可以在一个日期对象实例上调用`getTime()`得到它。但是在ES5中，一个更简单的方法是调用定义为`Date.now()`的静态帮助函数。而且在前ES5中填补它很容易

```js
if (!Date.now) {
	Date.now = function(){
		return (new Date()).getTime();
	};
}
```

2. 如果你不带`new`调用`Date()`，你将会得到一个那个时刻的日期/时间的字符串表达

### Error(..)

1. `Error(..)`构造器（很像上面的`Array()`）在有`new`与没有`new`时的行为是相同的

2. 你想要创建error对象的主要原因是，它会将当前的执行栈上下文捕捉进对象中（在大多数JS引擎中，在创建后使用只读的`.stack`属性表示）。这个栈上下文包含函数调用栈和error对象被创建时的行号，这使调试这个错误更简单

```js
function foo(x) {
	if (!x) {
		throw new Error( "x wasn't provided" );
	}
	// ..
}
```

3. Error对象实例一般拥有至少一个`message`属性，有时还有其他属性（你应当将它们作为只读的），比如`type`。然而，与其检视上面提到的`stack`属性，最好是在error对象上调用`toString()`（明确地调用，或者是通过强制转换隐含地调用）来得到一个格式友好的错误消息

4. 技术上讲，除了一般的`Error(..)`原生类型以外，还有几种特定错误的原生类型：`EvalError(..)`，`RangeError(..)`，`ReferenceError(..)`，`SyntaxError(..)`， `TypeError(..)`，和`URIError(..)`。但是手动使用这些特定错误原生类型十分少见。如果你的程序确实遭受了一个真实的异常，它们是会自动地被使用的

### Symbol(..)

1. Symbol可以用做属性名，但是你不能从你的程序中看到或访问一个symbol的实际值，从开发者控制台也不行。如果你在开发者控制台中对一个Symbol求值，将会显示`Symbol(Symbol.create)`之类的东西

2. 要定义你自己的Symbol，使用`Symbol(..)`原生类型。`Symbol(..)`原生类型“构造器”很独特，因为它不允许你将`new`与它一起使用，这么做会抛出一个错误

```js
var mysym = Symbol( "my own symbol" );
mysym;				// Symbol(my own symbol)
mysym.toString();	// "Symbol(my own symbol)"
typeof mysym; 		// "symbol"

var a = { };
a[mysym] = "foobar";

Object.getOwnPropertySymbols( a );
// [ Symbol(my own symbol) ]
```

3. 虽然Symbol实际上不是私有的（在对象上使用`Object.getOwnPropertySymbols(..)`反射，揭示了Symbol其实是相当公开的），但是它们的主要用途可能是私有属性，或者类似的特殊属性

### 原生类型原型

1. 每一个内建的原生构造器都拥有它自己的`.prototype`对象——`Array.prototype`，`String.prototype`等等

2. 做为文档惯例，`String.prototype.XYZ`会被缩写为`String#XYZ`，对于其它所有`.prototype`的属性都是如此

3. 一些原生类型的原型不仅仅是单纯的对象

```js
typeof Function.prototype;			// "function"
Function.prototype();				// 它是一个空函数！

RegExp.prototype.toString();		// "/(?:)/" —— 空的正则表达式
"abc".match( RegExp.prototype );	// [""]
```

如你所见，`Function.prototype`是一个函数，`RegExp.prototype`是一个正则表达式，而`Array.prototype`是一个数组。这使它们成了可以赋值给变量的，很好的“默认”值——如果这些类型的变量还没有值

```js
function isThisCool(vals,fn,rx) {
	vals = vals || Array.prototype;
	fn = fn || Function.prototype;
	rx = rx || RegExp.prototype;

	return rx.test(
		vals.map( fn ).join( "" )
	);
}

isThisCool();		// true

isThisCool(
	["a","b","c"],
	function(v){ return v.toUpperCase(); },
	/D/
);					// false
```

这个方式的一个微小的副作用是，`.prototype`已经被创建了，而且是内建的，因此它仅被创建 *一次*。相比之下，使用`[]`，`function(){}`，和`/(?:)/`这些值本身作为默认值，将会（很可能，要看引擎如何实现）在每次调用`isThisCool(..)`时重新创建这些值（而且稍可能要回收它们）。这可能会消耗内存/CPU

要非常小心不要对后续要被修改的值使用`Array.prototype`做为默认值。在这个例子中，`vals`是只读的，但如果你要在原地对`vals`进行修改，那你实际上修改的是`Array.prototype`本身，这将把你引到刚才提到的坑里
