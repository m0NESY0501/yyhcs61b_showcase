# Gitlet设计文档
# 概述
本项目实现了一个简化版的Git版本控制系统，名为Gitlet。该系统允许用户跟踪文件变化、创建提交、管理分支、合并代码以及恢复历史版本。Gitlet通过在项目目录中创建.gitlet目录来存储所有版本控制元数据，实现对文件系统的非侵入式管理。系统采用对象数据库设计，所有提交、文件内容和元数据都以对象形式存储并使用SHA-1哈希进行唯一标识。

# 核心功能
初始化(init)：在当前目录创建Gitlet仓库

添加(add)：将文件添加到暂存区

提交(commit)：保存当前暂存区状态为新提交

删除(rm)：从暂存区和工作区移除文件

### 分支管理：

创建新分支(branch)

删除分支(rm-branch)

切换分支(checkout)

合并(merge)：将指定分支合并到当前分支，自动处理冲突

重置(reset)：将当前分支指针移动到指定提交
### 状态查看：
显示当前状态(status)

显示提交历史(log, global-log)

按消息查找提交(find)

文件恢复：从历史提交中恢复文件(checkout)
# Classes and Data Structures
## Main.java
程序入口点，负责解析命令行参数并分发到相应功能。

### Fields/Methods:

main(String[] args): 解析命令行参数，调用Repository中的相应方法

异常处理：捕获并处理IO异常和GitletException
## Repository.java

实现Gitlet核心功能的类，包含所有版本控制操作。

### Fields:

CWD: 当前工作目录

GITLET_DIR: .gitlet目录路径
### 核心方法:

init(): 初始化仓库

add(String fileName): 将文件添加到暂存区

commit(String message): 创建新提交

rm(String fileName): 从版本控制中移除文件

log()/globallog(): 显示提交历史

status(): 显示当前仓库状态

checkoutBranch()/checkoutHeadFile()/checkoutHistoryFile(): 检出不同版本

branch()/rmBranch(): 分支管理

merge(String branchName): 合并分支

reset(String commitID): 重置到指定提交

find(String message): 按提交消息查找
### 辅助方法:

gitCheck(): 验证是否在Gitlet仓库中

createStagingMap()/createRMMap(): 加载暂存区和删除区

getCurrentCommit(): 获取当前提交

handleConflict(): 处理合并冲突

getAncestorShas()/findSplitPoint(): 查找共同祖先（用于合并）
## Commit.java
表示提交对象，存储文件状态、消息、时间戳和父提交引用。

### Fields:

message: 提交消息

timestamp: 提交时间

parent: 父提交哈希

viceParent: 合并时的第二父提交哈希

tree: 文件名到blob ID的映射（当前提交的文件状态）
#### 核心方法:

构造函数（初始提交、普通提交、合并提交）

getParentCommit()/getViceParentCommit(): 获取父提交

getTree(): 获取文件状态映射
## Utils.java
提供各种实用功能，包括文件操作、序列化和哈希计算。

### 核心方法:

sha1(): 计算SHA-1哈希

readContents()/writeContents(): 读写文件内容

readObject()/writeObject(): 序列化/反序列化对象

restrictedDelete(): 安全删除文件

plainFilenamesIn(): 获取目录中的文件列表

join(): 路径拼接

serialize(): 对象序列化

## GitletException.java

自定义异常类，用于处理Gitlet特定错误。

# Algorithms
### 对象存储

文件内容(blob)：文件内容被存储在.gitlet/blobs目录下，文件名是其SHA-1哈希

提交对象：每个提交被序列化并存储在.gitlet/commits目录下，文件名是其SHA-1哈希

分支指针：分支信息存储在.gitlet/refs/heads目录下，文件内容是指向提交的哈希

### 暂存区管理

暂存区是一个映射（文件名→blob ID），存储在.gitlet/staging文件中

删除区同样是一个映射，存储在.gitlet/rm文件中

提交时，暂存区和删除区的内容被合并到新的提交树中，然后清空
### 分支与HEAD
HEAD文件包含当前分支的路径（如refs/heads/master)

分支文件包含该分支指向的提交哈希

切换分支时，更新HEAD指向并检出相应提交的文件
### 合并算法
查找分割点：
通过BFS遍历提交图，找到当前分支和目标分支的最近共同祖先

三路合并：
比较分割点、当前分支和目标分支的文件状态
对于每个文件，根据三者状态决定合并结果

冲突处理：
当同一文件在两个分支中有不同修改时，生成冲突标记
将冲突内容写入工作区文件，标记为冲突状态

合并提交：
创建包含两个父提交引用的新提交，记录合并操作
### 查找分割点
使用BFS搜索算法遍历两个分支的提交历史

从当前分支构建祖先集合

从目标分支遍历，找到第一个也在当前分支祖先集合中的提交
# Persistence
对象持久化：使用Java序列化将对象保存到文件

文件存储：文件内容以原始字节形式存储在blobs目录

分支指针：文本文件，包含提交哈希

HEAD：文本文件，包含当前分支路径

暂存区/删除区：序列化HashMap，存储文件名和blob ID的映射
## 目录结构：
.gitlet/

├── blobs/        存储文件内容

├── commits/      存储提交对象

├── refs/

│   └── heads/    存储分支指针

├── HEAD          当前分支指针

├── staging       暂存区

└── rm            删除区





