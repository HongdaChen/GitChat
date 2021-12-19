## LSTM 三重门背后的故事

在前面的文章中我们知道了基本的 RNN 很容易出现梯度消失的问题，并且列出了梯度消失的几个解决方案，其中包括 LSTM 模型，今天就来看看 LSTM
是怎样解决这个问题的。

**本文知识点** ：

  * LSTM 的结构

    * 为什么 LSTM 要有三重门？
    * 如何将三道门相连？
  * LSTM 的前向计算

  * 反向传播

  * LSTM 是如何解决梯度消失的

在文章的末尾会列出几个关于 LSTM 的面试真题，大家在学习完本节内容后可以用这些题目检验一下自己是否能够回答上来。

首先我们来看一下 LSTM 的结构。

* * *

LSTM: Long short-term memory 长短期记忆网络也是一种 RNN 结构，因为它也具有反馈连接。在结构上，它和基本 RNN
的区别在于多了三个门控单元和一个长期状态，这样的结构使它可以学习到数据的长期时间依赖性，可以解决基本 RNN 的梯度消失问题。LSTM
的应用也很广，可以用于图像，音频，视频，金融数据等多种领域。

### 1\. LSTM 的结构

LSTM 的结构如下图所示：

![7eIclW](https://images.gitbook.cn/7eIclW)

用数学表达为：

$ i_t = \sigma (W_i h_{t-1} + U_i x_{t} + b_i)$ $ o_t = \sigma (W_o h_{t-1} +
U_o x_{t} + b_o)$ $ f_t = \sigma (W_f h_{t-1} + U_f x_{t} + b_f)$

$ \tilde{C}_t = \tanh (W_C h_{t-1} + U_C x_{t} + b_C)$ $ C_t = f_t \circ
C_{t-1} + i_t \circ \tilde{C}_t $

$ h_t = o_t \circ \tanh{C_t}$

$ y_t = h_t$

再回顾一下 RNN 的结构：

![y6pLF6](https://images.gitbook.cn/y6pLF6)

从图中可以看到它们的 **主要区别有两个** ：

**1\. LSTM 比 基本的 RNN 多了三个门控（即 sigmoid 函数）** ：

$i，o，f$ 这三个 sigmoid 对应了三个“门”：

![AS0KOE](https://images.gitbook.cn/AS0KOE)

$f$：forget，遗忘门，负责控制是否记忆过去的长期状态 $i$：input，输入门，负责控制是否将当前时刻的内容写入长期状态
$o$：output，输出门，负责控制长期状态是否作为当前时刻的输出，准备被下一时刻读取

**2\. LSTM 比基本 RNN 多了一个长期状态 C**

从表达式上可以看出，长期状态 $ C_t $ 等于之前的状态累积 $ C_{t-1} $ ，再加上当前时间步的状态 $ \tilde{C}_t
$，以及它是否被输出由一个门控制 $ o_t \circ \tanh{C_t}$ ，在输出时，就比 RNN 多了这个长期状态。

接下来让我们具体看看为什么 LSTM 要有这三重门？

* * *

### 1.1 为什么 LSTM 要有三重门？

前面已经知道 **RNN 具有记忆的能力，但是从它的结构图中可以看出过去的信息虽然被记忆了下来，却是变形之后的信息，原始的消息已经丢失**
，有些时候原始信息的改变可能对最终结果没有任何影响，但有些情况下一点小的改变也可能使结果完全不同，就像综艺节目中的传话游戏，经过几个人的不同演绎，最后一个人得到的信息可能已经和第一个人传递的截然不同了。

**而 LSTM 设计的基本原则，就是要保证信息的完整性** 。 如何保护？就是把它们 **“写”** 下来，将 LSTM
的状态的改变通过加减法明确地写下来，这样就可以避免信息的变形。

但如果只是单纯地不停地把每一步的信息都写下来，不停地累加，信息就会越来越多，越来越乱，而其实有些信息是有用的，有些信息没有用，一些重复的内容也是可以忽略的。
这也正是 **LSTM 的基本挑战：无法控制和无法协调的写会造成混乱和溢出，甚至难以修复** 。

**[Hochreiter 和 Schmidhuber
的这篇论文](https://www.bioinf.jku.at/publications/older/2604.pdf)中将这个无法协调的问题分成了几个子问题**，分别是“输入权重冲突
input weight conflict”，“输出权重冲突 output weight conflict”，“滥用问题 abuse
problem”和“内部状态漂移 internal state drift”， **LSTM 的结构设计就是为了解决这些问题** 。

解决方法用一个核心观点概括，就是 **选择性** 。

LSTM 的基本挑战的解决方案和它控制状态 state 的关键，就是要 **有选择性地做三件事** ：

  1. 我们写什么
  2. 我们读什么（和我们人类的学习一样，输出需要先输入，因为需要读一些东西才能知道要写什么 ）
  3. 我们应该忘记什么（因为有些信息是过时的，是分散注意力的，就应该被忘记）

> 小结：LSTM 通过写来保护信息的完整性，并通过有选择地写/读/遗忘来克服无限制写带来的问题。

* * *

#### 第一种选择性：有选择地写

和实际生活中一样，我们需要对所写的东西有所选择，例如 **上课时做笔记，我们只记录其中最重要的那些知识点** ，为了让 RNN
能够做到这一点，就需要一种选择性 write 的机制。

基本 RNN 的 state 变的混乱的原因之一就是它写入了 state
的每个元素，可以想象一下在一张白纸上不停写，写满了也还要继续一层一层地写，那么它只会变得越来越混乱。

这就是 Hochreiter 和 Schmidhuber 所说的“输入权重冲突 input weight conflict”：如果每个 unit
在每个时间步被所有 unit 写入，它将收集大量没有用的信息，使它的原始 state 无法使用， 因此 RNN 必须学习如何选择它的一部分 unit
而忽视没用的部分，并保护好 state。

**$i_t$，输入门，就是 write，负责选择什么样的新信息 $x_t$ 应该被存放在细胞状态中。**

例如我们想用 LSTM 生成一段歌词 I should go and see some friends, but they don't really
comprehend，有了前面这句 I should go and see some friends，下一步 LSTM 想写入到 state
中的是主语的单复数，这样就可以增加新的主语 “they” 来替代旧的主语“friends”。

![yqBe4q](https://images.gitbook.cn/yqBe4q)

* * *

#### 第二种选择性：有选择地读

这个也和现实生活中一样的道理，我们在读取信息时也要有选择性， **要去看最相关的知识内容** 。 为了使 RNN 具有这样的能力，需要一种选择性 read
的机制。

基本 RNN 的 state 会变混乱的第二个原因是，每次写时都是从 state 的每个元素中读取信息。假如我们在写一篇关于 LSTM
的文章，结果把当天的天气啊外面的风景啊家里来了什么客人啊全都写进去了，这篇文章就很混乱了，读入的不相关的信息越多，文章就会变得越来越不知所云。

这个是 Hochreiter 和 Schmidhuber 所说的“输出权重冲突 output weight conflict”：如果所有 unit
在每个时间步都读取了不相关的 unit，就可能产生大量的无关信息，因此，RNN 也必须学习如何只使用一部分 unit 而过滤掉不相关的信息。

**$o_t$，输出门，就是负责选择输出什么值 $o_t$ 来作为下一个单元要 read 的内容。**

继续前面的歌词例子，当前时刻已经知道了一个代词 “they”，接下来需要生成“do“ 或
”does”，那么此时就需要输出这个代词是单数还是复数的信息，这样下一个动词就知道如何进行形态变化，即选择“do“。

## ![j62V3H](https://images.gitbook.cn/j62V3H)

#### 第三种选择性：有选择地忘记

同样的，我们是不可能同时记住那么多事情的， **为了给新信息腾出空间，我们需要有选择地忘记最不相关的旧信息** 。 为了使 RNN
也做到这一点，就需要一种选择性 forget 的机制。

最初的 LSTM 论文中没有引入这个想法，导致原始的 LSTM 在涉及长序列的即使是很简单的任务中也遇到了麻烦，即 state
有时会无限增长，导致网络崩溃，整个 LSTM 信息过载，后来在 2000 年由
[Gers](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.55.5709&rep=rep1&type=pdf)
等人做出了改进。

**$f_t$，遗忘门，因为 forget 才能记忆新的。**
对应前面歌词的例子是，我门已经知道了当前主语的性别和单复数，后面就可以直接用正确的代词“they”，就可以忘记旧的主语“friends”，后面都用“they”来代替：I
should go and see some friends, but they don't really comprehend, they don't
need too much talking

![shgBTJ](https://images.gitbook.cn/shgBTJ)

* * *

#### 如何实现“选择性”

现在，我们知道了 LSTM 要做到有选择地写、读、忘记，那么 **如何实现这个选择性？就是用“门”来作为选择性的机制。**

有选择性地 read、write、forget 涉及到了 state 的每个元素的读，写，遗忘的决策，我们将利用向量来表示这些决策，这些向量的值要介于 0
和 1 之间，代表了我们 **为每个状态元素执行的读，写，遗忘的概率** 。

虽然这些决策是二元的，要么写要么不写，但需要通过可微的函数来实现这些决策，所以很自然地会想到 **用 sigmoid 函数** ，因为它是可微的，并且能产生
0 到 1 之间的连续值。

所以时刻 t 的三个门：

$i_t$，输入门，为了 write， $o_t$，输出门，为了下一个单元进行 read， $f_t$，遗忘门，为了 forget， 进而可以
remember。

**它们所对应的数学定义如下：**

$ i_t = \sigma (W_i h_{t-1} + U_i x_{t} + b_i)$

$ o_t = \sigma (W_o h_{t-1} + U_o x_{t} + b_o)$

$ f_t = \sigma (W_f h_{t-1} + U_f x_{t} + b_f)$

（关于门的名称： 我们通常理解 write 是一种输出，read 是一种输入，LSTM 中输入门和输出门这两个名称和它们的功能是相反的。
遗忘门用于遗忘，但它实际上的操作是记忆功能，遗忘门向量为 1 意味着记住所有东西，而不是忘记。）

> 小结：LSTM 通过三个 sigmoid 门控进行有选择地写/读/遗忘。

* * *

### 1.2 如何将三道门相连？

建立完上面的三个选择性机制，再将这三个机制粘在一起，就可以得到 LSTM 结构。

**1\. 先来看长期状态的写入：**

![Vncusl](https://images.gitbook.cn/Vncusl)

既然是写入，就会在先前状态的基础上有增量 $\Delta C_t$，即当前时刻的信息：

$ C_t = C_{t-1} + \Delta C_t $

但是右边这两项，都不是直接相加，而是要经过上面提到的选择机制的作用。

1\. 先用 $ \tilde{C}_t $ 代表候选的写入内容

计算 $ \tilde{C}_t $ 的方法和基本 RNN 中的类似，

$ \tilde{C}_t = \tanh (W_C h_{t-1} + U_C x_{t} + b_C)$

注意其中 $ h_{t-1} $ 为

$ h_{t-1} = o_{t-1} \circ \tanh{C_{t-1}}$

即在使用先前状态 $C_{t-1}$ 时，要和 read 门 $o_{t-1}$ 元素相乘来得到门控后的状态，因为前面提到过，读 自然地会出现在 写
之前，所以应该在读取先前状态时使用 read 门，作为 state 下一次写的内容。

2\. 同时，还要进行有选择性地写

于是用 write 门 $i_t$ 和 $ \tilde{C}_t $ 做元素乘法来得到真正要写入的信息: $ i_t \circ \tilde{C}_t
$

3\. 然后将这个增量 $\Delta C_t$ 添加到先前的状态上

别忘了这时我们还有一个遗忘机制，所以将先前状态和遗忘门 $f_t$ 做元素乘法，就得到了长期状态的表达式：

$ C_t = f_t \circ C_{t-1} + i_t \circ \tilde{C}_t $

所以当有一个新的输入时，模型首先忘掉那些用不上的长期记忆信息，然后学习新输入中有什么是值得写入的信息，然后存入长期记忆中。

**2\. 再来看选择性输出：**

![](https://images.gitbook.cn/CV6fEf)

  1. 在最初的 LSTM 模型中，单个细胞的输出就是这个长期状态 $C_t$，也就是对信息只有三个选择性，但实践证明这样的状态很有可能无界，即使选择很小的学习率，谨慎地选择 bias，模型还是会爆炸，于是在输出时 $C_t$ 要经过一个压缩函数来加以限制，通常用 tanh， 即 $ \tanh{C_t}$ 
  2. 另外，在单个 LSTM 细胞中，我门可以看到它实际的流程是 先写再读，这和实际生活是相反的，所以需要有一个隐藏状态，即 $h_t$ 将前一时刻的信息作为“读”的内容传递给下一时刻，并且是有选择性的读，所以有 $ h_t = o_t \circ \tanh{C_t}$ 

至此，我们就知道了 LSTM 的结构的来历。

* * *

### 2\. LSTM 的前向计算

前面很详细地介绍了 LSTM 的结构，它的前向计算就按照表达式计算即可，

整体流程如下图所示，就可以由左向右依次计算 遗忘门，输入门，输出门：

![zcKn2M](https://images.gitbook.cn/zcKn2M)

  1. 计算遗忘门，

计算： $ f_t = \sigma (W_f h_{t-1} + U_f x_{t} + b_f)$ 其中： `W_f`
是遗忘门的权重矩阵，`[h_t-1, x_t]` 表示把两个向量连接成一个更长的向量，`b_f` 是遗忘门的偏置项，`σ` 是 sigmoid 函数。

  2. 计算输入门，

计算：$ i_t = \sigma (W_i h_{t-1} + U_i x_{t} + b_i)$ 根据上一次的输出和本次输入来计算当前输入的单元状态：$
\tilde{C}_t = \tanh (W_C h_{t-1} + U_C x_{t} + b_C)$ 当前时刻的长期状态：$ C_t = f_t
\circ C_{t-1} + i_t \circ \tilde{C}_t $

  3. 计算输出门，

计算：$ o_t = \sigma (W_o h_{t-1} + U_o x_{t} + b_o)$ 再计算：$ h_t = o_t \circ
\tanh{C_t}$

* * *

### 3\. LSTM 的反向传播

反向传播的目标是要学习下图中的这 8 个参数：

![2OcvzD](https://images.gitbook.cn/2OcvzD)

我们以四个 W 为例，先计算当前步的，再计算前一步的，流程图如下所示：

![tbO1Tf](https://images.gitbook.cn/tbO1Tf)

### 1\. 要得到当前步 $\frac{\partial E_t}{\partial W}$

**前向计算的流程可以概括如下：**

  1. 先计算出 $f_t$，$i_t$，$ \tilde{C}_t $ ，
  2. 再计算 $c_t$，然后计算 $o_t$，
  3. 最后得到 $h_t$ 和输出 $y_t$：

**可知为了求 W 的梯度：**

  1. 需要先知道：$ \delta{f_t}$，$ \delta i_t$，$ \delta \tilde{C}_t $ ,$ \delta C_{t-1}$
  2. 再往前推，需要知道：$ \delta C_t$, $ \delta o_t$
  3. 再往前推，需要知道：$ \delta h_t$

**所以先求 $ \delta h_t$，再求 $ \delta C_t$, $ \delta o_t$，求 $ \delta{f_t}$，$ \delta
i_t$，$ \delta \tilde{C}_t $ ,$ \delta C_{t-1}$，最后得到 $ \delta W_f$, $ \delta
W_i$, $ \delta W_c$, $ \delta W_o$**

#### 1\. 计算 $ \delta h_t$ ：

$ \delta h_t = \frac{\partial E_t}{\partial h_t }$

其中 $ \partial E_t $ 指的是 t 时刻的误差，对每个时刻的误差都要计算一次。

#### 2\. 已知：$ \delta h_t$ ，求：$ \delta C_t$, $ \delta o_t$

这里推导会用到前向计算式：$ h_t = o_t \circ \tanh{C_t}$

可得：

$ \delta o_t = \frac{\partial E_t}{\partial o_t} = \frac{\partial
E_t}{\partial h_t } \frac{\partial h_t}{\partial o_t } = \delta h_t \circ
\tanh{C_t} $

$ \delta C_t = \frac{\partial E_t}{\partial C_t} = \frac{\partial
E_t}{\partial h_t } \frac{\partial h_t}{\partial C_t } = \delta h_t \circ o_t
\circ [1- \tanh^2{C_t}] $

#### 3\. 已知：$ \delta C_t$, $ \delta o_t$ ，求：$ \delta{f_t}$，$ \delta i_t$，$
\delta \tilde{C}_t $ , $ \delta C_{t-1}$

会用到前向计算式：$ C_t = f_t \circ C_{t-1} + i_t \circ \tilde{C}_t $

$ \delta f_t = \frac{\partial E_t}{\partial f_t} = \frac{\partial
E_t}{\partial C_t } \frac{\partial C_t}{\partial f_t } = \delta C_t \circ
C_{t-1} $

$ \delta i_t = \frac{\partial E_t}{\partial i_t} = \frac{\partial
E_t}{\partial C_t } \frac{\partial C_t}{\partial i_t } = \delta C_t \circ
\tilde{C}_t $

$ \delta C_{t-1} = \frac{\partial E_t}{\partial C_{t-1}} = \frac{\partial
E_t}{\partial C_t } \frac{\partial C_t}{\partial C_{t-1} } = \delta C_t \circ
f_t $

$ \delta \tilde{C}_t = \frac{\partial E_t}{\partial \tilde{C}_t} =
\frac{\partial E_t}{\partial C_t } \frac{\partial C_t}{\partial \tilde{C}_t }
= \delta C_t \circ i_t $

#### 4\. 已知：$ \delta{f_t}$，$ \delta i_t$，$ \delta \tilde{C}_t $ , $ \delta
C_{t-1}$，求：$ \delta W_f$, $ \delta W_i$, $ \delta W_c$, $ \delta W_o$

这里用到前向计算式中的 $ f_t $, $ i_t $, $ o_t $, $ \tilde{C}_t $，首先将它们做一下简写：

$ f_t = \sigma (W_f h_{t-1} + U_f x_{t} + b_f)$

将内部的矩阵乘法进一步简化：

$ \begin{align*} W_f h_{t-1} + U_f x_{t} + b_f & = \begin{bmatrix} W_f & U_f &
b_f \end{bmatrix} \begin{bmatrix} h_{t-1} \\\ x_{t} \\\ 1 \\\ \end{bmatrix}
\\\ & = W_f s_{t} \end{align*} $

所以：

$ f_t = \sigma ( W_f s_{t})$

同理：

$ i_t = \sigma ( W_i s_{t})$ $ o_t = \sigma ( W_o s_{t})$ $ \tilde{C}_t =
\sigma ( W_c s_{t})$

于是有：

$\frac{\partial E_t}{\partial W_f} = \frac{\partial E_t}{\partial f_t }
\frac{\partial f_t}{\partial W_f } = [ \delta f_t \circ f_t \circ (1 - f_t) ]
\cdot s_t$

$\frac{\partial E_t}{\partial W_i} = \frac{\partial E_t}{\partial i_t }
\frac{\partial i_t}{\partial W_i } = [ \delta i_t \circ i_t \circ (1 - i_t) ]
\cdot s_t$

$\frac{\partial E_t}{\partial W_o} = \frac{\partial E_t}{\partial o_t }
\frac{\partial o_t}{\partial W_o } = [ \delta o_t \circ o_t \circ (1 - o_t) ]
\cdot s_t$

$\frac{\partial E_t}{\partial W_c} = \frac{\partial E_t}{\partial \tilde{C}_t
} \frac{\partial \tilde{C}_t }{\partial W_c } = [ \delta \tilde{C}_t \circ ( 1
- \tilde{C}_t^2 ) ] \cdot s_t$

综上，将 W 的梯度合在一起为：

$ \begin{align*} \frac{\partial E_t}{\partial W} & = \begin{bmatrix} \delta
f_t \circ f_t \circ (1 - f_t) \\\ \delta i_t \circ i_t \circ (1 - i_t) \\\
\delta o_t \circ o_t \circ (1 - o_t) \\\ \delta \tilde{C}_t \circ ( 1 -
\tilde{C}_t^2 ) \end{bmatrix} \cdot s_{t} \end{align*} $

* * *

### 2\. 要得到前一步 $\frac{\partial E_t}{\partial h_{t-1}}$

由上面这个流程可知，求参数的关键一点在于 $ \delta h_t$ ， 所以为了求前一时间步的四个 w，需要知道 $ \delta h_{t-1}$ ，

在前向传播中， $ f_t $, $ i_t $, $ o_t $, $ \tilde{C}_t $ 是 $ h_{t-1}$ 的函数， 要用到：

$ h_t = o_t \circ \tanh{C_t}$ $ C_t = f_t \circ C_{t-1} + i_t \circ
\tilde{C}_t $

那么，已知： $ \delta{f_t}$，$ \delta i_t$，$ \delta \tilde{C}_t $ , $ \delta C_{t-1}$
，求：$ \delta h_{t-1}$

首先，由 $ h_t $ 传递给 $ h_{t-1}$, 然后根据 $ h_t = o_t \circ \tanh{C_t}$ 对 $ h_{t-1}$
求偏导， 再由 下面两项将式子合并，

$ \delta o_t = \delta h_t \circ \tanh{C_t} $ $ \delta C_t = \delta h_t \circ
o_t \circ [1- \tanh^2{C_t}] $

然后根据 $ C_t = f_t \circ C_{t-1} + i_t \circ \tilde{C}_t $ 对 $ h_{t-1}$ 求偏导，
最后根据下面三项合并，

$ \delta f_t = \delta C_t \circ C_{t-1} $ $ \delta i_t = \delta C_t \circ
\tilde{C}_t $ $ \delta \tilde{C}_t = \delta C_t \circ i_t $

得到最终结果：

$ \begin{align*} \frac{\partial E_t}{\partial h_{t-1}} & = \frac{\partial
E_t}{\partial h_t} \underbrace{ \frac{\partial h_t}{\partial h_{t-1}}} \\\ & =
\frac{\partial E_t}{\partial h_t} [ \tanh C_t \frac{\partial o_t}{\partial
h_{t-1}} + o_t \underbrace{\frac{\partial \tanh C_t}{\partial h_{t-1}}} ] \\\
& = \underbrace{\delta h_t \tanh C_t} \frac{\partial o_t}{\partial h_{t-1}} +
\underbrace{\delta h_t o_t ( 1 - \tanh^2 C_t )} \frac{\partial C_t}{\partial
h_{t-1}} \\\ & = \delta o_t \frac{\partial o_t}{\partial h_{t-1}} + \delta C_t
\underbrace{\frac{\partial C_t}{\partial h_{t-1}}} \\\ & = \delta o_t
\frac{\partial o_t}{\partial h_{t-1}} + \delta C_t [ C_{t-1} \frac{\partial
f_t}{\partial h_{t-1}} + \tilde{C}_t \frac{\partial i_t}{\partial h_{t-1}} +
i_t \frac{\partial \tilde{C}_t}{\partial h_{t-1}} ] \\\ & = \delta o_t
\frac{\partial o_t}{\partial h_{t-1}} + \underbrace{ \delta C_t C_{t-1} }
\frac{\partial f_t}{\partial h_{t-1}} + \underbrace{\delta C_t \tilde{C}_t}
\frac{\partial i_t}{\partial h_{t-1}} + \underbrace{\delta C_t i_t}
\frac{\partial \tilde{C}_t}{\partial h_{t-1}} \\\ & = \delta o_t
\frac{\partial o_t}{\partial h_{t-1}} + \delta f_t \frac{\partial
f_t}{\partial h_{t-1}} + \delta i_t \frac{\partial i_t}{\partial h_{t-1}} +
\delta \tilde{C}_t \frac{\partial \tilde{C}_t}{\partial h_{t-1}} \\\
\end{align*} $

进一步，因为：

$ \frac{\partial o_t}{\partial h_{t-1}} = o_t ( 1 - o_t) W_o $ $
\frac{\partial f_t}{\partial h_{t-1}} = f_t ( 1 - f_t) W_f $ $ \frac{\partial
i_t}{\partial h_{t-1}} = i_t ( 1 - i_t) W_i $ $ \frac{\partial
\tilde{C}_t}{\partial h_{t-1}} = ( 1 - \tilde{C}_t^2) W_c $

所以，最后的结果是：

$ \begin{align*} \frac{\partial E_t}{\partial h_{t-1}} & = \begin{bmatrix}
W^T_o & W^T_f & W^T_i & W^T_c \\\ \end{bmatrix} \begin{bmatrix} \delta o_t
\circ o_t \circ (1 - o_t) \\\ \delta f_t \circ f_t \circ (1 - f_t) \\\ \delta
i_t \circ i_t \circ (1 - i_t) \\\ \delta \tilde{C}_t \circ ( 1 - \tilde{C}_t^2
) \\\ \end{bmatrix} \\\ & = \begin{bmatrix} W_o \\\ W_f \\\ W_i \\\ W_c
\end{bmatrix}^T \begin{bmatrix} \delta o_t \circ o_t \circ (1 - o_t) \\\
\delta f_t \circ f_t \circ (1 - f_t) \\\ \delta i_t \circ i_t \circ (1 - i_t)
\\\ \delta \tilde{C}_t \circ ( 1 - \tilde{C}_t^2 ) \\\ \end{bmatrix} \\\ & =
W^T \begin{bmatrix} \delta o_t \circ o_t \circ (1 - o_t) \\\ \delta f_t \circ
f_t \circ (1 - f_t) \\\ \delta i_t \circ i_t \circ (1 - i_t) \\\ \delta
\tilde{C}_t \circ ( 1 - \tilde{C}_t^2 ) \\\ \end{bmatrix} \\\ \end{align*} $

* * *

### 3\. 总的梯度为：

$ \frac{\partial E}{\partial W} = \sum_{t=0}^T \frac{\partial E_t}{\partial W}
$

* * *

### 4\. LSTM 是如何解决梯度消失的？

我们在上一篇 RNN 中推导了梯度消失的原因，在计算梯度时有这么一串连乘：

$\frac{\partial h_t}{\partial h_{t-1}} \frac{\partial h_{t-1}}{\partial
h_{t-2}}\frac{\partial h_{t-2}}{\partial h_{t-3}}...$

并且知道如果激活函数是 sigmoid 的话，上面的每一项都小于 0.25，再加上权重如果也都是小于 1 的话，就会发生梯度消失。

在 LSTM 细胞中，内部的循环状态是这样的一个加法表达：

$ C_t = f_t \circ C_{t-1} + i_t \circ \tilde{C}_t $

如果我们像基本 RNN 一样对循环状态求偏导(下面是简化后的形式)，就会得到：

$ \frac{\partial C_t}{\partial C_{t-1}} = f$

这样我们就得到了若干个 $f$ 的连乘，即如果有三个时间步，那么就有三个 $f$ 相乘。

由此可知，如果 $f$ 的输出为 1，梯度就不会减少。

而 $f$ 是遗忘门，它通过输出 1 或 0，来判断门是打开还是关闭，是通过还是阻止，当门打开时，即 $f$ 输出值接近于 1，梯度就不会消失。

假设我们在第一个时间步只有一个输入，然后我们把未来时间步的输入门设置为 0，即把未来的输入都挡住，并且把之前状态的遗忘门都设置为
1，也就是会记得过去的所有状态，这样就会有 $C_t = C_{t-1}$ ，即状态 $C_t$ 永远不会减少，在反向传播时，error
进入到这样的循环中就也不会减少。

而在基本 RNN 中，如果做同样的事情，就算未来的输入被挡住，前面的状态 $C_t$
还是经过了未来时间步的激活函数，误差在反向传播时就仍会连续地降低，最终削减为 0。

* * *

好了，在这篇文章中我们讲了很多 LSTM 的理论，包括它的结构，结构的来历，前向计算，反向传播，还有为什么解决了基本 RNN
的梯度消失问题，大家可以自测一下看下列问题是否能够回答出来，下一篇我们将看看如何应用 LSTM。

**面试题：**

  * LSTM 的结构公式是什么？
  * LSTM 的模型参数都有什么？
  * 手推 LSTM 的反向传播？
  * LSTM 和 RNN 的区别是什么？优势是什么？
  * LSTM 是如何解决梯度消失的？

* * *

参考文献：

[Sepp Hochreiter and Jürgen Schmidhuber，《Long Short-Term
Memory》](https://www.bioinf.jku.at/publications/older/2604.pdf) [Long short-
term memory](https://en.wikipedia.org/wiki/Long_short-term_memory)
[Understanding LSTM
Networks](https://colah.github.io/posts/2015-08-Understanding-LSTMs/) [Written
Memories: Understanding, Deriving and Extending the
LSTM](https://r2rt.com/written-memories-understanding-deriving-and-extending-
the-lstm.html#dealing-with-vanishing-and-exploding-gradients)

