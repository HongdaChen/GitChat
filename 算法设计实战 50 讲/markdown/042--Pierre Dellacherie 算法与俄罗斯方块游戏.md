> 编写一个俄罗斯方块游戏，涉及键盘控制、定时器、UI 和复杂数据结构定义和使用，非常具有挑战性。很多编程爱好者自己也编写过俄罗斯方块游戏，该游戏有自己的
> AI 算法，基本原理还是状态搜索和评估。这一课我们介绍著名的估值算法、Pierre Dellacherie
> 算法。当然，建立游戏的数据模型仍然是我们的重点。

### 俄罗斯方块游戏 AI 的原理

在游戏过程中，当一个板块下落的时候，旁边还会提示下一个将要落下的板块形状，熟练的玩家会利用下一个板块的形状评估现在要如何摆放当前的板块。玩家玩俄罗斯方块游戏的目的是得更高的分数和玩更长时间，但是一般俄罗斯方块游戏的程序设计都会随着游戏的进行慢慢提高难度，比如加快板块的下落速度，随机增加一些带空格的行等。

在探讨计算机的俄罗斯方块游戏智能算法之前，先研究一下人类玩家玩这个游戏的一些基本策略。玩家玩这个游戏，首先要能够玩尽量长的时间，这就要求要尽可能地消除行，避免累积高度太高；其次是尽量多得分，利用规则消除加分的特点，尽量一次消除多行。

在遇到板块形状很难处理的情况，要选择产生空格子少的摆放方法，尽量避免出现“空洞”。在很多情况下，当一个板块可以摆放在多个位置的时候，玩家需要根据自己的经验选择一个对下一步操作最有利的位置摆放这个板块，这就涉及一个局面评估的问题。

对于此类问题通常的做法是设计一个局面评价函数，对各种可能的局面进行评估，根据评估结果选出最好的一个局面进行实施，那么要评估的局面是什么呢？如果我们把游戏区域这
10 × 20
个小格子视为“棋盘”的话，所谓的局面就是一个板块摆放在某个位置后这个“棋盘”的状态；如果一个板块有多个位置可以摆放，就会产生多个“棋盘”状态，使用评价函数对每个状态进行评估，根据评估的结果将板块摆放在最佳位置上，这就是俄罗斯方块游戏
AI 的原理。

#### 影响评价结果的因素

影响评价结果的因素是多方面的，对这些因素需要一个统一的考虑，选择一个合理的评估策略。每一个板块放在什么位置，都会造成一系列的状态参数变化，根据俄罗斯方块游戏的规则，我们整理出相关的参数有如下几个。

  * 当一个板块摆放后，与这个板块相接触的小方块数量是一个需要考虑的参数。很显然，与之接触的小方块越多，说明这个板块摆放在该位置后产生的空格或“空洞”越少，如果一个“棋盘”局面中空的小方格或“空洞”数量少则说明这个局面对玩家有利。
  * 当一个板块摆放在某个位置后，这个板块最高点的高度是一个需要考虑的参数。这个高度会影响整体的高度，当有两个位置可选择摆放板块时，应该优先选择放置在板块最高点的高度比较低的位置上。
  * 当一个板块摆放在某个位置后能消除的行数是一个重要参数，毫无疑问，能消除的行越多越好。
  * 游戏区域中已经被下落板块填充的区域中空的小方格数量也是评价游戏局面的一个重要参数，很显然，每一行中的空小方格数量越多，局面对玩家越不利。
  * 游戏区域中已经被下落板块填充的区域中“空洞”的数量也是一个重要参数。如果一个空的小方格上方被其他板块的小方格挡住，则这个小方格就形成“空洞”。“空洞”是俄罗斯方块游戏中最难处理的情况，必需等上层的小方块都被消除后才有可能填充“空洞”，很显然，这是一个能恶化局面的参数。

俄罗斯方块游戏的评估算法，就是对以上影响因素进行综合评估，然后给出一个可比较的量化结果，下面将介绍几种目前比较流行的评估算法。

#### 常见的评估算法

评估函数的优劣决定了游戏智能的强弱，这么多参数中，如何给出一种均衡的评估策略，使得评估函数总是能够给出最有利的评价结果？这个问题不好回答，首先要根据问题的本质确定这些参数和评估结果是线性关系还是各种非线性关系，其次是确定各种参数在评估结果曲线上的系数。

俄罗斯方块游戏一出现，就有一些喜欢“搞事情”的人研究俄罗斯方块游戏的“不死”算法，也有一些人开始研究怎么打败俄罗斯方块游戏的 AI 程序。1997
年，Heidi Burgiel 在他的论文 _How to lose at tetris_
中证明了完全随机的俄罗斯方块游戏最终一定会结束，于是人们把热情转移到如何让程序的 AI 能够获得更高的积分或消除更多的行（平均值）。

在这个过程中出现了很多著名的 AI 算法，比如 Pierre Dellacherie 算法、Colin Fahey 算法、Roger LLima /
Laurent Bercot / Sebastien Blondeel 算法、James & Glen 算法和 Thiery & Scherrer
算法等。Colin Fahey 算法和 James & Glen 算法都支持 two-piece 算法，所谓的 two-piece
算法，就是在评估的过程中将当前板块形状和下一块板块形状一起进行评估和计算。

Colin Fahey 在自己的网站上公开了算法的实现代码，同时还发布了一个算法模拟平台，各种俄罗斯方块游戏的 AI
算法可以在这个平台上进行对比和评估，大家可以到 [Colin Fahey
的网站](http://colinfahey.com)下载实现代码和这个模拟平台。在 2003 年之前，Colin Fahey
算法在这个模拟器平台上取得了非常好的效果。

在评估过程中只考虑当前板块形状的 one-piece 算法相对简单一些，但是取得的效果一点也不比 two-piece 算法差，甚至要强于 two-piece
算法。Pierre Dellacherie 算法和 Thiery & Scherrer 算法都是比较著名的 one-piece 算法，在 Thiery &
Scherrer 算法出现之前，Pierre Dellacherie 算法是最强的 one-piece 算法，Colin Fahey 在他的网站上非常推崇
Pierre Dellacherie 算法，称其是 one-piece 算法中最好的算法（2003 年）。Thiery & Scherrer 算法是
Bruno Scherrer 和 Christophe Thiery 两个人联合研究的一种算法，2009 年他们发布了这个算法的 1.4
版，据说平均可以消除 3500 万行。

### 数据模型

#### “棋盘”模型

评估算法的具体表现就是一个对局面的评价函数，因此，要实现评估算法，首先要定义局面的数据模型。俄罗斯方块游戏的板块摆放区域大小固定，以小方格为单位，一般有
20 个格子高、10 个格子宽。因为要考虑边界的情况，所以我们用一个 12 × 22 的二维数组表示“棋盘”，用 1 表示小方格被占用状态，用 0
表示小方格处于空的状态，用一个大于 0
的值表示边界方格，这是此类算法处理的常用技巧。当前已经消除的行数和当前得分是游戏进行过程中的两个状态，用整数分别表示它们就可以了。除此之外，增加一个表示当前最高行所在位置的标识：top_row；最后，RUSSIA_GAME
数据结构的定义如下：

    
    
    typedef struct
    {
        int board[BOARD_ROW][BOARD_COL];
        int top_row;
        int score;
        int lines;
    }RUSSIA_GAME;
    

#### 板块模型

如图（1）所示，板块的形状一共有七种，通常也会用不同的颜色标识出来。为了描述这些形状，人们为每种形状指定了一个大写英文字母作为名字，分别是
I、J、L、O、S、T 和 Z。

![](https://images.gitbook.cn/4e98bf30-08ef-11e9-84c5-8d0ac317b8b0)

图（1）俄罗斯方块图形示意图

每种形状的板块通过旋转可以产生几种不同的形态，O 型板块不管如何旋转都只有一种形态；I、S 和 Z 型板块通过旋转可以产生两种形态；L、J 和 T
型板块通过旋转可以产生 4 种形态。通过观察，可以发现每种板块形状无论怎么旋转，都不会超过 4 × 4 个小格子的空间，因此我们用一个 4 × 4
的矩阵描述这些形状经过旋转产生的各种形态，如图（2）所示，灰色显示的小方块表示板块的形状以及板块旋转可能产生的形态。

![](https://images.gitbook.cn/60b0b470-08ef-11e9-aed2-d1df6adec5a5)

图（2）板块形状数据定义示意图

由小方块组成的 4 × 4 矩阵一般用二维数组定义，这样，每种板块的形态的定义如 B_SHAPE 所示，其中 shape[][] 是个 4 × 4
的二维数组，每个位置的值如果是 1 表示这个位置被板块占用，对应图（2）中的灰色方格位置，0 表示这个位置是空，对应图（2）中的白色方格位置。width 和
height 是板块形状当前的宽和高，以 I 板块为例，其第一个形态的宽就是 4，高是 1，其第二个形态的宽就是 1，高是 4。

    
    
    typedef struct 
    {
        int shape[SHAPE_BOX][SHAPE_BOX];
        int width;
        int height;
    }B_SHAPE;
    

由于板块的旋转，一种板块可以有多个形态。我们定义 R_SHAPE 数据模型表示一种板块的所有形态信息，r_count
是这个板块旋转时可能产生的不同形态的个数，对于 O 型板块来说，r_count 就是 1，对于 T 型板块来说，r_count 就是 4。shape_r
是一个最大长度为 4 的数组，存放 r_count 对应的每一种旋转形态。

    
    
    typedef struct tagRShape
    {
        B_SHAPE shape_r[MAX_SHAPE_R];
        int r_count;
    }R_SHAPE;
    

根据以上数据结构定义，用 7 个 R_SHAPE
类型元素组成的数组存放事先准备好的板块旋转形态数据表，算法实现过程中需要引用这些数据时，首先根据板块的形状编号从 R_SHAPE 数组中找到板块对应的
R_SHAPE 数据，然后就可以根据旋转角度找到 R_SHAPE 数据中对应的 B_SHAPE 数据。如果要穷举 R_SHAPE
板块的所有旋转状态，只需遍历 shape_r 数组即可，这个数据表的部分内容如下所示：

    
    
    typedef struct
    {
        B_SHAPE shape_r[MAX_SHAPE_R];
        int r_count;
    }R_SHAPE;
    

### Pierre Dellacherie 算法

前面介绍“俄罗斯方块游戏 AI
原理”的时候，介绍了一些评价俄罗斯方块游戏局面的参数，但是这些参数都是一些抽象的，如何具体使用这些参数进行评估计算呢？对于这些参数，不同的算法有不同的使用策略，本节要介绍的
Pierre Dellacherie 评价算法就是其中一种评价策略。2003 年，法国人 Pierre Dellacherie 在 Colin Fahey
的平台上提交了一种 one-piece 算法，该算法的结果超过了 Colin Fahey 算法，取得了平均消除 65 万行的成绩，成为 2003
年智能程度最高的 one-piece 人工智能算法。

#### Pierre Dellacherie 算法的基本概念

Pierre Dellacherie 算法将影响俄罗斯方块游戏的抽象参数转化为 6 种具体的属性，并详细定义了这 6 种属性，具体如下。

  * landingHeight：指当前板块放置后，板块中点距离游戏区域底部的距离（以小方格为单位）。
  * erodedPieceCellsMetric：这是消除参数的体现，它代表的是消去行数与当前摆放的板块中被消去的小方格数的乘积。
  * boardRowTransitions：如果把每一行中的小方格从有小方块填充到无小方块，或从无小方块到有小方块填充视作一次“变换”的话，这个属性就是各行中发生变换的次数之和。
  * boardColTransitions：关于“变换”的定义和 boardRowTransitions 一样，只是以列为单位统计变换的次数。
  * boardBuriedHoles：各列中“空洞”的小方格数之和。
  * boardWells：各列中“井”的深度连加之和。

landingHeight 属性比较简单，无需多做说明。erodedPieceCellsMetric
属性体现了消除参数的影响，能消除一行，肯定是一种好的结果，但是如果单纯的考虑消除，会导致一些“不理智的选择”。如图（3）所示，在出现图（3-a）的局面时，最好的选择其实是图（3-c），但是如果评价算法给消除一个太高的评价，则会导致出现图（3-b）的选择，虽然消除了一行，但是产生了一个“空洞”。

![](https://images.gitbook.cn/7350c570-08ef-11e9-8d41-0bd0032d62d4)

图（3）单纯考虑消除产生的不理智结果

Pierre Dellacherie
算法对消除参数进行了适当的折中处理，考虑到每个板块由四个小方块组成（图（2）中的灰色小格子），如果能同时将这个板块的四个小方块都消除，其结果就是“消除的行数
× 4”，将取得明显优势。但是如果只能消除一个小方块，其结果就是 1
倍关系，所产生的影响很容易被形成一个“空洞”或引起一次“变换”所抵消，这样就可以避免类似图（3）所示的例子中的那种“不理智”的选择。boardRowTransitions
和 boardColTransitions
属性反映的是小方块摆放的紧密程度，这个也比较容易理解，小方块摆放的越紧密，期间的空格就越少，小方格状态之间的变换就越少。

> 但是需要注意一点，这种变换要考虑边界因素。可以这样理解，如果紧邻边界的行或列是空小方格，则视为一次“变换”，即边界作为被填充的小方格参与计算。

![](https://images.gitbook.cn/843591e0-08ef-11e9-84c5-8d0ac317b8b0)

图（4）“空洞”和“井”示意图

boardBuriedHoles
是一列中“空洞”的小方格数量之和。所谓的“空洞”就是某一列中顶端被小方块填堵住的空小方格，如图（4-a）所示，红色方框标识的就是“空洞”。形成空洞是俄罗斯方块游戏中最坏的局面，要极力避免这种情况，因此
Pierre Dellacherie 算法给“空洞”的系数是 −4。boardWells
是“井”的深度连加之和，首先来定义什么是“井”，“井”就是两边（包括边界）都由方块填充的空列，图（4-b）就是很多资料上常引用的“井”的示意图，其中蓝色方框标识的就是两个“井”。“井”的评价记分采用的是连加求和，一个“井”中连续的空小方格有
1 个就计 1，有两个就计 1 + 2 = 3，有三个就计 1 + 2 + 3 = 6，以此类推，如（4-b）中两个“井”的记分之和就是（1 + 2）+（
1 + 2 + 3）= 9。

#### Pierre Dellacherie 估值函数原理

Pierre Dellacherie 算法的评估函数就是将以上述 6 个属性作为输入参数，采用线性组合的方式，计算出最后的评估值，其计算公式如下：

value = −landingHeight + erodedPieceCellsMetric − boardRowTransitions −
boardColTransitions − (4 × boardBuriedHoles) − boardWells

对每个局面计算应用上述公式计算 value 值，取最大的一个作为最后的选择。如果两个局面的评分相同怎么办？两个局面的 value
值相同是一种很普遍的情况，为此 Pierre Dellacherie 算法又定义了一个优先度的概念，当两个局面的 value
值相同时，取优先度大的那个作为最后的选择。优先度的定义如下：

如果板块摆放在游戏区域的左侧（1 ~ 5 列）：

​ priority = 100 × 板块需要水平平移次数 + 10 + 板块需要旋转的次数

如果板块摆放在游戏区域的右侧（6 ~ 10 列）：

​ priority = 100 × 板块需要水平平移次数 + 板块需要旋转的次数

假如游戏中新的板块总是从游戏区域的中间开始落下，那么“板块需要水平平移次数”就是将板块摆放在所选位置时需要水平移动多少个小方格。每个板块最终摆放在指定位置后，其形态不一定就是初始形态，可能需要做一些旋转操作才能以此形态放置，这些旋转操作的次数就是“板块需要旋转的次数”。

以上就是 Pierre Dellacherie 评估算法的核心内容，主要就是 value 计算公式所代表的评估函数，正是这个估值计算公式决定了俄罗斯方块游戏
AI 的智能。

#### Pierre Dellacherie 算法实现

Pierre Dellacherie 估值算法的原理并不复杂，要实现这个算法的关键是识别出游戏局面中的 6
种属性，每种属性需要一个计算函数，这些函数有类似的接口，如下所示。game 是游戏的局面（状态），bs 是板块形状数据，row 是选中放置板块的行，col
是选中放置板块的列。

    
    
    int Get_XXXXX(RUSSIA_GAME *game, B_SHAPE *bs, int row, int col)
    {
        ......
    }
    

##### **landingHeight**

首先是 landingHeight，这个非常简单，由于数组的下标 row 与高度是反对称的，需要做个取反计算。

    
    
    int GetLandingHeight(RUSSIA_GAME *game, B_SHAPE *bs, int row, int col)
    {
        return (GAME_ROW - row);
    }
    

##### **erodedPieceCellsMetric**

GetErodedPieceCellsMetric() 的算法实现也很简单，就是从 top_row
开始遍历所有的行，如果发现某一行可以消除，则计算当前板块形状中有多少小方块属于这一行。erodedRow 记录可以消除多少行，erodedShape
记录消除的小格子中有多少小方块是属于当前摆放的板块 bs 上，最后返回它们的乘积。

erodedRow 比较容易理解，erodedShape
的计数如果前面没有看懂，这里我再啰嗦一下。无论形状如何，每个板块上都有四个小格子（自己看图（2）来理解），如果这个板块放下去以后，这个板块上的小格子随着行的消除能够被消除的个数就是
erodedShape。

一般来说，板块放置后，如果一行都不能消除，这个函数就返回
0。只要能消除一行，这个板块上至少能被消除一个小格子，理解一下，毕竟是这个板块摆放导致的消除嘛。如果运气好（技术好才对），这个板块放下去后，随着行的消除，自己也被完全消除，也就是说，新落下的这个板块没有给局面增加任何负担，当然要给与加倍的奖励。想象一下，你留了一个很深的“井”，这时候来了个“I”形板块，这酸爽，一下子消
4 行，并且这个“I”形板块的四个小格子也跟着被消除了，这个评估得分就是 4 × 4 = 16 分。IsFullRowStatus()
判断一行是否被填满，实现很简单，GitHub 上自己看。

    
    
    int GetErodedPieceCellsMetric(RUSSIA_GAME *game, B_SHAPE *bs, int row, int col)
    {
        int erodedRow = 0; //本次能消除的行数
        int erodedShape = 0; //当前板块 bs 中的小格子本次被消除了几块
        int i = game->top_row; //从当前最高行开始向下逐行搜索
        while(i < GAME_ROW)
        {
            if(IsFullRowStatus(game, i)) //这行是否填满
            {
                erodedRow++;
                if((i >= row) && (i <= (row + bs->height)))
                {
                    int sline = i - row;
                    for(int j = 0; j < bs->width; j++)
                    {
                        if(bs->shape[sline][j] != 0)
                        {
                            erodedShape++;
                        }
                    }
                }
            }
            i++;
        }
    
        return (erodedRow * erodedShape);
    }
    

##### **boardRowTransitions和boardColTransitions**

boardRowTransitions 和 boardColTransitions 的计算也非常简单。以 GetBoardRowTransitions()
函数的计算为例，从 top_row 开始遍历所有的行，对每一行统计“变换”。统计从左边界开始到右边界结束，注意这个算法里列下标是从 0 开始的，因为要从
board 区域中的边界开始计算，计算 boardColTransitions 的算法实现与 GetBoardRowTransitions()
函数类似，指示将遍历方法从按照行遍历改成按照列遍历。

    
    
    int GetBoardRowTransitions(RUSSIA_GAME *game, B_SHAPE *bs, int row, int col)
    {
        int transitions = 0;
        for(int i = game->top_row; i < GAME_ROW; i++)  //每一行
        {
            for(int j = 0; j < (BOARD_COL - 1); j++)
            {
                //小格子发生从“填充”到“空”的变化
                if((game->board[i + 1][j] != 0)&&(game->board[i + 1][j + 1] == 0)) 
                {
                    transitions++;
                }
                //小格子发生从“空”到“填充”的变化
                if((game->board[i + 1][j] == 0)&&(game->board[i + 1][j + 1] != 0)) 
                {
                    transitions++;
                }
            }
        }
    
        return transitions;
    }
    

##### **boardBuriedHoles**

“空洞”是一个很关键的属性，但是计算 boardBuriedHoles 的算法并不复杂。遍历 board 的每一列，对每一列从 top_row
开始找第一个填充的小方块（第一个 while 循环），找到之后在继续找这个小方块之下所有的空小方格，统计它们的数量之和（第二个 while 循环）。

    
    
    int GetBoardBuriedHoles(RUSSIA_GAME *game, B_SHAPE *bs, int row, int col)
    {
        int holes = 0;
        for(int j = 0; j < GAME_COL; j++)
        {
            int i = game->top_row;
            while((game->board[i + 1][j + 1] == 0) && (i < GAME_ROW))
                i++;
            while(i < GAME_ROW)
            {
                if(game->board[i + 1][j + 1] == 0)
                {
                    holes++;
                }
                i++;
            }
        }
    
        return holes;
    }
    

##### **boardWells**

“井”的计算仍然以列为单位进行扫描，对每一列从 top_row
开始处理，如果某个小方格是空状态，但是其左右相邻的两列（包括边界）都是填充的小方格，则统计井深的 wells 计数器
+1。当遇到一个小方块是填充状态时，一个井深的统计结束，根据 wells 计数器计算 sum，然后 wells 计数器清0，准备继续统计下一个“井”的深度。

关于“井”的积分，前面介绍过，“井”的得分与深度有关，如果“井”深是 n，则得分就是从 1 加到 n
求和，可见，对于不同的“井”深，其值是确定的。我们再次使用了以空间换时间的策略，预先计算好从 1 到 n 各数列的和，存放在 sum_n 表中，然后用 n
作为数组下标直接得到对应的和。游戏区域最高就是 20 行，因此井深不会超过 20，只需计算 20 个数列和存放在 sum_n 表即可。

    
    
    int sum_n[] = { 0, 1, 3, 6, 10, 15, 21, 28, 36, 45, 55, 66, 78, 91, 105, 120, 136, 153, 171, 190, 210 };
    
    int GetBoardWells(RUSSIA_GAME *game, B_SHAPE *bs, int row, int col)
    {
        int wells = 0;
        int sum = 0;
        for(int j = 0; j < GAME_COL; j++)
        {
            for(int i = game->top_row; i <= GAME_ROW; i++)
            {
                if(game->board[i + 1][j + 1] == 0)
                {
                    if((game->board[i + 1][j]!= 0)&&(game->board[i + 1][j + 2]!=0))
                    {
                        wells++;
                    }
                }
                else
                {
                    sum += sum_n[wells];
                    wells = 0;
                }
            }
        }
    
        return sum;
    }
    

##### **最后，估值**

最后的估值函数就是将前面原理部分介绍的 value 计算公式翻译成代码。

    
    
    int EvaluateFunction(RUSSIA_GAME *game, B_SHAPE *bs, int row, int col)
    {
        int evalue = 0;
    
        int lh = GetLandingHeight(game, bs, row, col);
        int epcm = GetErodedPieceCellsMetric(game, bs, row, col);
        int brt = GetBoardRowTransitions(game, bs, row, col);
        int bct = GetBoardColTransitions(game, bs, row, col);
        int bbh = GetBoardBuriedHoles(game, bs, row, col);
        int bw = GetBoardWells(game, bs, row, col);
    
        evalue = (-1) * lh + epcm - brt - bct - (4 * bbh) - bw;
    
        return evalue;
    }
    

### 总结

Demaine、Hohenberger 和 Liben-Nowell 在 2002 年提交了一篇名为“Tetris is Hard, Even to
Approximate”的论文，初步论证了俄罗斯方块游戏是 NP 完全问题（NP-
complete），这使得所有人都彻底放弃了寻找俄罗斯方块游戏的数学公式解法。目前主要的几种俄罗斯方块游戏人工智能算法，都采用了穷举算法，只是在穷举实现的细节上稍有不同。总的来说，俄罗斯方块游戏的人工智能算法都由两个核心部分组成，其一是板块摆放动作引擎，此引擎负责产生各种板块的摆放方法；其二是评估函数，对每种板块摆放方法进行评估。

本课重点介绍了 Pierre Dellacherie
估值算法，要实现一个完整的俄罗斯方块游戏还需要做很多工作。本课的代码中有一个驱动板块摆放的搜索引擎，还有一个生成一定数量的随机板块函数，用于测试我们的摆放引擎和
AI。我试过，随机生成 10 万个板块，有 10% 的概率能成功处理完，最好成绩是消除大约 4 万行，得 400 多万分，感兴趣的朋友可以玩一玩。

[请单击这里下载源码](https://github.com/inte2000/play_with_algo)

### 答疑与交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《算法应该怎么玩》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「259」给小助手-伽利略获取入群资格。**

