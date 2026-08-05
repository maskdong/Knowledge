> 学习主线：先掌握使用语义，再理解查找与扩容，最后做并发选型。

> 版本说明：Go 1.24 起内置 Map 使用 Swiss Table；`hmap`、`bmap`、overflow bucket 和渐进式 evacuation 主要对应 Go 1.23 及以前。讨论底层实现时先说明版本。

## 1. 先记住这些结论

| 问题                       | 结论                                                       |
| ------------------------ | -------------------------------------------------------- |
| Map 是引用传递吗？              | 不是。Map 是值传递，但复制后的 Map 值共享底层数据。                           |
| nil Map 能做什么？            | 可以读、遍历和删除，不能写入。                                          |
| Key 有什么要求？               | 必须可比较，因为哈希命中后还要比较完整 Key。                                 |
| 为什么不能改 Map 中 Struct 的字段？ | Map 元素不可寻址，应取出、修改、写回，或存储指针。                              |
| Map 遍历有序吗？               | 没有保证。需要稳定输出时，提取 Key 后排序。                                 |
| 普通 Map 能并发读写吗？           | 不能。只读并发安全的前提是整个期间绝对没有写入。                                 |
| 并发场景怎么选？                 | 一般使用 `map + Mutex/RWMutex`；特定访问模式再评估 `sync.Map` 或分片 Map。 |

## 2. 使用语义

### 2.1 创建与初始化

```go
var nilMap map[string]int              // nil Map
emptyMap := make(map[string]int)       // 已初始化
hintedMap := make(map[string]int, 100) // 容量提示
literalMap := map[string]int{"a": 1}  // 字面量
```

`make(map[K]V, n)` 的 `n` 是容量提示，不是固定容量，也没有 `cap(m)` 操作。预估数量时提供提示，可以减少后续增长和复制。

nil Map 与空 Map 的关键差别是写入：

```go
var a map[string]int
a["x"] = 1 // panic

b := make(map[string]int)
b["x"] = 1 // 正常
```

通常不需要 `*map[K]V`。Map 值本身就是一个指向运行时数据的描述符；只有需要让函数替换调用方持有的 Map 时，才需要返回新 Map 或使用指针。

### 2.2 读取、判断、删除与清空

不存在的 Key 返回 Value 的零值：

```go
count := m["count"]
```

需要区分“不存在”和“值为零”时使用 comma-ok：

```go
count, ok := m["count"]
if !ok {
    // key 不存在
}
```

```go
delete(m, "count") // 删除不存在的 Key 也安全
clear(m)            // 清空当前 Map 的所有元素
```

`clear` 会影响所有仍持有这张 Map 的变量，但不等价于立即把全部内部存储归还给操作系统。`m = make(map[K]V)` 只替换当前变量，不影响其他持有旧 Map 的变量。

### 2.3 Map 是值传递，但共享底层数据

```go
func update(m map[string]int) {
    m["a"] = 100       // 调用方可见
    m = make(map[string]int) // 只改变局部变量
    m["b"] = 200       // 写入新 Map
}
```

准确表述是：Map 参数按值传递，复制的是 Map 值而不是全部元素；复制后的值仍引用同一份底层数据。因此修改元素对调用方可见，局部变量重新赋值对调用方不可见。

### 2.4 Key 必须可比较

可以作为 Key：布尔值、整数、浮点数、复数、字符串、指针、Channel、数组，以及字段全部可比较的 Struct。Interface 也可以作为 Key，但其中保存的动态值必须可比较。

不能作为 Key：Slice、Map、Function，以及包含不可比较字段的 Struct 或数组。

### 2.5 Map 元素不可寻址

```go
type User struct { Age int }

users := map[string]User{"jerry": {Age: 30}}
// users["jerry"].Age = 31 // 编译错误
```

Map 的元素位置可能随增长或重组变化，语言不提供稳定地址。修改 Struct Value 要取出、修改、写回：

```go
user := users["jerry"]
user.Age = 31
users["jerry"] = user
```

也可以保存指针：

```go
users := map[string]*User{"jerry": {Age: 30}}
users["jerry"].Age = 31
```

`map[K]*V` 会引入共享修改、nil 指针、额外间接访问和 GC 压力，也不会带来并发安全。

### 2.6 遍历语义

Map 遍历顺序没有保证，不要依赖某次运行观察到的顺序。需要稳定输出时先提取并排序 Key：

```go
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)
for _, k := range keys {
    fmt.Println(k, m[k])
}
```

同一 Goroutine 在遍历时删除元素是允许的；尚未遍历到的元素若被删除，之后不会产生对应迭代值。遍历期间新增的元素可能被遍历到，也可能不会。并发写入仍然不安全。

## 3. 底层原理：一条查找链路

无论讨论哪个版本，都先抓住这个抽象：

```text
hash(key)
→ 定位候选位置
→ 快速筛选候选项
→ 比较完整 Key
→ 命中或继续探测
```

哈希只负责缩小范围，不能单独证明两个 Key 相等；冲突后仍必须比较完整 Key。这也是 Key 必须可比较的原因。

### 3.1 Go 1.24+：Swiss Table

Swiss Table 使用开放寻址，把数据按 Group 组织。可以简化为：

```text
Map
├── Directory：选择目标 Table
└── Table
    └── Group：控制信息 + 一组 Key/Value Slot
```

- **Slot**：保存真正的 Key/Value。
- **Group**：一次检查一组 Slot，控制字节记录空、删除和哈希指纹等状态。
- **Table**：负责定位、探测、插入和局部扩容。
- **Directory**：Map 较大时，根据哈希高位选择目标 Table。

单个 Table 的容量有上限（当前实现约为 1024 个 Slot）。继续增长时，运行时会把 Table 拆分为两个 Table，并让 Directory 增加相应的哈希前缀映射；因此大 Map 的增长不必每次都搬迁全部元素。

哈希可理解为两部分：

- `H1`：定位起始 Group，并参与探测；
- `H2`：保存为短指纹，批量筛选 Group 内的候选 Slot。

控制字节有三种关键状态：`Empty` 表示该 Slot 从未使用，`Deleted` 表示元素已删除但探测链不能在此处终止，其他值表示已占用并保存对应的 `H2`。一个 Group 通常包含 8 个 Slot，因此运行时可以一次比较一组指纹，而不是逐个读取完整 Key。

#### 读取链路

以 `v, ok := m[key]` 为例，逻辑上可以拆成以下步骤：

1. **检查 Map 状态**：nil Map 直接返回 Value 零值和 `ok=false`，不会分配；非 nil Map 继续查找。
2. **计算哈希**：运行时使用 Map 自己的 hash seed 计算 Key 的哈希值，并拆出用于定位的 `H1` 和用于筛选的 `H2`。
3. **Directory 选 Table**：Map 较大时，使用哈希高位和 Directory 的映射找到目标 Table；小 Map 可以直接落到单个 Table。
4. **H1 选起始 Group**：根据 `H1` 计算起始 Group，并按照探测序列依次检查后续 Group。
5. **批量匹配 H2**：读取当前 Group 的控制字节，一次得到所有可能匹配的 Slot。
6. **比较完整 Key**：只对 `H2` 匹配的 Slot 读取完整 Key，并执行真正的相等比较。`H2` 相同只说明“可能命中”。
7. **决定继续还是结束**：完整 Key 相等则返回 Value；没有命中但遇到 `Empty`，说明后续探测链上不可能有该 Key，返回零值；只有遇到已占用或 `Deleted` 且当前 Group 没有结果时，才继续探测。

可以把读取链路记成：

```text
nil 检查
→ hash(seed)
→ Directory 选 Table
→ H1 找 Group
→ 控制字节批量筛 H2
→ 完整 Key 比较
→ 命中返回 / Empty 终止 / 否则继续探测
```

#### 写入链路

以 `m[key] = value` 为例，写入除了要查找 Key，还要决定新元素放在哪里：

1. **检查可写状态**：nil Map 不能直接写入，会触发 panic；已初始化的 Map 才进入写入流程。
2. **计算哈希并选 Table**：这一步与读取相同，使用同一个 Map seed，通过 Directory、`H1` 找到起始 Group。
3. **扫描探测序列**：逐个 Group 批量匹配 `H2`，对候选 Slot 比较完整 Key。
4. **Key 已存在**：找到相等 Key 后，只更新对应 Value，不新增 Slot，也不改变 Map 长度。
5. **Key 不存在**：继续扫描直到找到可用位置。优先记录探测过程中遇到的第一个 `Deleted` Slot；如果没有，再使用第一个 `Empty` Slot。复用 `Deleted` 可以减少墓碑积累，但不能在遇到 `Deleted` 时提前结束查找。
6. **检查增长条件**：如果负载过高、可用位置不足，或当前 Table 达到规模限制，运行时先增长或拆分 Table，并把受影响的元素重新放置，再重新执行本次查找。若 Directory 也需要扩展，会同步增加哈希前缀到 Table 的映射。
7. **提交写入**：在选定 Slot 写入 Key 和 Value，最后更新控制字节中的 `H2` 和占用状态。写入完成后，后续读取才能把该 Slot 当作候选。

写入链路可以压缩为：

```text
nil 检查
→ hash(seed)
→ Directory 选 Table
→ H1 找 Group，批量筛 H2
→ 完整 Key 比较
→ 已存在则更新 Value
→ 不存在则选 Deleted/Empty
→ 必要时增长或拆分并重试
→ 写入 Key/Value，再提交控制字节
```

#### 删除与探测的关系

删除不是简单地把 Slot 当作从未使用。运行时需要留下 `Deleted` 状态，让后续查找继续穿过该位置；否则，某个 Key 因冲突被放在更后面的 Group 时，查找可能在删除点错误地提前返回“不存在”。因此，`Empty` 可以终止探测，`Deleted` 不能终止探测。

Swiss Table 的重点不是记住字段，而是理解“控制字节先筛选、完整 Key 再确认”，以及读写都沿同一条探测链路工作。

### 3.2 Go 1.23 及以前：bucket 实现

传统实现可以记住四个词：

- `hmap`：Map 元信息；
- `bmap`：bucket；
- `tophash`：哈希高位，用于快速筛选；
- overflow bucket：主 bucket 放满后的溢出链。

扩容期间通过 `oldbuckets` 和 `nevacuate` 做渐进式 evacuation。面试中提到这套模型时，应明确它是旧版本实现，不要泛化为当前所有 Go 版本。

## 4. 哈希冲突与扩容

### 4.1 冲突怎么处理

|版本|冲突处理|理解重点|
|---|---|---|
|Go 1.24+|开放寻址、Group 探测|连续布局，控制信息批量筛选|
|Go 1.23 及以前|bucket + overflow bucket|可能产生额外指针跳转|

平均情况下，查找、插入和删除接近 `O(1)`；冲突严重时探测距离或溢出链变长，最坏可退化到 `O(n)`。

### 4.2 为什么需要扩容

元素增加会让空闲位置变少、冲突和探测距离变长。Map 通过增长或重组降低负载，维持平均访问成本。

Go 1.24+ 可以让 Table 扩大，达到规模上限后拆分为多个 Table；Directory 负责把哈希空间映射到正确的 Table。拆分将一次大搬迁拆成更小的局部工作。

Go 1.23 及以前会保留新旧 bucket 数组，并在后续操作中渐进迁移。旧实现既可能翻倍扩容，也可能等量整理 overflow bucket。

### 4.3 删除后会自动缩容吗

不能依赖 Map 在大量删除后自动缩小。确实需要回收内部空间时，复制到新 Map：

```go
compact := make(map[string]int, len(m))
for k, v := range m {
    compact[k] = v
}
m = compact
```

复制有 CPU 和短期内存成本，应由实际内存收益证明其必要性。

## 5. 并发安全与工程选型

### 5.1 普通 Map 的并发边界

- 多个 Goroutine 只读，且整个期间没有写入：可以安全使用；
- 读写并发、写写并发、遍历期间写入：都不安全；
- 赋值、`delete`、`clear` 都属于写操作。

运行时可能报 `concurrent map read and map write` 或 `concurrent map writes`，但不能把运行时报错当作完整的安全保障；本质问题是 Data Race。测试时使用：

```bash
go test -race ./...
```

### 5.2 `map + Mutex/RWMutex`

这是默认方案：类型安全、边界清晰，适合大多数业务。

```go
type Counter struct {
    mu sync.Mutex
    m  map[string]int
}

func (c *Counter) Increment(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.m[key]++
}
```

锁保护的是业务不变量，不只是某一次 Map 访问。下面的写法虽然两个方法各自加锁，但整体不是原子的：

```go
v, _ := c.Get(key)
c.Set(key, v+1)
```

读改写、多 Key 更新等复合操作必须放进同一个临界区。`RWMutex` 只有在读占绝对优势且存在真实竞争时才值得评估，不能只凭“读多写少”机械选择。

### 5.3 `sync.Map`

```go
var m sync.Map
m.Store("name", "Jerry")
value, ok := m.Load("name")
if ok {
    fmt.Println(value.(string))
}
```

适合：Key 只写一次后大量读取，或不同 Goroutine 操作高度独立的 Key。

不适合：需要跨 Key 一致性、读改写原子性、热点 Key 高频写入，或强调编译期类型安全的场景。它使用 `any`，读取通常需要类型断言；泛型封装只能隐藏断言，不能改变适用边界。

### 5.4 分片 Map

当全局锁确认成为瓶颈时，可按 Key 哈希分配到多个 shard，每个 shard 使用独立 Map 和锁。分片会增加实现复杂度，只有 Benchmark、Mutex Profile 或线上指标证明锁竞争是瓶颈时才值得使用。

### 5.5 选型速查

|场景|优先方案|
|---|---|
|单 Goroutine|普通 Map|
|初始化后只读|普通 Map，完成初始化后再发布|
|一般并发读写|`map + Mutex`，必要时评估 `RWMutex`|
|多步原子业务操作|`map + Mutex`，把完整操作放进临界区|
|Key 写一次、之后大量读取|评估 `sync.Map`|
|不同 Key 高度独立且并发高|评估 `sync.Map` 或分片 Map|
|尚未确认瓶颈|先用清晰方案，再 Benchmark 和 Profile|


## 6. 面试附录

### 6.1 90 秒综合回答

> Go Map 是哈希表。Map 按值传递，但复制后的 Map 值仍共享底层数据，所以修改元素对调用方可见，局部变量重新赋值不可见。Key 必须可比较，因为哈希只能定位候选位置，冲突后还要比较完整 Key；Map 元素不可寻址，Struct Value 需要取出、修改、写回，或存储指针。
>
> 版本上要区分：Go 1.24 起使用 Swiss Table，通过 Directory、Table、Group 和开放寻址查找；哈希可理解为用于定位和探测的 H1，以及用于快速筛选候选的 H2，最后仍比较完整 Key。Go 1.23 及以前主要是 `hmap/bmap`、bucket、overflow bucket 和渐进式 evacuation。
>
> 普通 Map 只有在整个并发期间都没有写入时才支持并发只读。一般并发场景优先使用 `map + Mutex/RWMutex`；Key 写一次读很多或 Key 高度独立时再评估 `sync.Map`。涉及读改写和多个 Key 的不变量时，要把完整操作放进同一个同步边界。

### 6.2 高频追问

1. **Map 是值传递还是引用传递？** 值传递；复制后的值共享底层数据。
2. **nil Map 可以读写吗？** 可以读、遍历、删除，不能写入。
3. **为什么 Key 必须可比较？** 冲突后要比较完整 Key。
4. **为什么不能直接改 Struct 字段？** Map 元素不可寻址；取出修改再写回，或使用指针值。
5. **Go Map 如何处理冲突？** 1.24+ 使用开放寻址和 Group 探测；旧版使用 bucket 和 overflow bucket。
6. **扩容是否一次搬完？** 1.24+ 通过 Table 增长或拆分控制成本；旧版通过渐进式 evacuation。
7. **Get 和 Set 都加锁为何仍可能出错？** 方法之间有竞态窗口，复合操作不原子。
8. **为什么 sync.Map 不是通用替代？** 访问模式受限、类型为 any，复杂不变量仍需额外同步。
9. **删除很多元素后如何回收空间？** 必要时新建 Map 复制剩余元素，不能依赖自动缩容。

### 6.3 容易失分的表述

|不严谨说法|推荐说法|
|---|---|
|Map 是引用传递|Map 是值传递，复制后的值共享底层数据|
|Map 底层就是 `hmap/bmap`|旧版是 `hmap/bmap`，Go 1.24+ 使用 Swiss Table|
|Map 并发读是安全的|只有并发期间绝对没有写入时，多 Goroutine 只读才安全|
|给 Get/Set 加锁就线程安全|单方法安全不保证复合操作原子|
|sync.Map 一定更快|是否合适取决于访问模式，应通过 Benchmark 验证|
|Map 扩容后指针会失效|语言规定 Map 元素不可寻址，不应依赖元素稳定地址|

### 6.4 最终记忆主线

```text
值传递但共享底层数据
→ Key 必须可比较，元素不可寻址
→ hash 定位 → 指纹筛选 → 完整 Key 比较
→ 1.24+ Swiss Table；旧版 bucket + overflow bucket
→ 增长或重组维持负载
→ 普通 Map 不支持并发读写
→ 复合操作必须整体同步
→ sync.Map 只适合特定访问模式
→ 选型依赖语义、Benchmark 与 Profile
```
