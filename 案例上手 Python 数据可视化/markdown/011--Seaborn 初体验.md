Seaborn 是基于 Matploblib 发展而来的实现数据可视化的库，它提供了一些更高级的工具，使得应用起来比 Matplotlib
更简单。因此，目前应用非常广泛。在第0-3课中已经说明了 Seaborn 的安装方法，如果尚未安装好，可以参考有关内容，或者参考 [Seaborn
的官方网站](https://seaborn.pydata.org/)。

### 初步了解 Seaborn

Seaborn 的目的是通过对 Matplotlib 的更高级封装，可以自动处理来自数据集（包括 DataFrame 和数组等类型）的不同特征数据。

    
    
    %matplotlib inline
    import seaborn as sns    # ①
    sns.set()    # ②
    tips = sns.load_dataset("tips")    # ③
    sns.relplot(x='total_bill', y='tip', 
                col='time', hue='smoker', style='smoker', size='size', data=tips)    # ④
    

输出结果：

![enter image description
here](https://images.gitbook.cn/f9b63790-3cb0-11e9-8dd5-1b4e2ffa509f)

因为 Seaborn 是基于 Matplotlib 的，所以还需要在程序前面写上 %matplotlib inline，以实现将图示嵌入到当前的页面中。

观察生成的图示中坐标系的 title 和各个坐标轴上的说明，再对照代码，并没有出现 Matplotlib 中不得不有的函数，比如
title、set_xlabel 等。仅从这一点，就显示出来 Seaborn 是相对 Matplotlib 更高级的封装。更何况右侧的图例，代码中更看不到
legend 的使用。

Seaborn 以非常简捷的代码实现了数据可视化。

下面详细研习这段程序。

语句 ① 引入 seaborn 库，注意，将其更名为 sns 是业界习惯。

语句 ②，意味着所得可视化结果采用 Seaborn 默认的风格，包括主题、色彩搭配等。

语句 ③，这是 Seaborn 相对 Matplotlib 的一大特征，Seaborn 中集成了一些数据集，这些数据集都是 DataFrame
类型，可以通过 load_dataset 获得，参数 'tips' 意味着获得 tips 数据集——③ 最终返回了 DataFrame
对象，这个数据集的主要特征和基本数据状况如下：

    
    
    tips.sample(5)
    

![avatar](https://images.gitbook.cn/FvDuycx8ycUwZiD5ODhVqmbmrjSp)

    
    
    tips.dtypes
    
    #out
    total_bill     float64
    tip            float64
    sex           category
    smoker        category
    day           category
    time          category
    size             int64
    

在 tips 数据集中，有三个特征是数字类型数据，有四个特征是分类数据，例如 sex 特征下的数据只有 Male 和 Female 两类。

语句 ④ sns.relplot(x='total_bill', y='tip', col='time', hue='smoker',
style='smoker', size='size', data=tips)，对数据集中的五个特征进行了可视化。

  * 坐标系的横轴和纵轴分别为数字类型的特征 total_bill 和 tip，即参数 x='total_bill'，y='tip'。
  * 另外一个数字类型的特征 size，则用图示中点的大小表示，即参数 size='size'。
  * 图示中有两个分区，分别表示了特征 time 中的两类数据，即参数 col='time'。
  * 每个分区中的坐标系内的点，分别代表了特征 smoker 中的两类数据，即参数 hue='smoker'，style='smoker'。

慢慢回味，暂不求甚解，只需要对 sns.replot 函数有初步感受，简单地设置几个参数，就得到了信息丰富的可视化结果。

还记得在 Matplotlib 中如何绘制簇状的柱形图吗？那时我们要计算一下每簇的中心，然后使用两次 bar
函数，依据柱子的不同位置，来完成簇状柱形图的绘制。

在 Seaborn 中，把这个过程优化了。

    
    
    titanic = sns.load_dataset("titanic")
    titanic.head()
    

![avatar](https://images.gitbook.cn/Fir5C1YiM0yVwO7JkUooJCjdPUkC)

> 这是著名的泰坦尼克号豪华邮轮的有关数据——1997 年上映了一部电影《泰坦尼克号》，在第 70 届奥斯卡金像奖中获得 14 项提名，最终获得 11
> 个奖项。这是一个非常好的电影，推荐欣赏。

下面要做的是用 Seaborn 对上述数据中的部分特征绘制簇状柱形图，显示不同等级的船舱不同性别的人获救情况。

    
    
    sns.barplot(x = "class", y = "survived", hue = "sex", data=titanic)
    

输出结果：

![enter image description
here](https://images.gitbook.cn/e9fab9c0-3d7d-11e9-ab86-d9b554a5bb95)

居然如此简单。是的，就是这么简单。

因此，一定要掌握 Seaborn 这个库，它值得拥有。

Seaborn 不仅仅优化了 Matplotlib 中已有的统计图的绘制流程，它还提供了一些新型的统计图，比如：

    
    
    iris = sns.load_dataset("iris")
    iris.head()
    

![avatar](https://images.gitbook.cn/FjmJTd2s5nPFJN5mfwrzLmslJX_I)

这是著名的鸢尾花的数据集，因为这个数据集太有名，而且在机器学习和数据分析中经常会用到，所以有必要对这个数据集进行较为详细地介绍。以下内容参考了维基百科
[“Iris flower data set”](https://en.wikipedia.org/wiki/Iris_flower_data_set)
词条：

> The Iris flower data set or Fisher's Iris data set is a multivariate data
> set introduced by the British statistician and biologist Ronald Fisher in
> his 1936 paper The use of multiple measurements in taxonomic problems as an
> example of linear discriminant analysis. It is sometimes called Anderson's
> Iris data set because Edgar Anderson collected the data to quantify the
> morphologic variation of Iris flowers of three related species. Two of the
> three species were collected in the Gaspé Peninsula "all from the same
> pasture, and picked on the same day and measured at the same time by the
> same person with the same apparatus".

这个数据集一共有五个特征：

  * 花萼长度，sepal_length
  * 花萼宽度，sepal_width
  * 花瓣长度，petal_length
  * 花瓣宽度，petal_width
  * 种类，species

对鸢尾花数据集有了初步了解之后，就绘制 Seaborn 提供的一种新型统计图——相对 Matplotlib 而言。

    
    
    sns.swarmplot(x="species", y="petal_length", data=iris)
    

输出结果：

![enter image description
here](https://images.gitbook.cn/e8a03d60-3d7e-11e9-9de9-d79e7402baad)

图示的结果，就类似函数的名字 swarmplot，这种类型的图，后面还会用到，此处算是第一面，不必深入，只要浅尝辄止即可。

本课对 Seaborn 仅做了初步介绍，目的在于感受它相对 Matplotlib 的优势，也就是知道它存在的必要性。

下一课开始，要通过示例对 Seaborn 的常用功能进行详细介绍。

### 总结

本课对 Seaborn 有了初步了解，特别要知道 Seaborn 是对 Matplotlib
更高级的封装，不仅简化了原来的制图函数，而且还提供了一些新型的绘图函数。因此，用 Seaborn
制图的时候，允许我们将更多的注意力集中到数据本身，当需要将数据可视化的时候，不必刻意去思考复杂的实现过程，随手调用函数就可以达成目标，让数据分析的全过程一气呵成。

### 答疑与交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《Python数据可视化》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「288」给小助手伽利略获取入群资格。**

