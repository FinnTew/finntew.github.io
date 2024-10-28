---
title: Demos - BloomFilter 实现
description: BloomFilter 基本实现的笔记记录
slug: bloomfliter-impl
date: 2024-10-27T15:15:46+08:00
math: true
image:
tags:
- Golang
- Demos
weight: 1
---

## BloomFilter

### 工作原理 & 优/缺点

BloomFilter 本质上就是由一个二进制位数组和一系列的哈希函数组成的一个集合数据结构，特殊的点在于，和其他集合数据结构不同，我们并不会在 BloomFilter 中直接存储集合中的元素，而是在位数组中记录每个元素的存在性。

工作原理：
- 插入时，对当前插入的目标对象计算多次哈希值，得到 $Object \rightarrow IndexOfBinaryArray$ 的映射，将这些下标置1。
- 查找时，同理对当前插入的目标对象计算多次哈希值，得到 $Object \rightarrow IndexOfBinaryArray$ 的映射，判断这些下标是否全部为1。

这样做的好处是：
- 占用空间少。
- 插入以及查询时间复杂度都非常低，为 $O(Cnt_{HashFunc})$。

但相应的缺点也非常明显：
- **查询得到的结果是概率性正确的，而非绝对正确**。
- 删除困难。

且缺点随着 BloomFilter 中存放对象数量的增加而扩大。

不过可以保证的是：BloomFilter 查询得到存在性为否时，该对象确保不在集合里。

类似于：如果某天下雨，那么老寒腿的人前一天一定腿疼了，但是老寒腿的人腿疼的时候第二天并不一定会下雨。

BloomFilter 也是这样，得到某个对象不存在的结果时一定正确，但得到某个对象存在的结果时只是有正确的概率，并不保证一定正确。

需要注意的是：我们选择哈希函数的 seed 时，注意素数距离的同时尽可能选择接近 2 的幂的素数，尽可能的使得哈希分布均匀。

### 使用场景

1. 存在性判定：
   - 判断某个元素是否在一个很大数据量的集合中。
   - 防止缓存穿透：判断请求数据是否有效避免直接绕过缓存请求数据库等。
   - 垃圾邮箱过滤：判断目标邮箱是否在垃圾邮件列表中。
   - 黑名单判定：判断IP / 手机号等等是否在黑名单中。
   - 短链接过滤：判断某个短链是否已经被使用。
   - ...
2. 去重：
   - 爬虫的目标 URL 去重。
   - 订单号去重。
   - ...

## 实现

### 定义 & Hash

我们前面提到 "BloomFilter 本质上就是由一个二进制位数组和一系列的哈希函数组成的一个集合数据结构"。

显然需要我们需要设置如下四个字段：

```go
type BloomFilter[Type comparable] struct {
	bitset []bool // 位数组
	size   int // 位数组大小
	hashes int // 哈希函数个数
	seeds  [14]int // 哈希函数使用的 seed
}

func NewBloomFilter[Type comparable]() *BloomFilter[Type] {
	return &BloomFilter[Type]{
		bitset: make([]bool, 2<<24),
		size:   2 << 24,
		hashes: 14,
		seeds:  [14]int{3, 13, 47, 67, 89, 137, 193, 257, 331, 421, 521, 631, 761, 907},
	}
}
```

哈希函数：

```go
func hash(value interface{}, seed, cap int) (int, error) {
	if value == nil {
		return 0, fmt.Errorf("value is nil")
	}
	hasher := fnv.New32a()
	_, err := hasher.Write([]byte(fmt.Sprintf("%v", value)))
	if err != nil {
		return 0, fmt.Errorf("unable to hasher value: %w", err)
	}
	h := int(hasher.Sum32())
	result := ((cap-1)&seed*(h^(h>>16)) + cap) % cap // 确保到下标的映射不会越界
	if result < 0 {
		return -result, nil
	}
	return result, nil
}
```

### 新增 & 查询

```go
func (bf *BloomFilter[Type]) Add(value Type) {
	for i := 0; i < bf.hashes; i++ {
		index, err := hash(value, bf.seeds[i], bf.size)
		if err != nil {
			panic(err)
		}
		bf.bitset[index] = true // 修改位数组
	}
}

func (bf *BloomFilter[Type]) Contains(value Type) (bool, error) {
	result := true
	for i := 0; i < bf.hashes; i++ {
		index, err := hash(value, bf.seeds[i], bf.size)
		if err != nil {
			return false, err
		}
		result = result && bf.bitset[index]
	}
	return result, nil
}
```