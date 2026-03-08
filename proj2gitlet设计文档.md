# Gitlet 设计文档

## 1. 项目概述

**目标**：实现一个简化版 Git 版本控制系统，支持本地仓库的核心功能（提交历史、分支管理、三路合并）。

**实现范围**：

- ✅ 支持：初始化、暂存、提交、分支、合并、历史查询、文件恢复
- ❌ 不支持：远程仓库（push/pull/fetch）、变基（rebase）、子模块、Git LFS

**工程指标**：

- 代码量：1000+ 行 Java 代码（不含测试）
- 核心类：5 个（Main, Repository, Commit, Utils, GitletException）
- 支持命令：13 个完整实现的 Git 命令

---

## 2. 系统架构

### 2.1 核心实体及职责

#### **Repository** — 命令调度中心

- **职责**：所有 Git 命令的唯一入口，负责仓库状态管理、命令参数解析与转发
- **关键逻辑**：维护暂存区（staging area）和删除区（removal area）的内存映射

#### **Commit** — 不可变提交对象

- **职责**：表示文件系统某一时刻的快照，构成 DAG（有向无环图）结构
- **核心字段**：
  - `parent`: 单亲提交的 SHA-1 哈希
  - `viceParent`: 合并提交的第二父提交（merge 场景）
  - `tree`: HashMap<文件名, Blob SHA-1>，文件快照索引
  - `message`: 提交消息
  - `timestamp`: UTC 时间戳

**设计决策**：

- 为什么用 `viceParent` 而非 `List<parent>`？因为 Gitlet 仅支持两路合并（与真实 Git 的章鱼合并 Octopus Merge 不同），固定两个 parent 字段避免泛型复杂性。

#### **Blob** — 文件内容存储单元

- **职责**：通过 SHA-1 哈希实现内容寻址存储，相同内容的文件共享同一 Blob 对象
- **存储位置**：`.gitlet/blobs/<sha1前2位>/<sha1剩余部分>`（与 Git 一致）

**设计优势**：

- 去重存储：修改同一文件 100 次，若内容相同则仅存 1 个 Blob
- 完整性校验：SHA-1 冲突概率为 $2^{-160}$，可忽略不计

#### **Staging Area** — 暂存区

- **实现**：HashMap<String, String>（文件名 → Blob SHA-1）
- **持久化**：序列化至 `.gitlet/staging` 文件
- **作用**：记录 `add` 操作的待提交变更，`commit` 时清空

#### **Branch / HEAD** — 分支指针与当前分支追踪

- **Branch**：文本文件（`.gitlet/refs/heads/<branch_name>`），内容为该分支指向的 Commit SHA-1
- **HEAD**：符号引用文件（`.gitlet/HEAD`），内容为 `ref: refs/heads/<branch_name>`
- **设计理由**：采用符号引用而非直接存 Commit SHA-1，支持 `detached HEAD` 状态（虽然 Gitlet 未实现，但保持架构可扩展性）

---

### 2.2 实体关系

```
Commit Graph (DAG):
  c0 ←─ c1 ←─ c2 ←─ c4 (master)
         ↖         ↗
           c3 (feature)
```

- **Commit → Commit**：通过 `parent` 和 `viceParent` 构成 DAG
- **Commit → Blob**：通过 `tree` 映射，间接引用文件内容
- **Branch → Commit**：分支文件内容为 Commit SHA-1
- **HEAD → Branch**：HEAD 内容为分支路径（`ref: refs/heads/master`）

---

## 3. 存储设计

### 3.1 目录结构

```
.gitlet/
├── HEAD                    # 当前分支指针（内容：ref: refs/heads/master）
├── staging                 # 序列化的暂存区 HashMap
├── rm                      # 序列化的删除区 HashMap
├── blobs/                  # 文件内容存储（内容寻址）
│   ├── a1/                 # SHA-1 前 2 位作为目录（减少单目录文件数）
│   │   └── 23c4f...        # Blob 文件（SHA-1 剩余部分）
│   └── b2/
│       └── 8d9a1...
├── commits/                # Commit 对象存储
│   ├── 0f1e2d3c...         # Commit 文件（序列化对象）
│   └── 9a8b7c6d...
└── refs/
    └── heads/              # 分支指针目录
        ├── master          # master 分支（内容：Commit SHA-1）
        └── feature         # feature 分支
```

**设计对齐**：

- 与真实 Git 的 `.git/` 目录结构高度一致（学习目的）
- `blobs/` 和 `commits/` 使用两级目录，避免单目录文件过多导致文件系统性能下降（实际 Git 也用此策略）

---

### 3.2 序列化策略

| 对象类型 | 序列化方式 | 原因 |
|---------|-----------|------|
| Commit | Java Serializable | 包含复杂嵌套结构（HashMap、String、Date），标准序列化最简单 |
| Blob | 原始字节流 | 文件内容不需要结构化，直接写入减少开销 |
| Staging / RM | Java Serializable | HashMap 需要保留键值关系 |
| Branch / HEAD | 纯文本 | 仅存储字符串，文本文件便于调试 |

**权衡分析**：

- 未使用 JSON/XML：避免引入第三方库，保持纯 JDK 实现
- 未使用自定义二进制格式：工程复杂度 vs 性能提升不成比例

---

## 4. 核心算法

### 4.1 暂存区管理算法

#### **add 命令流程**

```java
1. 计算文件内容的 SHA-1 → blobHash
2. 读取当前 HEAD 指向的 Commit → currentCommit
3. 比较：
   - 若 currentCommit.tree.get(文件名) == blobHash
     → 文件未修改，从暂存区移除（若存在）
   - 否则：
     → 将文件内容写入 .gitlet/blobs/<blobHash>
     → 更新 staging.put(文件名, blobHash)
4. 序列化 staging 至磁盘
```

**关键优化**：

- 先检查文件是否已在当前 Commit 中，避免重复添加未修改文件
- 若文件恢复到 HEAD 版本，自动从暂存区移除（与 Git 行为一致）

---

### 4.2 三路合并算法

#### **核心思路**

合并两个分支时，需找到它们的"分割点"（split point），即最近公共祖先（LCA, Lowest Common Ancestor）。基于分割点、当前分支、目标分支的三方文件状态，决定合并结果。

#### **步骤 1: 查找分割点（BFS 算法）**

```java
splitPoint = findSplitPoint(currentCommit, targetCommit):
    1. ancestors = BFS遍历 currentCommit 的所有祖先 → 存入 HashSet
    2. queue = new Queue(targetCommit)
    3. while queue 不为空:
         commit = queue.dequeue()
         if commit in ancestors:
             return commit  // 找到第一个共同祖先
         queue.enqueue(commit.parent, commit.viceParent)
    4. return null  // 无共同祖先（不同仓库的分支）
```

**算法复杂度**：

- 时间：$O(C_1 + C_2)$，其中 $C_1, C_2$ 为两分支的提交数
- 空间：$O(C_1)$（存储祖先集合）

**为什么用 BFS 而非 DFS**：

- BFS 找到的第一个共同祖先必然是"最近"的（按拓扑距离）
- DFS 可能先遍历到更远的祖先，需额外逻辑判断"最近性"

---

#### **步骤 2: 三路文件比较**

对工作目录中的所有文件，根据三方状态决定操作：

| Split Point | Current Branch | Target Branch | 操作 |
|-------------|----------------|---------------|------|
| A | A | A | 不变 |
| A | B | A | 保持 Current（B） |
| A | A | C | 检出 Target（C） |
| A | B | C | **冲突** |
| 无 | A | B | **冲突** |
| A | 无 | A | 删除文件 |
| A | 无 | C | **冲突** |
| A | B | 无 | 保持 Current（B） |
| 无 | A | 无 | 保持 Current（A） |

**关键规则**：

- 若某一方未修改（与 Split Point 相同），采用另一方
- 若双方都修改且不同，标记冲突

---

#### **步骤 3: 冲突处理**

当检测到冲突时，生成冲突标记文件：

```java
<<<<<<< HEAD
Current Branch 的文件内容
=======
Target Branch 的文件内容
>>>>>>> <target_branch_name>
```

**实现细节**：

- 若某一方文件不存在，对应部分留空
- 冲突文件写入工作目录，用户手动解决后需 `add` 再提交
- 合并提交的 `viceParent` 字段指向目标分支的 Commit

---

### 4.3 SHA-1 哈希实现

**使用场景**：

1. Blob 文件名（内容寻址）
2. Commit 对象唯一标识
3. 完整性校验

**计算方式**：

```java
sha1Hash = SHA1(对象序列化字节流)
```

**为什么选择 SHA-1**：

- 与真实 Git 保持一致（学习目的）
- 虽然 SHA-1 已被证明存在碰撞攻击（2017 年 SHAttered），但对版本控制系统的实际影响可忽略
- 工业界已迁移至 SHA-256（Git 2.29+ 支持），但 Gitlet 作为教学项目保留 SHA-1

---

## 5. 命令实现细节

### 5.1 核心命令

| 命令 | 功能 | 关键实现 |
|------|------|---------|
| `init` | 初始化仓库 | 创建 `.gitlet/` 目录结构，生成初始 Commit（时间戳为 Unix Epoch） |
| `add <file>` | 添加到暂存区 | 计算 SHA-1，更新 staging HashMap，写入 blob |
| `commit <msg>` | 创建提交 | 合并 staging 和当前 Commit 的 tree，清空 staging 和 rm |
| `rm <file>` | 移除文件 | 从 staging 删除，添加到 rm 区，若文件在 HEAD 中则删除工作目录文件 |
| `log` | 显示当前分支历史 | 从 HEAD 开始，递归遍历 parent 指针，按时间倒序输出 |
| `global-log` | 显示所有提交 | 遍历 `.gitlet/commits/` 目录下所有 Commit 文件 |
| `find <msg>` | 按消息查找提交 | 遍历所有 Commit，过滤 message 字段 |
| `status` | 显示仓库状态 | 列出分支、暂存区、未暂存修改、未追踪文件（四个区域） |
| `checkout` | 恢复文件/切换分支 | 三种模式：`-- <file>`（从 HEAD 恢复）、`<commit> -- <file>`（从历史恢复）、`<branch>`（切换分支） |
| `branch <name>` | 创建分支 | 在 `.gitlet/refs/heads/` 创建文件，内容为当前 HEAD 的 Commit SHA-1 |
| `rm-branch <name>` | 删除分支 | 删除分支文件，禁止删除当前分支 |
| `reset <commit>` | 回退到指定提交 | 更新 HEAD 指向，检出 Commit 的所有文件，清空 staging |
| `merge <branch>` | 合并分支 | 执行三路合并算法，生成合并提交（包含 viceParent） |

---

### 5.2 边界情况处理

#### **未初始化仓库**

- 所有命令（除 `init`）需先调用 `gitCheck()`，检查 `.gitlet/` 目录是否存在
- 若不存在，抛出 `GitletException("Not in an initialized Gitlet directory.")`

#### **重复 `init`**

- 检测到 `.gitlet/` 已存在，抛出异常，避免覆盖现有仓库

#### **`add` 不存在的文件**

- 调用 `File.exists()`，若返回 false，抛出异常

#### **`commit` 无变更**

- 检查 staging 和 rm 是否均为空，若是，拒绝提交并提示 "No changes added to the commit."

#### **`merge` 冲突后的提交**

- 检测到冲突时，输出冲突文件列表，允许用户解决后再提交
- 合并提交的 message 格式：`Merged <target_branch> into <current_branch>.`

---

## 6. 测试策略

### 6.1 单元测试

- **Commit 对象序列化/反序列化**：验证 parent、tree、timestamp 等字段的正确性
- **SHA-1 哈希碰撞**：生成大量随机文件，统计冲突率（预期为 0）
- **暂存区逻辑**：测试 add → commit → add 同一文件的不同版本

### 6.2 集成测试

- **分支与合并**：创建多分支，模拟冲突和非冲突合并场景
- **文件恢复**：提交 → 修改 → checkout，验证工作目录恢复正确
- **历史遍历**：创建 100+ 提交的长链，验证 log 性能和正确性

### 6.3 压力测试

- **大文件**：测试 100MB+ 文件的 Blob 存储和 SHA-1 计算性能
- **深层提交树**：构造 1000 层深的提交链，验证栈溢出风险
- **并发冲突**：模拟多进程同时修改暂存区（虽然 Gitlet 不支持并发，但可测试文件锁机制）

---

## 7. 已知限制与未来改进

### 7.1 当前限制

- ❌ 不支持远程仓库（需实现网络协议和权限管理）
- ❌ 不支持 `.gitignore`（需解析规则并过滤文件）
- ❌ 不支持子模块（需嵌套仓库管理）
- ❌ 不支持 `git rebase`（需重写 Commit 历史）
- ❌ 不支持文件权限（Unix 权限位）

### 7.2 可能的优化方向

1. **性能优化**：
   - 使用 Pack 文件压缩存储（真实 Git 的增量存储）
   - 缓存常用 Commit 对象，减少磁盘 I/O

2. **功能扩展**：
   - 实现 `git stash`（临时保存未提交变更）
   - 支持交互式暂存（`git add -p`）

3. **代码质量**：
   - 引入设计模式（Command 模式统一命令接口）
   - 使用日志框架（SLF4J）替代 System.out.println

---

## 8. 参考资料

- [Git Internals - Plumbing and Porcelain](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain)
- [CS61B Spring 2021 Project 2 Spec](https://sp21.datastructur.es/materials/proj/proj2/proj2)
- Pro Git (Scott Chacon & Ben Straub)

---

**最终总结**：本项目完整实现了 Git 的核心数据结构和算法，包括 DAG 提交图、内容寻址存储、三路合并等关键技术。代码量虽仅千行，但涵盖了版本控制系统的本质挑战，为理解 Git 内部机制和分布式系统设计奠定了坚实基础。


