---
title: golang迭代变量
date: 2019-10-08
categories: golang
---
在学习 golang 的过程中，遇到一个比较有意思的问题，也是书中指出的需要重点注意的问题。

```
arr := []int{2,3,4,5}
var ap []func()int
// 循环把求值的函数放入到函数数组
for _, v := range arr {
	ap = append(ap, func() int {
		return v * v
	})
}

// 此处遍历并依次调用函数
for _, v := range ap {
	fmt.Print(v())
}
```

预期结果我想应该是 4,9,16,25 吧？结果实际输出出乎意料，竟然输出了 25 25 25 25

原因是 v 是在 range 的块作用域里面，在循环里`创建的所有函数变量共享相同的变量` - `一个可访问的存储位置，而不是固定的值`

可以简单的输出一下 v 的地址来直观的看待此问题

```
func main() {
	f := test()
	fmt.Println(f())
	fmt.Println(f())

	arr := []int{2,3,4,5}
	var ap []func()int
	for _, v := range arr {
		fmt.Println(&v) // 输出 v 的地址
		ap = append(ap, func() int {
			return v * v
		})
	}

	for _, v := range ap {
		fmt.Print(v())
	}
}
// 输出
0xc00005c0a8
0xc00005c0a8
0xc00005c0a8
0xc00005c0a8
```

可以看到，每次迭代过程， v 的地址都是一样的，所以，匿名函数里面保存的 v 也是一样的，循环结束后，v 最终的值变成了 25 ，所以，在遍历 ap 的时候，输出的结果全都是 25（因为引用的都是同一个变量）

