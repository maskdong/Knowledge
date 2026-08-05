

> 学习主线：使用语义 → 底层结构 → 哈希与冲突 → 扩容与迁移 → 并发安全 → `sync.Map` → 工程实践 → 面试表达
>
> 版本说明：Go 1.24 起，内置 Map 的运行时实现改为 Swiss Table。传统面试资料中的 `hmap`、`bmap`、overflow bucket、渐进式 evacuation 主要对应 Go 1.23 及以前。讨论底层实现时，应先说明 Go 版本。

## 1. 使用语义

### 1.1 创建与初始化

```go
var nilMap map[string]int              // nil Map
emptyMap := make(map[string]int)       // 已初始化的空 Map
hintedMap := make(map[string]int, 100) // 容量提示，不是固定容量
literalMap := map[string]int{"a": 1}  // 字面量初始化
```

`make(map[K]V, n)` 中的 `n` 是容量提示，可以减少增长次数，但不保证一次性分配恰好容纳 `n` 个元素的空间。Map 没有 `cap` 操作。

nil Map 可以读，不能写。空 Map 已初始化，可以正常写入。

通常不需要 `*map[K]V`，直接使用 `make` 创建 Map 即可。

### 1.3 读取、存在性判断与删除

直接读取不存在的 Key 会得到 Value 的零值：

```go
count := m["count"]
```

这无法区分“Key 不存在”和“Key 存在但值为 0”。需要使用 comma-ok：

```go
count, ok := m["count"]
if !ok {
    // key 不存在
}
```

删除和清空：

```go
delete(m, "count")
clear(m)
```

- `delete` 删除不存在的 Key 也是安全的。
- `clear(m)` 清除共享底层 Map 中的所有元素，其他持有者也能观察到结果。
- `m = make(map[K]V)` 只是让当前变量改为引用一张新 Map，不影响旧 Map 的其他持有者。
- `clear` 不等价于立即向操作系统归还全部内部存储。

### 1.4 Map 是值传递

Go 的参数传递只有值传递，Map 也不例外。复制 Map 值后，两个变量仍然引用同一份底层数据。

```go
func update(m map[string]int) {
    m["a"] = 100
}
```

`update` 会影响调用方看到的元素。但局部变量重新赋值不会改变调用方变量：

```go
func reset(m map[string]int) {
    m["a"] = 100              // 修改共享的底层 Map
    m = make(map[string]int)   // 只改变局部变量 m
    m["b"] = 200              // 写入新 Map
}
```

准确的面试表述是：

> Map 是值传递。复制后的 Map 值仍然共享底层数据，因此修改元素对调用方可见；给局部 Map 变量重新赋值，不会改变调用方变量。


### 1.5 Key 必须可比较

Map 的 Key 必须支持 `==` 和 `!=`。

可以作为 Key：

- 布尔值、整数、浮点数、复数和字符串；
- 指针、Channel；
- 数组；
- 所有字段均可比较的 Struct；
- Interface，但其中保存的动态值也必须可比较。

不能作为 Key：

- Slice；
- Map；
- Function；
- 包含不可比较字段的 Struct 或数组。

### 1.6 Map 元素不可寻址

```go
type User struct {
    Age int
}

users := map[string]User{"jerry": {Age: 30}}

// users["jerry"].Age = 31 // 编译错误
// p := &users["jerry"]    // 编译错误
```

Map 的内部组织可能在增长或重组时变化，元素没有语言层面保证的稳定地址。因此，不能取得 Map 元素的地址，也不能直接修改 Struct Value 的字段。

方案一：取出、修改、写回。

```go
user := users["jerry"]
user.Age = 31
users["jerry"] = user
```

方案二：保存指针。

```go
users := map[string]*User{
    "jerry": {Age: 30},
}
users["jerry"].Age = 31
```

`map[K]*V` 修改方便，但会引入共享修改、nil 指针、额外间接访问和 GC 压力。保存指针也不会让 Map 具备并发安全性。

### 1.7 遍历语义

**Map 的遍历顺序没有保证**，不应依赖某次运行观察到的顺序。

需要稳定输出时，应先提取并排序 Key：

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

同一 Goroutine 在遍历时删除元素是允许的。尚未遍历到的元素若被删除，之后不会产生对应迭代值；遍历期间新增的元素可能被遍历到，也可能不会。

## 2. 底层结构

### 2.1 Go 1.24+：Swiss Table

从 Go 1.24 开始，内置 Map 使用 Swiss Table 思路。理解时可抓住以下层次：

```text
Map
├── Directory
└── 一个或多个 Table
    └── 多个 Group
        ├── Control Word
        └── 8 个 Key/Value Slot
```

#### slot
一个 Slot 保存一组KV，真正存放数据的结构。
Slot 本身不负责快速查询。它旁边有对应的 Control Byte，记录：
- Slot 是否为空；
- Slot 是否已删除；
- Slot 被占用时，对应 Key 的 `H2`。

#### Group
一个 Group 包含，一次检查 8 个 Slot。

```
Group
├── Control Word
│   ├── ctrl[0]
│   ├── ctrl[1]
│   └── ... ctrl[7]
└── Slots
    ├── slot[0]
    ├── slot[1]
    └── ... slot[7]
```

示例：

```
Control: [23][89][50][E ][D ][E ][E ][E ]
Slot:    [K1][K2][K3][  ][  ][  ][  ][  ]
```

其中：
- `23、89、50`：Slot 已占用，数字表示对应 Key 的 `H2`；
- `E`：Empty，从未被占用；
- `D`：Deleted，曾被占用但已经删除。
Group 是查询和探测的基本单位。查询不会先逐个比较 8 个完整 Key，而是先一次性比较 8 个 Control Byte。

#### Table
由多个 Group 组成的子哈希表
Table 是一张可以独立完成查询、插入和扩容的哈希表：

```
Table
├── Group 0：8 Slots
├── Group 1：8 Slots
├── Group 2：8 Slots
└── Group 3：8 Slots
```

Table 内部负责：

- 根据 `H1` 找到起始 Group；
- 按探测序列检查其他 Group；
- 统计已使用元素和增长额度；
- 容量不足时进行扩容或者分裂。

Go 将单个 Table 的最大容量限制为 1024 个 Slot。达到上限后，不再无限扩大这张 Table，而是拆成两张 Table。


#### Directory
当 Map 比较大、存在多张 Table 时，Directory 使用哈希值的高位选择目标 Table：
Directory 只保存“哈希前缀到 Table”的映射。

```
Directory
├── entry[0] → Table A
├── entry[1] → Table A
├── entry[2] → Table B
└── entry[3] → Table C
```

#### 写入&读取流程
```
写入流程：

计算 hash
→ Directory 选择 Table
→ H1 定位起始 Group （H1 64位 hash 值的前 57 位）
→ H2 筛选候选 Slot  （H2 64位 hash 值的后 7 位）
→ 比较完整 Key
├── Key 已存在：更新 Value
└── Key 不存在
    → 沿探测序列寻找 Empty/Deleted
    → 必要时扩容
    → 写入 Key、Value 和 H2
    
读取流程：
hash(key)
→ Directory 选择 Table
→ H1 定位起始 Group
→ H2 批量筛选 Slot
→ 比较候选完整 Key
├── 命中：返回 Value
├── 未命中但遇到 Empty：返回不存在
└── 未命中且没有 Empty：继续探测
```

#### Swiss Table 描述

Swiss Table 本质上是一种开放寻址哈希表。它将 8 个 Key/Value Slot 组成一个 Group，并为每个 Group 维护一个 8 字节的 Control Word。Key 的哈希被拆为 H1 和 H2：H1 用来定位起始 Group并生成探测序列，H2 是低 7 位，存储在对应的控制字节中。查询时可以一次比较 Group 内 8 个 H2，只对匹配的候选 Slot 比较完整 Key，因此减少了 Key 比较和内存访问。发生冲突时通过二次探测寻找其他 Group，而不是使用旧版 Go Map 的 overflow bucket。Go 还通过 Directory 管理多个独立 Table，使扩容只影响部分数据，控制单次扩容带来的延迟。
最后记住这条主线即可：

```
hash(key)
→ 高位选择 Table
→ H1 定位起始 Group
→ H2 同时筛选 8 个 Slot
→ 完整 Key 确认
→ 未命中且没有 Empty 就继续探测
```


### 2.2 Go 1.23 及以前：bucket 实现

传统面试资料描述的是旧实现：

- `hmap` 保存 Map 元信息；
- `bmap` 表示 bucket；
- 一个 bucket 通常容纳 8 个 Key/Value；
- `tophash` 保存哈希高位，用于快速筛选；
- 冲突严重时连接 overflow bucket；
- `oldbuckets` 和 `nevacuate` 用于扩容期间的渐进迁移。

旧知识仍可能被面试官追问，但回答时要明确版本：

> `hmap/bmap + overflow bucket` 是 Go 1.23 及以前的主要实现；Go 1.24 起已经切换为 Swiss Table。



## 3. 哈希与冲突

### 3.1 为什么既要哈希又要比较 Key？

哈希值用于快速定位候选位置，但不同 Key 可能产生相同哈希，因此命中候选后还必须比较完整 Key。

```text
hash(key) → 定位候选位置 → 比较完整 Key → 确认是否命中
```

这也是 Key 必须可比较的底层原因。

### 3.2 Swiss Table 的 H1 与 H2

可以把 Key 的哈希理解为两部分：

- `H1`：用于选择 Table、确定起始 Group 并进行探测；
- `H2`：通常取低 7 位，保存在控制字节中，用于快速筛选 Group 内的候选 Slot。

查找流程：

1. 计算 Key 的哈希值；
2. 根据哈希高位定位 Table；
3. 根据 H1 确定起始 Group；
4. 比较 Group 控制信息中的 H2；
5. 对匹配 H2 的候选 Slot 比较完整 Key；
6. 未命中且未遇到终止条件时，沿探测序列检查后续 Group。

H2 相同只意味着“可能匹配”，并不代表 Key 相等。

### 3.3 如何解决哈希冲突？

Go 1.24+ 的 Swiss Table 使用开放寻址：目标位置被占用时，按照探测序列继续寻找其他 Group/Slot。

旧版 Map 使用 bucket 加 overflow bucket：主 bucket 放不下时，通过溢出桶继续存储。

| 版本 | 主要冲突处理方式 | 特点 |
|---|---|---|
| Go 1.24+ | 开放寻址、Group 探测 | 连续布局，缓存友好 |
| Go 1.23 及以前 | bucket + overflow bucket | 可能产生额外指针跳转 |

平均情况下，查找、插入和删除接近 `O(1)`；大量冲突时探测或链路变长，最坏可退化到 `O(n)`。


## 4. 扩容与迁移

### 4.1 为什么需要扩容？

元素持续增加后：

- 空闲 Slot 减少；
- 冲突概率提高；
- 平均探测距离变长；
- 查询和插入成本上升。

Map 需要通过增长或重组维持合适的负载和访问效率。

### 4.2 Go 1.24+ 的增长方式

Swiss Table Map 可以包含多个 Table：

1. 较小 Table 可以扩大自身容量并重新组织元素；
2. Table 增长到一定规模后可拆分为两个 Table；
3. Directory 必要时同步增长；
4. 元素根据哈希高位归入拆分后的不同 Table。

```text
原 Table → Table A + Table B
```

多 Table 拆分限制了单次增长所需处理的数据规模，降低一次操作搬迁整张超大 Map 所造成的延迟尖峰。

### 4.3 旧版的渐进式迁移

Go 1.23 及以前扩容后会同时保留新旧 bucket 数组，并在后续 Map 操作中逐步执行 evacuation，而不是一次搬完。常见扩容包括：

- 翻倍扩容：负载过高，需要增加 bucket 数量；
- 等量扩容：bucket 数量基本不变，主要整理过多的 overflow bucket。

面试中若被问到 `oldbuckets`、`nevacuate`、渐进搬迁，应明确这是旧实现。

### 4.4 删除后会自动缩容吗？

不能依赖 Map 在大量删除后自动收缩到很小。若一个 Map 曾经很大、现在只剩少量数据，并且确实需要回收内部空间，可以新建 Map 并复制剩余元素：

```go
compact := make(map[string]int, len(m))
for k, v := range m {
    compact[k] = v
}
m = compact
```

这只替换当前变量；其他持有旧 Map 的变量仍然引用旧 Map。

## 5. 并发安全

### 5.1 普通 Map 的并发边界

普通 Map 不是并发读写安全的。

- 多个 Goroutine 只读，且整个期间没有任何写入：可以安全使用。
- 读写并发：不安全。
- 写写并发：不安全。
- 一个 Goroutine 遍历，另一个写入：不安全。
- 赋值、`delete`、`clear` 都属于写操作。

可能观察到：

```text
fatal error: concurrent map read and map write
fatal error: concurrent map writes
```

但 Runtime 报错不是完整的安全保障；并发读写本质上是 Data Race，程序行为不受保证。测试时应启用 Race Detector：

```bash
go test -race ./...
```

### 5.2 为什么普通 Map 不内置锁？

- 单线程和只读场景不应承担强制同步成本；
- 不同业务需要不同锁粒度；
- 多步骤复合操作即使每个 Map 方法带锁，通常仍需要外部事务边界；
- 某些场景更适合不可变快照、分片或 `sync.Map`。

因此，Go 把同步策略交给调用方。

### 5.3 `map + Mutex/RWMutex`

这是最通用、类型安全且容易维护的方案：

```go
type Counter struct {
    mu sync.RWMutex
    m  map[string]int
}

func NewCounter() *Counter {
    return &Counter{m: make(map[string]int)}
}

func (c *Counter) Get(key string) (int, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.m[key]
    return v, ok
}

func (c *Counter) Set(key string, value int) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.m[key] = value
}
```

锁保护的不只是 Map，而是业务不变量。

下面虽然 `Get` 和 `Set` 各自安全，但组合并不原子：

```go
v, _ := c.Get(key)
c.Set(key, v+1)
```

两个 Goroutine 可能同时读到旧值并覆盖彼此更新。正确方式是让整个复合操作处于同一个临界区：

```go
func (c *Counter) Increment(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.m[key]++
}
```

### 5.4 Mutex、RWMutex 与分片

- 临界区短、竞争不高时，`Mutex` 往往更简单，性能也未必差。
- 读操作明显占主导且存在真实竞争时，可考虑 `RWMutex`。
- 超大 Map 的全局锁已经成为瓶颈时，可以按 Key 哈希将数据划分到多个 shard，每个 shard 使用独立 Map 和锁。

不要仅凭“读多写少”就机械使用 `RWMutex`，应结合 Benchmark、Mutex Profile 和实际流量分布判断。

## 6. `sync.Map`

### 6.1 基本 API

```go
var m sync.Map

m.Store("name", "Jerry")

value, ok := m.Load("name")
if ok {
    fmt.Println(value.(string))
}

actual, loaded := m.LoadOrStore("name", "Tom")
_ = actual
_ = loaded

m.Delete("name")
```


### 6.2 适用场景

`sync.Map` 主要针对两类场景优化：

1. 某个 Key 只写一次、之后大量读取，例如只增不改的缓存；
2. 多个 Goroutine 读取、写入和覆盖互不相交的 Key。


### 6.3 不适用场景

- 需要维护多个 Key 之间的一致性；
- 需要读改写等复杂复合操作；
- 写入频繁且热点集中在少数 Key；
- 强依赖编译期类型安全；
- 普通 `map + Mutex` 已经足够清晰。

`sync.Map` 使用 `any`，读取时常需类型断言：

```go
value, ok := m.Load(id)
if !ok {
    return
}
user, ok := value.(*User)
if !ok {
    return
}
```

可以在业务层封装泛型类型，隐藏断言细节，但这不改变其适用边界。


### 6.6 选型速查

| 场景               | 推荐方案                  |
| ---------------- | --------------------- |
| 单 Goroutine 使用   | 普通 Map                |
| 初始化完成后只读         | 普通 Map，发布前完成初始化       |
| 一般并发读写           | `map + Mutex/RWMutex` |
| 多步原子业务操作         | `map + Mutex`         |
| Key 写一次、之后大量读取   | 可考虑 `sync.Map`        |
| 不同 Key 高度独立、并发度高 | 可考虑 `sync.Map`        |
| 已确认全局锁竞争严重       | 可考虑分片 Map             |

## 8. 面试表达

### 8.1 90 秒综合回答

> Go Map 是哈希表类型。Go 只有值传递，Map 传参时复制的是 Map 值，复制后的变量仍共享底层数据，所以修改元素对调用方可见，但给局部 Map 变量重新赋值不会影响调用方。Map 的 Key 必须可比较，因为哈希只能定位候选位置，冲突后还需要比较完整 Key。Map 元素不可寻址，所以 Struct Value 要取出、修改再写回，或者存储指针。
>
> 底层实现需要区分版本。Go 1.24 起使用 Swiss Table，通过 Directory、Table、Group、Control Word 和开放寻址组织数据。哈希可理解为 H1 和 H2：H1 用于定位和探测，H2 用于快速筛选 Group 内候选 Slot，最后仍比较完整 Key。Table 会通过增长或拆分控制负载。Go 1.23 及以前则主要是 `hmap/bmap`、bucket、overflow bucket 和渐进式 evacuation。
>
> 普通 Map 在没有任何写入时可以并发只读，但读写并发、写写并发以及遍历期间写入都存在 Data Race。一般并发场景优先使用 `map + Mutex/RWMutex`；写一次读很多，或者不同 Goroutine 操作相互独立的 Key 时，可以评估 `sync.Map`。涉及读改写或多个 Key 的业务不变量时，应把完整操作放在同一个同步边界中。

### 8.2 高频追问与回答要点

#### 1. Map 是值传递还是引用传递？

值传递。复制后的 Map 值仍共享底层数据；修改元素可见，局部变量重新赋值不可见。

#### 2. nil Map 可以读写吗？

可以读、遍历、删除和清空；直接写入会 Panic。

#### 3. 为什么 Key 必须可比较？

哈希只能定位候选位置，冲突后必须使用相等比较确认实际 Key。

#### 4. 为什么不能直接修改 Map 中 Struct 的字段？

Map 元素不可寻址，其内部位置可能因增长或重组变化。应取出、修改、写回，或存储指针。

#### 5. Go Map 如何处理哈希冲突？

Go 1.24+ 的 Swiss Table 使用**开放寻址和 Group 探测**；旧版使用 bucket 和 overflow bucket。最终都要比较完整 Key。

#### 6. 扩容是否一次搬完所有元素？

不要脱离版本回答。Go 1.24+ 通过 Table 增长或拆分限制单次增长成本；旧版通过 `oldbuckets` 渐进 evacuation。

#### 7. 普通 Map 什么时候可以并发使用？

只有所有 Goroutine 都只读，并且整个并发期间绝对没有写入时。其他情况需要同步。

#### 8. 为什么 `Get` 和 `Set` 都加锁，`Get` 后 `Set` 仍可能出错？

两个方法之间存在竞态窗口；单次方法线程安全不等于复合业务操作原子。读改写必须位于同一临界区。

#### 9. `sync.Map` 为什么不是普通 Map 的通用替代？

它针对特定并发模式优化，类型为 `any`，复杂复合操作和跨 Key 不变量仍需额外同步。普通 `map + Mutex` 往往更清晰。

#### 10. Map 删除大量元素后如何回收空间？

不能依赖自动缩容；如果内存收益值得复制成本，可创建容量接近剩余元素数的新 Map，并复制仍需保留的数据。

### 8.3 容易失分的表述

| 不严谨或过时的说法 | 推荐表达 |
|---|---|
| “Map 是引用传递” | “Map 是值传递，复制后的 Map 值共享底层数据” |
| “Map 底层就是 `hmap/bmap`” | “旧版是 `hmap/bmap`；Go 1.24+ 使用 Swiss Table” |
| “Map 并发读是安全的” | “只有并发期间不存在任何写入时，多 Goroutine 只读才安全” |
| “加了 `Get/Set` 锁就是线程安全” | “单方法安全不保证复合操作原子” |
| “`sync.Map` 比加锁 Map 快” | “是否更合适取决于访问模式，应通过 Benchmark 验证” |
| “Map 扩容后指针会失效” | “语言规定 Map 元素不可寻址；底层增长和重组使稳定地址无法保证” |

### 8.4 最终记忆主线

```text
值传递但共享底层数据
→ Key 必须可比较
→ 元素不可寻址
→ Go 1.24+ 使用 Swiss Table
→ H1 定位与探测，H2 筛选候选
→ 开放寻址处理冲突
→ Table 增长或拆分
→ 普通 Map 不支持并发读写
→ 复合操作必须整体同步
→ sync.Map 只适合特定访问模式
→ 工程选型依赖语义、Profile 与 Benchmark
```

## 参考资料

- [Go Blog: Faster Go maps with Swiss Tables](https://go.dev/blog/swisstable)
- [Go Runtime Map 源码说明](https://go.dev/src/internal/runtime/maps/map.go)
- [Go Language Specification: Map types](https://go.dev/ref/spec#Map_types)
- [Go `sync.Map` 文档](https://pkg.go.dev/sync#Map)

