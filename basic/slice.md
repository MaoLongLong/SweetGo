# 切片

Go 语言中操作切片的函数只有两个：`append` 和 `copy`，很符合 Go 的设计哲学，**“简单”** :innocent:。只需要记忆这两个函数，但是各种各样的切片操作也只能用这两个函数进行组合。`go1.18` 之后应该会出现一些基于泛型的工具包，简化这些操作（比如最近比较火的：[samber/lo](https://github.com/samber/lo)）

> 以下内容基本来自 [Golang 官方 Wiki](https://github.com/golang/go/wiki/SliceTricks)，加了些自己的描述

## AppendVector

切片类型为 `[]byte` 的话，不仅可以追加另一个 `[]byte`，还可以直接追加 `string`

```go
a = append(a, b...)
```

## Copy

错误示范！！！`copy` 是根据 `min(len(dst), len(src))` 进行复制

```go
var b []byte
copy(b, a) // b 永远为空
```

正确示范

```go
b := make([]T, len(a)) // cap 也固定为 len(a)
copy(b, a)

// 这两种方法通常比上一种慢些，但是复制之后如果还有追加操作，
// 它们可能更高效，例如，复制 9 个 byte 它们会提前分配 cap=16
// 接下来的几次追加可能就不用重新扩容
b = append([]T(nil), a...)
b = append(a[:0:0], a...)

// 同第一种方法，固定 cap
b = append(make([]T, 0, len(a)), a...)
```

## Cut

删除下标范围 `[i, j)` 的元素

```go
a = append(a[:i], a[j:]...)
```

## Delete

删除下标为 `i` 的元素

```go
a = append(a[:i], a[i+1:]...)
// or 下标 i 之后的元素前移 1 位
a = a[:i+copy(a[i:], a[i+1:])]

// 如果不用保持原有顺序
a[i] = a[len(a)-1]
a = a[:len(a)-1]
```

> 由于切片的底层就是长度固定的普通数组，上面的 **Cut** 和 **Delete** 方法有潜在的内存泄露问题，底层数组可能依然持有被删除元素的引用，可以通过手动断开引用解决

## Cut (GC)

```go
copy(a[i:], a[j:])
for k, n := len(a)-j+i, len(a); k < n; k++ {
	a[k] = nil
	// 在 go1.18 可以通过泛型获取更通用的 zero value
	// var zero T
	// a[k] = zero
}
a = a[:len(a)-j+i]
```

## Delete (GC)

```go
copy(a[i:], a[i+1:])
a[len(a)-1] = nil
a = a[:len(a)-1]

// 如果不用保持原有顺序
a[i] = a[len(a)-1]
a[len(a)-1] = nil
a = a[:len(a)-1]
```

## Expand

在下标 `i` 处初始化 `n` 个元素

```go
a = append(a[:i], append(make([]T, n), a[i:]...)...)
```

## Extend

在末尾初始化 `n` 个元素

```go
a = append(a, make([]T, n)...)
```

## Filter

刷过力扣应该有印象 :joy:，27 题

```go
n := 0
for _, x := range a {
	if keep(x) {
		a[n] = x
		n++
	}
}
a = a[:n]
```

## Insert

```go
a = append(a[:i], append([]T{x}, a[i:]...)...)
```

第二个 `append` 创建了一个新的切片，`a[i:]` 复制到这个切片上之后，整个新切片又重新复制回 `a`，可以利用其他方法避免额外切片的创建

```go
s = append(s, nil)
copy(s[i+1:], s[i:])
s[i] = x
```

## InsertVector

在下标 `i` 处插入 `n` 个元素

```go
a = append(a[:i], append(b, a[i:]...)...)

// 和 Insert 单个元素，同理，为了避免创建额外的切片，还可以写出更啰嗦的版本
func Insert(s []int, k int, vs ...int) []int {
	// 如果 cap 足够，不用扩容
	if n := len(s) + len(vs); n <= cap(s) {
		s2 := s[:n]
		copy(s2[k+len(vs):], s[k:]) // 后移
		copy(s2[k:], vs) // 插入
		return s2
	}
	s2 := make([]int, len(s) + len(vs))
	copy(s2, s[:k])
	copy(s2[k:], vs)
	copy(s2[k+len(vs):], s[k:])
	return s2
}
```

## Push

```go
a = append(a, x)
```

## Pop

```go
x, a = a[len(a)-1], a[:len(a)-1]
```

## Push Front

```go
a = append([]T{x}, a...)
```

## Pop Front

```go
x, a = a[0], a[1:]
```
