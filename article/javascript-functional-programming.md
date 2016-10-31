
# js中与FP有关的概念与运用

## 目录
1. 高阶函数
2. 柯里化、组合
3. Functor、Monad

--

## 1. 高阶函数指，函数与普通变量一样
* 可以作为参数；
* 可以作为返回值；

这2条特性可以实现：
* OOP：比如代理、单例等；
* 柯里化、组合、Functor、Monad；
* 甚至 AOP

### 例子 AOP
场景：页面用了某第三方库，需要执行某库方法的前后处理一些事情

```
// def
function TheLib() {
	this.data = "/origin data/"
}
TheLib.prototype.originFn = function (arg1) {
	console.log("exec origin: ", arg1)
	return "/output origin/"
}
var expObj = new TheLib()

// hook
function hook(origin, before, after) {
	return function () {
		var args = [].slice.call(arguments)
		before && before.apply(this, args)
		var rst = origin.apply(this, arguments)
		after && after.apply(this, [rst].concat(args))
		return rst
	}
}

expObj.originFn = hook(expObj.originFn, function (originInput) {
	console.log("exec before: ", originInput, this.data)
}, function (originOutput, originInput) {
	console.log("exec after: ", originOutput, originInput, this.data)
})

// exec
expObj.originFn("x")

```


## 2. 柯里化、组合

柯里化：**一种对参数进行缓存，得到一个记住这些参数的新函数。**

### 例子 isType bind

```
// isType
function isType(typeName) {
	return function (value) {
		return Object.prototype.toString.call(value) === "[object "+typeName+"]"
	}
}

var isFunction = isType("Function")

log(isFunction({})) // false
log(isFunction(new Function())) // true
log(isFunction(function () { // true
	
}))

// bind
Function.prototype._bind = function (ctx) {
	var self = this
	return function () {
		self.apply(ctx, arguments)
	}
}

function print() {
	log(this.name)
}
var printA = print._bind({name: "a"})
printA() // "a"

```

组合：如其名字，一种组合函数的方法；从简单的函数构造更复杂的函数、并将它们当作单元去操作。

### 组合例子

见下面 Maybe Functor


## 3. Functor、Monad

概念出自`Haskell`，目的是解决：函数副效应、错误处理、异步调用等问题。

Functor 
* 对外界来说，Functor 是一个容器，封装了值，提供`map`接口让外界访问/影响里面的值。
* 对Functor自身来说，是一个对于函数调用的抽象，我们赋予容器自己去调用`map`传递过来的函数的能力。
* 当 `map` 一个函数时，我们让容器自己来运行这个函数，这样容器就可以自由地选择何时何地如何操作这个函数，以致于拥有惰性求值`IO`、错误处理`Either`、异步调用`Future` 等特性。

Monad 
* 在`IO`、`Future`的例子中，map 是可以返回 `Functor`实例的，所以每`map`一次就升维一次，需要有个方法`join`在`map`后自动执行来降维。

### Functor 容器

```
// common
var fs = require("fs")
var _ = require("lodash")
var log = console.log

function log(raw) {
	console.log(raw)
}
var add = _.curry(_.add)

// def
var Box = function (list) {
	this.__value = list
}
Box.prototype.map = function (f) {
	var newList = []
	for (var i = 0; i < this.__value.length; i++) {
		newList.push(f(this.__value[i]))
	}
	return new Box(newList)
}

// exec
new Box([1,2,3])
	.map(add(10))
	.map(log)

```

### Maybe

```
// def
var Maybe = function (raw) {
	this.__value = raw
}
Maybe.prototype.map = function (f) {
	return this.isNothing() ? new Maybe(null) : new Maybe(f(this.__value))
}
Maybe.prototype.isNothing = function () {
	return (this.__value === null || this.__value === undefined)
}

// exec
new Maybe({name: "adi", age:100})
	.map(_.flowRight(add(10), _.property("age")))
	.map(log)
```

### Either

```
// def

function Left(raw) {
	this.__value = raw
}
function Right(raw) {
	this.__value = raw
}
Left.prototype.map = function (f) {
	return this
}
Right.prototype.map = function (f) {
	return new Right(f(this.__value))
}
Left.prototype.catch = function (f) {
	f(this.__value)
}
Right.prototype.catch = function (f) {
}

// exec
var getAge = function (user) {
	return user.age ? new Right(user.age) : new Left("Error")
}
getAge({name: "adi", age1: 100})
	.map(add(10))
	.map(log)
	.catch(log)

```


### IO Functor

```
// def
function IO(objFn) {
	this.__value = objFn
}
IO.prototype.map = function (f) {
	return new IO(_.flowRight(f, this.__value))
}

// exec
var ops = new IO(function (_) {
		return global.process
	})
	.map(function (raw) {
		return raw.version
	})
	.map(function (raw) {
		return "【"+raw+"】"
	})
	
console.log(ops.__value())

```

### Future Functor

```
// def
function Future() {
	this.slots = []
}

Future.prototype.ready = function (slot) {
	if (this.__completed) {
		spot(this.__value)
	}
	else {
		this.slots.push(slot)
	}
}

Future.prototype.complete = function (raw) {
	if (this.__completed) throw "already completed"
	this.__value = raw
	this.__completed = true
	// emitter
	for (var i = 0; i < this.slots.length; i++) {
		this.slots[i](raw)
	}
}

Future.prototype.map = function (op) {
	var next = new Future()
	this.ready(function (raw) {
		next.complete( op(raw) ) // op(raw) maybe FutureInstance, so need join to unWrap
	})
	return next
}

readFile("files/README.md")
	.map(function (raw) {
		var f = new Future()
		setTimeout(function () {
			console.log(raw)
			f.complete(1)
		}, 1000)
		return f
	})
	.map(function (raw) {
		console.log(raw) // Future(Future(raw))
	})

```

### Future Monad

```
// def
Future.prototype.join = function () {
	var next = new Future()
	this.ready(function (rawWithFuture) {
		rawWithFuture.ready(function (raw) {
			next.complete(raw)
		})
	})
	return next
}

Future.prototype.then = function (op) {
	var next = new Future()
	this.ready(function (raw) {
		var result = op(raw)
		if (result && result.slots){
			result.ready(function (raw) {
				next.complete( raw )
			})
		}
		else {
			next.complete( result )
		}
	})
	return next
}

// exec

readFile("files/README.md")
	.then(function (raw) {
		var f = new Future()
		setTimeout(function () {
			console.log(1, raw)
			f.complete(1)
		}, 1000)
		
		var next = new Future()
		f.then(function () {
			setTimeout(function () {
				console.log(1.5, raw)
				next.complete(1.5)
			}, 1000)
		})
		return next
	})
	.then(function (raw) {
		console.log(2, raw)
		return 3
	})
	.then(function (raw) {
		var f = new Future()
		setTimeout(function () {
			console.log(3, ++raw)
			f.complete(++raw)
		}, 1000)
		return f
	})
```

### Future +lift
lift 处理并行的情况，类似 Promise.all 接口。

```
// def
Future.lift = function (op) {
	return function () {
		var futures = Array.prototype.slice.call(arguments)
		var next = new Future()
		bindFutures(0, [])
		return next
		
		function bindFutures(index, raws) {
			return futures[index].then(function (raw) {
				return (index < futures.length - 1) ?
					bindFutures(index + 1, raws.concat(raw)) :
					next.complete( op.apply(null, raws.concat(raw)) )
			})
		}
	}
}

// exec
var mergeFiles = Future.lift(function (str1, str2, str3) {
	return str1 + "_" + str2 + "__" + str3
})
var f1 = readFile("files/README.md")
var f2 = readFile("files/README2.md")
var f3 = readFile("files/README3.md")
mergeFiles(f1, f2, f3).then(log)

```

--

* [JavaScript函数式编程][1]
* [monads-in-pictures][2]
* [from-callback-to-future-functor-monad][3]
* [promises-are-the-monad-of-asynchronous-programming][4]
* [mostly-adequate-guide-chinese][5]


[1]: https://zhuanlan.zhihu.com/p/21714695?refer=starkwang
[2]: http://jiyinyiyong.github.io/monads-in-pictures/
[3]: https://hackernoon.com/from-callback-to-future-functor-monad-6c86d9c16cb5
[4]: https://blog.jcoglan.com/2011/03/11/promises-are-the-monad-of-asynchronous-programming/
[5]: https://github.com/llh911001/mostly-adequate-guide-chinese

