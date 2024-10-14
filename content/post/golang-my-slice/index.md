---
title: Golang 底层原理 - Slice 的底层原理 / 简单实现
description: Golang Slice 的学习笔记，包括 切片的底层结构、和数组的区别、扩容机制、以及简单的代码实现等
slug: golang-my-slice
date: 2024-10-07T19:16:47+08:00
math: true
image: 
tags: 
  - Golang
  - Golang底层原理
weight: 1
---

## 一些介绍 ～

### 底层结构

切片的底层结构如下：

```go
type slice struct {
	array unsafe.Pointer // 指向底层数组的指针
	len   int            // 长度/当前元素个数
	cap   int            // 容量/最大元素个数
}
```

可以看到 *slice* 本质上是从**数组**抽象出来的一个复合类型

### 切片和数组的区别

这种设计使得切片相对数组更加灵活高效：

- 长度、容量可以动态变化，可以追加元素，动态扩容
- 切片是一个引用类型，在传递时采用浅拷贝，只复制*len*和*cap*，而使用同一个底层数组
- 操作过程中更新*len*和*cap*，做到 $O(1)$ 的计算长度，而不像数组需要遍历计算长度

### 扩容机制

我们刚在提到切片可以追加元素，动态扩容

所谓的扩容大致做法就是：当前长度加上追加元素的数量大于当前容量的时候，重新开一个新的满足长度要求的数组指针，将原数组元素依次拷贝到新数组后，再追加新的元素

值的一提的是对于确定扩容后容量的做法，在1.18版本以前和1.18版本及以后版本的实现稍有不同：

#### 1.18 版本之前

- 期望的新容量大于当前容量的**2倍**：直接扩充为新容量
- 否则：
  - 原容量小于**1024**：扩充为原来的**2倍**
  - 原容量大于**1024**：循环若干次，每次扩充为原来的**1.25倍**，直到新容量大于等于期望容量为止

**代码：**

```go
newCap := s.capacity
doubleCap := newCap + newCap
if expCap > doubleCap {
	newCap = expCap
} else {
	if s.capacity < 1024 {
		newCap = doubleCap
	} else {
		for 0 < newCap && newCap < expCap {
			newCap += newCap / 4
		}
		if newCap <= 0 {
			newCap = expCap
		}
	}
}
```

由于这种扩容方式容量扩大不太平滑，循环扩容时会出现后一次的计算值比前一次的计算值小的情况，而且扩容后容量和长度差距过大

所以在1.18版本之后，修改为了另一种更平滑的扩容方式，容量和长度也更加接近

#### 1.18 版本及之后版本

- 期望的新容量大于当前容量的**2倍**：直接扩充为新容量
- 否则：
  - 原容量小于**256**：扩充为原来的**2倍**
  - 原容量大于**256**：循环若干次，每次扩充增加 **(旧容量+3*256)/4**，直到新容量大于等于期望容量为止

**代码：**

```go
newCap := s.capacity
doubleCap := newCap + newCap
if expCap > doubleCap {
    newCap = expCap
} else {
    if s.capacity < 256 {
        newCap = doubleCap
    } else {
        for 0 < newCap && newCap < expCap {
            newCap += (newCap + 3*256) / 4
		}
        if newCap <= 0 {
            newCap = expCap
        }
    }
}
```

## 手写实现 ～

### 结构定义 & 构造 ...

```go
type Slice[T any] struct {
	array    unsafe.Pointer // 底层数组
	length   int            // 长度
	capacity int            // 容量
}

func NewSlice[T any](capacity int) *Slice[T] {
    arrPtr := unsafe.Pointer(&make([]T, capacity)[0])
    return &Slice[T]{
        array:    arrPtr,
		length:   0,
        capacity: capacity,
    }
}

func (s *Slice[T]) Len() int {
    return s.length
}

func (s *Slice[T]) Cap() int {
    return s.capacity
}

func (s *Slice[T]) Array() []T {
    return (*(*[]T)(unsafe.Pointer(&s.array)))[:s.length]
}
```

### 追加元素

```go
func (s *Slice[T]) Append(elems ...T) {
	if s.length >= s.capacity {
		s.grow(s.length + 1) // 扩容
	}
	for i := 0; i < len(elems); i++ { // 追加元素
		elemPtr := unsafe.Pointer(uintptr(s.array) + uintptr(s.length)*unsafe.Sizeof(*new(T)))
		*(*T)(elemPtr) = elems[i]
		s.length += 1 // 更新长度
	}
}
```

### 扩容

```go
func (s *Slice[T]) grow(expCap int) {
	newCap := s.capacity
	doubleCap := newCap + newCap
	// 1.18 以前
	//if expCap > doubleCap {
	//	newCap = expCap
	//} else {
	//	if s.capacity < 1024 {
	//		newCap = doubleCap
	//	} else {
	//		for 0 < newCap && newCap < expCap {
	//			newCap += newCap / 4
	//		}
	//		if newCap <= 0 {
	//			newCap = expCap
	//		}
	//	}
	//}

	// 1.18 及以后
	// 计算新容量
	if expCap > doubleCap {
		newCap = expCap
	} else {
		if s.capacity < 256 {
			newCap = doubleCap
		} else {
			for 0 < newCap && newCap < expCap {
				newCap += (newCap + 3*256) / 4
			}
			if newCap <= 0 {
				newCap = expCap
			}
		}
	}
	// 创新新的底层数组
	newArrPtr := unsafe.Pointer(&make([]T, newCap)[0])
	// 拷贝元素
	for i := 0; i < s.length; i++ {
		oldElemPtr := unsafe.Pointer(uintptr(s.array) + uintptr(i)*unsafe.Sizeof(*new(T)))
		newElemPtr := unsafe.Pointer(uintptr(newArrPtr) + uintptr(i)*unsafe.Sizeof(*new(T)))
		*(*T)(newElemPtr) = *(*T)(oldElemPtr)
	}
	// 更新指针
	s.array = newArrPtr
	s.capacity = newCap
}
```

### 取值 & 修改

```go
func (s *Slice[T]) Get(idx int) T {
	if idx < 0 || idx >= s.length {
		panic("index out of range")
	}
	elemPtr := unsafe.Pointer(uintptr(s.array) + uintptr(idx)*unsafe.Sizeof(*new(T)))
	return *(*T)(elemPtr)
}

func (s *Slice[T]) Set(idx int, elem T) {
	if idx < 0 || idx >= s.length {
		panic("index out of range")
	}
	elemPtr := unsafe.Pointer(uintptr(s.array) + uintptr(idx)*unsafe.Sizeof(*new(T)))
	*(*T)(elemPtr) = elem
}
```

### 截取 & 删除


```go
func (s *Slice[T]) Slice(leftBound, rightBound int) *Slice[T] {
    // 检查区间合法性
    if leftBound < 0 || rightBound > s.length || leftBound > rightBound {
        panic("invalid slice bounds")
    }
    // 创建新的切片
    return &Slice[T]{
        // 这里只需要将向左边界偏移后的指针设为新切片的起始指针即可
        array:    unsafe.Pointer(uintptr(s.array) + uintptr(leftBound)*unsafe.Sizeof(*new(T))),
        length:   rightBound - leftBound,
        capacity: s.capacity - leftBound,
    }
}

func (s *Slice[T]) Del(idx int) {
    if idx < 0 || idx >= s.length {
        panic("index out of range")
    }
    // 删除操作：拼接 [0, idx) 和 (idx, s.length-1]两个切片即可
    newArrPtr := s.Slice(0, idx)
    newArrPtr.Append(s.Slice(idx+1, s.length).Array()...)
    s.array = newArrPtr.array
    s.length -= 1
    s.capacity = newArrPtr.capacity
}
```

### 完整代码

```go
package my

import "unsafe"

type Slice[T any] struct {
	array    unsafe.Pointer // 底层数组
	length   int            // 长度
	capacity int            // 容量
}

func NewSlice[T any](capacity int) *Slice[T] {
	arrPtr := unsafe.Pointer(&make([]T, capacity)[0])
	return &Slice[T]{
		array:    arrPtr,
		length:   0,
		capacity: capacity,
	}
}

func (s *Slice[T]) Len() int {
	return s.length
}

func (s *Slice[T]) Cap() int {
	return s.capacity
}

func (s *Slice[T]) Array() []T {
	return (*(*[]T)(unsafe.Pointer(&s.array)))[:s.length]
}

func (s *Slice[T]) Append(elems ...T) {
	if s.length >= s.capacity {
		s.grow(s.length + 1)
	}
	for i := 0; i < len(elems); i++ {
		elemPtr := unsafe.Pointer(uintptr(s.array) + uintptr(s.length)*unsafe.Sizeof(*new(T)))
		*(*T)(elemPtr) = elems[i]
		s.length += 1
	}
}

func (s *Slice[T]) grow(expCap int) {
	newCap := s.capacity
	doubleCap := newCap + newCap
	// 1.18 以前
	//if expCap > doubleCap {
	//	newCap = expCap
	//} else {
	//	if s.capacity < 1024 {
	//		newCap = doubleCap
	//	} else {
	//		for 0 < newCap && newCap < expCap {
	//			newCap += newCap / 4
	//		}
	//		if newCap <= 0 {
	//			newCap = expCap
	//		}
	//	}
	//}

	// 1.18 及以后
	if expCap > doubleCap {
		newCap = expCap
	} else {
		if s.capacity < 256 {
			newCap = doubleCap
		} else {
			for 0 < newCap && newCap < expCap {
				newCap += (newCap + 3*256) / 4
			}
			if newCap <= 0 {
				newCap = expCap
			}
		}
	}
	newArrPtr := unsafe.Pointer(&make([]T, newCap)[0])
	for i := 0; i < s.length; i++ {
		oldElemPtr := unsafe.Pointer(uintptr(s.array) + uintptr(i)*unsafe.Sizeof(*new(T)))
		newElemPtr := unsafe.Pointer(uintptr(newArrPtr) + uintptr(i)*unsafe.Sizeof(*new(T)))
		*(*T)(newElemPtr) = *(*T)(oldElemPtr)
	}
	s.array = newArrPtr
	s.capacity = newCap
}

func (s *Slice[T]) Get(idx int) T {
	if idx < 0 || idx >= s.length {
		panic("index out of range")
	}
	elemPtr := unsafe.Pointer(uintptr(s.array) + uintptr(idx)*unsafe.Sizeof(*new(T)))
	return *(*T)(elemPtr)
}

func (s *Slice[T]) Set(idx int, elem T) {
	if idx < 0 || idx >= s.length {
		panic("index out of range")
	}
	elemPtr := unsafe.Pointer(uintptr(s.array) + uintptr(idx)*unsafe.Sizeof(*new(T)))
	*(*T)(elemPtr) = elem
}

func (s *Slice[T]) Slice(leftBound, rightBound int) *Slice[T] {
	if leftBound < 0 || rightBound > s.length || leftBound > rightBound {
		panic("invalid slice bounds")
	}
	return &Slice[T]{
		array:    unsafe.Pointer(uintptr(s.array) + uintptr(leftBound)*unsafe.Sizeof(*new(T))),
		length:   rightBound - leftBound,
		capacity: s.capacity - leftBound,
	}
}

func (s *Slice[T]) Del(idx int) {
	if idx < 0 || idx >= s.length {
		panic("index out of range")
	}
	newArrPtr := s.Slice(0, idx)
	newArrPtr.Append(s.Slice(idx+1, s.length).Array()...)
	s.array = newArrPtr.array
	s.length -= 1
	s.capacity = newArrPtr.capacity
}

// 测试
package main

import (
    "fmt"
    "some-go-demos/pkgs/my"
    "strconv"
)

func main() {
	mySlice := my.NewSlice[string](2)
	for i := 0; i < 10; i++ {
		mySlice.Append(strconv.Itoa(i))
	}
	fmt.Println(mySlice.Get(5))
	fmt.Println(mySlice.Slice(3, 6).Array())
	mySlice.Del(5)
	fmt.Println(mySlice.Slice(3, 6).Array())
	mySlice.Set(5, "hello")
	fmt.Println(mySlice.Slice(3, 6).Array())
	fmt.Println("Len: ", mySlice.Len(), " Cap: ", mySlice.Cap())
}

```