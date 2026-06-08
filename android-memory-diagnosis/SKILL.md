---
name: android-memory-diagnosis
description: Diagnose Android memory issues — leaks, churn, and large allocations — using Android Studio Profiler and MAT. Covers capture → analyze → locate → fix end-to-end.
category: software-development
---

# Android 内存问题诊断

## 触发条件

- 用户要求诊断、排查或修复 Android 内存问题
- 用户遇到 OOM、ANR 或应用逐渐变慢
- 用户想知道针对某种症状该用什么工具
- 用户有 `.hprof` 文件需要分析

---

## 一、认问题：泄漏、溢出、抖动

三种内存问题的本质不同，选错工具等于白干。

### 对比

| | 泄漏 (Leak) | 抖动 (Churn) | 溢出 (OOM) |
|---|---|---|---|
| **对象是否回收** | ❌ 永不回收 | ✅ 正常回收 | —（已崩溃） |
| **根因** | GC Root 不该持有但持有了 | GC 太频繁，暂停累积 | 堆空间不够 |
| **内存曲线** | 持续 ↗ 不降 | 锯齿状剧烈振荡 | — |
| **用户感受** | 最终 OOM 崩溃 | 掉帧、卡顿 | 立刻崩溃 |
| **首选工具** | 堆转储 (Heap Dump) | 分配记录 (Allocations) | 实时视图 + 堆转储 |

### 类比

```
泄漏 = 仓库堆满废品，没人清 → 最终塞爆
溢出 = 仓库爆了，新货进不来 → 立刻崩
抖动 = 仓管疯狂进出盘点——仓库没爆，但正常作业被频繁打断 → 卡
```

### 内存泄漏

对象已不需要，但被 GC Root 间接持有，GC 判定它"还活着"，永远不回收。泄漏的终局通常是 OOM——但不是所有 OOM 都因为泄漏。

### 内存溢出 (OutOfMemoryError)

JVM/ART 尝试分配内存时堆里找不到足够大的连续空间，抛出 `OutOfMemoryError`。

**Android 堆上限**（远小于 PC）：

| 设备 | 典型上限 |
|------|---------|
| 低端机 2GB RAM | 128-192 MB |
| 中端机 4GB RAM | 256-384 MB |
| 高端机 8GB+ RAM | 512 MB+ |

**OOM 五大场景**（按频率）：

1. **Bitmap 过大** — 4000×3000 ARGB_8888 = 48MB，一张照片
2. **泄漏累积** — 单次 1MB × 30 分钟 = 30MB → 再加载图片 → OOM
3. **大对象一次性分配** — `ByteArray(100MB)` 在 192MB 设备上
4. **线程过多** — 每线程 1MB 栈，200 线程 = 200MB
5. **代码膨胀** — 方法数超 64K 或过多类加载

**急救代码模板**：

```kotlin
try {
    val bitmap = BitmapFactory.decodeFile(path)
} catch (e: OutOfMemoryError) {
    // ① 降级：更小采样率
    val opts = BitmapFactory.Options().apply { inSampleSize = 4 }
    val smaller = BitmapFactory.decodeFile(path, opts)
    // ② 清缓存
    imageCache.evictAll()
    // ③ 建议 GC
    Runtime.getRuntime().gc()
    // ④ 重试
    val retry = BitmapFactory.decodeFile(path, opts)
}
```

### 内存抖动 (Memory Churn)

短时间内大量创建临时对象 → 立即丢弃 → GC 被反复触发 → 暂停时间累积 → UI 掉帧。**对象在回收，但回收本身成了问题。**

**常见场景**：

| 场景 | 为什么抖 |
|------|---------|
| `onDraw` / Canvas 里 `new Paint()` | 每帧创建，60fps = 60 次/秒 |
| Compose 高频重组里直接 `new` 大对象 | 每帧重组 = 每帧分配 |
| 列表 item 里创建临时对象 | 滚动时高频创建/丢弃 |
| 字符串拼接循环 `result += "item_$x"` | 每次 `+=` = 新 String |
| 自动装箱 `Integer count = 0; for (...) count++` | 每次 ++ = 新 Integer |

**严重程度衡量**（GC 日志）：

```
adb logcat | grep "art.*GC"

GC: concurrent copying freed 8542(512KB), paused 1.2ms total 18ms
GC: concurrent copying freed 12034(768KB), paused 2.5ms total 28ms
...（每秒多次 = 严重）
```

- GC 日志刷屏 = 必须优化
- `paused > 5ms` = 用户可感知掉帧（60fps 每帧仅 16ms）

---

## 二、防问题：开发阶段预防清单

| 问题 | 预防 |
|------|------|
| Bitmap 过大 | `BitmapFactory.Options.inSampleSize`；用 Glide/Coil |
| 图片列表 | RecyclerView + LRU 缓存，不可见图及时回收 |
| 大文件 | 流式读、分块，别一次性 `readBytes()` |
| 静态持有 Context | `object` 里用 `WeakReference` 或 `ApplicationContext` |
| Handler | `onDestroy` 里 `removeCallbacks(null)` |
| Compose DisposableEffect | `onDispose` 里必须反注册 |
| 线程/协程 | 用 `lifecycleScope`，禁用 `GlobalScope` |
| Canvas 里 new 对象 | `remember { Paint() }` 复用 |
| 字符串拼接循环 | 用 `StringBuilder` 或预构建 |

**监控手段**（运行时发现苗头）：

| 工具 | 用途 | 建议 |
|------|------|------|
| **LeakCanary** | Activity/Fragment 泄漏 | **强烈推荐集成** |
| AS Profiler 实时视图 | 内存曲线趋势 | 开发阶段主动看 |
| GC 日志 | GC 频率/暂停 | 频率异常高 = 泄漏或抖动 |
| `dumpsys meminfo` | 精确 Pss | >200MB 警惕 |

---

## 三、查问题：诊断流程

### 决策树

```
症状："App 越来越慢 / OOM 了"
  │
  ├── 第一步：实时视图（profileable 即可）
  │     Profiler → View Live Telemetry → 看 MEMORY 曲线
  │       ├── 持续爬升不降 → 泄漏 → 第三步 堆转储
  │       ├── 锯齿剧烈起伏 → 抖动 → 第二步 分配记录
  │       └── 突然尖峰     → 大对象 → 第二步 分配记录
  │
  ├── 第二步：录制 Java/Kotlin 分配（需 debuggable）
  │     开始录制 → 操作 App → 停止
  │       ├── Visualizations：哪个类色块剧烈涌现？
  │       ├── Allocations 标签页：按 Count 降序 → 右键 → Jump to Source
  │       └── 火焰图：最宽的条 = 分配最重的方法
  │
  └── 第三步：捕获堆转储（需 debuggable）
        抓取 → 解析 → 类列表
          ├── "Show activity/fragment leaks" 过滤
          ├── 搜可疑类 → 实例数 > 1（Activity）/ > 0（Fragment）= 泄漏
          ├── 右键实例 → References → 追到 GC Root
          └── 搞不定 → 导出 .hprof → MAT
```

### 工具速查

| 任务 | 模式 | 看什么 | 核心操作 |
|------|------|--------|---------|
| View Live Telemetry | profileable | 曲线趋势 | 第一步筛查 |
| Track Memory Consumption | debuggable | 对象分配时间线 | Count 降序 → Jump to Source |
| Analyze Memory Usage | debuggable | 所有存活对象 | References → 追 GC Root |
| MAT | 离线 | 深度分析 | Dominator Tree + OQL |

---

## 四、读结果：各视图解读

### 堆转储 — References 标签页

```
Reference             Depth   Native   Shallow   Retained
────────────────────────────────────────────────────────
Bitmap@2000847480      5      1,048,577   50      1,048,627   ← 目标
  Index 8 in Object[]   4          0       40     10,486,310   ← ⚠️ 主泄漏链
  referent in Cleaner  768         0       36          92      ← 忽略
  mBitmap in Canvas      -        99       44         143      ← 不可达 (Depth -)
```

- **Retained Size**：断开这条引用能释放多少。**找最大的那行 → 展开 → 追到 GC Root。**
- **Depth**：到 GC Root 跳数。`-` = 此路不通 GC Root，不是主因。
- 勾选 `Show nearest GC root only` 只显示最短路径。

### 堆转储 — 类列表

- `Show activity/fragment leaks`：自动检测可疑实例
- 按 Instance Count 降序 → Activity > 1 / Fragment > 0 = 泄漏
- `Show project classes`：只看自己项目

### 分配记录 — Table + Visualizations + 火焰图

**Table**：按 Count 降序 → 右键 → Jump to Source

**Visualizations**：横轴=时间，纵轴=分配量。色块闪现即消失 = 抖动；色块持续涨 = 持续分配。

**火焰图**：横轴宽度 = 分配次数（不是时间），最宽的条 = 最重的分配者。

### GC Root 类型

| 类型 | 例子 | 存活 |
|------|------|------|
| 静态字段 | `object LeakRegistry { val list = ... }` | JVM 全生命周期 |
| 活跃线程 | `Thread { ... }.start()` 未 interrupt | 线程存活期间 |
| 栈帧局部变量 | `val x = ...; Handler.post { use(x) }` | 方法执行期间 |
| JNI 引用 | Native `NewGlobalRef` | 直到 `DeleteGlobalRef` |

References 链终点是以上四种之一 = 找到根因。

---

## 五、修问题：常见案例

### 泄漏案例

**案例 1：静态字段持有 Context**

```kotlin
object AnalyticsTracker {
    private val screens = mutableListOf<Activity>()  // BUG
    fun track(activity: Activity) { screens.add(activity) }
}
// 检测：堆转储 → Activity 实例 > 1 → References → 追到 AnalyticsTracker
// 修复：用 WeakReference，或 onDestroy 时 detach
```

**案例 2：DisposableEffect 忘 onDispose（Compose）**

```kotlin
DisposableEffect(Unit) {
    registerListener(context)
    onDispose { /* BUG: 空的 */ }
}
// 检测：堆转储 → Show activity/fragment leaks → 追到 LeakRegistry
// 修复：onDispose 里加 unregisterListener(context)
```

**案例 3：线程超出作用域**

```kotlin
fun loadData() {
    val data = fetchBigData()  // 局部变量
    thread { process(data); Thread.sleep(60000) }  // BUG
}
// 检测：堆转储 → byte[] → Retained 降序 → 追到运行中的线程
// 修复：用 lifecycleScope，onDestroy 时 cancel
```

**案例 4：Handler/内部类**

```kotlin
class MyActivity : Activity() {
    private val handler = Handler(Looper.getMainLooper())
    fun schedule() { handler.postDelayed({ doWork(this) }, 60000) }
}
// 检测：Activity > 1 → References → this$0 → Handler → MessageQueue
// 修复：onDestroy 里 handler.removeCallbacks(null)
```

### 抖动案例

**案例 5：Canvas 每帧 new 对象**

```kotlin
Canvas {
    for (i in 0..100) { drawText("$i", Paint(), ...) }
}
// 检测：分配记录 → Visualizations → Paint 色块反复闪现 → Table → Count 降序 → Jump to Source
// 修复：remember { Paint() } 缓存，循环外预构建字符串
```

---

## 六、验问题：修复后验证清单

1. **重新抓堆转储** → 泄漏类型实例数恢复为 1（或 0）
2. **重新录分配记录** → 抖动类分配次数大幅下降
3. **旋转测试**：旋转 5 次 → 堆转储 → Activity 数仍为 1
4. **切后台测试**：回桌面 → 返回 → Activity 数仍为 1
5. MAT：修复前后 .hprof 加入 Compare Basket 对比

---

## 七、Agent 操作手册

### 用户报内存问题时

```
用户："App 越来越卡 / OOM 了"
  │
  ├── ① 先问：有没有 hprof？
  │     ├── 有 → 要文件路径 + 操作描述 → 直接分析
  │     └── 没有 → 指导抓取
  │
  ├── ② 指导抓取（用户不会的傻瓜式指令）：
  │     "1. Run Configuration → 以可调试模式运行
  │      2. 点 Profile → 等 App 启动 → 做有问题的操作
  │      3. Profiler → Home → Analyze Memory Usage → 抓堆转储
  │      4. Past Recordings → 导出按钮 → 保存 .hprof"
  │
  ├── ③ 同时问操作步骤 + 症状：
  │     "做了什么之后卡？内存一直涨还是反复跳？"
  │
  └── ④ 分析 → 给结果：
        "你的 MainActivity 有 5 个实例。
         引用链：LeakRegistry（静态字段）→ ArrayList → MainActivity。
         修复：在 DisposableEffect 的 onDispose 里加 detach(context)。"
```

### 诊断所需材料（优先级）

**第一梯队（必填）**：`.hprof` 文件 + 操作描述

**第二梯队（加分）**：References 截图 / 类列表截图 / 实时曲线截图 / GC 日志

**第三梯队（锦上添花）**：`dumpsys meminfo` 输出 / 用户指出的可疑代码区域

---

## 八、MAT 深度分析

### MAT 核心功能

| 功能 | 时机 | 操作 |
|------|------|------|
| Leak Suspects | 打开 hprof 自动生成 | 读顶部嫌疑 → Details |
| Dominator Tree | 找最大 Retained Heap 持有者 | Retained Heap 降序 → 展开 |
| Histogram | 按类统计 | 包名过滤 → Retained 排序 |
| OQL | 自定义查询 | 见下方 |
| Compare Basket | 两次 hprof 对比 | 加篮 → Compare |

### Dominator Tree 读法

```
行                               Retained Heap   含义
────────────────────────────────────────────────────────
LeakRegistry.class               16,315,731       ← GC Root 落脚点
  └─ LeakRegistry 单例            16,315,211       ← 泄漏持有者
       └─ ArrayList               16,315,163       ← 列表
            └─ Object[]           16,315,139       ← 数组
                 └─ MainActivity     648,182       ← 泄漏实例
```

- 缩进 = "被支配"（父回收，子必回收）
- Retained Heap = 自身 + 所有被支配对象
- 右键 → Path to GC Roots → exclude weak references

### 常用 OQL

```
SELECT * FROM INSTANCEOF android.app.Activity
SELECT * FROM android.graphics.Bitmap WHERE retainedHeapSize > 1000000
SELECT className, COUNT(*) FROM INSTANCEOF com.example.* GROUP BY className
SELECT * FROM INSTANCEOF com.example.LeakRegistry
```

---

## 九、核心原则

- **不要猜，先抓数据。** 跑 profiler 比读代码快。
- **实时视图 → 分配记录 → 堆转储** 是递进链，每一步回答一个更具体的问题。
- **找泄漏看 Retained Size，别看 Shallow Size。**
- **References 里 Depth = 1 或 2 = 主阻塞路径。** Depth > 100 = 框架内部链。
- **内存分析必须 debuggable 模式。**
- **抓堆转储时 Java 内存临时增加** — 正常现象。
- **LEAK 的 GC Root 最常见是 STATIC 字段** — 搜 `object` / `companion object` 优先。
