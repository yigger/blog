---
title: javascript - 闭包
date: 2021-03-03
categories: javascript
---

## 闭包的概念
一个函数和对其周围状态（lexical environment，词法环境）的引用捆绑在一起（或者说函数被引用包围），这样的组合就是闭包（closure）。

也就是说，闭包让你可以在一个内层函数中访问到其外层函数的作用域。在 JavaScript 中，每当创建一个函数，闭包就会在函数创建的同时被创建出来。


## 痛点
JS 的作用域分为函数作用域，全局作用域，块作用域。因 JS 的特殊性，在函数作用域内，外层作用域是无法获取到函数内的变量和函数的：

```js
function result() {
    var count = 0;
    console.log(count)
}
// 无法访问 result 函数中的 count
console.log(count) // ReferenceError
```

## 解决方案

假如现在希望在函数外访问 `result` 函数内的 count 呢？可以做到吗？

可以！把代码改为如下形式就可以了

```js
function result() {
    var count = 0;
    function getCount() {
        console.log(count)
        return count;
    }
    return getCount
}

var getCount = result()
getCount() // 0
```

上述代码，(count变量 + getCount函数)就形成了一个闭包，使得外层作用域可以通过某种形式进行读取

## 实际场景
- 面向对象  
在 ES6 之前，JS 都没有“类”的语法的，导致很难写出面向对象的模式，但是，勉强可以借助闭包来实现：
```js
function Student() {
    var name = ''
    var age = 0
    function getName() {
        return name
    }
    function setName(newName) {
        name = newName
    }
    function sayHi() {
        return 'hi, ' + name
    }
    
    return {
        getName: getName,
        setName: setName,
        sayHi: sayHi
    }
}

var stu1 = Student()
var stu2 = Student()
stu1.setName('yigger01')
stu2.setName('yigger02')
console.log(stu1.sayHi()) // hi, yigger01
console.log(stu2.sayHi()) // hi, yigger02
```

虽然并不是特别严谨，但总算是达到了**隐藏变量和方法**的效果，通过这样的“封装”，外部无法直接读取/写入 name 变量，只能通过暴露出来的方法进行变更。

另外，值得一提的是，在一些编程语言中，一个函数中的局部变量仅存在于此函数的执行期间。一旦 setName() 执行完毕，你可能会认为 name 变量将不能再被访问。然而，因为代码仍按预期运行，所以在 JavaScript 中情况显然与此不同。


## 闭包的循环“陷阱”
其实也称不上是陷阱，如果不了解闭包特性的人，踩坑一点也不奇怪，早期用 JQ 一顿写代码的时候也经常遇到过，直接上代码：
```js
function caculate() {
  var nums = ['one', 'two', 'three'];

  for (var i = 0; i < nums.length; i++) {
    setTimeout(function() {
    	console.log(i, nums[i])
    }, 100)
  }
}

caculate()
// 输出:
//   3, undefined
//   3, undefined
//   3, undefined
```

### 原因是什么？
因为 `i` 是用 var 声明的，由于变量的提升，实际上 i 是属于 caculate 的函数作用域，以上代码等同于：
```js
function caculate() {
  var nums = ['one', 'two', 'three'];
  var i
  for (i = 0; i < nums.length; i++) {
    setTimeout(function() {
    	console.log(i, nums[i])
    }, 100)
  }
}

caculate()

// 实际调用栈：
第 1 次循环：
i = 0, 由于 setTimeout，挂起执行`console.log(i, nums[i]`
第 2 次循环：
i = 1, 由于 setTimeout，挂起执行`console.log(i, nums[i]`
第 3 次循环：
i = 2, 由于 setTimeout，挂起执行`console.log(i, nums[i]`
// 执行完最后一次循环后，i++  ---->  i = 3

由于以上代码是立即执行的，所以一瞬间就完成了，可以假设 需要 1ms 的时间

... 随后 cpu 喝了杯咖啡，过去了 99ms ...

随后开始执行定时器中的代码

由于闭包的特性，函数执行完成以后还保留着 i 的引用，而此时的 i = 3

所以，最后实际执行的结果：
console.log(3, nums[3]) // 3, undefined

```

### 解决方案
- 方案 1：使用 let 关键字
```js
function caculate() {
  var nums = ['one', 'two', 'three'];
  for (var i = 0; i < nums.length; i++) {
	let j = i // let 不会存在变量提升
    setTimeout(function() {
    	console.log(j, nums[j])
    }, 100)
  }
}
caculate()
```

- 方案 2：消除对 i 的引用
```js
function caculate() {
  var nums = ['one', 'two', 'three'];
  for (var i = 0; i < nums.length; i++) {
    // 匿名作用域，j 是 i 的复制，不是引用
    (function(j) {
      setTimeout(function() {
        console.log(j, nums[j])
      }, 100)
    })(i)
  }
}
caculate();
```

方案还有挺多，由于 es6 的基本都兼容主流浏览器了，一般都会写成方案 1

```
for (let i = 0; i < nums.length; i++) {
    setTimeout(function() {
    	console.log(i, nums[i])
    }, 100)
  }
```


## 总结
1. 闭包的形成条件：内部函数引用外部函数的变量/参数
2. 闭包的结果：内部函数的使用外部函数的那些变量和参数仍然会保存
3. 返回的函数并非孤立的函数，而是连同周围的环境打了一个包，成了一个封闭的环境包，共同返回出来 ----> 闭包
4. 函数的作用域，取决于声明时而不取决于调用时



参考资料：  
https://juejin.cn/post/6844903777808416776
https://www.cnblogs.com/rubylouvre/p/3345294.html
https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Closures
https://segmentfault.com/a/1190000003818163