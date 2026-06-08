# Android Memory Diagnosis

Android 内存问题诊断技能——覆盖泄漏、抖动、大对象分配，从采集 → 分析 → 定位 → 修复全流程。

## 安装

```bash
hermes skills install ttzhou616/android-memory-diagnosis
```

## 适用于

- Android Studio Profiler
- Eclipse MAT (Memory Analyzer Tool)
- Jetpack Compose / View-based 应用

## 涵盖内容

- 诊断决策树（根据症状自动选工具）
- 工具速查表（Profiler 三件套 + MAT）
- 各视图解读（References、火焰图、Dominator Tree）
- GC Root 类型表
- 5 种常见泄漏模式（含代码 + 检测 + 修复）
- MAT OQL 查询示例
- 修复后验证清单
- 诊断所需材料（三个梯队）
- Agent 问诊流程 + 傻瓜式 hprof 抓取指令
