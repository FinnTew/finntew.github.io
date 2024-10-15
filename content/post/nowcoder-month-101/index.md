---
title: 题解 - 牛客小白月赛 101
description: 牛客小白月赛 101 题解记录
slug: nowcoder-month-101
date: 2024-10-15T14:55:25+08:00
math: true
image:
tags:
  - 题解记录
weight: 1
---

# 比赛链接：[牛客小白月赛 101](https://ac.nowcoder.com/acm/contest/90072)

## A. tb的区间问题

### 思路

签到，问题等价于求长度为k的连续子数组的最大和，预处理前缀和然后枚举左端点即可。

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

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}

func Solve() {
	defer func() {
		_ = writer.Flush()
	}()
	var n, k int
	_, _ = fmt.Fscan(reader, &n, &k)
	arr := make([]int, n+1)
	preSum := make([]int, n+1)
	for i := 1; i <= n; i++ {
		_, _ = fmt.Fscan(reader, &arr[i])
		preSum[i] = preSum[i-1] + arr[i]
	}
	res := 0
	k = n - k
	for i := k + 1; i <= n; i++ {
		res = max(res, preSum[i]-preSum[i-k])
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

## B. tb的字符串问题

### 思路

类似括号匹配问题，维护一个栈，如果栈顶元素为`f`且当前字符为`c`，则出栈，否则入栈，"tb"同理。

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
	var (
		n   int
		str string
	)
	_, _ = fmt.Fscan(reader, &n)
	_, _ = fmt.Fscan(reader, &str)
	stk := make([]rune, 0, n)
	for _, char := range str {
		length := len(stk)
		if length >= 1 {
			last := stk[length-1]
			if (last == 'f' && char == 'c') || (last == 't' && char == 'b') {
				stk = stk[:length-1]
				continue
			}
		}
		stk = append(stk, char)
	}
	_, _ = fmt.Fprintf(writer, "%v\n", len(stk))
}

func main() {
	T := 1
	//_, _ = fmt.Fscan(reader, &T)
	for i := 0; i < T; i++ {
		Solve()
	}
}
```

## C. tb的路径问题

### 思路

规律题，手玩一下样例，由 $gcd(a, b) = gcd(a, a-b)$，最近的路径为 $(1, 1) \rightarrow (2, 2) \rightarrow (2k-2, 2k)$，特判一下$n$比较小的情况。

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
	var n int
	_, _ = fmt.Fscan(reader, &n)
	if n <= 3 {
		_, _ = fmt.Fprintln(writer, (n - 1) * 2)
		return
	}
	if n % 2 == 0 {
		_, _ = fmt.Fprintln(writer, 4)
		return
	}
	_, _ = fmt.Fprintln(writer, 6)
}

func main() {
	T := 1
	//_, _ = fmt.Fscan(reader, &T)
	for i := 0; i < T; i++ {
		Solve()
	}
}
```

## D. tb的平方问题

### 思路

题目问和为完全平方数且包含 $x$ 的区间个数，所以需要处理两个信息：完全平方数，对应区间。

完全平方数我们可以 $O(\sqrt{Max(a)})$ 的求出。

而对于区间，注意到 $n \in [1, 1000]$，可以直接枚举，复杂度为 $O(n^2)$。 

但是显然不能在线的去回答每次询问，考虑预处理，维护一个差分数组 $diff$，我们预处理完全平方数以及$a$的前缀和，然后枚举所有区间，对于每个区间，判断其和是否为完全平方数，是则更新差分数组 $[l, r]$ 区间。

对于每个询问的答案即为 $diff[x]$。

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
	var n, q int
	_, _ = fmt.Fscan(reader, &n, &q)
	arr := make([]int, n+1)
	preSum := make([]int, n+1)
	for i := 1; i <= n; i++ {
		_, _ = fmt.Fscan(reader, &arr[i])
		preSum[i] = preSum[i-1] + arr[i]
	}
	vis := make(map[int]struct{})
	val := 1
	for val*val <= preSum[n] {
		vis[val*val] = struct{}{}
		val += 1
	}
	diff := make([]int, n+1)
	insert := func(l, r int) {
		diff[l] += 1
		if r < n {
			diff[r+1] -= 1
		}
	}
	for l := 1; l <= n; l++ {
		for r := l; r <= n; r++ {
			sum := preSum[r] - preSum[l-1]
			if _, ok := vis[sum]; ok {
				insert(l, r)
			}
		}
	}
	for i := 1; i <= n; i++ {
		diff[i] += diff[i-1]
	}
	for q > 0 {
		var x int
		_, _ = fmt.Fscan(reader, &x)
		_, _ = fmt.Fprintln(writer, diff[x])
		q -= 1
	}
}

func main() {
	T := 1
// 	_, _ = fmt.Fscan(reader, &T)
	for i := 0; i < T; i++ {
		Solve()
	}
}
```

## E. tb的数数问题

### 思路

找到所有没有出现过的数及其倍数，除去这些数，剩余的数的个数即为答案。

类似埃氏筛写法，时间复杂度为调和级数。

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

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}

func Solve() {
	defer func() {
		_ = writer.Flush()
	}()
	var n int
	_, _ = fmt.Fscan(reader, &n)
	arr := make([]int, n+1)
	vis := make(map[int]struct{})
	flag := false
	maxVal := -1
	for i := 1; i <= n; i++ {
		_, _ = fmt.Fscan(reader, &arr[i])
		if arr[i] == 1 {
			flag = true
		}
		maxVal = max(maxVal, arr[i])
		vis[arr[i]] = struct{}{}
	}
	if !flag {
		_, _ = fmt.Fprintln(writer, "0")
		return
	}
	f := make(map[int]struct{})
	for i := 2; i <= maxVal; i++ {
		_, isVis := vis[i]
		_, isF := f[i]
		if !isVis && !isF {
			for j := 2*i; j <= maxVal; j += i {
				delete(vis, j)
				f[j] = struct{}{}
			}
		}
	}
	res := 0
	for i := 1; i <= maxVal; i++ {
		if _, ok := vis[i]; ok {
			res += 1
		}
	}
	_, _ = fmt.Fprintln(writer, res)
}

func main() {
	T := 1
	//_, _ = fmt.Fscan(reader, &T)
	for i := 0; i < T; i++ {
		Solve()
	}
}
```
