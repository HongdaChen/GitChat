### 关于排序算法

今天与大家一起探讨计算机科学中的基础算法之排序算法。排序算法是非常基础同时又应用非常广泛的算法，无论在工作还是在生活中，比如：

  * 数据库脚本，如 MSSQL、MySQL、NoSQL 中按多个关键词的升序或降序排序，例如，学生按照考试分数排名次，分数相等的再按照学号排序。
  * 前端界面和后端写服务时经常要调用排序接口。
  * 计算机科学很多算法都是基于排序算法的，例如二分查找算法实现逻辑中一般都会先对原序列应用排序操作。
  * 还有很多其他应用……

我们使用的高层排序接口，比如 Python 内置的 sorted
函数，可能会结合我们今天介绍的几种排序算法，比如待排序列表内元素个数较少时，使用插入排序；较多时使用快速排序和归并排序算法。

### 实际案例

排序算法选择主要考虑如下因素：

  * 算法的执行效率
  * 排序的稳定性
  * 排序元素的个数
  * 递归调用的开销

例如 Java 语言中，首先根据元素的类型选择排序算法，当为基本类型时，排序实现逻辑如下：

  * 待排序的数组中的元素个数小于 7 时，采用插入排序（不用快排是因为递归开销代价更大）；
  * 当待排序的数组中的元素个数大于 或等于 7 时，采用快速排序，选择合适的划分元是极为重要的：
    * 当数组大小 7 时，取首、中、末三个元素中间大小的元素作为划分元；
    * 当数组大小 40 时，从待排数组中较均匀的选择 9 个元素，选出一个伪中数做为划分元。

当为引用类型时，排序实现逻辑如下：

  * 采用归并排序，对于引用类型排序的稳定性非常重要，而快速排序是无法保证排序的稳定性的，归并排序是稳定的排序算法，并且时间复杂也较优 nlog(n)。

接下来先介绍这三种排序算法：插入排序、快速排序、归并排序。

### 插入排序

直接插入排序，英文名称 straight insertion
sort，它是一种依次将无序区的元素在有序区内找到合适位置依次插入的算法，如同打扑克牌抽排时使用到的方法一样。

**基本思想** ：每次从无序表中取出第一个元素，把它插入到有序表的合适位置，使有序表仍然有序，直到无序表内所有元素插入为止。

**插入排序过程演示** ：用待排序列 **3 2 5 9 2** ，演示直接插入排序的过程，至此结束插入排序的过程，可以看到直接插入排序共经过 4
轮的操作。

![插入排序演示](http://images.gitbook.cn/1e2c6650-cc6c-11e7-9ba1-e7847f17191c)

![enter image description
here](http://images.gitbook.cn/2a274380-cc6c-11e7-9aa5-23cd4e74d481)

**插入排序评价** ：插入排序的最坏时间复杂度为 O(n^2)，属于稳定排序算法，对于处理小批量数据时高效，因此 Java 在排序元素个数小于 7
时，选择了这种算法。

### 快速排序

快速排序（Quicksort）是对冒泡排序的一种改进。通过一趟排序将要排序的数据分割成独立的两部分，其中一部分的所有数据都比另外一部分的所有数据都要小。然后再按此方法对这其中一部分数据进行快速排序，这是递归调用。再按此方法对另一部分数据进行快速排序，这也是递归调用。

**快速排序过程演示** ：假设待排序的序列仍为： **3 2 5 9 2** 。第一轮比较，选取第一个关键码 3 为 pivot，初始值 i = 0, j
= 4，整个的比较过程如下图所示：

![enter image description
here](http://images.gitbook.cn/b2227140-cc6e-11e7-9ba1-e7847f17191c)

快速排序评价：最坏时间复杂度为 O(n^2)，这是一种退化的情况，在大多数情况下只要选取了合适的划分元后，时间复杂度为 nlog(n)，快速排序通常比其他
O(nlogn) 算法更快，属于速度很快的非稳定排序算法。

### 归并排序

归并排序，英文名称是 MERGE-SORT。它是建立在归并操作上的一种有效的排序算法，该算法采用分治法（Divide and
Conquer）的一个非常典型的应用。将已有序的子序列合并，得到完全有序的序列；即先使每个子序列有序，再使子序列段间有序。

二路归并：比较 a[i] 和 b[j] 的大小，若 a[i]≤b[j]，则将第一个有序表中的元素 a[i] 复制到 r[k] 中，并令 i 和 k 分别加上
1；否则将第二个有序表中的元素b[j]复制到 r[k] 中，并令 j 和 k 分别加上
1；如此循环下去，直到其中一个有序表取完；然后再将另一个有序表中剩余的元素复制到 r 中从下标 k 到下标 t 的单元。

这个过程，请见下面的演示：

![enter image description
here](http://images.gitbook.cn/5c8b09a0-cccc-11e7-b630-b97450fd73b8)

![enter image description
here](http://images.gitbook.cn/6a131680-cccc-11e7-bf41-c9c5b004a42a)

![enter image description
here](http://images.gitbook.cn/7e0c4ee0-cccc-11e7-a1aa-212b011b68f6)

![enter image description
here](http://images.gitbook.cn/88f2aed0-cccc-11e7-bf41-c9c5b004a42a)

![enter image description
here](http://images.gitbook.cn/93b13b70-cccc-11e7-bf41-c9c5b004a42a)

![enter image description
here](http://images.gitbook.cn/a60d65a0-cccc-11e7-a1aa-212b011b68f6)

![enter image description
here](http://images.gitbook.cn/ada2ece0-cccc-11e7-b630-b97450fd73b8)

归并排序的最坏时间复杂度为 O(nlogn)，是一种稳定排序算法。

### 希尔排序

缩小增量排序，是对上面介绍的插入排序算法的一种更高效的改进版本，可以看做是分组版插入排序算法。

希尔排序思想，先取一个正整数 d1，以 d1 间隔分组，先对每个分组内的元素使用插入排序操作，重复上述分组和直接插入排序操作；直至 di =
1，即所有记录放进一个组中排序为止。

案例演示：仍然用待排序列 **3 2 5 9 2** 。在第一轮中，选取增量为2，即分为两组，第一组为 [3 5 2]，另一组为 [2
9]，分别对这两组做直接插入排序，第一组插入排序的结果为[2 3
5]，第二组不动，这样导致原来位于最后的元素2经过第一轮插入排序后跑到最开始，所以希尔排序不是稳定的排序算法。

再经过第二轮排序，此时的增量为 1，所以一共只有一组，相当于直接插入排序，9后移1步，5插入到原9的位置。

这样整个的希尔排序结束，得到如下图所示的非降序序列。

![enter image description
here](http://images.gitbook.cn/dd8f0ba0-ccd1-11e7-b630-b97450fd73b8)

希尔排序评价的最坏时间复杂度为 O(n^2)，对中等规模的问题表现良好，但对规模非常大的数据排序不是最优选择，并且如上所述希尔排序不是稳定的排序算法。

### 冒泡排序

冒泡排序算法 Day 42 中已经有比较详细的介绍，请参考。算法的时间复杂度为 O(n^2)，对于大批量数据处理效率低，是稳定的排序算法。

### 选择排序

直接选择排序，英文名称：Straight Select Sorting，是一个直接从未排序序列选择最值到已排序序列的过程。选择排序思想：

  * 第一次从 R[0]~R[n-1] 中选取最小值，与 R[0] 交换；
  * 第二次从 R[1]~R[n-1] 中选取最小值，与 R[1] 交换；
  * ……；
  * 第 i 次从 R[i-1]~R[n-1] 中选取最小值，与 R[i-1] 交换；
  * 总共通过 n-1 次，得到一个按关键码从小到大排列的有序序列。

选择排序举例，待排序列 3 2 5 9 2，演示如何用直接选择排序得到升序序列。

  * 第一轮，从所有关键码中选择最小值与 R[0] 交换，3 与 2 交换，如下图所示；
  * 第二轮，从 R[1]~R[n-1] 中选择最小值与 R[1] 交换，3 与 2 交换；
  * 第三轮，从 R[2]~R[n-1] 中选择最小值与 R[2] 交换，5 与 3 交换；
  * 第四轮，从 R[3]~R[n-1] 中选择最小值与 R[3] 交换，9 与 5 交换；
  * 终止。

![enter image description
here](http://images.gitbook.cn/56bd0ce0-ccd5-11e7-8003-bb3919ccc53a)

直接选择排序的最坏时间复杂度为 O(n^2)，处理小批量数据时，直接选择排序所需要的比较次数也比直接插入排序多，它是稳定排序算法。

### 堆排序

堆排序，英文名称
Heapsort，利用二叉树（堆）数据结构设计的一种排序算法，是对直接选择排序的一种改进算法。在逻辑结构上是按照二叉树存储结构，正是这种结构优化了选择排序的性能。在物理存储上是连续的数组存储，它利用了数组的特点快速定位指定索引的元素。

堆排序思想，以大根堆排序为例，即要得到非降序序列：

  1. 先将初始文件 R[0..n-1] 建成一个大根堆，此堆为初始的无序区。
  2. 再将关键字最大的记录 R[0]（即堆顶）和无序区的最后一个记录 R[n-1] 交换，由此得到新的无序区 R[0..n-2] 和有序区 R[n-1]，且满足 R[0..n-2] ≤ R[n-1]。由于交换后新的根 R[0] 可能违反堆性质，故应将当前无序区R[0..n-2] 调整为堆。
  3. 然后再次将 R[0..n-2] 中关键字最大的记录R[0]和该区间的最后一个记录 R[n-2] 交换，由此得到新的无序区R[0..n-3] 和有序区 R[n-2..n-1]，且仍满足关系 R[0..n-3] ≤ R[n-2..n-1]。
  4. 重复步骤 2 和步骤 3，直到无序区只有一个元素为止。

堆排序举例，待排序列 **3 2 5 9 2** 。第一步，首先以上待排序列的物理存储结构和逻辑存储结构的示意图如下所示：

![enter image description
here](http://images.gitbook.cn/932993e0-ccd7-11e7-ac9c-a19c9dd475d6)

构建初始堆是从 length/2-1，即从索引 1 处开始构建，2 的左右孩子等于 9,2，它们三个比较后，父节点 2 与左孩子 9 交换，如下图所示：

![enter image description
here](http://images.gitbook.cn/c1306a20-ccd7-11e7-83f1-771948855d27)

接下来索引 1 减去 1，即索引 0 处，开始与其左右孩子比较，比较后父节点 3 与左孩子节点 9 交换，如下所示：

![enter image description
here](http://images.gitbook.cn/d6ff25d0-ccd7-11e7-a0f7-53d86e5a89e6)

因为索引等于 0，所以构建堆结束，得到大根堆，第一步工作结束。

下面开始第二步调整堆，也就是不断地交换堆顶节点和未排序区的最后一个元素，然后再构建大根堆。

交换堆顶元素 9 和未排序区的最后一个元素 2，如下图所示，现在有序区仅有一个元素 9 归位。

![enter image description
here](http://images.gitbook.cn/f1ceb420-ccd7-11e7-a788-b7cd43ed6965)

接下来拿掉元素 9，未排序区变成 2,3,5,2，然后从堆顶 2 进行堆的再构建，比较父节点 2 与左右子节点 3 和 5，父节点 2 和右孩子 5
交换位置，如下图所示，这样就再次得到了大根堆。

![enter image description
here](http://images.gitbook.cn/0b5c9ab0-ccd8-11e7-a788-b7cd43ed6965)

再交换堆顶 5 和未排序区的最后一个元素 2，这样 5 又就位了，这样未排序区变为了 2,3,2，已排序区为
5,9，交换后的位置又破坏了大根堆，已经不再是大根堆了，如下图所示：

![enter image description
here](http://images.gitbook.cn/2fcdfb00-ccd8-11e7-ac9c-a19c9dd475d6)

所以需要再次调整，然后堆顶 2 和左孩子 3 交换，交换后的位置如下图所示，这样二叉树又重新变为了大根堆，再把堆顶 3 和此时最后一个元素也就是右孩子 2
交换。

![enter image description
here](http://images.gitbook.cn/555b52a0-ccd8-11e7-83f1-771948855d27)

接下来再构建堆，不再赘述，见下图：

![enter image description
here](http://images.gitbook.cn/72281080-ccd8-11e7-a0f7-53d86e5a89e6)

堆排序的时间主要由建立初始堆和反复重建堆这两部分的时间开销构成，堆排序的最坏时间复杂度是
O(nlogn)，堆排序是不稳定的排序方法。由于建初始堆所需的比较次数较多，所以堆排序不适宜于记录数较少的排序序列。

### 算法兑现

冒泡排序：

    
    
    def bubble_sort(a):
        n, swap_size = len(a), 1
        while swap_size > 0:
            swap_size = 0
            for i in range(n-1):
                if a[i] > a[i+1]:
                    a[i],a[i+1] = a[i+1],a[i]
                    swap_size += 1
            n-=1
        return a
    

快速排序：

    
    
    def quick_sort(a, left=None, right=None):
        left = 0 if not isinstance(left,(int, float)) else left
        right = len(a)-1 if not isinstance(right,(int, float)) else right
        if left < right:
            part_index = part(a, left, right)
            quick_sort(a, left, part_index-1)
            quick_sort(a, part_index+1, right)
        return a
    
    def part(a, left, right):
        pivot = left
        index = pivot+1
        i = index
        while  i <= right:
            if a[i] < a[pivot]:
                a[i],a[index] = a[index],a[i]
                index+=1
            i+=1
        a[pivot],a[index-1] = a[index-1],a[pivot]
        return index-1
    

插入排序：

    
    
    def insertion_sort(a):
        for i in range(len(a)):
            pre_index = i-1
            current = a[i]
            while pre_index >= 0 and a[pre_index] > current:
                a[pre_index+1] = a[pre_index]
                pre_index-=1
            a[pre_index+1] = current
        return a
    

希尔排序：

    
    
    import math
    
    def shell_sort(a):
        gl=1
        while(gl < len(a)/3):
            gl = gl*3+1
        while gl > 0:
            for i in range(gl,len(a)):
                temp = a[i]
                j = i-gl
                while j >=0 and a[j] > temp:
                    a[j+gl]=a[j]
                    j-=gl
                a[j+gl] = temp
            gl = math.floor(gl/3)
        return a
    

直接选择排序：

    
    
    def selection_sort(a):
        for i in range(len(a) - 1):
            min_index = i
            for j in range(i + 1, len(a)):
                if a[j] < a[min_index]:
                    min_index = j
            if i != min_index:
                a[i], a[min_index] = a[min_index], a[i]
        return a
    

堆排序：

    
    
    import math
    
    def heap_sort(a):
        al = len(a)
    
        def heapify(a, i):
            left = 2 * i + 1
            right = 2 * i + 2
            largest = i
            if left < al and a[left] > a[largest]:
                largest = left
            if right < al and a[right] > a[largest]:
                largest = right
    
            if largest != i:
                a[i], a[largest] = a[largest], a[i]
                heapify(a, largest)
    
        # 建堆
        for i in range(math.floor(len(a) / 2), -1, -1):
            heapify(a, i)
    
        # 不断调整堆：根与最后一个元素
        for i in range(len(a) - 1, 0, -1):
            a[0], a[i] = a[i], a[0]
            al -= 1
            heapify(a, 0)
        return a
    

归并排序：

    
    
    import math
    
    def merge_sort(a):
        if(len(a)<2):
            return a
        middle = math.floor(len(a)/2)
        left, right = a[0:middle], a[middle:]
        return merge(merge_sort(left), merge_sort(right))
    
    def merge(left,right):
        result = []
        while left and right:
            if left[0] <= right[0]:
                result.append(left.pop(0));
            else:
                result.append(right.pop(0));
        while left:
            result.append(left.pop(0));
        while right:
            result.append(right.pop(0));
        return result
    

依次调用以上各个排序算法：

    
    
    sort_funs = [bubble_sort,quick_sort,insertion_sort,shell_sort,selection_sort,heap_sort,merge_sort]
    
    a = [10, 7, 11, 7, 18, 17, 9, 17, 0, 7, 3, 20, 20, 15]
    for fun in sort_funs:
        a_sorted = fun(a)
        print(a_sorted)
    

结果都一致，变为非降序序列。

    
    
    [0, 3, 7, 7, 7, 9, 10, 11, 15, 17, 17, 18, 20, 20]
    [0, 3, 7, 7, 7, 9, 10, 11, 15, 17, 17, 18, 20, 20]
    [0, 3, 7, 7, 7, 9, 10, 11, 15, 17, 17, 18, 20, 20]
    [0, 3, 7, 7, 7, 9, 10, 11, 15, 17, 17, 18, 20, 20]
    [0, 3, 7, 7, 7, 9, 10, 11, 15, 17, 17, 18, 20, 20]
    [0, 3, 7, 7, 7, 9, 10, 11, 15, 17, 17, 18, 20, 20]
    [0, 3, 7, 7, 7, 9, 10, 11, 15, 17, 17, 18, 20, 20]
    

### 小结

今天一起学习 7
个经典的排序算法，这些算法思想会在以后的算法学习和实践中经常被使用，希望读者们能举一反三，熟能生巧，提到某个排序算法便能在脑海中形成对它们排序过程的画面。

今天完整代码的 Jupyter Notebook 下载链接：

> <https://pan.baidu.com/s/1O7i_Lgi9MFIcB_0W-GGvXg>
>
> 提取码：ffts

