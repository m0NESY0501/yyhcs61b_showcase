# CS61B：数据结构（Spring 2021）

![Java](https://img.shields.io/badge/Java-ED8B00?style=flat&logo=java&logoColor=white)
![Data Structures](https://img.shields.io/badge/Data_Structures-Algorithm-blue)
![All Labs Completed](https://img.shields.io/badge/Labs-8/8_Completed-success)
![All Projects Completed](https://img.shields.io/badge/Projects-4/4_Completed-success)

本仓库完整记录了我独立完成 UC Berkeley CS61B（Spring 2021）全部 labs 与 projects 的实现过程，核心内容覆盖 Java、数据结构、算法与基础软件工程实践。整个仓库在约 2 个月内完成，累计 107+ commits。

其中两个代表性项目分别是：**Gitlet**（简化版 Git 版本控制系统）与 **BYOW**（基于随机种子的 2D 世界生成游戏），它们集中体现了我在系统设计、持久化、图算法与工程实现上的能力。

---

## 🔥 核心项目

### **Gitlet —— 简化版 Git 版本控制系统**

该项目实现了一个可用的本地版本控制系统，覆盖 Git 的核心工作流与关键内部机制。

**项目要点：**

- ✅ **完整支持 13 个核心命令**：`init`、`add`、`commit`、`rm`、`log`、`global-log`、`status`、`checkout`、`branch`、`rm-branch`、`reset`、`merge`、`find`
- ✅ **对象模型设计清晰**：以 Commit DAG、Blob 内容寻址存储、Staging Area 暂存区为核心抽象
- ✅ **基于 SHA-1 的内容寻址存储**：相同内容文件共享同一 Blob，减少重复存储并便于完整性校验
- ✅ **实现三路合并算法**：通过 BFS 查找 split point，再进行三方文件比较与冲突标记写回
- ✅ **支持持久化仓库结构**：使用 `.gitlet` 目录组织 commits、blobs、refs、HEAD 与暂存区数据
- ✅ **代码规模 1000+ 行**：从零独立完成，覆盖版本控制系统的关键功能链路

**技术亮点：**

- 使用 HashMap 管理暂存区中的文件名到 Blob 哈希映射
- 通过 Java Serializable 完成 Commit 等对象持久化
- 合并冲突时生成标准冲突标记并直接写回工作区文件

→ **[详见 proj2README.md](proj2README.md)**

---

### **BYOW —— Build Your Own World（2D 随机世界生成游戏）**

该项目实现了一个可交互的 2D 地牢探索游戏，重点在于确定性随机生成、地图连通性与轻量化存档设计。

**项目要点：**

- ✅ **基于 seed 的确定性随机生成**：相同种子输入必然生成完全一致的地图
- ✅ **房间生成算法**：随机采样位置与尺寸，结合重叠检测与多次重试机制构建有效房间集合
- ✅ **走廊连接策略**：按房间顺序建立 L 型走廊，保证整体地图连通性
- ✅ **自动墙体封闭算法**：通过 `putWALLsafely()` 仅在 `NOTHING` 区域补墙，避免破坏已有房间结构
- ✅ **键盘交互系统**：支持 WASD 移动、菜单 UI 与完整游戏状态切换流程
- ✅ **存档设计采用操作序列重放**：保存输入字符串而非序列化整张地图，读档时通过重放恢复状态

**技术亮点：**

- 存档方案选择“操作序列”而非“对象快照”，以换取更小文件体积、更强可调试性与确定性验证能力
- 房间顺序连接策略在实现复杂度较低的前提下保证了世界连通性
- 基于 StdDraw 实现 2D 瓦片渲染与菜单交互

→ **[详见 proj3README.md](proj3README.md)**

---

## 📚 其他作业完成情况

| 类别 | 项目 | 核心内容 |
|------|------|---------|
| **proj0** | 2048 | 游戏状态管理、棋盘更新逻辑、基础搜索思路 |
| **proj1** | Deques & Guitar Hero | LinkedList / ArrayDeque 实现、Karplus-Strong 音频合成 |
| **lab1-lab2** | Git & Debugging | Git 工作流、调试基础、JUnit 单元测试 |
| **lab3** | Randomized Testing | 随机化测试、性质测试、Bug 复现与定位 |
| **lab4** | Debugging Practice | Java 整数缓存陷阱、测试设计 |
| **lab6** | Capers | 文件 I/O、Java 序列化、目录结构设计 |
| **lab7** | BSTMap | 二叉搜索树、Map 接口实现、迭代器 |
| **lab8** | HashMap | 哈希表实现、冲突处理、装载因子控制 |

---

## 🎯 能力映射

| 技术能力 | 本仓库中的证据 |
|---------|--------------|
| **数据结构** | BST（lab7）、HashMap（lab8）、LinkedList / ArrayDeque（proj1） |
| **图论与图算法** | 使用 BFS 查找 merge split point（proj2） |
| **面向对象设计** | `Commit` / `Blob` / `Repository` 与 `Engine` / `World` / `Room` 的职责拆分 |
| **序列化与持久化** | Java Serializable、`.gitlet` 文件系统设计、操作序列重放存档（proj2 / proj3） |
| **测试与调试** | JUnit 单元测试、随机化测试框架、调试练习（lab3 / lab4） |
| **2D 图形渲染** | StdDraw 渲染、菜单系统、游戏状态控制（proj3） |
| **算法实现** | 三路合并、程序化生成、内容寻址存储 |

---

## 🗂️ 仓库结构

```
cs61b/
├── proj0/          # 2048 游戏实现
├── proj1/          # Deques + Guitar Hero
├── proj2/          # Gitlet 版本控制系统
├── proj3/          # BYOW 2D 世界生成游戏
├── lab1-lab8/      # 各周实验
├── proj2README.md  # Gitlet 设计文档
├── proj3README.md  # BYOW 设计文档
└── README.md       # 仓库总览
```

---

## 🔗 补充材料

- **课程**：CS61B Data Structures（Spring 2021）
- **学校**：UC Berkeley
- **授课教师**：Josh Hug
- **主题**：Java、OOP、数据结构、算法、软件工程基础
- **知乎专栏**：<https://www.zhihu.com/column/c_1982810555128512732>

---

本仓库为私有仓库，主要用于科研实习 / 技术面试场景下的项目能力展示。所有代码均为本人独立完成。
