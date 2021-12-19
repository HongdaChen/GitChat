今天一起同大家刷刷 LeetCode
中最经典的练习题，它们代表几大类常用的数据结构和基本算法思想。作为程序员必须要熟练掌握，因为它们经常在面试中被考察到。就算不为面试，掌握它们也会有助于我们编写出更高质量的代码，提升自己的计算机思维。

### 链表反转

链表作为经典的数据结构，应用极其广泛，列表、二叉树等都会看到它的身影，面试也是经常被问道。不要小瞧链表，老鸟也会经常翻车，所以不要掉以轻心，需要格外小心。

链表 ListNode 对象通常被定义为如下，val 为 ListNode 对象的值域，next 指向下一个 Node，下一个 Node 又指向下下个
Node，从而形成一条链式结构。

    
    
    class ListNode:
        def __init__(self, x):
            self.val = x
            self.next = None
    

链表反转是指反转后原最后 Node 变为头节点，并指向原倒数第二个节点，依次类推。

那么，如何实现链表反转？代码其实只有几行，关键如何利用链表的特点构思出链表的反转，下面解释思考的过程，同时理解链表的精髓所在。

首先给出递归版本的求解代码。假设原链表序列如下图所示：

![](https://images.gitbook.cn/73fa6160-db8a-11ea-b89a-ffa3ff92b2c1)

求解代码 reverseList 函数的前两行表示边界条件或递归终止的条件是 head 节点为 None 或 head 指向的下一个节点为 None。

令变量 tmp 指向链表的第二个节点，如下图所示，然后递归调用 reverseList 函数，反转 tmp 指向的链表：

![](https://images.gitbook.cn/869dcdc0-db8a-11ea-bedf-6d5f234f6331)

反转后的结果如下图所示，newhead 指向反转后链表的头部：

![](https://images.gitbook.cn/9a868b60-db8a-11ea-9d4f-fd9909d9aad4)

那么，如何将值等于 1 的节点经过 O(1) 时间复杂度链接到 newhead 链表的最后呢？

这就用到已经标记下来的 tmp 节点，因为反转后 tmp 节点的指向不正是等于 1 的节点吗！所以 `tmp.next = head` 操作实现建立值等于
2 的节点和值等于 1 的节点的链接：

![](https://images.gitbook.cn/acd54810-db8a-11ea-856c-1b0ca96fbf95)

但是务必要小心，此时值为 1 的节点还是会指向其他节点，所以务必将其 next 值置为 None。

到此完成递归版链表反转的解释。

    
    
    class Solution:
        def reverseList(self, head):
            if head is None or head.next is None:
                return head
            tmp = head.next
            newhead = self.reverseList(tmp)
            tmp.next = head
            head.next = None
            return newhead
    

链表反转的迭代版本应该怎么写，平时链表使用多的读者会深有体会，链表操作最容易犯错的地方，就是有意无意中形成一个环（cycle），如下 5->4->3->5
的环结构：

![](https://images.gitbook.cn/c2286b20-db8a-11ea-b89a-ffa3ff92b2c1)

如下链表反转的迭代版，一不小心就写出一个带环的链表结构。下面借助示意图分析这个环结构如何形成的。

    
    
    class Solution:
        def reverseList(self, head: ListNode) -> ListNode:
            pre = head
            cur = head.next
            while cur:
                tmp = cur.next
                cur.next = pre
                pre = cur
                cur = tmp
            return pre
    

当执行到 `cur.next = pre` 时，已经形成一个 1->2->1 的环形结构。

![](https://images.gitbook.cn/d9d583c0-db8a-11ea-bedf-6d5f234f6331)

要想避免入坑，使用链表需要慎之又慎。pre 和 cur 的初始指向非常重要，此处只需对以上代码稍作修改，仅仅修改 pre 和 cur
的初始指向，便得到链表反转的迭代版本。

    
    
    class Solution:
        def reverseList(self, head: ListNode) -> ListNode:
            pre = None
            cur = head
            while cur:
                tmp = cur.next
                cur.next = pre
                pre = cur
                cur = tmp
            return pre
    

修改后的代码，链表反转的迭代步骤示意图。初始状态如下图所示，pre 指向一个虚拟的 None 节点，这是关键一步，保证避免环的存在。

![](https://images.gitbook.cn/f2d8b680-db8a-11ea-98dc-69751c28f996)

迭代步骤中，先保存住 tmp 节点，通过 `cur.next = pre` 建立 1 节点和 None 节点的指向，然后令 pre 和 cur
的指向分别前移一位，至此完成一轮迭代。迭代时的临时变量为 tmp，起到修改 cur.next 值前缓存 cur 节点的 next 域的作用。

![](https://images.gitbook.cn/05acf6e0-db8b-11ea-9369-b5210af59199)

### 二叉树

链是单向的，节点的 next
域指向下一个节点。二叉树在单向的链表基础上，又增加出一个维度，它的节点一次指向两个节点，并称它们为儿子节点。以此递归，形成一棵树，并被称作二叉树，也就不难理解了。链表和二叉树实则神态上相似，前者为一维指向，后者为二维指向。

因此，在实际使用二叉树时，以上我们讲解的链表思维都能应用到二叉树中，并且二叉树因为指向两个节点，求解的时间性能往往更优于链表。

二叉树的应用案例，我们先从一道二叉树的路径求和题目开始。

二叉树的节点对象定义为：

    
    
    class TreeNode:
        def __init__(self, x):
            self.val = x
            self.left = None
            self.right = None
    

二叉树示意图如下，父节点对象有三个值，val、left、right 分别为当前节点的值，指向左儿子、右儿子的变量。

![](https://images.gitbook.cn/1a0fc680-db8b-11ea-bedf-6d5f234f6331)

读者们请注意，val 这个取值不仅仅局限于我们通常理解的一个数值，它可以扩展为更加具有表达力和想象力的其他各种对象，包括自定义对象。

本题目中求解二叉树从根节点（最开始的头节点）到叶节点（既没有左孩子也没有右孩子的节点）的路径求和等于 22，返回所有满足此条件的路径。如
5->4->11->2 求和等于 22，就是满足条件的一条路径。

![](https://images.gitbook.cn/2afd4580-db8b-11ea-bd81-030d631eb386)

如链表的解题思路相似，二叉树的解题构思一般也是先前进一步，使得问题的求解规模减少一点，然后递归调用，注意递归调用的终止条件，求得满足条件的解。

完整代码如下，下面分布介绍如何构思出这些代码。

    
    
    class Solution:
        def pathSum(self, root: TreeNode, sum: int) -> List[List[int]]:
            paths = []
            if root is None:
                return paths
            if root.val == sum and root.left is None and root.right is None:
                paths.append([root.val])
                return paths
    
            ls = self.pathSum(root.left,sum - root.val)
            for l in ls:
                l.insert(0,root.val)
            if len(ls)>0:
                paths.extend(ls)
    
            rs = self.pathSum(root.right,sum-root.val)
            for r in rs:
                r.insert(0,root.val) 
            if len(rs)>0:
                paths.extend(rs)
    
            return paths
    

pathSum 函数的两个参数，root 路径和 sum 值，边界条件也是递归调用的终止条件为如下。注意，`if root.val == sum and
root.left is None and root.right is None:` 中 `root.left is None and root.right
is None` 条件非常重要，确保已搜索到叶节点，如果遗漏此条件，可能搜索出某条从根节点到非叶结点的路径，这与题目中要求的从根节点到叶结点的路径相违背。

    
    
            paths = []
            if root is None:
                return paths
            if root.val == sum and root.left is None and root.right is None:
                paths.append([root.val])
                return paths
    

调用 pathSum 一次后，如果以上 2 个边界条件都不满足，则表明需要继续向下搜索，此时剩余部分的路径总和为 `sum -
root.val`，分别探索以 root.lef 节点为根节点的子树对应路径，表达为代码就是：

    
    
    rs = self.pathSum(root.right,sum-root.val)
    

rs 等于剩余部分的路径，然后需要插入它的父节点 root 到索引 0 处；搜索以 root.right 为根节点的子树原理相似。最后 paths
就是所有可能的路径。

### 动态规划

动态规划（简写为 DP）作为比较难的算法代表，此专栏在前几天单独有一讲。实话讲，这仍然不够，DP 非常灵活，一旦求解问题找到 DP
的解，通常都会带来时间复杂度的大幅改善。苦难是有的，带来的好处也非常明显，所以想培养对 DP 的敏锐度，多做一些题目是必要的。

下面再分析一道使用 DP 求解的问题：连续最大子列表的乘积，元素类型为整型。

    
    
    输入: [2,3,-2,4]
    输出: 6
    解释: [2,3] 子数组连续乘积最大为 6  
    

如果元素都为非负，求解相对容易，因为是连续子数组，所以 `max(it, it * tmp)` 意味着要么重新从 it 开始，要么一直在 tmp 上累积：

    
    
    class Solution(object):
        def maxProduct(self, nums):
            ret,tmp = nums[0],nums[0]
            for it in nums[1:]:
                tmp = max(it, it * tmp)
                ret = max(tmp,ret)
            return ret
    

如果列表中也存在负数，相比上面代码需要考虑当前遍历到的元素 it 的正负，如果为负数则 tmp_max 被赋值为当前得到的最小累积
tmp_min（注意它可能为非负或为负），如果 tmp_min 为非负，则乘以 it，大概率不会得到一个新的最大值，所以运行到 `ret =
max(tmp_max, ret)` 时，会抛弃 tmp_max，但是如果 tmp_min 为负，乘以 it 后有可能得到一个新的最大值。

    
    
    class Solution(object):
        def maxProduct(self, nums):
            ret, tmp_min, max_prod = [nums[0]]*3
            for it in nums[1:]:
                if it < 0:
                    tmp_min,tmp_max = tmp_max,tmp_min
                tmp_max = max(it, it * tmp_max)
                tmp_min = min(it, it * tmp_min)            
                ret = max(tmp_max, ret)
            return ret
    

### 贪心

贪心算法有时得到的只是可行解，而不是最优解。但在满足某些假定条件下，能获得最优解。如下买卖股票的经典题目，在满足假定条件：卖股票发生前只能先买一次股票下，使用贪心求解便能求出最大收益。

[7,1,5,3,6,4] 的最大收益为 7，如下图所示。

![](https://images.gitbook.cn/453fd610-db8b-11ea-8b08-a305d0150119)

绘图代码：

    
    
    import plotly.graph_objects as go
    fig = go.Figure()
    fig.add_trace(
        go.Scatter(
            x=[0, 1, 2, 3, 4, 5],
            y=[7,1,5,3,6,4]
        ))
    fig.show()
    

也就是贪婪的找到所有相邻的呈现上升的点对，并求累加和就是最优解，也就是最大收益。

代码如下所示：

    
    
    class Solution(object):
        def maxProfit(self, prices):
            pair = zip(prices[:-1],prices[1:])
            return sum( x1 - x0 for x0, x1 in pair if x0 < x1 )
    

### 回溯 + DFS

回溯就是带有规律的来回尝试搜寻完整解结构的过程，所谓的搜索规律，常见的一种就是结合 DFS 深度优先搜索。比如求解集合的所有子集。

已知列表 [1,2,3]，里面没有重复元素，它的所有子集为：

    
    
    [[], [1], [2], [3], [1, 2], [1, 3], [2, 3], [1, 2, 3]]
    

如何使用回溯 + DFS，构思出这个问题的解呢？关键是定义的 dfs 方法，第一个参数 e 表示一个子集，beg 表示待搜索路径的起始位置，digits
表示当前子集包括的元素个数。

    
    
    class Solution:
        def subsets(self, nums):
            if nums is None:
                return None
            ret, e = [[]], []
    
            def dfs(e, beg, digits):
                if len(e) == digits: # dfs 递归的终止条件
                    ret.append(e.copy()) # 满足条件表示找到一个子集
                    return
                while digits <= len(nums): # 求解框架的第一层
                    for i in range(beg, len(nums)): # 求解框架的第二层
                        e.append(nums[i])
                        dfs(e, i+1, digits) # 递归：头部分为 e，路径起始位置 i+1, 元素个数 digits
                        e.remove(e[-1]) # 下面五行代码实现对 e 部分的出栈控制 
                    if len(e) == 0:
                        digits += 1
                    else:
                        return
    
            dfs(e, 0, 1) # 调用 dfs
            return ret
    

求解过程的解释说明，请参考代码注释。

### 字符串、位运算、栈、队列

字符串类型题目，比如求由空格分隔的字符串的最后一个单词的字符个数：

    
    
    class Solution:
        def lengthOfLastWord(self, s: str) -> int:
            if len(s)==0:
                return 0
            return len(s.strip().split(' ')[-1])
    

需要注意如果输入的字符串为 `'a '` 需要先使用 strip 函数去掉前后的空格。

常见的位运算符号 `&` 位与，`|` 位或，`^` 异或，介绍一个使用异或求解的问题，列表中只有 1 个元素出现 1 次，其他都出现 2 次，找出出现 1
次的元素。

    
    
    class Solution(object):
        def singleNumber(self, nums):
            a = 0
            for i in nums:
                a ^= i
            return a
    

栈先进后出，队列先进先出，两者可以相互模拟，下面我们使用栈模拟队列，用 head, tail 分别指向 s 的头和尾，以实现队列的
push、pop、peek 都为 O(1) 时间复杂度。下面假定不会在空队列中使用 pop、peek 操作。

    
    
    class MyQueue:
    
        def __init__(self):
            """
            初始化
            """
            self.s = [] # 模拟栈：先进后出
            self.head = 0 # 头索引
            self.tail = 0 # 尾索引
    
        def push(self, x: int) -> None:
            """
            Push 元素到队列的尾部
            """
            self.s.append(x)
            self.tail += 1
    
    
        def pop(self) -> int:
            """
            移除并返回队列的头元素
            """
            pe = self.s[self.head]
            self.head += 1
            return pe
    
        def peek(self) -> int:
            """
            返回队列的头元素
            """
            return self.s[self.head]
    
    
        def empty(self) -> bool:
            """
            返回队列是否为空
            """
            return self.head == self.tail
    

使用队列：

    
    
    obj = MyQueue()
    obj.push(x)
    p2 = obj.pop()
    p3 = obj.peek()
    p4 = obj.empty()
    

### 小结

今天分类总结基础算法的经典练习题，首先要熟悉主要的算法和数据结构：

  * 单维度链表
  * 双维度二叉树
  * 高效的动态规划解法
  * 贪心有时也能求得最优解
  * 回溯和 DFS
  * 字符串、位运算、栈和队列

以上每一类我们都列举 1 到 2
个经典的题目，最好提到某类数据结构和算法时，马上在脑海中闪现出对应的题目，进而联想到这类问题常见的解题思路，多想多练习，最后彻底理解透这些数据结构和算法的精髓。

要想做到举一反三，并非易事，刻意练习真的很有必要，最后祝愿读者们都能理解透今天的所有题目。

