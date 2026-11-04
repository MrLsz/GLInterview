# GLInterview

> 面试模拟记录仓库：从 [Glimmer](https://github.com/MrLsz/Glimmer) 知识库随机出题，模拟面试作答并打分，沉淀标准答案话术。

## 归档规则

- 一个知识点对应**一个大文件**，文件内按「题目一 / 题目二 / …」累积多道面试题。
- 分类对齐 Glimmer 知识库（不涉及 HarmonyOS）。

## 分类目录

```
GLInterview/
├── android/                 # Android 开发体系
│   ├── boot/                # 系统架构与启动
│   ├── binder/              # Binder IPC 机制
│   ├── component/           # 四大组件
│   ├── window-input/        # Window 与输入体系
│   ├── view-render/         # View 绘制与渲染
│   ├── build-system/        # 构建与编译体系
│   ├── java/                # Java 语言
│   ├── kotlin/              # Kotlin 语言
│   ├── jvm/                 # JVM 原理
│   └── matrix/              # Matrix APM 源码分析
├── ios/                     # iOS 开发体系
│   ├── objective-c/         # Objective-C 语言
│   ├── swift/               # Swift 语言
│   ├── runtime/             # 语言与运行时
│   ├── foundation/          # 系统底层机制
│   ├── uikit/               # UIKit 框架
│   └── startup/             # 启动与构建
└── ai/                      # AI 开发
    └── python/              # Python / Agent / LLM
```

## 面试题索引

| 文件 | 分类 | 题量 | 最新得分 |
| ---- | :---: | :---: | :---: |
| [应用启动分析面试题](android/boot/应用启动分析面试题.md) | Android · boot | 1 | 59 ❌ |
| [OC 语法面试题](ios/objective-c/OC语法面试题.md) | iOS · Objective-C | 1 | 88 ✅ |

## 评分标准

每题满分 100，80 及格，五维度：

| 维度 | 满分 |
| :---: | :---: |
| 概念准确性 | 25 |
| 核心区别完整度（流程题：流程完整度） | 30 |
| 适用场景（流程题：关键机制理解） | 20 |
| 语言流畅性 | 15 |
| 语言精准度 | 10 |
