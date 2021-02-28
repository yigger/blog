---
title: javascript - 作用域与变量提升
date: 2021-02-28
categories: javascript
---

[toc]

## 函数作用域
函数作用域还是比较简单，但凡写过 JS 的小伙伴也知道下面的示例代码为什么会输出这种结果，因为变量 `count` 是在函数作用域内定义的，那么 `count` 的作用范围则仅在 myFunction 内生效，所以外部是无法访问到 count 变量的。
```js
function myFunction() {
	var count = 1;
	count++;
	console.log(count); // 2
}
myFunction();
console.log(count); // a is not defined
```

## 块作用域
对于后端的开发者来说，特别是用 ruby  语言的，块作用域应该是用得比较多，因为在 ruby 中，可以往函数传递一个块来进行执行。但是 js 就 相对来说没这么复杂，甚至还很少显式的用到块作用域。
```js
var right = true
if (right) {
	var count = 0;
  count ++;
  console.log(count); // 1
}
console.log(count); // ?
```

对于 `?` 处会打印什么？在我第一次看到这段代码的时候，我也会直觉性的认为他会打印 `count is undefined`，因为我认为 `count` 的包含在 if 的块作用域内的，然而事实是 `?` 处的 `count` 也打印了 **1**，问题在于我忽略了**变量提升**这茬事了，稍后也会描述到，现在可以简单理解为在 if 块内用 var 声明的变量，最终也会等同于在 if 外层声明

## let
基于上述的块作用域提到的示例代码片段，产生了与预期不一致的行为 -- **预期是 count 仅在 if 块内可用**，如何才能产生预期的情况？很简单，把 var 改为 let 即可。

let 是 es6 引入的新的关键字，提供了除 var 以外的另一种变量声明的方式，let 最有用的地方在于**声明的变量仅在块内起作用**。

再次执行上述代码，发现与预期情况一致了：count 仅在 if 作用域内生效
```
var right = true
if (right) {
	let count = 0;
  count ++;
  console.log(count); // 1
}
console.log(count); // Uncaught ReferenceError: count is not defined
```

除此以外 ，也可以 显示的定义块作用域，如下代码也能产生预期效果：
```js
var right = true
if (right) {
	{
    let count = 0;
    count ++;
    console.log(count); // 1
  }
  console.log(count); // Uncaught ReferenceError: count is not defined
}
```

## const

除了 let 的声明 ，const 也是 es6 引入的新的变量声明方式，最初学习的时候，我认为用 const 方式声明的变量后，随后就不能被改动了。但慢慢我认识到这是错误的。**const 实际保证的，不是变量的值不能改变，而是变量指向的内存地址不能改动。**因此，在使用复合对象类型的时候需要尤其注意，比如对象和数组
示例片段一：
```js
const count = 0
count++ // count 不可变，报错
console.log(count)
```

示例片段二：
```js
const arr = [0]
arr.push(1)
console.log(arr) // [0, 1]

const arr1 = [0]
arr1 = [2,3] // 报错
```

片段三：
```js
const stu = {
	name: 'yigger',
  sex: 1
}
stu.age = 18
console.log(stu) // {name: "yigger", sex: 1, age: 18}
```

片段四：
```js
const stu = {
	name: 'yigger',
  sex: 1
}
stu.age = 18

// 无法重新赋值，报错
stu = {
	site: 'yigger.cn'
}
```

## try/catch
另外，值得注意的是，try/catch 也会创建一个块作用域，其中声明的变量仅在 catch 内部有效。
```js
try {
	null();	
} catch (e) {
	console.log(e) // TypeError: null is not a function
}
console.log(e) // Uncaught ReferenceError: e is not defined
```

## 变量提升
在最初写 JS 代码的时候，以下代码让我感到非常困惑，我还一度误以为 JS 是自带多线程执行代码的，否则怎么可以在变量/函数声明之前进行调用呢？ 后来才慢慢了解到，原来 JS 有变量提升的机制。
```js
sayHi() // hi, yigger
console.log(myName) // undefined

var myName = 'yigger'
function sayHi() {
	console.log('hi, yigger')
}
```

首先，JavaScript是单线程语言，所以执行肯定是按顺序执行。但是并不是逐行的分析和执行，而是一段一段地分析执行，会先进行编译阶段然后才是执行阶段。

在编译阶段阶段，代码真正执行前的几毫秒，会检测到所有的变量和函数声明，所有这些函数和变量声明都被添加到名为Lexical Environment的JavaScript数据结构内的内存中，所以这些变量和函数能在它们真正被声明之前使用。



- var 变量提升

```js
console.log(myName) // undefined
var myName = 'yigger'
console.log(myName) // yigger

//  由于变量提升，以上代码等同于

var myName;
console.log(myName) // undefined
myName = 'yigger'
console.log(myName) // yigger
```

代码这么写就清晰了许多，具体原因是当 JavaScript 在编译阶段会找到 var 关键字声明的变量会添加到词法环境中，并初始化一个值 undefined，在之后执行代码到赋值语句时，会把值赋值到这个变量

- let/const 的声明会变量提升吗？
答案是会的，事实上所有的声明（function, var, let, const, class）都会被“提升”，那你可能会问 ，既然会提升，下述代码理应会输出 undefined，而不是 error

```js
console.log(myName) // Uncaught ReferenceError: Cannot access 'myName' before initialization
let myName = 'yigger'
```

这是因为在编译阶段，JavaScript引擎遇到变量 myName 并将它存到词法环境中，但因为使用 let  关键字声明的，JavaScript引擎并不会为它初始化值（而 var 会初始为 undefined），所以在编译阶段，此刻的词法环境像这样:
```
lexicalEnvironment = {
  myName: <uninitialized>
}
```

如果我们要在变量声明之前使用变量，JavaScript引擎会从词法环境中获取变量的值，但是变量此时还是uninitialized状态，所以会返回一个错误ReferenceError。


完。

参考链接：
- https://juejin.cn/post/6844903895341219854
- 《你不知道的 JS - 上卷》
