> 上一课简单介绍了博弈树，从编程实现算法的角度看，博弈树是三种树中最简单的一种，无论是原理还是实现都不复杂。这一课，我们就以简单的井字棋（Tic-Tac-
> Toe）游戏为例，介绍一下如何用博弈树实现一个简单的井字棋
> AI，最后的结果并不复杂，我希望大家把关注点放在如何设计数据模型、如何确定落子，以及将博弈树的理论应用到具体的问题等这样的过程上，而不是最后的结果。

### Tic-Tac-Toe——规则简单的博弈

小时候，我不知道这游戏的真名是 Tic-Tac-
Toe，只知道无聊的时候，和小伙伴们找块沙地，用小树枝画个井字就可以玩半天。井字棋总体来说没什么难度，只要稍微动动脑，两个人玩基本上都是平局，现在想想也没啥意思，但正因为井字棋规则非常简单，所以适合用来做博弈树算法讲解的例子。

这一课的重点依然是实现过程，也就是说如何将博弈树的理论应用到一个简单的游戏实现中，将上一课介绍的算法伪代码翻译成具体的算法实现。

### 棋盘与棋子的数学模型

井字棋游戏的棋盘类似一个 3 × 3 的九宫格形状，直观上很容易想到用一个 3 × 3
的二维数组表示棋盘，二维数组每个元素的值代表棋盘上对应位置的状态，有三种，分别是空、O 型棋子或 X
型棋子。使用二维数组的好处是数据访问比较直观，二维数组的两个下标可以直接表示棋子的位置，但缺点也很明显，首先是遍历棋盘需要用两重循环处理两个下标，其次是行、列以及斜线方向上都要做是否满足三点连线的判断，根据行、列和斜线的下标变化特点，需要用几套不同的方法处理。

我之前多次说过，很多高效的棋类游戏都用一维数组建模，这一课用井字棋游戏来体验一下一维数组表达棋盘模型的好处。用一个长度为 9 的一维数组表示 3 × 3
的棋盘，失去了数据访问的直观性，但是好处更多。首先是处理数据的方法更简洁，只需对数组一维遍历就可以得到棋盘的当前状态，不需要关注两个下标的计算；其次，也是很重要的一点，就是结合一点小技巧可以用一套统一的算法非常简洁地处理上面提到的判断三子一线问题。

作为一个游戏，玩家的信息也很重要，我们用一个整数作为玩家的 ID，玩家当然还有很多其他信息，如名字、等级、头像图片等，但是没必要都存在这里。这里的玩家 ID
其实就是一个占位符（Placeholder），或者可以理解为一个索引，通过这个玩家 ID
可以找到他的其他信息，这是数据模型设计常用的伎俩。对于博弈游戏，定义两个常量 PLAYER_A 和 PLAYER_B 分别代表参与博弈的两个玩家的 ID。

m_board[i] 的值本来是表示棋盘格的状态，但是我们取个巧，用玩家 ID 表示这个位置是某个玩家落下的棋子，至于空位置，再定义一个
PLAYER_NULL，就算是偷懒偷到底吧。这里要特别说明一下，正确的设计姿势应该是设置 EMPTY、O 和 X 三个状态值作为 m_board[i]
的值，然后将 O 和 X 分别与玩家建立绑定关系，这样的好处是可以为玩家指定他的棋子是 O 还是X，当然，通过棋盘状态 m_board[i]
反推玩家需要多一个查询。

    
    
    class GameState
    {
        ……
        int m_playerId;
        int m_board[BOARD_CELLS];
    };
    

以上就是棋局状态的数据结构定义，现在来看看上面提到的判断三子一线的小技巧。观察井字棋游戏的棋盘，能够排成三子一线的情况一共有三横、三竖加两条斜交叉线 8
种情况，我们事先把这 8 种情况的数组下标组织成一个表，表中的每个值代表 m_board[i] 对应的位置：

    
    
    int line_idx_tbl[LINE_DIRECTION][LINE_CELLS] = 
    {
        {0, 1, 2}, //第一行
        {3, 4, 5}, //第二行
        {6, 7, 8}, //第三行
        {0, 3, 6}, //第一列
        {1, 4, 7}, //第二列
        {2, 5, 8}, //第三列
        {0, 4, 8}, //正交叉线
        {2, 4, 6}, //反交叉线
    };
    

这样在判断三子一线时，不需要做复杂的数组下标计算，直接查这张表就可以依次判断 8 条线上是否有三子一线的情况。GameState
类中判断三子一线的算法就可以用一个循环把行、列和交叉线都处理了，这就是算法设计中常用的用数据表进行一致性处理的技巧，在本课程的其他部分中也多次用到这种技巧。使用精心构造的数据表，可以让很多棘手问题的实现代码变得无比简单。

    
    
    bool GameState::CountThreeLine(int player_id)
    {
        for(int i = 0; i < LINE_DIRECTION; i++)
        {
            if( (m_board[line_idx_tbl[i][0]] == player_id)
                && (m_board[line_idx_tbl[i][1]] == player_id)
                && (m_board[line_idx_tbl[i][2]] == player_id) )
            {
                return true;
            }
        }
    
        return false;
    }
    

### 估值函数与估值算法

对于大多数棋类游戏来说，通过完整搜索博弈树直到叶子节点确定落子的方法是不现实的，因为当前阶段计算机的计算能力和存储空间都不足以支撑这种实现方式。不过换句话说，如果计算机的能力真的能够实现这样的搜索，那么看计算机博弈棋类游戏将毫无任何趣味可言，因为对于先手有优势的棋类游戏来说，先手必赢，对于比较公平的棋类游戏来说，结局总是平局。

在计算机还不能完整搜索博弈树的现阶段，棋类游戏的 AI
设计通常是根据计算能力设定博弈树搜索深度，在达到搜索深度的时候，对局面进行评估，判断该局面对自己是否有利以及有利的程度。如果在某个位置落子，最后能给自己带来最有利的局面，那么就选择这个位置落子。

因此，一个棋类游戏 AI
的水平主要由两个方面决定，一个是在相同时间内能搜索的博弈树深度，另一个就是对棋盘局面的评价算法。前者体现程序员的代码设计水平和优化能力，和人类棋手下棋一样，棋类
AI 在博弈的时候，能多考虑一步就有多一步的优势；至于后者，则更是决定棋类
AI“智商”的关键因素，如果评价算法能准确地反映棋局的状态，将对落子的判断提供更准确的依据。

![enter image description
here](https://images.gitbook.cn/eec3bc50-04ce-11e9-bb88-23e1984004bc)

图（1）井字棋局面示意图

对棋盘局面的评价算法通常被称为估值函数，要研究井字棋游戏的估值函数，需要理解井字棋游戏的一些棋局现象。首先是空行数的概念，所谓棋子占据的空行数，指的是棋子所在的行、列或斜线方向上只有己方的棋子或空格子的行（列、斜线）数
＋ 全是空格的行（列、斜线）数。如图（1）所示，第一个棋局中 X 棋子的空行数是 5，O 棋子的空行数是 4；第二个棋局中 X 棋子的空行数是 4，O
棋子的空行数是
3。根据对井字棋游戏规则的理解，对于一个井字棋的棋局，每个玩家的棋子占据的空行越多，就说明该玩家有更大的可能性凑成三子一线的结果，因此空行数是井字棋游戏估值函数评估棋局的一个重要因素。

如果说空行数代表是玩家的潜在优势，那么双连子则代表了玩家当前的直接优势。所谓的双连子，指的是在一行（列、斜线）上有两个己方的棋子而没有对方棋子的情况，如图（1）所示的第二个棋局，O
的棋子就形成了双连子。对于双连子，如果对手不堵的话，下一手直接就赢了，因此，在某些情况下，一方棋子占据的空行数多并不一定说明局面占优势，图（1）的第二个棋局，O
的棋子形成了双连子而 X 的棋子不是双连子，尽管 X 棋子的空行数比 O 棋子的空行数多，但是 O 棋子形成了双连子，比 X 棋子更有优势。

显然，所有对于双连子应该给予更高的估值，以体现出对空行数的估值优势，那么对于具体的算法而言，怎么提高这种估值优势呢？我们给空行数定义评价值是
1，对于一个玩家来说，如果一个棋局比另一个棋局的空行数多一个，则估值也相应多 1。对于双连子，我们给予一定倍数的评价值放大，对于井字棋游戏来说，给予 10
倍的估值加倍，就足以使其傲视空行数，因为一个井字棋的空行数怎么也不会超过 10。

最后是三子连线的情况，如果玩家的棋子出现三子连线，则说明这是当前玩家必赢的局面，相反，如果对手的棋子出现三子连线，则说明这是当前玩家必输的局面。对于玩家必赢的局面，要给予一个足够高的奖励评价分，使得这个局面能从众多优势局面中“脱颖而出”。当然，对于一个必输的局面，要给一个足够低的“惩罚”评价分，使得这个局面无论如何都不会被选择。

综上所述，我们给出了一种井字棋游戏的估值函数计算方法，对于执 X 棋子的一方来说，其评估函数是：

![avatar](https://images.gitbook.cn/FiWejn5Nf9Oadot2dXS7624G8nvf)

根据这个估值函数，我们设计了 FeEvaluator 算子，以下是 FeEvaluator 算子估值函数的算法实现：

    
    
    int FeEvaluator::Evaluate(GameState& state, int player_id)
    {
        int min =  GetPeerPlayer(player_id);
    
        int aOne, aTwo, aThree, bOne, bTwo, bThree;
        CountPlayerChess(state, player_id, aOne, aTwo, aThree);
        CountPlayerChess(state, min, bOne, bTwo, bThree);
    
        if(aThree > 0)
        {
            return GAME_INF;
        }
        if(bThree > 0)
        {
            return -GAME_INF;
        }
    
        return (aTwo - bTwo) * DOUBLE_WEIGHT + (aOne - bOne);
    }
    

CountPlayerChess() 的作用是从 state 局面中统计出 player_id 所代表的玩家三子连线数、双连子数和空行数，仍然是利用
line_idx_tbl 数据表的处理优势对棋局做个统计，详情请参考 GitHub 上的代码。

估值函数在很大程度上决定了棋类 AI 的智商，在 GitHub
上的例子代码中，我还给出了一个评估算子：WzEvaluator，这个估值算法只考虑三子连线和双连子，不考虑空行。使用这个估值函数时，棋力稍弱，与
FeEvaluator 算子对弈时，处于绝对劣势，连续对弈 1000 局，输掉三分之二局，平三分之一局，基本无胜局。

### 如何产生走法（落子方法）

棋类游戏的规则多种多样，有的只能放置棋子，不能移动棋子，有的只能移动已有的棋子，因此走法产生的算法也是多种多样，很难找到通用的算法。井字棋游戏的走法比较简单，就是找个空位置落子，如果有多个空位置的话，就调用搜索算法对这些空位置进行估值，然后选一个估值最高的位置落子。

我们设计的走法产生算法配合搜索算法，成为搜索算法的一部分。下面就以“极大极小值”搜索算法为例，介绍一下如何结合本课的数学模型，实现井字棋游戏的“极大极小值”搜索算法实现。

MiniMax() 函数是个递归实现，前几行就是递归退出条件和退出处理（这是我写代码的风格），退出处理当然就是用评估算子对局面进行估值嘛。接下来的 for
循环就是尝试对当前玩家产生走法（当前玩家是随着搜索过程中来回切换的，这个动作在 SwitchPlayer() 中），IsEmptyCell()
函数判断该位置是否为空，如果是就对这个位置进行搜索和估值。

max_player_id 表示当前搜索是要找对 max_player_id 代表的玩家有利的位置。current_player
表示当前搜索的这一层是哪个玩家，如果 max_player_id 代表玩家，这一层就是极大值节点；如果 max_player_id
不代表玩家，这一层就是极小值节点。score 是估值，根据玩家 ID 确定 score 初始值。在搜索的过程中，取 max 还是取 min，也是要根据
tryState 临时状态的玩家 ID 来决定，因此，搜索过程中有个 SwitchPlayer() 切换玩家的过程。

    
    
    int MinimaxSearcher::MiniMax(GameState& state, int depth, int max_player_id)
    {
        if(state.IsGameOver() || (depth == 0))
        {
            return m_evaluator->Evaluate(state, max_player_id);
        }
    
        int current_player = state.GetCurrentPlayer(); //这一层是搜索哪个玩家
        int score = (current_player == max_player_id) ? -GAME_INF : GAME_INF;
        for(int i = 0; i < BOARD_CELLS; i++)
        {
            if(state.IsEmptyCell(i))/*此位置可以落子*/
            {
                state.SetGameCell(i, state.GetCurrentPlayer());
                state.SwitchPlayer();
                int value = MiniMax(state, depth - 1, max_player_id);
                state.SwitchPlayer(); //恢复递归搜索之前的状态
                state.ClearGameCell(i);
                if(current_player == max_player_id)
                {
                    score = std::max(score, value);
                }
                else
                {
                    score = std::min(score, value);
                }
            }
        }
    
        return score;
    }
    

SearchBestPlay() 函数调用 MiniMax() 函数，搜索并评估最佳落子位置，返回最佳落子位置。for 循环搜索棋盘上的空位置，然后代表
max_player_id 在这个位置落子，并将状态切换到对手落子状态，调用 MiniMax() 函数对当前落子后的状态进行搜索评估。

    
    
    int MinimaxSearcher::SearchBestPlay(const GameState& state, int depth)
    {
        int bestValue = -GAME_INF;
        int bestPos = 0;
    
        int max_player_id = state.GetCurrentPlayer();
        for(int i = 0; i < BOARD_CELLS; i++)
        {
            GameState tryState = state;
            if(tryState.IsEmptyCell(i))
            {
                tryState.SetGameCell(i, max_player_id); //假设当前玩家在此位置落子
                tryState.SwitchPlayer(); //状态切换到对手落子
                int value = MiniMax(tryState, depth - 1, max_player_id); //对玩家的这个状态搜索评估
                if(value >= bestValue)
                {
                    bestValue = value;
                    bestPos = i;
                }
            }
        }
    
        return bestPos;
    }
    

为了配合上一课的内容，井字棋游戏还实现了带“α-β”剪枝的搜索算法 AlphaBetaSearcher 和“负极大值”搜索算法
NegamaxSearcher，有兴趣的读者可以研究 GitHub
上的代码，了解这两种搜索算法的实现细节。通过对比，“α-β”剪枝算法对提高搜索效率方面确实有非常好的效果。对于空盘状态的棋局，假如设置搜索深度是
6，“极大极小值”搜索算法搜索棋局的次数是 65000 多次，但是应用“α-β”剪枝后搜索棋局的次数只有 6500 多次。

### 最后，开始下棋

我做了个小的棋类游戏框架，GameControl 允许 PLAYER_A 和 PLAYER_B
两个玩家对弈，可以是电脑玩家，也可以是人类玩家。电脑玩家可以设置搜索算法和评估算子，并通过搜索算法的 SearchBestPlay()
函数获取最佳落子位置，人类玩家则接受用户输入落子的行和列位置。这个框架的用法很简单：

    
    
    FeEvaluator feFunc;
        MinimaxSearcher ms(&feFunc);
    
        HumanPlayer human("张三");
        ComputerPlayer computer("DELL 7577");
        computer.SetSearcher(&ms, SEARCH_DEPTH);
    
        GameState init_state;
        init_state.InitGameState(PLAYER_A);
    
        GameControl gc;
        gc.SetPlayer(&computer, PLAYER_A);
        gc.SetPlayer(&human, PLAYER_B);
        gc.InitGameState(init_state);
    
        int winner = gc.Run();
        if (winner == PLAYER_NULL)
        {
            std::cout << "GameOver, Draw!" << std::endl;
        }
        else
        {
            Player *winnerPlayer = gc.GetPlayer(winner);
            std::cout << "GameOver, " << winnerPlayer->GetPlayerName() << " Win!" << std::endl;
        }
    

下面是游戏过程的输出：

    
    
    MinimaxSearcher 56160 (without Alpha-Beta)
    DELL 7577 play at [2 , 2]
    Current game state :
    ---
    -o-
    ---
    
    Please select your position (row = 1-3,col = 1-3): 1 2
    Current game state :
    -x-
    -o-
    ---
    
    MinimaxSearcher 3270 (without Alpha-Beta)
    DELL 7577 play at [3 , 3]
    Current game state :
    -x-
    -o-
    --o
    
    Please select your position (row = 1-3,col = 1-3):
    

[请单击这里下载源码](https://github.com/inte2000/play_with_algo)

### 答疑与交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《算法应该怎么玩》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「259」给小助手-伽利略获取入群资格。**

