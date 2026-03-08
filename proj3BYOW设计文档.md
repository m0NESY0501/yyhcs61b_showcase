# BYOW (Build Your Own World) 设计文档

## 1. 项目概述

**目标**：实现一个 2D 程序化生成地牢探索游戏，支持基于种子的确定性世界生成、玩家交互与游戏状态持久化。

**核心特性**：

- ✅ 确定性世界生成：相同种子 → 完全相同的地图布局（关键测试需求）
- ✅ 基于房间的地牢设计：随机房间 + 走廊连接 + 墙体自动封闭
- ✅ 键盘交互系统：WASD 移动 + 菜单 UI（New Game / Load Game / Quit）
- ✅ 持久化策略：存储操作序列而非地图对象，通过重放恢复游戏状态

**技术栈**：

- Java + StdDraw 库（2D 瓦片渲染）
- 无第三方依赖（纯 JDK 实现）

---

## 2. 系统架构

### 2.1 核心类职责

#### **Main.java** — 程序入口

- **职责**：解析命令行参数，决定运行模式
  - 无参数 → 调用 `Engine.interactWithKeyboard()`（交互模式）
  - `-s <input>` → 调用 `Engine.interactWithInputString(input)`（自动测试模式）

#### **Engine.java** — 游戏引擎核心

- **职责**：处理世界生成、游戏状态管理、存档/读档逻辑
- **关键方法**：
  1. `interactWithInputString(String input)` — 命令行模式，解析输入字符串（如 `N1234SWWAD:Q`）
  2. `interactWithKeyboard()` — 键盘交互模式，启动 `GalaGame` UI 系统
  3. `constructFinalWorld(long seed)` — 世界生成主流程，调用房间生成 + 走廊连接
  4. `saveGame(String operations)` — 将操作序列写入 `save.txt`
  5. `loadGame()` — 从 `save.txt` 读取操作序列，重放还原状态
  6. `moveAvatar(World world, char direction)` — 处理玩家移动逻辑（WASD）

**设计亮点**：

- 分离了"世界生成"和"游戏交互"两个模块，便于单元测试
- 统一输入接口：无论是命令行字符串还是键盘输入，最终都转换为操作序列字符串处理

---

#### **Room.java** — 房间生成与连接

- **职责**：房间的创建、碰撞检测、墙体填充、走廊连接
- **核心方法**：
  1. `Room(int x, int y, int width, int height)` — 构造函数，定义房间左下角坐标和尺寸
  2. `checkRoom(TETile[][] world)` — 碰撞检测，检查房间区域是否全为 `NOTHING`（未被占用）
  3. `addRoom(TETile[][] world)` — 在地图上绘制房间（填充 `FLOOR`，周围填充 `WALL`）
  4. `addPath(Room target, TETile[][] world)` — 连接当前房间与目标房间，生成 L 型走廊
  5. `putWALLsafely(TETile[][] world, int x, int y)` — 安全填充墙体（仅当目标位置为 `NOTHING` 时填充）

**内部类**：

- `Point` — 存储房间中心点坐标 `(x, y)`，用于走廊连接时的路径计算

---

#### **World.java** — 世界状态封装

- **职责**：将地图数组 `TETile[][]` 和玩家位置 `(x, y)` 封装为单一对象，便于传递
- **字段**：
  - `TETile[][] map` — 游戏世界的瓦片数组
  - `int avatarX, avatarY` — 玩家当前坐标

**设计理由**：

- 避免在多个方法间传递多个参数（`world`, `playerX`, `playerY`），提升代码可读性
- 为未来扩展留余地（如添加敌人列表、道具位置等）

---

#### **GalaGame.java** — 图形界面与游戏循环

- **职责**：基于 StdDraw 实现菜单系统、游戏主循环、实时渲染
- **核心方法**：
  1. `startGame()` — 显示主菜单（New Game / Load Game / Quit），监听用户输入
  2. `receiveSEED()` — 接收用户输入的随机种子，逐字符显示并验证
  3. `runGame(String input)` — 游戏主循环，监听 WASD 键盘输入，实时更新地图
  4. `drawFrame(String message)` — 渲染当前帧（绘制地图 + 顶部 HUD 信息）

**技术细节**：

- 使用 `StdDraw.pause(50)` 控制帧率（约 20 FPS）
- 菜单输入通过 `StdDraw.hasNextKeyTyped()` 轮询实现
- 游戏状态机：`MENU` → `SEED_INPUT` → `PLAYING` → `PAUSED`（保存时）

---

## 3. 核心算法

### 3.1 确定性世界生成算法

#### **目标**

确保相同种子输入产生完全相同的地图布局（关键测试需求）。

#### **实现策略**

使用 **Java Random 类** 并固定种子：

```java
Random rand = new Random(seed);
```

**关键保证**：

- 所有随机操作（房间位置、尺寸、连接顺序）必须按固定顺序调用 `rand.nextInt()`
- 避免使用 `System.currentTimeMillis()` 或其他非确定性输入

---

### 3.2 房间生成算法

#### **流程**

```java
1. 初始化空地图 (80×50)，所有瓦片设为 NOTHING
2. 创建 roomList = []
3. 重复 30 次：
   a. 随机生成房间参数：
      - x = rand.nextInt(WIDTH - 10)
      - y = rand.nextInt(HEIGHT - 10)
      - width = rand.nextInt(7) + 3   // 宽度 3-9
      - height = rand.nextInt(7) + 3  // 高度 3-9
   b. 创建 Room(x, y, width, height)
   c. 调用 checkRoom() 检测碰撞
   d. 若无碰撞：
      - 调用 addRoom() 绘制房间
      - roomList.add(room)
4. 返回 roomList
```

**参数选择依据**：

- **30 次重试**：经验值，平衡地图密度与性能（更多重试 → 更多房间，但可能导致长时间失败循环）
- **尺寸 3-9**：确保房间不会过小（无法放置玩家）或过大（占据过多空间）

**优化方向**（未实现）：

- 使用空间分割算法（BSP Tree）替代随机重试，保证房间均匀分布
- 动态调整重试次数，根据已生成房间数量决定是否继续

---

### 3.3 走廊连接算法

#### **目标**

连接所有房间，保证地图连通性（任意两房间可达）。

#### **策略：顺序连接法**

```java
for i = 0 to roomList.size() - 2:
    roomList[i].addPath(roomList[i + 1], world)
```

**连通性保证**：

- 将房间构成一条链：`Room0 ↔ Room1 ↔ Room2 ↔ ... ↔ RoomN`
- 基于图论：链式结构必然连通（无需额外的 BFS 验证）

#### **L 型走廊生成算法**

```java
addPath(Room target, TETile[][] world):
    1. 获取当前房间中心点 (x1, y1)
    2. 获取目标房间中心点 (x2, y2)
    3. 绘制水平线段：从 (x1, y1) 到 (x2, y1)
       - 填充 FLOOR
       - 调用 putWALLsafely() 在上下两侧填充 WALL
    4. 绘制垂直线段：从 (x2, y1) 到 (x2, y2)
       - 填充 FLOOR
       - 调用 putWALLsafely() 在左右两侧填充 WALL
```

**关键设计**：`putWALLsafely()` 方法

```java
putWALLsafely(TETile[][] world, int x, int y):
    if world[x][y] == NOTHING:  // 仅当未被占用时填充
        world[x][y] = WALL
```

**设计理由**：

- 避免走廊墙体覆盖已有房间的地板（保持房间完整性）
- 走廊穿过房间时，不会在房间内部生成墙体

---

### 3.4 玩家移动逻辑

#### **流程**

```java
moveAvatar(World world, char direction):
    1. 根据 direction 计算目标坐标 (newX, newY)
       - 'W': (x, y+1)
       - 'A': (x-1, y)
       - 'S': (x, y-1)
       - 'D': (x+1, y)
    2. 检查目标瓦片类型：
       - 若 world.map[newX][newY] == FLOOR:
           a. world.map[oldX][oldY] = FLOOR   // 清除旧位置
           b. world.map[newX][newY] = AVATAR  // 绘制新位置
           c. world.avatarX = newX
           d. world.avatarY = newY
       - 否则：忽略移动（不允许穿墙或超出地图边界）
```

**边界情况处理**：

- 地图边缘：坐标检查 `0 <= newX < WIDTH` 和 `0 <= newY < HEIGHT`
- 墙体碰撞：仅允许移动到 `FLOOR` 瓦片
- 连续输入：每次仅处理一步移动（避免"加速"bug）

---

## 4. 持久化策略

### 4.1 设计决策：操作序列 vs 对象序列化

#### **方案 A：序列化 World 对象**

```java
saveGame():
    ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("save.txt"));
    out.writeObject(world);
```

**优点**：

- ✅ 实现简单（一行代码）
- ✅ 读档速度快（O(1) 反序列化）

**缺点**：

- ❌ 文件体积大（80×50 瓦片数组 + 对象开销 ≈ 50KB）
- ❌ 难以调试（二进制格式不可读）
- ❌ 不可回放（无法验证操作序列正确性）

---

#### **方案 B：存储操作序列（已采用）**

```java
saveGame(String operations):
    Files.write(Paths.get("save.txt"), operations.getBytes());
```

**示例操作序列**：`N1234SWWADDSS:Q`  

- `N1234` — 新游戏，种子 1234
- `SWWADDSS` — 玩家移动序列
- `:Q` — 退出并保存

**优点**：

- ✅ 文件体积小（~50 字节 vs 50KB，减少 1000 倍）
- ✅ 可读可调试（纯文本格式）
- ✅ 保证正确性（重放操作序列，必然还原到相同状态）
- ✅ 支持回放功能（如游戏录像、作弊检测）

**缺点**：

- ❌ 读档慢（需重新生成世界 + 重放所有操作，时间复杂度 O(n)）
- ❌ 依赖确定性（必须保证随机数生成器严格确定性）

---

### 4.2 实现细节

#### **保存流程**

```java
// 用户按下 :Q 时
String operations = "N" + seed + movementHistory;
saveGame(operations);
```

#### **读档流程**

```java
loadGame():
    1. String savedOps = new String(Files.readAllBytes(Paths.get("save.txt")))
    2. return interactWithInputString(savedOps)  // 重放操作
```

**关键点**：

- 读档时 **不保留** `":Q"`，因为这是保存命令而非游戏操作
- 若用户选择 Load Game 后输入 `L`，实际执行的是 `savedOps + "L"`（去掉 `:Q` 后追加新输入）

---

### 4.3 与工业界实践的对应

| 方案 | 对应系统 | 应用场景 |
|------|---------|---------|
| 操作序列存储 | Git（存储 diff 而非完整文件）| 节省空间、支持版本控制 |
| 操作序列存储 | 游戏回放系统（如《星际争霸》录像）| 验证比赛公平性、分析策略 |
| 操作序列存储 | 数据库 Write-Ahead Log (WAL) | 崩溃恢复、保证持久性 |
| 对象序列化 | Java 对象持久化、Redis RDB | 快照备份、快速恢复 |

**本项目选择理由**：

- 作为教学项目，优先考虑 **正确性** 和 **可测试性**（重放保证完全一致）
- 存档文件不频繁读取，O(n) 重放开销可接受

---

## 5. 测试策略

### 5.1 确定性验证

```java
@Test
public void testDeterminism() {
    World world1 = Engine.interactWithInputString("N1234S");
    World world2 = Engine.interactWithInputString("N1234S");
    assertArrayEquals(world1.map, world2.map);
}
```

### 5.2 碰撞检测测试

- 生成大量随机房间，统计碰撞率（预期 < 30%）
- 边界情况：房间贴地图边缘、房间重叠 1 像素

### 5.3 连通性测试

- BFS 遍历地图，检查所有 `FLOOR` 瓦片是否可达
- 测试用例：种子 1-1000，预期连通率 100%

### 5.4 存档正确性测试

```java
@Test
public void testSaveLoad() {
    String ops1 = "N5678SWWAD:Q";
    World world1 = Engine.interactWithInputString(ops1);
    saveGame(ops1);
    
    World world2 = loadGame();
    assertArrayEquals(world1.map, world2.map);
    assertEquals(world1.avatarX, world2.avatarX);
}
```

---

## 6. 已知限制与改进方向

### 6.1 当前限制

- ❌ 无敌人 AI（仅静态地图探索）
- ❌ 无道具系统（钥匙、宝箱等）
- ❌ 无多楼层设计（仅单层地牢）
- ❌ 无视野系统（Fog of War）

### 6.2 可能的优化

1. **世界生成**：
   - 使用 Perlin Noise 生成自然地形（河流、山脉）
   - 实现 BSP Tree 房间分割算法，替代随机重试

2. **游戏玩法**：
   - 添加敌人巡逻 AI（A* 寻路算法）
   - 道具系统（背包管理、装备系统）
   - 多楼层设计（楼梯连接、进度保存）

3. **性能优化**：
   - 使用脏矩形技术（Dirty Rectangle），仅重绘变化区域
   - 预计算房间连通性，缓存路径信息

---

## 7. 参考资料

- [Procedural Generation Wiki](http://pcg.wikidot.com/)  
- [Roguelike Development Guide](https://www.roguebasin.com/)  
- [Binary Space Partitioning (BSP) Trees](https://en.wikipedia.org/wiki/Binary_space_partitioning)  
- CS61B Spring 2021 Project 3 Spec

---

**最终总结**：本项目展示了如何通过简洁的算法（顺序连接、L 型走廊）实现复杂的游戏系统。持久化策略的权衡分析（操作序列 vs 对象序列化）体现了工程决策中的核心思想：在正确性、性能、可维护性之间找到平衡点。代码虽不复杂，但涵盖了游戏开发的关键技术（程序化生成、状态管理、实时渲染），为后续的游戏引擎开发和算法优化奠定了基础。

