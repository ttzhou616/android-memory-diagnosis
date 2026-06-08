---
name: android-memory-diagnosis
description: Diagnose Android memory issues — leaks, churn, and large allocations — using Android Studio Profiler and MAT. Covers capture → analyze → locate → fix end-to-end.
category: software-development
---

# Android 内存问题诊断

## 触发条件

以下情况加载此技能：
- 用户要求诊断、排查或修复 Android 内存问题
- 用户遇到 OOM、ANR 或应用逐渐变慢
- 用户想知道针对某种症状该用什么工具
- 用户有 `.hprof` 文件需要分析

## 概念基础：泄漏 vs 溢出

### 类比

```
一个漏水的水桶：

  内存泄漏 = 桶底有个小洞，水一直在漏
    → 桶里可用空间越来越小
    → 漏得慢可能几小时才见底，不是立刻致命的

  内存溢出 = 桶里的水完全干了，你还想往外舀一瓢
    → 系统说："没了！一滴都没了！"
    → OutOfMemoryError → 应用崩溃
```

### 技术定义

> **内存泄漏（Memory Leak）**：对象已不再需要，但被 GC Root 间接持有，GC 判定它"还活着"，永远不回收。堆内存被僵尸对象占据，可用空间持续缩小。**泄漏通常导致溢出，但溢出不一定因为泄漏。**

> **内存溢出（OutOfMemory, OOM）**：JVM/ART 尝试分配一块内存时，堆里找不到足够大的连续空间，抛出 `OutOfMemoryError`。**这是泄漏最常见的终局表现，但不是唯一原因。**

示例：一次加载一张 100MB 的图片——没有泄漏，但堆不够大，直接 OOM。

### Android 堆内存硬上限

Android 给每个进程分配固定的堆上限，远小于 PC：

| 设备类型 | 典型堆上限 |
|---------|-----------|
| 低端机（2GB RAM） | 128-192 MB |
| 中端机（4GB RAM） | 256-384 MB |
| 高端机（8GB+ RAM） | 512 MB+ |

这意味着 PC 上不痛不痒的分配，在 Android 上可能就是致命一击。

### OOM 五大常见场景

| 排名 | 场景 | 为什么 |
|------|------|--------|
| 1 | **Bitmap 加载过大** | 4000×3000 ARGB_8888 = 48MB，一张照片吃掉堆的 1/4 |
| 2 | **内存泄漏累积** | 单次泄漏 1MB × 用户使用 30 分钟 = 30MB → 再加载图片 → OOM |
| 3 | **大对象一次性分配** | `ByteArray(100MB)` 在 192MB 堆设备上 → 直接 OOM |
| 4 | **线程过多** | 每个线程默认 1MB 栈，200 个线程 = 200MB |
| 5 | **代码膨胀** | 方法数超 64K（MultiDex）或加载过多类 → Code 内存区溢出 |

## 三层策略：预防 → 监控 → 急救

### 第一层：预防（写代码时就应该做的）

| 问题 | 预防手段 |
|------|---------|
| Bitmap 过大 | `BitmapFactory.Options.inSampleSize` 降采样；用 Glide/Coil 自动处理 |
| 图片列表 | RecyclerView + 图片库 LRU 缓存，不可见图及时回收 |
| 大文件读取 | 流式读、mmap、分块处理，别一次性 `readBytes()` |
| 静态持有 Context | `object` 里存 `Activity` 用 `WeakReference` 或 `ApplicationContext` |
| Handler/Runnable | `onDestroy` 里 `removeCallbacks(null)` |
| Compose DisposableEffect | `onDispose` 里必须反注册 |
| 线程/协程 | 用 `lifecycleScope` / `viewModelScope`，禁用 `GlobalScope` |

### 第二层：监控（运行时发现苗头）

| 手段 | 查出什么 | 建议 |
|------|---------|------|
| **LeakCanary** | Activity/Fragment 泄漏 | **强烈推荐集成**，通知栏直接弹 |
| **Android Studio Profiler** | 实时内存曲线 + 堆转储 | 开发阶段主动跑 |
| **`dumpsys meminfo`** | Pss 精确值 | Pss > 200MB 要警惕 |
| **GC 日志** | GC 频率 + 回收量 + 暂停时长 | `adb logcat \| grep "art.*GC"` |

### 第三层：急救（已经 OOM 了怎么办）

以下代码放在可能触发 OOM 的地方：

```kotlin
try {
    val bitmap = BitmapFactory.decodeFile(path)
} catch (e: OutOfMemoryError) {
    // ① 降级：用更小的采样率重试
    val opts = BitmapFactory.Options().apply { inSampleSize = 4 }
    val smallerBitmap = BitmapFactory.decodeFile(path, opts)

    // ② 清缓存：把 LRU 缓存清掉腾空间
    imageCache.evictAll()

    // ③ 建议 GC（只是建议，不保证立即执行）
    Runtime.getRuntime().gc()

    // ④ 重试
    val retryBitmap = BitmapFactory.decodeFile(path, opts)
}
```

**治本**：急救之后再按诊断决策树抓堆转储 → 分析 → 修复，不要止步于 `catch`。

### 三层关系

```
预防（第一层）
  └── 写代码时做好 → 90% 的 OOM 不会发生
        │
        └── 漏网之鱼 → 监控（第二层）在运行时抓住
              │
              └── 监控没拦住 → 急救（第三层）保用户不崩溃
                    │
                    └── 然后按诊断决策树治本
```

## 诊断决策树

按优先级排序——**有一个第一梯队就能干活，下面的是加分项。**

### 第一梯队：必填（有一个即可开始）

| 材料 | 优先级 | 怎么获取 | 为什么重要 |
|------|--------|---------|-----------|
| **`.hprof` 堆转储文件** | ⭐⭐⭐ | AS Profiler → Past Recordings → 右键堆转储 → Export → 存为 `.hprof` | 包含所有存活对象 + 引用关系 + Retained Size，能推演出完整泄漏链 |
| **操作复现步骤 + 堆转储** | ⭐⭐⭐ | 如："进首页 → 点知识图谱 → 拖动节点 10 次 → 返回首页 → 抓堆转储" | 操作 → 内存变化的因果关系，比单纯一个 hprof 更精确 |

### 第二梯队：缩小范围

| 材料 | 怎么获取 | 用途 |
|------|---------|------|
| **References 标签页截图**（展开到 GC Root） | 堆转储中点开可疑实例 → References → 展开到最深 → 截图 | 直接看到完整泄漏链 |
| **类列表截图**（All classes，实例数降序） | 堆转储类列表 → 切 All classes → 点 Instance Count 列标题排序 → 截图 | 一眼看出哪些类实例数反常 |
| **实时视图 MEMORY 曲线截图** | Profiler → View Live Telemetry → 操作期间截图 | 判断是泄漏（持续升）还是抖动（锯齿） |
| **Allocations 列表截图**（Count 降序） | 分配记录 → Allocations 标签页 → 按 Count 降序 → 截图 | 定位 churn 源头 |
| **Logcat GC 日志** | `adb logcat -d \| grep "art.*GC"` | 看 GC 频率、回收量、暂停时长 |

### 第三梯队：锦上添花

| 材料 | 怎么获取 | 用途 |
|------|---------|------|
| **`dumpsys meminfo` 输出** | `adb shell dumpsys meminfo <包名>` | 精确的 Pss/Rss 和各内存类别数值 |
| **可疑代码区域** | 用户指出怀疑的文件/函数 | 加速定位，但没有 hprof 也能自己找到 |
| **MAT Leak Suspects 截图** | MAT 打开 hprof → Leak Suspects 报告 → 截图 | 自动化泄漏嫌疑分析结果 |

### 问诊流程（Agent 执行）

当用户报告内存问题时，按以下步骤索要材料：

```
用户："App 越来越卡 / OOM 了"
  │
  ├── ① 先问：有没有 hprof 文件？
  │     ├── 有 → 要文件路径 + 操作描述 → 直接分析
  │     └── 没有 → 进入下一步
  │
  ├── ② 指导抓取：
  │     "Run Configuration → 以可调试模式运行 → Profile
  │      → Home 标签页 → Analyze Memory Usage → 抓堆转储
  │      → Past Recordings → Export → 保存 .hprof"
  │
  ├── ③ 同时问：操作步骤 + 症状细节
  │     "你做了什么操作之后感觉到卡的？"
  │     "内存是一直升还是反复跳？"
  │     "有没有崩溃日志？"
  │
  └── ④ 拿到 hprof → 分析 → 给出结果：
        "你的 MainActivity 有 5 个实例，正常应该 1 个。
         引用链：LeakRegistry（静态字段）→ ArrayList → MainActivity。
         修复：在 DisposableEffect 的 onDispose 里加 LeakRegistry.detach(context)。"
```

### 用户不会抓 hprof？用傻瓜式指令

如果用户不知道怎么抓，给以下指令（逐条复制执行即可）：

```
1. Android Studio → 顶部 Run Configuration 下拉 → 选"以可调试模式运行 'app'（完整数据）"
2. 点 Profile 按钮（不是 Run）
3. 等 App 启动后，做你怀疑有问题的那几步操作
4. 底部 Profiler 面板 → Home 标签页 → 点 "Analyze Memory Usage (Heap Dump)" 的 ▶
5. 等解析完成 → 点 "Past Recordings" → 找到刚才的记录 → 点导出按钮（⤓）
6. 保存到桌面，文件名叫 memory.hprof
```

## 诊断决策树

```
症状："应用越用越慢 / OOM 崩溃"
  │
  ├── 第一步：实时视图（profileable 即可，无需 debuggable）
  │     View → Tool Windows → Profiler → View Live Telemetry
  │     看 MEMORY 曲线：
  │       ├── 持续爬升不降 → 泄漏或缓存膨胀 → 跳第三步
  │       ├── 剧烈锯齿起伏 → 内存抖动 → 跳第二步
  │       └── 突然尖峰     → 大对象分配 → 跳第二步
  │
  ├── 第二步：录制 Java/Kotlin 分配（需 debuggable）
  │     开始录制 → 操作 App → 停止 → 分析：
  │       ├── Visualizations 标签页：哪个类的色块在剧烈涌现？
  │       ├── Allocations 标签页：按 Count 降序排列
  │       │     └── 右键排第一的类 → Jump to Source → 修复分配点
  │       └── 火焰图：最宽的条 = 分配最重的方法
  │
  └── 第三步：捕获堆转储（需 debuggable）
        抓取 → 等待解析 → 调查：
          ├── 类列表 → "Show activity/fragment leaks" 过滤
          ├── 搜索可疑类 → 检查实例数量
          │    Activity 实例 > 1 或 Fragment > 0 = 泄漏
          ├── 右键实例 → References → 沿引用链追到 GC Root
          └── 搞不定 → 导出 .hprof → MAT → Leak Suspects / Dominator Tree
```

## 工具速查表

### Android Studio Profiler（覆盖 90% 场景）

| 任务 | 所需模式 | 能看到什么 | 核心操作 |
|------|---------|-----------|---------|
| View Live Telemetry | profileable | CPU/内存/线程 实时曲线 | 第一步筛查——看趋势 |
| Track Memory Consumption (Java/Kotlin 分配) | debuggable | 对象分配记录（时间线） | 按 Count 降序 → Jump to Source |
| Analyze Memory Usage (堆转储) | debuggable | 所有存活对象快照 | References → 追到 GC Root |

### MAT（深度排查，10% 场景）

| 功能 | 使用时机 | 核心操作 |
|------|---------|---------|
| Leak Suspects 报告 | 打开 hprof 自动生成 | 读顶部嫌疑对象，点 "Details" |
| Dominator Tree | 找最大的 Retained Heap 持有者 | 按 Retained Heap 降序 → 展开树 |
| Histogram | 按类统计实例数 | 按包名过滤，按 Retained 排序 |
| OQL | 自定义查询 | `SELECT * FROM com.example.MyClass` |
| Compare Basket | 对比两次堆转储 | 加入篮 → "Compare" |

## 各视图解读方法

### 堆转储 — References 标签页

```
Reference             Depth   Native   Shallow   Retained
────────────────────────────────────────────────────────
Bitmap@2000847480      5      1,048,577   50      1,048,627   ← 选中的目标
  Index 8 in Object[]   4          0       40     10,486,310   ← ⚠️ 主泄漏链
  referent in Cleaner  768         0       36          92      ← 内部机制，忽略
  mBitmap in Canvas      -        99       44         143      ← 不可达路径（Depth "-"）
```

- **Depth**：到 GC Root 的跳数。越小越接近根。`-` = 此路径走不到 GC Root，不是主因。
- **Retained Size**：断开这条引用能释放多少内存。**找 Retained Size 最大的那行——那就是主泄漏链。**
- **展开** Retained 最大的行 → 继续展开 → 追到 GC Root → 反向读："GC Root → ... → 持有者 → 泄漏对象"。
- 勾选 `Show nearest GC root only` 只显示最短路径。

### 堆转储 — 类列表

- `Show activity/fragment leaks`：自动检测可疑的 Activity/Fragment 泄漏实例。
- `All classes` → 按 Instance Count 降序 → 数量反常的（Activity > 1、Fragment > 0）= 泄漏。
- `Show project classes`：只看自己项目的类。

### 分配记录 — Table 标签页

- 按 **Count** 降序 → 排第一的类 = 分配最多的对象。
- 右键 → Jump to Source → 修复分配点。
- 拖动时间范围选择器，框选尖峰时段，只看那段时间的数据。

### 分配记录 — Visualizations 标签页

- 横轴 = 时间，纵轴 = 分配量。
- 每个颜色块 = 一种对象类型。
- **闪现即消失、反复出现** = 内存抖动（短命对象大量分配又立即回收）。
- **色块持续变宽不缩** = 持续分配。
- **堆叠色块越堆越高不下落** = 疑似泄漏。

### 分配记录 — 火焰图

- 横轴宽度 = 分配次数（Count），不是时间。
- 纵轴 = 调用栈深度（上方调用方、下方被调用方）。
- 最宽的条 = 分配最重的方法。右键 → Jump to Source。

### MAT — Dominator Tree

```
行                               Retained Heap   含义
────────────────────────────────────────────────────────
LeakRegistry.class               16,315,731       ← GC Root 落脚点（静态字段）
  └─ LeakRegistry 单例            16,315,211       ← 泄漏持有者
       └─ ArrayList               16,315,163       ← leakedActivities 列表
            └─ Object[]           16,315,139       ← 列表底层数组
                 └─ MainActivity     648,182       ← 泄漏的 Activity #1
                      └─ Bitmap     1,048,627      ← 泄漏的 1MB Bitmap
                 └─ MainActivity     648,182       ← 泄漏的 Activity #2
```

- 每缩进一级 = "被支配"（父回收，子必回收）。
- **Retained Heap** = 这个对象 + 所有被它支配的对象的内存总和。
- 右键任意行 → **Path to GC Roots → exclude weak references** 看引用链。

## GC Root 类型

| GC Root | 例子 | 存活时间 |
|---------|------|---------|
| 静态字段 | `object LeakRegistry { val list = ... }` | JVM 全生命周期 |
| 活跃线程 | `Thread { ... }.start()` — 未 interrupt 或 daemon | 线程存活期间 |
| 栈帧局部变量 | `val x = ...; Handler.post { use(x) }` | 方法执行期间 |
| JNI 引用 | Native 代码通过 `NewGlobalRef` 持有 Java 对象 | 直到 `DeleteGlobalRef` |

读 References 链时，终点是以上四种之一 = 找到了"为什么不能被 GC"的根因。

## 常见泄漏模式

### 模式 1：静态字段持有 Context/Activity

```kotlin
object AnalyticsTracker {
    private val screens = mutableListOf<Activity>()  // BUG：永远不清空
    fun track(activity: Activity) { screens.add(activity) }
}
// 修复：用 WeakReference，或在 onDestroy 时 detach
```

**检测**：堆转储 → 按 Activity 实例数排序 → >1 = 泄漏 → References → 追到 `AnalyticsTracker`。

### 模式 2：DisposableEffect 忘写 onDispose（Compose）

```kotlin
DisposableEffect(Unit) {
    registerListener(context)
    onDispose { /* BUG：空的——忘了 unregisterListener() */ }
}
// 修复：在 onDispose 里加 unregisterListener(context)
```

**检测**：堆转储 → Show activity/fragment leaks → References → 追到 LeakRegistry 或 listener 列表。

### 模式 3：协程/线程超出作用域

```kotlin
fun loadData() {  // 方法返回了，但线程还在跑
    val data = fetchBigData()  // 10MB 局部变量
    thread { process(data); Thread.sleep(60000) }  // BUG：线程没被终止
}
// 修复：用 lifecycleScope，在 onDestroy 时 cancel
```

**检测**：堆转储 → 搜 `byte[]` → 按 Retained Size 降序 → 找到 10MB 的数组 → References → 追到一个还在跑的线程。

### 模式 4：内存抖动（非泄漏）

```kotlin
Canvas {
    for (i in 0..1000) {
        drawText("label_$i", ...)  // 每帧创建 1000 个 String！
    }
}
// 修复：预构建字符串，用 Paint 对象缓存
```

**检测**：分配记录 → Visualizations → 某类色块反复闪现 → Table → 按 Count 降序 → Jump to Source → 移到循环外面或加缓存。

### 模式 5：Handler/内部类泄漏

```kotlin
class MyActivity : Activity() {
    private val handler = Handler(Looper.getMainLooper())
    fun schedule() {
        handler.postDelayed({ doWork(this) }, 60000)  // 'this' 被捕获
    }
}
// 修复：onDestroy 里 handler.removeCallbacks(null)
```

**检测**：堆转储 → Activity 实例数 >1 → References → `this$0` 在匿名类中 → 被 Handler → MessageQueue 持有。

## MAT OQL 示例

```
-- 查找所有 Activity 实例
SELECT * FROM INSTANCEOF android.app.Activity

-- 查找大于 1MB 的 Bitmap
SELECT * FROM android.graphics.Bitmap WHERE retainedHeapSize > 1000000

-- 查找项目中实例数最多的类
SELECT className, COUNT(*) FROM INSTANCEOF com.example.* GROUP BY className

-- 查找被特定类持有的所有对象
SELECT * FROM INSTANCEOF com.example.knowledgepalace.LeakRegistry
```

## 修复后验证清单

修复代码后：

1. **重新抓堆转储** → 验证泄漏类型的实例数恢复为 1（或 0）
2. **重新录分配记录** → 验证抖动类的分配次数大幅下降
3. **旋转测试**：设备旋转 5 次 → 堆转储 → Activity 实例数仍为 1
4. **切后台测试**：回桌面 → 返回 → Activity 仍为 1
5. 如果用 MAT：把修复后的 hprof 加入 Compare Basket → 对比修复前后的差异

## 核心原则

- **不要猜，先抓数据。** 看代码之前，先跑一次 profiler 录制。
- **实时视图 → 分配记录 → 堆转储** 是递进链条，每一步回答一个更具体的问题。
- **找泄漏看 Retained Size，别看 Shallow Size。** 浅层大小对深层持有大量子对象的对象有误导性。
- **References 里 Depth = 1 或 2 = 这是主阻塞路径。** Depth > 100 = 框架内部链，通常不是问题。
- **分析内存必须在 Run Configuration 切到 debuggable。** 内存分析任务都需要可调试模式。
- **抓堆转储时 Java 内存会临时增加** — 正常现象，它和 App 在同一进程中。
