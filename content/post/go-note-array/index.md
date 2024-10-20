---
title: "Golang 笔记 - 数组的底层实现"
description: "Golang 数组底层实现的学习笔记"
slug: go-note-array
date: 2024-10-19T19:31:17+08:00
math: true
image:
tags:
  - Golang
  - Golang 笔记
weight: 1
---

## 一些介绍

我们知道，数组是由确定长度的相同类型元素的集合组成的一种数据结构，整体占用一片连续的内存空间。

```go
arr := [5]int{2, 4, 6, 8, 10}
curr := unsafe.Pointer(&arr[0])
end := unsafe.Pointer(uintptr(unsafe.Pointer(&arr[0]))+unsafe.Sizeof(*new(int))*uintptr(5))
for {
	if curr == end {
		break
	}
	print(*(*int)(curr), " ")
	curr = unsafe.Pointer(uintptr(curr) + unsafe.Sizeof(*new(int)))
}	
println()
```

上面的代码可以验证这一点，而对于确定长度和相同类型，可以参照源代码 `cmd/compile/internal/types/type.go` 中的 `types.NewArray` 实现：

```go
/*
 * type Array struct {
 *   Elem      *Type
 *   Bound     int64
 * }
 */

func NewArray(elem *Type, bound int64) *Type {
    if bound < 0 {
        base.Fatalf("NewArray: invalid bound %v", bound)
    }
    t := newType(TARRAY)
    t.extra = &Array{Elem: elem, Bound: bound}
    if elem.HasShape() {
        t.SetHasShape(true)
    }
    if elem.NotInHeap() {
        t.SetNotInHeap(true)
    }
    return t
}
```

且对于 **是否在堆栈中初始化** 也是在编译期就确定了，由数组存储元素的类型决定。

## 数组初始化

### 创建

在 Golang 中，数组有 `[Len]Type{elems...}` 和 `[...]Type{elems...}` 两种创建方式。前者显式的指定长度，后者在编译期推导其长度。

两种方式构造的数组在运行时是完全相同的。

`cmd/compile/internal/typecheck/expr.go` `typecheck.tcCompLit`

```go
func tcCompLit(n *ir.CompLitExpr) (res ir.Node) {
	//...
	t := n.Type()
	//...
	switch t.Kind() {
	default:
	    base.Errorf("invalid composite literal type %v", t)
        n.SetType(nil)
    case types.TARRAY:
        typecheckarraylit(t.Elem(), t.NumElem(), n.List, "array literal")
        n.SetOp(ir.OARRAYLIT)
	//...
	}
	//...
	return n
}
```

最终都会通过 `typecheckarraylit` 函数计算实际长度/检查是否越界。`[...]Type` 只是一种提供便利的语法糖。

### 优化

编译器会根据数组元素数量的范围进行不同的优化：


- $len > 4$ 时，先获取一个保证唯一的 `staticname`，在静态存储区上初始化数组的元素，然后拷贝到栈上。
  `cmd/compile/internal/walk/complit.go` `walk.anylit`
  ```go
  func anylit(n ir.Node, var_ ir.Node, init *ir.Nodes) {
	  t := n.Type()
	  switch t.Kind() {
	  //...
      case ir.OSTRUCTLIT, ir.OARRAYLIT:
		  n := n.(*ir.CompLitExpr)
		  if !t.IsStruct() && !t.IsArray() {
			  base.Fatalf("anylit: not struct/array")
		  }

		  if isSimpleName(var_) && len(n.List) > 4 {
			  // lay out static data
			  vstat := readonlystaticname(t)

			  ctxt := inInitFunction
			  if n.Op() == ir.OARRAYLIT {
				  ctxt = inNonInitFunction
			  }
			  fixedlit(ctxt, initKindStatic, n, vstat, init)

			  // copy static to var
			  appendWalkStmt(init, ir.NewAssignStmt(base.Pos, var_, vstat))

			  // add expressions to automatic
			  fixedlit(inInitFunction, initKindDynamic, n, var_, init)
			  break
		  } 
        //...
	  }
  }
  ``` 
  可以理解为下面这样：
  ```go
  var arr [5]int
  //...
  staticname_var_0[0] = 1
  staticname_var_0[1] = 2
  staticname_var_0[2] = 3
  staticname_var_0[3] = 4
  staticname_var_0[4] = 5
  //...
  arr = staticname_var_0
  ```

- $len \leq 4$ 时，将其转化为更原始的语句，并将元素直接放在栈上。
  `cmd/compile/internal/walk/complit.go` `walk.anylit`
  ```go
  func anylit(n ir.Node, var_ ir.Node, init *ir.Nodes) {
	  t := n.Type()
	  switch t.Kind() {
	  //...
      case ir.OSTRUCTLIT, ir.OARRAYLIT:
		  n := n.(*ir.CompLitExpr)
		  if !t.IsStruct() && !t.IsArray() {
			  base.Fatalf("anylit: not struct/array")
		  }

		  if isSimpleName(var_) && len(n.List) > 4 {
			  //...
			  break
		  }
		  var components int64
		  if n.Op() == ir.OARRAYLIT {
			  components = t.NumElem()
		  } else {
			  components = int64(t.NumFields())
		  }
		  // initialization of an array or struct with unspecified components (missing fields or arrays)
		  if isSimpleName(var_) || int64(len(n.List)) < components {
			  appendWalkStmt(init, ir.NewAssignStmt(base.Pos, var_, nil))
		  }

		  fixedlit(inInitFunction, initKindLocalCode, n, var_, init)
	  }
  }
  ``` 
  
  `cmd/compile/internal/walk/complit.go` `walk.fixedlit`
  ```go
  func fixedlit(ctxt initContext, kind initKind, n *ir.CompLitExpr, var_ ir.Node, init *ir.Nodes) {
	  isBlank := var_ == ir.BlankNode
	  var splitnode func(ir.Node) (a ir.Node, value ir.Node)
	  //...

	  for _, r := range n.List {
		  a, value := splitnode(r)
		  if a == ir.BlankNode && !staticinit.AnySideEffects(value) {
			  // Discard.
			  continue
		  }

		  islit := ir.IsConstNode(value)
		  if (kind == initKindStatic && !islit) || (kind == initKindDynamic && islit) {
			  continue
		  }

		  // build list of assignments: var[index] = expr
		  ir.SetPos(a)
		  as := ir.NewAssignStmt(base.Pos, a, value)
		  as = typecheck.Stmt(as).(*ir.AssignStmt)
		  switch kind {
		  case initKindStatic:
			  genAsStatic(as)
            case initKindDynamic, initKindLocalCode:
			  appendWalkStmt(init, orderStmtInPlace(as, map[string][]*ir.Name{}))
            default:
			  base.Fatalf("fixedlit: bad kind %d", kind)
		  }
	  }
  }
  ```
  可以理解为下面这样：
  ```go
  var arr [4]int
  arr[0] = 1
  arr[1] = 2
  arr[2] = 3
  arr[3] = 4
  ```
  
## 访问 & 赋值

介绍部分的代码已经有所验证：数组是一片连续的内存空间(无论是在栈上还是静态存储区)，已知首指针、长度、元素类型的size就可以确定整个数组。


### 访问
通过索引访问数组元素时，会基于首地址加上索引乘以元素大小快速定位到特定元素。

Golang 在编译期以及运行时都会进行越界检查，确保访问合法。

### 赋值

同理，当我们对数组中的某个元素赋值时，新值直接存储到计算的对应内存地址。

需要注意的是，数组赋值给另一个数组时会复制整个数组，因为数组是值类型。这意味着每次赋值都会创建一个新的数组副本。

同样在其传递时也会复制数组本身，采用值传递的方式。