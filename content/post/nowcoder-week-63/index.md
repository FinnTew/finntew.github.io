---
title: 题解 - 牛客周赛 63
description: 牛客周赛 63 题解记录
slug: nowcoder-week-63
date: 2024-10-14T20:37:13+08:00
math: true
image:
tags:
  - 题解记录
weight: 1
---

# 比赛连接： [牛客周赛 63](https://ac.nowcoder.com/acm/contest/91592)

## A. 小红的好数

### 思路

签到，按字符串读入，先判断位数然后判断两位是否相等即可。

### 代码

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

var (
	reader = bufio.NewReader(os.Stdin)
	writer = bufio.NewWriter(os.Stdout)
)

func Solve() {
	defer func() {
		_ = writer.Flush()
	}()
	var x string
	_, _ = fmt.Fscan(reader, &x)
	if len(x) == 2 && x[0] == x[1] {
		_, _ = fmt.Fprint(writer, "Yes")
	} else {
		_, _ = fmt.Fprint(writer, "No")
	}
}

func main() {
	T := 1
	//_, _ = fmt.Fscan(reader, &T)
	for i := 0; i < T; i++ {
		Solve()
	}
}
```

## B. 小红的好数组

### 思路

n, k 范围都只有 $[1, 1000]$，枚举左端点 $l \in [0, n-k+1]$, 然后对于所有 $l$ 判断 $s[l:l+k]$ 是否可以恰好修改一位使得它回文即可。

### 代码

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

var (
	reader = bufio.NewReader(os.Stdin)
	writer = bufio.NewWriter(os.Stdout)
)

func Solve() {
	defer func() {
		_ = writer.Flush()
	}()
	var n, k int
	_, _ = fmt.Fscan(reader, &n, &k)
	arr := make([]int, n)
	for i := 0; i < n; i++ {
		_, _ = fmt.Fscan(reader, &arr[i])
	}
	res := 0
	for i := 0; i < n-k+1; i++ {
		l, r := i, i+k-1
		cnt := 0
		for l < r {
			if arr[l] != arr[r] {
				cnt += 1
			}
			l += 1
			r -= 1
		}
		if cnt == 1 {
			res += 1
		}
	}
	_, _ = fmt.Fprintf(writer, "%d\n", res)
}

func main() {
	T := 1
	//_, _ = fmt.Fscan(reader, &T)
	for i := 0; i < T; i++ {
		Solve()
	}
}
```

## C. 小红的矩阵行走

### 思路

问能否从左上角走到右下角，且下一个格子颜色必须与当前颜色一致。

bfs 模拟题目要求即可，从 $(0, 0)$ 开始，每次向颜色相同的格子走，看是否可以到达 $(n-1, m-1)$ 即可。

### 代码

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

var (
	reader = bufio.NewReader(os.Stdin)
	writer = bufio.NewWriter(os.Stdout)
)

func Solve() {
	defer func() {
		_ = writer.Flush()
	}()
	var n, m int
	_, _ = fmt.Fscan(reader, &n, &m)
	a := make([][]int, n)
	for i := 0; i < n; i++ {
		a[i] = make([]int, m)
		for j := 0; j < m; j++ {
			_, _ = fmt.Fscan(reader, &a[i][j])
		}
	}
	q := [][2]int{{0, 0}}
	for len(q) > 0 {
		x, y := q[0][0], q[0][1]
		if x == n-1 && y == m-1 {
			_, _ = fmt.Fprintf(writer, "Yes\n")
			return
		}
		q = q[1:]
		for _, d := range [][2]int{{1, 0}, {0, 1}} {
			nx, ny := x+d[0], y+d[1]
			if nx < n && ny < m && a[nx][ny] == a[x][y] {
				q = append(q, [2]int{nx, ny})
			}
		}
	}
	_, _ = fmt.Fprintf(writer, "No\n")
}

func main() {
	T := 1
	_, _ = fmt.Fscan(reader, &T)
	for i := 0; i < T; i++ {
		Solve()
	}
}
```

## D. 小红的行列式构造

### 思路

要构造一个值为 x 且**所有元素都不为0**的三阶行列式。

换个想法，我们可以构造任意一个值为 1 的三阶行列式，然后乘以一个常数 x 即可。

比如 $$ A = \left | \begin{matrix} 1 & 1 & 1 \\ 1 & 2 & 1 \\ 1 & 1 & 2 \end{matrix} \right | = 1$$

我们构造 $ x \times A $ 即可，注意特判 $x = 0$ 时输出一个元素全部为 1 的行列式。

### 代码

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

var (
	reader = bufio.NewReader(os.Stdin)
	writer = bufio.NewWriter(os.Stdout)
)

func Solve() {
	defer func() {
		_ = writer.Flush()
	}()
	var x int
	_, _ = fmt.Fscan(reader, &x)
	if x == 0 {
		_, _ = fmt.Fprint(writer, "1 1 1\n1 1 1\n1 1 1\n")
	} else {
		_, _ = fmt.Fprintf(writer, "%d %d %d\n1 2 1\n1 1 2\n", x, x, x)
	}
}

func main() {
	T := 1
	//_, _ = fmt.Fscan(reader, &T)
	for i := 0; i < T; i++ {
		Solve()
	}
}
```

## E. 小红的 red 计数

说是线段树balabala的，待补...

## F. 小红开灯

### 思路

首先，按照题意分别记录下所有同行/列的坐标，方便变换操作。

对于每盏灯，我们按下他的同时，所有相关的灯置1，其他位置都置0，可以得到这个灯的状态01串。

我们设 $ S_f[i] = S[i] \oplus 1 $，则题目可以转化为，能否用求出的所有状态串凑出 $ S_f $。

使用线性基维护所有状态，最后验证答案即可。

### 代码

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"strings"
)

var (
	reader = bufio.NewReader(os.Stdin)
	writer = bufio.NewWriter(os.Stdout)
)

const B = 100

type Bitset struct {
	bits [B + 1]bool
}

func (bs *Bitset) Reset() {
	for i := range bs.bits {
		bs.bits[i] = false
	}
}

func (bs *Bitset) Set(i int) {
	bs.bits[i] = true
}

func (bs *Bitset) Xor(other Bitset) {
	for i := range bs.bits {
		bs.bits[i] = bs.bits[i] != other.bits[i]
	}
}

func (bs *Bitset) HasBits() bool {
	for _, bit := range bs.bits {
		if bit {
			return true
		}
	}
	return false
}

type LinerBase struct {
	a   []Bitset
	b   [][]int
	cnt int
}

func NewLinerBase() *LinerBase {
	return &LinerBase{
		a: make([]Bitset, B+1),
		b: make([][]int, B+1),
	}
}

func (lb *LinerBase) Insert(x Bitset, id int) int {
	v := []int{id}
	for i := B; i >= 0; i-- {
		if x.bits[i] {
			if !lb.a[i].HasBits() {
				lb.a[i] = x
				lb.b[i] = v
				lb.cnt++
				return 1
			} else {
				x.Xor(lb.a[i])
				v = append(v, lb.b[i]...)
			}
		}
	}
	return 0
}

func (lb *LinerBase) Size() int {
	return lb.cnt
}

func (lb *LinerBase) Ask(x Bitset) []int {
	var res []int
	for i := B; i >= 0; i-- {
		if x.bits[i] {
			x.Xor(lb.a[i])
			res = append(res, lb.b[i]...)
		}
	}
	if x.HasBits() {
		return nil
	}
	return res
}

func Solve() {
	defer func() {
		_ = writer.Flush()
	}()
	var n int
	_, _ = fmt.Fscan(reader, &n)
	var str string
	_, _ = fmt.Fscan(reader, &str)
	str = " " + str
	xArr := make(map[int][]int)
	yArr := make(map[int][]int)
	p := make([][2]int, n+1)
	for i := 1; i <= n; i++ {
		var x, y int
		_, _ = fmt.Fscan(reader, &x, &y)
		xArr[x] = append(xArr[x], i)
		yArr[y] = append(yArr[y], i)
		p[i] = [2]int{x, y}
	}
	bs1 := Bitset{}
	lb := NewLinerBase()
	for i := 1; i <= n; i++ {
		x, y := p[i][0], p[i][1]
		bs1.Reset()
		for _, j := range xArr[x] {
			bs1.Set(j)
		}
		for _, j := range yArr[y] {
			bs1.Set(j)
		}
		lb.Insert(bs1, i)
	}
	bs2 := Bitset{}
	for i := 1; i <= n; i++ {
		if str[i] == '0' {
			bs2.Set(i)
		}
	}
	if strings.Count(str[1:], "1") == n {
		_, _ = fmt.Fprintln(writer, "0")
		return
	}
	res := lb.Ask(bs2)
	if res == nil {
		_, _ = fmt.Fprintln(writer, "-1")
	} else {
		ans := make([]int, n+1)
		for _, i := range res {
			ans[i] ^= 1
		}
		sum := 0
		for i := 1; i <= n; i++ {
			if ans[i] != 0 {
				sum++
			}
		}
		_, _ = fmt.Fprintln(writer, sum)
		for i := 1; i <= n; i++ {
			if ans[i] != 0 {
				_, _ = fmt.Fprintf(writer, "%d ", i)
			}
		}
		_, _ = fmt.Fprintln(writer)
	}
}

func main() {
	T := 1
	//_, _ = fmt.Fscan(reader, &T)
	for i := 0; i < T; i++ {
		Solve()
	}
}
```