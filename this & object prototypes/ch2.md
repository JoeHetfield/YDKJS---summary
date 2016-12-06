# This豁然开朗

## 调用点

1. 调用栈
2. 如何找到调用点

## 默认绑定

是在没有其他规则适用时的默认规则。

```js
function foo() {
	console.log( this.a );
}

var a = 2;

foo(); // 2
```

如果`strict mode`生效，那么对于默认绑定来说全局对象是不合法的，所以`this`将被设置为`undefined`。

```js
function foo() {
	"use strict";

	console.log( this.a );
}

var a = 2;

foo(); // TypeError: `this` is `undefined`
```

如果`foo()`的内容没有在`strint mode`下执行，对于默认绑定来说全局对象是唯一合法的；`foo()`的调用点的`strict mode`状态与此无关。

```js
function foo() {
	console.log( this.a );
}

var a = 2;

(function(){
	"use strict";

	foo(); // 2
})();
```

## 隐含绑定

当一个方法引用存在一个环境对象时，这个对象应当被用于这个函数调用的`this`绑定。

```js
function foo() {
	console.log( this.a );
}

var obj = {
	a: 2,
	foo: foo
};

obj.foo(); // 2
```

只有对象属性引用链的最后一层是影响调用点的。

```js
function foo() {
	console.log( this.a );
}

var obj2 = {
	a: 42,
	foo: foo
};

var obj1 = {
	a: 2,
	obj2: obj2
};

obj1.obj2.foo(); // 42
```

当一个隐含绑定丢失了它的绑定，这通常意味着它会退回到默认绑定。

```js
function foo() {
	console.log( this.a );
}

var obj = {
	a: 2,
	foo: foo
};

var bar = obj.foo; // 函数引用！

var a = "oops, global"; // `a`也是一个全局对象的属性

bar(); // "oops, global"
```

尽管`bar`似乎是`obj.foo`的引用，但实际上它只是另一个`foo`自己的引用而已。起作用的调用点是`bar()`，一个直白，毫无修饰的调用。因此 *默认绑定* 适用于这里。

```js
function foo() {
	console.log( this.a );
}

function doFoo(fn) {
	// `fn` 只不过`foo`的另一个引用

	fn(); // <-- 调用点!
}

var obj = {
	a: 2,
	foo: foo
};

var a = "oops, global"; // `a`也是一个全局对象的属性

doFoo( obj.foo ); // "oops, global"
```

参数传递仅仅是一种隐含的赋值，而且因为我们在传递一个函数，它是一个隐含的引用赋值。

## 明确绑定

`call(..)`和`apply(..)`接收的第一个参数都是一个用于`this`的对象，之后使用这个指定的`this`来调用函数。

```js
function foo() {
	console.log( this.a );
}

var obj = {
	a: 2
};

foo.call( obj ); // 2
```

如果你传递一个简单原始类型值作为`this`绑定，那么这个原始类型值会被包装在它的对象类型中。

单独依靠明确绑定仍然不能解决函数“丢失”自己原本的`this`绑定，或者被第三方框架覆盖，等等问题。

```js
function foo() {
	console.log( this.a );
}

var obj = {
	a: 2
};

var bar = function() {
	foo.call( obj );
};

bar(); // 2
setTimeout( bar, 100 ); // 2

// `bar`将`foo`的`this`硬绑定到`obj`
// 所以它不可以被覆盖
bar.call( window ); // 2
```

```js
function foo(something) {
	console.log( this.a, something );
	return this.a + something;
}

// 简单的`bind`帮助函数
function bind(fn, obj) {
	return function() {
		return fn.apply( obj, arguments );
	};
}

var obj = {
	a: 2
};

var bar = bind( foo, obj );

var b = bar( 3 ); // 2 3
console.log( b ); // 5
```

```js
function foo(something) {
	console.log( this.a, something );
	return this.a + something;
}

var obj = {
	a: 2
};

var bar = foo.bind( obj );

var b = bar( 3 ); // 2 3
console.log( b ); // 5
```

`bind(..)`生成的硬绑定函数有一个名为`.name`的属性，它源自于原始的目标函数。

API调用的“环境”：

```js
function foo(el) {
	console.log( el, this.id );
}

var obj = {
	id: "awesome"
};

// 使用`obj`作为`this`来调用`foo(..)`
[1, 2, 3].forEach( foo, obj ); // 1 awesome  2 awesome  3 awesome
```

## new绑定

实际上不存在“构造器函数”这样的东西，而只有函数的构造器调用。

当对函数进行构造器调用时，这些事情会自动完成：

1. 一个全新的对象会凭空创建
2. 这个新构建的对象会被接入原形链
3. 这个新构建的对象被设置为函数调用的`this`绑定
4. 除非函数返回一个它自己的其他对象，这个被`new`调用的函数将自动返回这个新构建的对象

```js
function foo(a) {
	this.a = a;
}

var bar = new foo( 2 );
console.log( bar.a ); // 2
```

## 优先顺序

明确绑定的优先权高于隐含绑定。

```js
function foo() {
	console.log( this.a );
}

var obj1 = {
	a: 2,
	foo: foo
};

var obj2 = {
	a: 3,
	foo: foo
};

obj1.foo(); // 2
obj2.foo(); // 3

obj1.foo.call( obj2 ); // 3
obj2.foo.call( obj1 ); // 2
```

new绑定的优先级高于隐含绑定。

```js
function foo(something) {
	this.a = something;
}

var obj1 = {
	foo: foo
};

var obj2 = {};

obj1.foo( 2 );
console.log( obj1.a ); // 2

obj1.foo.call( obj2, 3 );
console.log( obj2.a ); // 3

var bar = new obj1.foo( 4 );
console.log( obj1.a ); // 2
console.log( bar.a ); // 4
```

new绑定的优先级高于明确绑定，`new`和`call`/`apply`不能同时使用，但可以使用硬绑定来测试这两个规则的优先级。

```js
function foo(something) {
	this.a = something;
}

var obj1 = {};

var bar = foo.bind( obj1 );
bar( 2 );
console.log( obj1.a ); // 2

var baz = new bar( 3 );
console.log( obj1.a ); // 2
console.log( baz.a ); // 3
```

ES5的内建`Function.prototype.bind(..)`：

```js
if (!Function.prototype.bind) {
	Function.prototype.bind = function(oThis) {
		if (typeof this !== "function") {
			// 可能的与 ECMAScript 5 内部的 IsCallable 函数最接近的东西，
			throw new TypeError( "Function.prototype.bind - what " +
				"is trying to be bound is not callable"
			);
		}

		var aArgs = Array.prototype.slice.call( arguments, 1 ),
			fToBind = this,
			fNOP = function(){},
			fBound = function(){
				return fToBind.apply(
					(
						this instanceof fNOP &&
						oThis ? this : oThis
					),
					aArgs.concat( Array.prototype.slice.call( arguments ) )
				);
			}
		;

		fNOP.prototype = this.prototype;
		fBound.prototype = new fNOP();

		return fBound;
	};
}
```

为什么`new`可以覆盖硬绑定这件事很有用：

```js
function foo(p1,p2) {
	this.val = p1 + p2;
}

// 在这里使用`null`是因为在这种场景下我们不关心`this`的hard-binding
// 而且反正它将会被`new`调用覆盖掉！
var bar = foo.bind( null, "p1" );

var baz = new bar( "p2" );

baz.val; // p1p2
```

## 绑定的特例

### 被忽略的`this`

如果你传递`null`或`undefined`作为`call`，`apply`或`bind`的`this`绑定参数，那么这些值会被忽略掉，取而代之的是 *默认绑定* 规则将适用于这个调用。

1. 使用`apply(..)`来将一个数组散开，从而作为函数调用的参数
2. `bind(..)`可以curry参数（预设值）

```js
function foo(a,b) {
	console.log( "a:" + a + ", b:" + b );
}

// 将数组散开作为参数
foo.apply( null, [2, 3] ); // a:2, b:3

// 用`bind(..)`进行柯里化
var bar = foo.bind( null, 2 );
bar( 3 ); // a:2, b:3
```

ES6中有一个扩散操作符：`...`。它让你无需使用`apply(..)`而在语法上将一个数组“散开”作为参数。

更安全的`this`，创建完全为空的对象的最简单方法。

```js
function foo(a,b) {
	console.log( "a:" + a + ", b:" + b );
}

// 我们的DMZ空对象
var ø = Object.create( null );

// 将数组散开作为参数
foo.apply( ø, [2, 3] ); // a:2, b:3

// 用`bind(..)`进行currying
var bar = foo.bind( ø, 2 );
bar( 3 ); // a:2, b:3
```

间接引用：

```js
function foo() {
	console.log( this.a );
}

var a = 2;
var o = { a: 3, foo: foo };
var p = { a: 4 };

o.foo(); // 3
(p.foo = o.foo)(); // 2
```

软化绑定：

```js
if (!Function.prototype.softBind) {
	Function.prototype.softBind = function(obj) {
		var fn = this,
			curried = [].slice.call( arguments, 1 ),
			bound = function bound() {
				return fn.apply(
					(!this ||
						(typeof window !== "undefined" &&
							this === window) ||
						(typeof global !== "undefined" &&
							this === global)
					) ? obj : this,
					curried.concat.apply( curried, arguments )
				);
			};
		bound.prototype = Object.create( fn.prototype );
		return bound;
	};
}
```

```js
function foo() {
   console.log("name: " + this.name);
}

var obj = { name: "obj" },
    obj2 = { name: "obj2" },
    obj3 = { name: "obj3" };

var fooOBJ = foo.softBind( obj );

fooOBJ(); // name: obj

obj2.foo = foo.softBind(obj);
obj2.foo(); // name: obj2   <---- 看!!!

fooOBJ.call( obj3 ); // name: obj3   <---- 看!

setTimeout( obj2.foo, 10 ); // name: obj   <---- 退回到软绑定
```

## 词法`this`

箭头函数，一个箭头函数的词法绑定是不能被覆盖的（就连`new`也不行！），本质是用被广泛理解的词法作用域来禁止了传统的`this`机制。

```js
function foo() {
  // 返回一个arrow function
	return (a) => {
    // 这里的`this`是词法上从`foo()`采用
		console.log( this.a );
	};
}

var obj1 = {
	a: 2
};

var obj2 = {
	a: 3
};

var bar = foo.call( obj1 );
bar.call( obj2 ); // 2, 不是3!
```
