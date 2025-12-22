# 概述
本项目实现了一个2D基于像素的世界生成器和冒险游戏，玩家可以探索程序化生成的世界。游戏采用基于房间的关卡设计，房间之间有走廊连接，支持玩家角色移动，以及通过保存/加载功能实现游戏状态持久化。世界生成基于用户提供的种子，保证了可重现性。

## 核心功能
基于随机种子的程序化世界生成

基于房间的地牢生成，房间之间有连接走廊

通过键盘交互控制玩家角色移动

保存和加载游戏功能

基于文本的用户界面和菜单系统

确定性世界生成（相同种子产生相同世界）
## Classes and Data Structures
### Main.java
程序入口点。负责解析命令行参数并决定运行模式：

键盘模式：无参数时调用 Engine.interactWithKeyboard()。

字符串输入模式：使用 -s 参数时调用 Engine.interactWithInputString(String input)，用于自动测试。

### Engine.Class
真正的游戏启动引擎，包含interactWithKeyboard()，interactWithInputString(String input)两个核心交互方法，一个与命令行交互，一个与键盘交互

其中与键盘交互的主要调用galaGame类启动游戏，而galaGame类则是通过与命令行交互

interactWithInputString(String input)方法处理了世界创建，保存和加载

创建调用了此类中的constructFinalWorld(SEED)

保存调用了此类中saveGame(String)的方法，我没有采取将World.Class序列化而是将所有交互等价的字符串存入save.txt中

加载调用了此类中loadGame()的方法，将以L开头的字符串去掉L与被保存的历史进度字符串合并

角色移动则是调用了moveAvatar的方法

#### Fields
1. interactWithKeyboard()
2. interactWithInputString(String input)
3. constructFinalWorld()
4. constructRoom(finalWorldTile, SEED)
5. constructPath(finalWorldTile, SEED, roomList)
6. saveGame()
7. loadGame()
8. moveAvatar(World, direction)
9. generateAvatar(World, Room)

### Room.Class

这个类是用来生成地图独立房间的关键，包含检查room是否可以放置，保存Room的中心点，在地图上添加Room，在两个房间之间添加一条路

添加路的时候为了满足每个房间都可以达到，我选择了先加房子再加路，因为当路穿过时所受限制更小，更容易判断条件，我们不应该在房间内修建走廊的围墙故添加了方法putWALLsafely(finalWorldTile, x, y)

子类point.Class用于存储房间的中心点

#### Fields

1. Room()
2. Room(x, y, width, height)
3. checkRoom()
4. addRoom()
5. getPoint()
6. putWALLsafely(finalWorldTile, x, y)
7. addPath()

### World.Class
为解决地图中找不到角色本人的问题，我创造了一个简单的World.Class类，专门用于存放地图和人物的位置，包含了两个初始化方法
#### Fields
1. World()
2. World(finalWorld, x, y)

### galaGame.Class

galaGame.Class是为了满足与键盘交互和游戏菜单界面UI基于StdDraw实现的

startGame()是游戏开始的地方，提供了菜单界面和三种输入的应对，NewGame(创建新游戏)，LoadGame(加载历史游戏)，Quit(退出游戏)

这三个功能的实现我充分调用了前面写的与命令行交互的方法，具体实现参照Engine.java

#### Fields
1. galaGame(int width, int height)
2. startGame()
3. loadGame()
4. runGame(String input)
5. receiveSEED()
6. drawFrame(String s)

## Algorithms
随机世界生成

世界生成是基于种子（SEED）的确定性过程：

房间放置：constructRoom 尝试进行 30 次随机尝试。每次随机生成位置和尺寸（3-9 之间），并通过 checkRoom 验证该区域是否全是 Tileset.NOTHING。若检测到重叠则跳过该房间。

路径连接：

所有的有效房间按顺序存储在 roomList 中。

遍历列表，将 room(i) 与 room(i+1) 的中心点通过水平和垂直的 FLOOR 路径连接起来。

墙壁补全：在铺设路径地板时，putWALLsafely 会检查路径周围坐标，如果是 NOTHING 则自动填充为 WALL，确保环境封闭。

玩家初始化：玩家（Avatar）默认生成在房间列表中第一个房间（bornRoom）的中心位置。

移动逻辑
程序监听用户按键（W/A/S/D）。

移动前，moveAvatar 检查目标坐标的 TETile 类型。只有当目标是 Tileset.FLOOR 时，才执行更新：将原位置设为 FLOOR，新位置设为 AVATAR，并更新 World 类中的坐标值。

## Persistence
持久化通过存储“操作序列”而非整个地图数组来实现：

存储机制：当用户输入 :Q 时，系统将完整的指令字符串（例如 N1234SWSAD）写入 save.txt 文件。

读取机制：

用户选择 Load Game 时，loadGame() 从文件中读取该字符串。

该字符串被重新喂给 Engine 处理，由于种子相同且操作序列一致，系统会“回放”所有步骤，从而恢复到退出前的精确状态。

技术实现：利用 java.nio.file.Files 进行字节流的读写操作。
