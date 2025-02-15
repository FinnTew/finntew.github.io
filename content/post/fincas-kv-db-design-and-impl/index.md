---
title: FincasKV - DB层设计与实现
description: FincasKV DB层相关的一些记录，包括 TTL，Batch，Redis DataStructure 等。
slug: fincas-kv-db-design-and-impl
date: 2025-02-14T18:00:02+08:00
math: true
image:
tags:
- Golang
- FincasKV
- KV Storage
- Redis DataStructure
weight: 1
---

## 闲言碎语

前置参考：

- [FincasKV - 存储层设计与实现](./fincas-kv-storage-design-and-impl)

本篇继续对 FincasKV 的 DB 层实现做一些记录。

## Put With TTL

这里实现 TTL 操作之前有想到两个实现方案：

- 对存储层进行一些修改，在原本 DataFile 的基础上，对于每个 Record 新增一个 expireAt 字段。
- 在进行 TTL 相关操作时新增 TTL 元信息文件，按照 {key, expireAt} 的方式存储。

最终使用了第二种方案，将 TTL 操作解耦出来完全放在 DB 层实现，降低一些维护成本，同时更加方便清理过期数据。

然后考虑怎么处理过期的 key，同样的想到这里有两种方案：

- 定时扫描，定期扫描所有 key，删除过期的 key。
- 惰性删除，在查询某个 key 时，如果设置了 TTL，且已经过期，则删除该 key。

这里使用惰性删除的方式，避免定时扫描全量数据造成的性能问题。不过缺点也比较明显，如果设置了 TTL 的 key 一直没有被访问，那么该 key 就会一直存在于内存中，造成内存浪费。

在之后了解到 redis 中一个优秀的实现：结合定时扫描和惰性删除的方式，使用了一种混合的方式：

- 定时扫描每次只抽出一部分 key 进行检查。避免全量扫描时间过长。
- 对于没有扫描到的 key 进行惰性删除。

这样可以均衡两种方式的优缺点，后续可能考虑使用这种方式优化一下这部分的实现。

一些代码摘要：

```go
type DB struct {
	// ...
	expireMap map[string]time.Time
	expireMu sync.RWMutex
	// ...
	ttlPath string
	needFlush bool
	// ...
}

func (db *DB) Get(key string) (string, error) {
    if db.isExpired(key) {
        _ = db.deleteExpiredKey(key)
        return "", err_def.ErrKeyNotFound
    }
	// ...
}

func (db *DB) Expire(key string, ttl time.Duration) error {
    if ttl <= 0 { ... }
    ex, err := db.Exists(key)
    if err != nil {
        return err
    }
    if !ex {
        return err_def.ErrKeyNotFound
    }
	expireAt := time.Now().Add(ttl)

    db.expireMu.Lock()
    db.expireMap[key] = expireAt
    db.needFlush = db.needFlush || db.dbOpts.FlushTTLOnChange
    db.expireMu.Unlock()

    if db.dbOpts.FlushTTLOnChange {
        _ = db.saveTTLMetadata()
    }

    return nil
}

func (db *DB) Persist(key string) error {
    ex, err := db.Exists(key)
    if err != nil { ... }
    if !ex {
        return err_def.ErrKeyNotFound
    }
    db.expireMu.Lock()
    delete(db.expireMap, key)
    db.needFlush = db.needFlush || db.dbOpts.FlushTTLOnChange
    db.expireMu.Unlock()

    if db.dbOpts.FlushTTLOnChange {
        _ = db.saveTTLMetadata()
    }
	
    return nil
}

func (db *DB) isExpired(key string) bool {
    db.expireMu.RLock()
    expAt, ok := db.expireMap[key]
    db.expireMu.RUnlock()
    return ok && time.Now().After(expAt)
}

func (db *DB) deleteExpiredKey(key string) error {
    err := db.bc.Del(key)
    db.expireMu.Lock()
    delete(db.expireMap, key)
    db.needFlush = db.needFlush || db.dbOpts.FlushTTLOnChange
    db.expireMu.Unlock()

    if db.dbOpts.FlushTTLOnChange {
    _ = db.saveTTLMetadata()
    }
    return err
}
```

## Write Batch

在整个项目完结后来看，Batch 操作实际还是应该在存储层实现的。

因为 Batch 最大的好处应该是去减少 IO 次数，降低磁盘 IO 压力。但是在 DB 层实现的话，目前可以看到的收益大概只有减少锁资源的使用。

之后会对这部分进行一些优化。

这里感觉比较好理解，可以直接看一下代码摘要：

```go
type operation struct {
	typ      OpType
	key      string
	value    string
	ttl      time.Duration
	created  time.Time
	priority int // 操作优先级
}

type WriteBatch struct {
    // ...
	operations []operation
    mu         sync.Mutex
    committed  bool
	// ...
}

// 省略 Put/Delete等方法

func (wb *WriteBatch) Commit() error {
    ctx, cancel := context.WithTimeout(context.Background(), wb.opts.CommitTimeout)
    defer cancel()
    return wb.CommitWithContext(ctx)
}

func (wb *WriteBatch) CommitWithContext(ctx context.Context) error {
    wb.mu.Lock()
    defer wb.mu.Unlock()

    if wb.committed { ... }

    if len(wb.operations) == 0 {
        wb.committed = true
        return nil
    }

    // 操作排序和合并
    wb.optimizeOperations()

    // 批量写入准备，过滤非法操作
    if err := wb.prepareBatch(); err != nil {
        return err
    }

    // 异步处理
    if wb.opts.AsyncCommit {
        go wb.executeOperations(ctx)
        return nil
    }

    return wb.executeOperations(ctx)
}

func (wb *WriteBatch) optimizeOperations() {
    // 按优先级排序
    sort.Slice(wb.operations, func(i, j int) bool {
        return wb.operations[i].priority > wb.operations[j].priority
    })

    // 合并相同key的操作
    optimized := make([]operation, 0, len(wb.operations))
    keyOps := make(map[string]operation)

    for _, op := range wb.operations {
        existing, exists := keyOps[op.key]
		if !exists {
            keyOps[op.key] = op
            continue
        }

        // 合并操作
        // 下面两种组合不需要合并
        if existing.typ == OpPut && op.typ == OpExpire {
            continue
        }
        if existing.typ == OpExpire && op.typ == OpPersist {
            continue
        }
        merged := wb.mergeOperations(existing, op)
        keyOps[op.key] = merged
    }

    // 构建操作列表
    for _, op := range keyOps {
        optimized = append(optimized, op)
    }

    wb.operations = optimized
}

func (wb *WriteBatch) mergeOperations(op1, op2 operation) operation {
    // 只需要保留最后一次操作
    if op2.created.After(op1.created) {
        return op2
    }
    return op1
}

func (wb *WriteBatch) prepareBatch() error {
    keysMap := make(map[string]struct{})
    for _, op := range wb.operations {
        keysMap[op.key] = struct{}{}
    }

    for _, op := range wb.operations {
        switch op.typ {
        case OpPut:
            continue
        case OpDelete, OpExpire, OpPersist:
            exists, err := wb.db.Exists(op.key)
            if err != nil { ... }
            if _, ok := keysMap[op.key]; !exists && !ok { ... }
        }
    }
    return nil
}

func (wb *WriteBatch) executeOperations(ctx context.Context) error {
    wb.db.expireMu.Lock()
    defer wb.db.expireMu.Unlock()

    if err := ctx.Err(); err != nil { ... }

    for _, op := range wb.operations {
        select {
        case <-ctx.Done():
            wb.rollback(op)
            return ctx.Err()
        default:
            if err := wb.executeOperation(op); err != nil {
                wb.rollback(op)
                return err
            }
        }
    }

    if wb.db.needFlush && wb.db.dbOpts.FlushTTLOnChange {
        if err := wb.db.saveTTLMetadata(); err != nil { ... }
        wb.db.needFlush = false
    }

    wb.committed = true
    return nil
}

func (wb *WriteBatch) executeOperation(op operation) error {
    var err error
    switch op.typ {
    case OpPut:
        err = wb.db.bc.Put(op.key, []byte(op.value))
    case OpDelete:
        err = wb.db.bc.Del(op.key)
        if err == nil {
            delete(wb.db.expireMap, op.key)
        }
    case OpExpire:
		expireAt := op.created.Add(op.ttl)
        wb.db.expireMap[op.key] = expireAt
        wb.db.needFlush = true
    case OpPersist:
        delete(wb.db.expireMap, op.key)
        wb.db.needFlush = true
    }

    return err
}

func (wb *WriteBatch) rollback(failedOp operation) {
    // 倒序遍历已执行的操作回滚
    for i := len(wb.operations) - 1; i >= 0; i-- {
        op := wb.operations[i]
        if op == failedOp {
            break
        }

        switch op.typ {
        case OpPut:
            // 删除已写入的数据
            _ = wb.db.bc.Del(op.key)
        case OpDelete:
            // 恢复删除的数据
            if val, err := wb.db.Get(op.key); err == nil {
                _ = wb.db.bc.Put(op.key, []byte(val))
            }
        case OpExpire:
            // 取消过期
            delete(wb.db.expireMap, op.key)
        case OpPersist:
            // 恢复过期
            if expAt, ok := wb.db.expireMap[op.key]; ok {
                wb.db.expireMap[op.key] = expAt
            }
        }
    }
}
```

可以使用 `sync.Pool` 优化一下 `WriteBatch` 的创建和释放，这里就不展开了。

其中省略了一些错误日志之类的部分。

## Redis DataStructure

这里记录一下 Redis 中的一些常见数据结构在 FincasKV 中的设计。（只对数据格式做一些记录，详细实现可以参考仓库中的具体代码）

我们首先可以想到一种最直观最好实现的方式，即使用序列化的方式组织 value，比如 JSON。

但是不难想到这么做有一些非常明显的弊端：

- 序列化的方式会导致一些对内容无意义的格式字符额外占用空间。
- 我们在每次读写时都需要进行序列化 or 反序列化操作，会有一定的额外性能开销。

所以这里采用另一种实现方式：多 Key，也就是在 FincasKV 中，每个 Redis 数据结构的对象会由多个小 Key 构成，而非一个整体的大 Key。

下面是对于不同数据结构的具体设计。

### String

String 的实现比较简单，我们按照 `string:{key} -> {value}` 的格式组织即可。

### Hash

对于 Hash，我们发现它需要额外维护一个长度元信息，所以我们可以将 Hash 拆分为两个部分：

- 一个长度键值对：`hash:{key}:__len__ -> {len}`
- 若干个字段键值对：`hash:{key}:{field} -> {value}`

### List

我们可以将 List 类比到一个链表结构，不难想到我们需要维护以下元信息：

- 一个长度键值对：`list:{key}:__len__ -> {len}`
- 一个头指针键值对：`list:{key}:__head__ -> {head}`
- 一个尾指针键值对：`list:{key}:__tail__ -> {tail}`

以及若干个节点键值对：`list:{key}:{index} -> {value}`。

### Set

Set 和 Hash 类似，区别在与 member 并非键值对形式，按照以下格式组织：

- 一个长度键值对：`set:{key}:__len__ -> {len}`
- 若干个成员键值对：`set:{key}:{member} -> ""`

### ZSet

对于 ZSet，它是一个有序集合，且 member 需要满足 `member -> score`，我们将数据存储和排序拆分为两个部分考虑：

- 数据键值对：`zset:{key}:{member} -> {score}`
- 排序键值对：`zset:{key}:s:{score}:{member} -> {member}`

注意：这里 score 直接使用 float 显然对于排序是无意义的，我们需要进行以下处理：

```go
func float64ToOrderedString(score float64) string {
	bits := math.Float64bits(score)
	if (bits & (1 << 63)) != 0 {
		bits = ^bits
	} else {
		bits = bits | (1 << 63)
	}
	return fmt.Sprintf("%016x", bits)
}
```

### 一些可优化的点

目前想到的只有 ZSet 部分的优化。

现在 ZSet 实现是很暴力的直接排序进行的，这里可以使用 SkipList 进行优化，大概记录如下：

- 首次读取 or 修改已有键/写入新键时构造 SkipList。
- 后续写入使用类似 WAL 的方式（Bitcask 本身就是类 WAL 的），所以写入流程为：先写入 Bitcask，然后向 SkipList 中插入。
- 读取时因为已经构造好了 SkipList，所以直接读取 SkipList 即可。

--- 

结束。