## 概述

中国象棋通用引擎协议 (Universal Chinese Chess Protocol，简称UCCI)，是一种象棋界面和象棋引擎之间的基于文本的通讯协议。

规范通用引擎协议 UCCI 协议，为棋软的 AI 引擎与界面分享提供标准接口，一些团队可以实现优美的界面，而另一些团队可以关注于 AI 引擎算法提升。

中国象棋的软件理论主要参考了国际象棋，通用引擎协议也是这样。UCCI 参考国际象棋的 UCI 来制定的。

## 通讯方法

通常情况下，引擎程序以独立进程方式运行，以标准输入输出与界面程序进行通信。

独立进程运行的情况下，象棋软件使用输入输出管道方式与作为子进程的引擎程序进行通信，通信内容为文本。
/Users/linmi/Documents/Work/GitChat/专栏/照云/flutter 实战：中国象棋
引擎也可以象棋程序的线程方式运行，象棋软件和引擎运行于同一个内存空间，可以使用更灵活的方式进行通信。

象棋程序与引擎之间的通信过程一般是这样的：

  1. 象棋软件向引擎发送的「指令」信息
  2. 引擎理解并执行指令，发送的「反馈」信息给象棋程序
  3. UCCI 协议以文件为基础，一个指令或反馈都在同一行

## 局面和着法表示

### 局面表示

象棋程序以 `position` 指令告知引擎局面信息。局面信息以 FEN 方式标记，一个典型的局面指令：

    
    
    position fen rnbakabnr/9/1c5c1/p1p1p1p1p/9/9/P1P1P1P1P/1C5C1/9/RNBAKABNR w - - 0 1
    

为了让引擎自动判断禁手（重复等），可以在 `position`
指令中分两部分反应局面信息，一部分是上一次吃子（或开局）后的局面，另一部分是上一次吃子后的招法列表。

    
    
    position fen rnbakabnr/9/1c5c1/p1p1p1p1p/9/9/P1P1P1P1P/1C5C1/9/RNBAKABNR w - - 0 1
    

Case 2 - 红方走出了炮二平五，请求引擎为黑方出着：无咋子局面，使用开局局面，有一步着法：

    
    
    position fen rnbakabnr/9/1c5c1/p1p1p1p1p/9/9/P1P1P1P1P/1C5C1/9/RNBAKABNR w - - 0 1 moves h2e2
    

Case 3 - 黑方炮8平5，请引擎为红方出着：无吃子过程，有两步着法：

    
    
    position fen rnbakabnr/9/1c5c1/p1p1p1p1p/9/9/P1P1P1P1P/1C5C1/9/RNBAKABNR w - - 0 1 moves h2e2 h7e7
    

Case 4 - 红方炮五进四，请引擎为黑方出着：前一步红方吃子，局面变化最最近吃子局面，无着法列表：

    
    
    position fen rnbakabnr/9/1c2c4/p1p1C1p1p/9/9/P1P1P1P1P/1C7/9/RNBAKABNR b - - 0 2
    

Case 5 - 黑方走出士4进5，请引擎为红方出着，使用上一个咋子局面，着法列表中是上一步黑方「士4进5」：

    
    
    position fen rnbakabnr/9/1c2c4/p1p1C1p1p/9/9/P1P1P1P1P/1C7/9/RNBAKABNR b - - 0 2 moves d9e8
    

### 着法表示

象棋应用和引擎程序之间对着法的表示使用 ICCS 格式。

ICCS 格式用 4 个字符表示一步着法的，例如 `b0c2` ：

  * ICCS 着法基于左下角坐标系
  * 用 a ~ i 表示从左到右的 9 列
  * 用 0 ~ 9 表示从下到上 10 行

举个例子，引擎返回着法 `b0c2`，表示将`第2列第1行的棋子移动到第3列第3行`。

## 指令和反馈

### 指令

    
    
    ucci
    

这是引擎启动后，界面需要给引擎发送的第一条指令，通知引擎现在使用的协议是UCCI。

### 响应1

    
    
    id {name | copyright | author | user} <信息>
    

返回引擎的版本号、版权、作者和授权用户，例如：

    
    
    id name ElephantEye 1.6 Beta
    id copyright 2004-2006 www.xqbase.com
    id author Morning Yellow
    id user ElephantEye Test Team
    

### 响应2

    
    
    option <选项> type <类型> [min <最小值>] [max <最大值>] [var <可选项> [var <可选项> [...]]] [default <默认值>]
    

显示引擎所支持的选项，UCCI 引擎支持以下选项：

  * usemillisec true，通知界面采用毫秒模式
  * batch false，批处理模式，默认是关闭的；
  * debug false，调试模式，默认是关闭的；
  * ponder false，是否使用后台思考的时间策略，默认是关闭的；
  * usebook true，是否使用开局库的着法，默认是启用的；
  * useegtb true，是否使用残局库，默认是启用的；
  * bookfiles <bookfile-name>，设定开局库文件的名称，可指定多个开局库文件，用分号「;」隔开；
  * egtbpaths <bookfile-name>，设定残局库路径的名称，和 bookfiles 类似；
  * evalapi <lib-name>，设定局面评价API函数库文件的名称，和 bookfiles 类似，但只能是一个文件；
  * hashsize <number>，以MB为单位规定Hash表的大小，0表示让引擎自动分配Hash表；
  * threads <number>，支持多处理器并行运算(SMP)的引擎可指定线程数(即最多可运行在多少处理器上)，0表示让引擎自动分配线程数；
  * idle <none|small|medium|large>，设定处理器的空闲状态，通常有none(满负荷)、small(高负荷)、medium(中负荷)、large(低符合)四种选项，引擎默认总是以满负荷状态运行的；
  * promotion false，是否允许仕(士)相(象)升变成兵(卒)，这是一种中国象棋的改良玩法，默认是不允许的；
  * pruning <none|small|medium|large>，设定裁剪程度，裁剪越多则引擎的搜索速度越快，但搜索结果不准确的可能性越大；
  * knowledge <none|small|medium|large>，设定知识大小，通常知识量越多则程序的静态局面评价越准确，但的运算速度会变慢；
  * randomness <none|small|medium|large>，设定随机性系数，一般都设为none，以保证引擎走出它认为最好的着法，但为了增强走棋的趣味性，可以把这个参数调高；
  * style <solid|normal|risky>，设定下棋的风格；
  * newgame，设置新局或新的局面，引擎收到该指令时，可以执行导入开局库、清空Hash表等操作；

### 响应3

    
    
    ucciok
    

引导状态的反馈，此后引擎进入空闲状态。

### 指令

    
    
    isready
    

检测引擎是否处于就绪状态

### 响应

    
    
    readyok
    

表明引擎处于就绪状态。

### 指令

    
    
    setoption <选项> [<值>]
    

设置引擎参数，这些参数都应该是前述 ucci 指令的 option 反馈的参数，例如：

setoption usebook false，不让引擎使用开局库； setoption selectivity large，把选择性设成最大；
setoption style risky，指定冒进的走棋风格； setoption loadbook，初始化开局库。

### 指令

    
    
    position {fen <FEN串> | startpos} [moves <后续着法列表>]
    

设置局面，用 fen 来指定 FEN 格式串，moves 后面跟的是随后走过的着法。startpos表示开始局面，它等价于
`rnbakabnr/9/1c5c1/p1p1p1p1p/9/9/P1P1P1P1P/1C5C1/9/RNBAKABNR w - - 0 1`

### 指令

    
    
    banmoves <禁止着法列表>
    

为当前局面设置禁手，以解决引擎无法处理的长打问题

### 指令

    
    
    go [ponder | draw] <思考模式>
    

要求引擎根据 position 指令设定的棋盘来思考，各选项为思考方式，有三种模式可供选择：

  * depth <深度> | infinite：限定搜索深度，infinite 表示无限制思考（直到找到杀棋或用 stop 指令中止）。如果深度设定为 0，那么引擎可以只列出当前局面静态评价的分数，并且反馈 nobestmove。
  * nodes <结点数>：限定搜索结点数。
  * time <时间> [movestogo <剩余步数> | increment <每步加时>] [opptime <对方时间> [oppmovestogo <对方剩余步数> | oppincrement <对方每步加时>]]：限定时间，时间单位毫秒
  * movestogo适用于时段制
  * increment适用于加时制
  * opptime、oppmovestogo 和 oppincrement 可以让界面把对方的用时情况告诉引擎。
  * 如果指定 ponder 选项，则引擎思考时时钟不走，直到接受到 ponderhit 指令后才计时，该选项用于后台思考，它只对限定时间的思考模式有效。
  * 指定 draw 选项表示向引擎提和，引擎以 bestmove 提供的选项作为反馈，参阅 bestmove 指令。
  * ponder 和 draw 选项不能同时使用，如果界面向正在后台思考中的引擎求和，则使用 ponderhit draw 指令。

### 响应1

    
    
    info <思考信息>
    

显示引擎思考信息，通常有以下几种信息：

  * time <已花费的时间> nodes <已搜索的结点数>
  * depth <当前搜索深度> [score <分值> pv <主要变例>]：输出引擎思考到的深度及其思考路线和好坏
  * currmove <当前搜索着法>：输出引擎正在思考的着法
  * message <提示信息>：输出引擎要直接告诉用户的信息

### 中间指令

    
    
    ponderhit [draw]
    

思考过程中的指令，告诉引擎后台思考命中，现在转入正常思考模式（引擎继续处于思考状态，此时go指令设定的时限开始起作用）。draw 选项表示向引擎提和，引擎以
bestmove 提供的选项作为反馈。

### 中间指令

    
    
    stop
    

中止引擎的思考。注意：发出该指令并不意味着引擎将立即回到空闲状态，而是要等到引擎反馈bestmove或nobestmove后才表示回到空闲状态。

### 响应2-1

    
    
    bestmove <最佳着法> [ponder <后台思考的猜测着法>] [draw | resign]
    

思考结果反馈，以及猜测在这个着法后对手会有怎样的应对。

通常，最佳着法是思考路线、中的第一个着法，而后台思考的猜测着法则是第二个着法。在对手尚未落子时，可以根据该着法来设定局面，并作后台思考。当对手走出的着法和后台思考的猜测着法吻合时，称为“后台思考命中”。

draw 选项表示引擎提和或者接受界面向引擎发送的提和请求

resign 选项表示引擎认输。

### 响应2-2

    
    
    nobestmove
    

反馈思考结果，但引擎一步着法也没计算，表示当前局面是死局面，或者接收到诸如 go depth 0 等只让引擎给出静态局面评价的指令。

### 指令

    
    
    quit
    

让引擎退出运转。

### 响应

    
    
    bye
    

接收到 quit 指令后的反馈

## 电脑象棋联赛中的 UCCI

电脑象棋联赛使用 UCCI 引擎，参赛引擎必须能够识别并正确处理以下的指令：

  * ucci；
  * position fen … [moves …]；
  * banmoves …；
  * go [draw] time … increment … [opptime … oppincrement …]；
  * quit。

参赛引擎必须能够反馈的信息有：

  * ucciok；
  * bestmove … [draw | resign]。

为了更好地让引擎适应模拟器，引擎最好能够实现以下功能：

  * 支持毫秒制，即启动时有 option usemillisec 的反馈，能够识别并处理 setoption usemillisec true 的指令
  * 支持认输和提和，即当引擎觉得没有机会获胜时，可以用 bestmove … draw 提和或接受提和
  * 支持 stop 指令

另外，识别 setoption 指令不是必须的，但在联赛中也会有用，例如用 setoption bookfiles …来导入开局库。

