在前面我们很细致地介绍了基本 LSTM 的理论，其实学者们还提出了多种 LSTM 的变种，如 Coupled LSTM、Peephole LSTM、GRU
等等，今天就来看看其中两个比较流行的变体 **Peephole connections 和 GRU**
，它们都可应对梯度消失问题，也都可用于构建深度神经网络，此外我们还会学习一个高效的搜索策略 **Beam Search** 。

首先来回顾一下 LSTM 的结构：

![](https://images.gitbook.cn/46d147e0-6460-11ea-99a6-09fef5109e7b)

LSTM 有三个门控，还有一个长期状态 C。

数学表达为：

$ i_t = \sigma (W_i h_{t-1} + U_i x_{t} + b_i)$ $ o_t = \sigma (W_o h_{t-1} +
U_o x_{t} + b_o)$ $ f_t = \sigma (W_f h_{t-1} + U_f x_{t} + b_f)$

$ \tilde{C}_t = \tanh (W_C h_{t-1} + U_C x_{t} + b_C)$ $ C_t = f_t \circ
C_{t-1} + i_t \circ \tilde{C}_t $

$ h_t = o_t \circ \tanh{C_t}$

$ y_t = h_t$

其中：

  * f：forget，遗忘门，负责控制是否记忆过去的长期状态。 
  * i：input，输入门，负责控制是否将当前时刻的内容写入长期状态。
  * o：output，输出门，负责控制长期状态是否作为当前时刻的输出，准备被下一时刻读取。

下面这张图是基本 LSTM、Peephole LSTM 和 GRU 的结构图对比：

![](https://images.gitbook.cn/af14b7b0-6460-11ea-9b0c-5b7924682571)

从图中可以看到，Peephole LSTM 的虚线部分使这两个门可以观察到单元状态 C；而 GRU
将隐层状态和单元状态合二为一，接下来我们将详细介绍这两种变体的原理。

### Peephole connections

#### **什么是 Peephole connections**

**Peephole connections** 是由 [Gers & Schmidhuber
(2000)](ftp://ftp.idsia.ch/pub/juergen/TimeCount-IJCNN2000.pdf) 提出的，是在原有的 LSTM
上面增加了一种连接，就是图中虚线部分所示。

![](https://images.gitbook.cn/e8fe2830-6460-11ea-9aca-b107169e8218)

有了这种改进，模型不仅可以看到当前的输入和前一步的最终隐层状态，还能看到前一步的单元状态，使模型的表现可以更好。

#### **基本原理**

peephole LSTM 的单元结构比较简单，就是在每个门那里连接状态 $c_{t-1}$，这个连接就叫做 Peephole connections：

![](https://images.gitbook.cn/0be6ecb0-6461-11ea-ae2f-87d428aaa5f7)

数学表达为：

$ i_t = \sigma (W_i h_{t-1} + U_i x_{t} + P_i c_{t-1} + b_i)$ $ o_t = \sigma
(W_o h_{t-1} + U_o x_{t} + P_o c_{t-1} + b_o)$ $ f_t = \sigma (W_f h_{t-1} +
U_f x_{t} + P_f c_{t-1} + b_f)$

$ \tilde{C}_t = \tanh (W_C h_{t-1} + U_C x_{t} + b_C)$ $ C_t = f_t \circ
C_{t-1} + i_t \circ \tilde{C}_t $

$ h_t = o_t \circ \tanh{C_t}$

$ y_t = h_t$

在基本 LSTM 单元中，门控制器只能看输入 $x_t$ 和前一个短期状态 $h_{t-1}$，而 Peephole 可以让模型多看一下长期状态
$c_{t-1}$，这样每个门的输入都会有三个来源。

#### **Peephole 为什么会比 LSTM 效果好**

基本 LSTM 模型是看不到单元状态 $c_t$ 的，假如这时输出门接近 0，即使单元状态 $c_t$
中包含了很重要的信息，比如携带着能够影响到模型最终表现的内容，那么由于输出门接近 0，导致最终隐层状态也会接近于 0，状态 $c_t$
中的重要信息就永远不会被学习到。所以，如果直接把单元状态 $c_t$ 加入到门的计算中，就能够更好地利用单元状态的信息，即使输出门接近
0，模型也可以表现得很好。

### GRU

#### **什么是 GRU**

**GRU (Gated Recurrent Unit)** 由 [Cho 在 2014
年提出](https://arxiv.org/pdf/1406.1078v3.pdf)，它的主要目的是要解决标准 RNN 的梯度消失问题。

此外我们知道 LSTM 有三个门和两个状态，这样就会产生大量参数，GRU 是 LSTM 的简化版本，它比 LSTM 的结构更简单，所以参数的数量也就比
LSTM 少了很多，而且达到的效果也差不多，这使得 GRU 比较受欢迎。

#### **GRU 的基本原理**

首先看一下 GRU 的结构图：

![](https://images.gitbook.cn/abf99f90-6461-11ea-8be0-ebada4c42c48)

GRU 的数学表达为：

$$z_t = \sigma(W_z \text{h}_{t-1} + U_z \text{x}_t )$$

$$r_t = \sigma(W_t \text{h}_{t-1} + U_t \text{x}_t)$$

$$\widetilde{h}_t = \text{tanh}(W (r_t \circ \text{h}_{t-1}) + U \text{x}_t)$$

$$\text{h}_t = (1 - z_t) \circ \text{h}_{t-1} + z_t \circ \widetilde{h}_t$$

  * 其中 $z_t$ 叫做 update gate 更新门；
  * $r_t$ 叫做 reset gate 重置门；
  * $\widetilde{h}_t$ 是 GRU 的候选状态，由 $[x_t, h_{t−1}]$ 计算得到；
  * $h_t$ 是 GRU 的隐层状态，由 $[h_{t−1}$、$\widetilde{h}_t]$ 计算而得。

**1\. 更新门 $z_t$ 的含义**

$$z_t = \sigma(W_z \text{h}_{t-1} + U_z \text{x}_t )$$

  * 在更新门中，$x_t$ 是当前时刻的输入，$U_z$ 是它的权重；
  * $h_{t-1}$ 是前一时刻 t-1 的信息，$W_z$ 是它的权重，在更新门中，$W_z$ 就决定了需要将多少过去的信息 $h_{t-1}$ 传递给未来；
  * 然后将两个乘积相加，并经过一个 Sigmoid 激活函数将结果映射到 0~1 之间。

通过最后 $\text{h}_t$ 的表达式中可以看出，更新门的作用是决定着候选状态 $\widetilde{h}_t$ 对新隐层状态的影响程度大小。

**2\. 重置门 $r_t$ 的含义**

$$r_t = \sigma(W_t \text{h}_{t-1} + U_t \text{x}_t) $$

它的形式和更新门的一模一样，区别只是权重不一样：

  * 这里 $x_t$ 的权重是 $U_t$；
  * $h_{t-1}$ 的权重是 $W_t$，重置门中的 $W_t$ 决定了要忘掉多少过去的信息 $h_{t-1}$。

**3\. 当前记忆内容**

然后计算候选隐层状态 $\widetilde{h}_t$：

$$\widetilde{h}_t = \text{tanh}(W (r_t \circ \text{h}_{t-1}) + U \text{x}_t)$$

  * 先用权重 U 乘以 $x_t$，用重置门 $r_t$ 和 $h_{t-1}$ 做逐元素点积，再乘以权重 W；
  * 然后再经过 tanh 激活函数。

这一步的作用是在建立一个新的记忆内容，它用重置门来存储过去的相关信息，重置门 $r_t$ 决定了 $h_{t-1}$ 在候选状态中的重要性。

由上面公式我们可以知道，如果重置门 $r_t$ 接近于零，在计算候选隐层状态 $\widetilde{h}_t$ 时，这些单元的先前隐层状态
$h_{t-1}$ 就会被忽略，模型就可以丢弃将来不会用到的信息。

**4\. 当前时间步的最终记忆**

再计算隐层记忆状态 $h_{t}$：

$$\text{h}_t = (1 - z_t) \circ \text{h}_{t-1} + z_t \circ \widetilde{h}_t$$

这一步是要从携带当前记忆内容的 $\widetilde{h}_t$ 和携带过去信息的 $h_{t-1}$ 中 **选择有用的内容**
组成最终的当前记忆，并向下一步传递。

由这个公式我们可以知道，当更新门 $z_t$ 接近零时，模型就会将前一步隐层状态 $h_{t-1}$ 完全复制到当前步。

如果更新门 $z_t$ 接近 1，例如在一个自然语言处理任务中，如果与当前最相关的信息位于文本的最开头，那么模型就会学习出 $z_t$ 接近 1，所以
$1-z_t$ 就会接近 0，这样模型就会选择忘掉不相关的历史信息，而只是记住开头的内容。

#### **GRU 和 LSTM 的主要区别**

1\. 首先，GRU 将单元状态 $\text{h}_{t-1}$ 和最终隐层状态 $\widetilde{h}_t$ 这两个状态组合成为了一个隐层状态
$h_t$：

![](https://images.gitbook.cn/9a893f80-6462-11ea-8414-7dfc6c82284c)

  * LSTM 中：$ h_t = o_t \circ \tanh{C_t}$ 
  * GRU 中：$ \text{h}_t = (1 - z_t) \circ \text{h}_{t-1} + z_t \circ \widetilde{h}_t $

也就是说 GRU 没有单独的记忆状态 C，每个时刻的隐层记忆状态 $h_t$ 是先前隐层记忆状态 $h_{t-1}$ 和候选隐层状态
$\widetilde{h}_t $ 之间的线性组合，这样做就避开了输出门 $o_t$，我们知道输出门的作用就只是决定了将多少单元状态 C
读入到最终隐藏状态 $h_t$ 中，GRU 的这种改进就减少了参数个数，而且还保持作用没变。

2\. 然后，GRU 引入一个 reset gate 重置门 $r_t$：

![](https://images.gitbook.cn/bef94040-6462-11ea-a93b-ddc7a86b90fd)

  * LSTM 中：$ f_t = \sigma (W_f h_{t-1} + U_f x_{t} + b_f)$ 
  * GRU 中：$r_t = \sigma(W_t \text{h}_{t-1} + U_t \text{x}_t) $

当重置门接近 1 时，在计算候选状态时就保留完整的先前状态信息，当重置门接近 0
时，就完全忽略先前的状态信息，即它的作用就是决定了要遗忘多少过去的信息，类似 LSTM 的遗忘门。

3\. 此外，GRU 还将 LSTM 的输入门和遗忘门组合成了一个 update gate 更新门 $z_t$：

![](https://images.gitbook.cn/21a25c90-6463-11ea-a93b-ddc7a86b90fd)

  * LSTM 中：$ i_t = \sigma (W_i h_{t-1} + U_i x_{t} + b_i)$，$ f_t = \sigma (W_f h_{t-1} + U_f x_{t} + b_f)$ 
  * GRU 中：$z_t = \sigma(W_z \text{h}_{t-1} + U_z \text{x}_t )$

标准 LSTM 中，输入门决定了将多少 **当前的输入** 读入到单元状态中，遗忘门决定了将多少 **先前的信息** 读入到当前单元状态中。

GRU 将这两个门组成一个后：

  * 如果更新门是 0，则将全部的 **先前信息** 读入到当前单元状态中，同时不会读入任何 **当前输入** ；
  * 如果更新门是 1，则将全部的 **当前输入** 读入到当前单元状态中，而不会读入任何 **先前信息** 。 

这样更新门的作用也就类似 LSTM 的遗忘门和输入门，它可以决定要扔掉什么信息，要增加什么信息。

有了上面这些改进，GRU 的参数更少，因此会比 LSTM 更快一点。

#### **GRU 如何解决梯度消失问题的**

前面我们提到过 GRU 的目的是解决标准 RNN 可能存在的梯度消失问题，那么它是如何做的呢？

GRU
主要是通过使用更新门和重置门来解决这个问题，更新门控制着流入单元的信息，重置门控制流出单元的信息，它们可以决定要保留哪些相关信息，并将其传递给网络的下一个时间步，不需要通过清洗或者删除与预测无关的信息，就能保持较长时间的内容。

从数学的角度来讲，在前面的文章我们推导过，在标准 RNN 的反向传播过程中，这项 $\frac{\partial h_t^i}{\partial
h_k^i} $ 可能导致梯度消失或爆炸，这一项的作用是将时间步 t 的误差反向传播到时间步 k，使得模型能够学习长距离的依赖性或相关性。

它等于

$$ \frac{\partial h_t^i}{\partial h_k^i} = (u_{ii})^{(t-k)} \prod_{g=k+1}^t
\sigma' (z_g^i)$$

  * 当 t-k 很大时，如果隐层状态的激活函数的梯度小于 1，或者权重小于 1 时，那么将出现梯度消失问题，因为 t-k 个小于 1 的乘积会使得整个结果接近零。
  * 而激活函数 Sigmoid 和 Tanh 的梯度通常都小于 1，并且在接近零梯度的地方快速饱和，这会使梯度消失的问题更严重。
  * 同理，当 $u_ii$ 大于 1 时，就可能会发生梯度爆炸，因为 t-k 会使 $\frac{\partial h_t^i}{\partial h_k^i} $ 这一项呈指数级增长。

**在 GRU 中** ，当 $z_t$ 更新门接近 0 时，则从公式

$$\text{h}_t = (1 - z_t) \circ \text{h}_{t-1} + z_t \circ \widetilde{h}_t$$

可知 $\text{h}_t^i \approx \text{h}_{t-1}^i$，$\forall i \in K$，其中 K 是 $z_t^i
\approx 0$ 的所有隐藏单元的集合。

这样在求偏导数时，我们会得到以下结果：

$$ \frac{\partial h_t^i}{\partial h_{i-1}^i} \approx 1 $$

由于：

$$ \frac{\partial h_t^i}{\partial h_{k}^i} = \prod_{g=k+1}^t \frac{\partial
h_g^i}{\partial h_{g-1}^i} $$

可知 $ \frac{\partial h_t^i}{\partial h_{k}^i} $ 这一项也接近于 1。

**这样隐层状态可以无损地被复制很多时间步，梯度消失的机会减少，模型就能够学习长距离的相关性。**

### Beam Search

接下来我们再介绍另一种 LSTM 的改进方案，叫做 Beam Search，它也经常和 seq2seq 一起用，Beam Search 方法可以帮助提高
LSTM 的预测结果的质量。

**Beam Search** 是一种近似搜索策略，大家肯定都知道 Greedy search，它是每次找出预测概率最大的一个单词，而 Beam
Search 是 **每次找出预测概率最大的 b 个单词** 。

这个 b 是 Beam 的长度，每次产生的 b 个输出被称为光束 Beam，这 b 个结果的选择是依据最大的联合概率
$p(y_t，y_{t+1}，...，y_{t+b} | x_t)$，而不是最大的 $p(y_t | x_t)$。

在下一篇我们会实践一个语言模型的应用，会用到 Beam Search 对 LSTM 改进，现在我们先看一下它在语言模型上面是如何搜索的。

例如，我们想要一个一个字地生成 `Lilei gave Hanmeimei an apple pie`。

![](https://images.gitbook.cn/de866130-6463-11ea-9b0c-5b7924682571)

**1\. 最开始我们有这个单词：Lilei**

使用 LSTM + Beam Search 生成下一个单词它会先从词汇表中找出最有可能是下一个单词的 b 个预测词，这里 b 我们取 3 为例：
`gave, Hanmeimei, apple`，这一步的概率为：$p(y_1 | x)$。

**2\. 然后再以这 Lilei + 这 b 个词为条件继续搜索下一个单词**

即以 `Lilei gave，Lilei Hanmeimei，Lilei apple` 为条件，用同样的方式预测下一个词的最大可能的 b 个：

    
    
    Lilei gave Hanmeimei，Lilei Hanmeimei apple，Lilei apple pie
    

这一步的概率为：$p(y_2 | y_1, x)$。

则新生成的两个单词的 3 种情况的概率为：

$$p（y_2, y_1 | x） = p（y_2 | y_1, x）* p（y_1 | x）$$

**3\. 以此类推，每次都取概率最大的 b 种预测词** ，最后我们可以得到概率最大的那一组序列：

    
    
    Lilei gave Hanmeimei an apple pie
    

上面我们介绍的这两种模型和一个搜索方法都可以作为对基本 LSTM 的改进，在做具体任务时，可以都尝试一下，将基本 LSTM 单元换成 Peephole
LSTM 或者 GRU，比较一下效果并选择最好的方案，在下一篇文章中，我们就会将这几种改进方案应用起来。

**面试题：**

  1. 什么是 GRU？
  2. RNN、LSTM、GRU 的异同点？
  3. GRU 和 LSTM 的优劣势？
  4. GRU 是如何改进 LSTM 的？
  5. GRU 是如何避免梯度消失的？

参考文献：

  * colah，[Understanding LSTM Networks](https://colah.github.io/posts/2015-08-Understanding-LSTMs/)

