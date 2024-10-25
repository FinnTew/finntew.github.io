---
title: 题解 - 牛客小白月赛 103
description: 牛客小白月赛 103 题解记录
slug: nowcoder-month-103
date: 2024-10-25T21:32:40+08:00
math: true
image:
tags:
  - 题解记录
weight: 1
---

# 比赛链接：[牛客小白月赛 103](https://ac.nowcoder.com/acm/contest/93218)

## A. 冰冰的正多边形

### 思路

签到，选择任意数量的木棍拼多边形，显然周长最小的是正三角形，所以记录每种长度木棍的数量，答案即为 $Min_{i=1}^{n} {3*Len_i} (Cnt_{Len_i} \geq 3)$，如果没有数量大于等于3的则为"no"，注意特判 $ n \leq 2 $。

### 代码

```go
func Solve() {
	defer func() {
		_ = writer.Flush()
	}()
	var n int
	_, _ = fmt.Fscan(reader, &n)
	cnt := make(map[int]int)
	for i := 0; i < n; i++ {
		var x int
		_, _ = fmt.Fscan(reader, &x)
		cnt[x] += 1
	}
	if n <= 2 {
		_, _ = fmt.Fprintf(writer, "no\n")
		return
	}
	res := 100000000
	for k, v := range cnt {
		if v < 3 {
			continue
		}
		res = min(res, k * 3)
	}
	if res == 100000000 {
		_, _ = fmt.Fprintf(writer, "no\n")
		return
	}
	_, _ = fmt.Fprintf(writer, "yes\n%d\n", res)
}
```

## B. 冰冰的电子邮箱

### 思路

模拟题，将 $s$ 按 "@" 分割为两个部分，分别检查 *local_part* 和 *domain* 部分是否合法即可。

### 代码

```go
func Check(email string) bool {
	p := strings.Split(email, "@")
	if len(p) != 2 {
		return false
	}
	lp, dm := p[0], p[1]
	if len(lp) < 1 || len(lp) > 64 {
		return false
	}
	if len(dm) < 1 || len(dm) > 255 {
		return false
	}
	if !CheckLP(lp) || !CheckDM(dm) {
		return false
	}
	return true
}

func CheckLP(lp string) bool {
	if lp[0] == '.' || lp[len(lp)-1] == '.' {
		return false
	}
	for _, ch := range lp {
		if !unicode.IsLetter(ch) && !unicode.IsDigit(ch) && ch != '.' {
			return false
		}
	}
	return true
}

func CheckDM(dm string) bool {
	if dm[0] == '.' || dm[len(dm)-1] == '.' || dm[0] == '-' || dm[len(dm)-1] == '-' {
		return false
	}
	for _, ch := range dm {
		if !unicode.IsLetter(ch) && !unicode.IsDigit(ch) && ch != '.' && ch != '-' {
			return false
		}
	}
	return true
}

func Solve() {
	defer func() {
		_ = writer.Flush()
	}()
	var s string
	_, _ = fmt.Fscan(reader, &s)
	if Check(s) {
		fmt.Fprintln(writer, "Yes")
	} else {
		fmt.Fprintln(writer, "No")
	}
}
```

## C. 冰冰的异或

### 思路

找规律，模拟题目操作打表可以发现:

- $ n \leq 2 $ 时，答案为 1。
- $ n > 2$ 时，答案为大于等于 n 的第一个2的幂。

打表代码：

```go
func Test() {
	defer func() {
		_ = writer.Flush()
	}()
	for n := 1; n <= 100; n++ {
		vis := make(map[int]struct{})
		for i := 1; i <= n; i++ {
			for j := 1; j <= n; j++ {
				vis[i^j] = struct{}{}
			}
		}
		for i := 0; i <= n*2; i++ {
			if _, ok := vis[i]; !ok {
				_, _ = fmt.Fprintf(writer, "N = %d, Ans = %d\n", n, i)
				break
			}
		}
	}
}
```

### 代码

```go
func Solve() {
	defer func() {
		_ = writer.Flush()
	}()
	var n int
	_, _ = fmt.Fscan(reader, &n)
	if n <= 2 {
		_, _ = fmt.Fprintln(writer, 1)
	} else {
		_, _ = fmt.Fprintln(writer, 1<<bits.Len(uint(n-1)))
	}
}
```

## D. 冰冰的分界线

### 思路

易知，题目要求的分界线即为两点连线的中垂线，考虑到实数精度问题，考虑计算点集里点两两之间的中垂线方程的一般式，用 `map` 去重，其长度即为答案。

### 代码

```go
type Line struct {
	a, b, c int
}

func gcd(a, b int) int {
	if b == 0 {
		return a
	}
	return gcd(b, a%b)
}

func calc(a, b, c int) (int, int, int) {
	if a < 0 || (a == 0 && b < 0) {
		a, b, c = -a, -b, -c
	}
	g := gcd(gcd(abs(a), abs(b)), abs(c))
	return a / g, b / g, c / g
}

func abs(x int) int {
	if x < 0 {
		return -x
	}
	return x
}

func Solve() {
	defer func() {
		_ = writer.Flush()
	}()
	var n int
	_, _ = fmt.Fscan(reader, &n)
	x := make([]int, n)
	y := make([]int, n)
	for i := 0; i < n; i++ {
		_, _ = fmt.Fscan(reader, &x[i])
	}
	for i := 0; i < n; i++ {
		_, _ = fmt.Fscan(reader, &y[i])
	}
	l := make(map[Line]struct{})
	for i := 0; i < n; i++ {
		for j := i + 1; j < n; j++ {
			a, b, c := 2*(x[j]-x[i]), 2*(y[j]-y[i]), y[j]*y[j]-y[i]*y[i]+x[j]*x[j]-x[i]*x[i]
			a, b, c = calc(a, b, c)
			l[Line{a, b, c}] = struct{}{}
		}
	}
	_, _ = fmt.Fprintln(writer, len(l))
}
```