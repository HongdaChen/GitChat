### 猜数游戏与二分查找

还记得猜数游戏吗？之前的问题是：如果你是攻击者，你能保证在10次（含）之内猜出对方的数字吗？

因为$2^{10} = 1024 > 1000$, 所以如果运用二分查找的话，我们作为攻击者是肯定可以赢得比赛的。

实现1-1000数列的二分查找

既然二分查找适用于猜数游戏，那让我们先套用一下上节课学习的二分查找算法实现代码，来实现以下1-1000数列的二分查找。

代码如下：

    
    
    arr = list(range(1, 1001)) # 生成一个List，里面顺序存储着1-1000这一千个元素
    
    tn = 635  # 可以随便改
    
    low = 0
    high = len(arr) - 1
    
    while (low <= high):
        m = int((high - low) / 2) + low
    
        if (arr[m] == tn):
            # 把打印出目标数所在的位置下标改成直接打印出目标数
            print("Succeeded! The target number is: ", arr[m]) 
            break
        else:
            if (arr[m] < tn):
                low = m + 1
            else:
                high = m - 1
    
    if (low > high):
        print("Search failed.")
    

代码-1

大家可以仔细比较一下，就会发现，相对于上一章最后的代码，代码-1只修改了第一行和成功时的打印语句。

第一行用到了Python3中的两个内置函数：list()和range()，用它们直接生成了一个包含1000个元素的List，且其中的元素从1开始依次数值递增1。

查找成功时的打印语句是因为在当前这个arr中，所有的元素值和元素下标有一一对应的关系，因此不必打印位置信息，直接打印元素值即可。这样，也可以和稍后的修改相一致。

1-1000数列二分查找的另一种写法

在代码-1的实现中，每一个元素的位置和元素值有直接的对应关系——下标+1 ==
元素值。既然如此，我们又何必非要把这1000个数字放在一个List变量中呢？我们完全可以用下面的方法来实现同一个算法：

    
    
    tn = 635  # 可以随便改
    
    low = 1
    high = 1000
    
    while (low <= high):
        m = int((high - low) / 2) + low
    
        if (m == tn):
            # 打印出目标数
            print("Succeeded! The target number is: ", m)
            break
        else:
            if (m < tn):
                low = m + 1
            else:
                high = m - 1
    
    if (low > high):
        print("Search failed.")
    

代码-2

让low，high和m直接代表元素值本身就好了，反正自然数的本来就是有序配列的等差数列。

### 编写猜数游戏攻击者辅助程序

辅助攻击者的程序

上面我们用代码实现了在1-1000范围内进行二分查找的功能。但如果仅仅只是这样的程序，它自己作为攻击者倒是可以，却无法变成人类攻击者的辅助工具。

在玩猜数游戏的时候，人类攻击者人肉运用二分查找会怎么样呢？肯定每次迭代都要：

  * 都要在头脑中计算出当前查找区间的中位数；
  * 然后告知对方（防守者）；
  * 等对方报出或大或小的结果后，再计算下次查找的区间
  * 然后进入下一轮迭代——好累啊，要是有个程序，替我们来做这个计算就好了。

我们希望有这么一个程序：

  * 攻击者运行它之后，它自动提出一个猜测值；
  * 攻击者把这个猜测值报给对手，然后将对手的反馈告知程序；
  * 程序再据此产生下一个猜测值
  * ……

如此反复多轮，直到猜出来为止。

由二分查找算法而来

这个辅助程序的基础是一个二分查找算法，这个程序自己是不知道目标数的！

它只是在1-1000的范畴上运行二分查找，不过每次循环得出当前的中位数之后，就要将其作为猜测数打印出来；然后不是继续运行，而是等着用户告诉程序：这个当前的猜测数是大、小还是正好？再根据用户的反馈进行下一轮循环。

这里就涉及到了输入和输出（打印）。打印好办啊，就用我们已经学过的print()，可是怎么能让用户告诉程序信息呢？

不用急，Python有个非常好用的内置函数：input（），用这个函数，我们可以读取用户的输入信息。 比如下面的代码：

    
    
    userInput = input("Your input: ")
    print(userInput)
    

第一行的功能是在屏幕上输出“Your input：”
这一串字，然后等着用户输入。用户输入一行字，然后打回车后，用户输入的内容就成了变量userInput的值，第二行把这个值打印出来。

下面是上面两行代码的一次运行：

    
    
    Your input: I'm Ye Mengmeng. I'm Ye Mengmeng.
    

由输入输出控制的二分查找，所以，我们要编写的程序功能是——在二分查找中:

1）每次计算出中间数后，将其打印出来，并询问与猜测数的大小关系；

2）读取用户输入，以确定当前猜测数和目标数的大小关系。

这里还有一个小问题，用户应该如何输入呢？

假使允许用户随便输入文字，例如：“Unfortunately, your guess is less than my secret number. Try
again!” 之类的文字，固然用户是痛快了，但是程序要读懂还得去理解自然语言，这会需要很多额外的技术和工作 ——
自然语言处理（NLP）是当前人工智能领域的主要研究范畴之一。

一个小游戏没必要搞那么复杂，我们就用传统计算机程序最常用的：命令式输入——用固定的输入代替固定的含义——让用户用输入1，2或者3来代替“猜中了”，“太小了”，和“太大了”！

还有个问题——人类经常会犯错，很可能输入成了别的数字或者字母，这可怎么办呢？

不如我们加一个判断吧，判断用户输入的是否是1,2或者3，如果都不是，则说明用户的输入不合法，要求用户重新输入。

辅助程序的实现

辅助程序代码如下：

    
    
    low = 1
    high = 1000   
    
    while low <= high:
    
        m = int((high - low) / 2) + low
        print("My guess is", m)
    
        # userInput是循环条件中被判断的变量，因此需要在循环之前先有个值，否则循环会出错。
        userInput = ""  
    
        while userInput != '1' and userInput != '2' and userInput != '3':
            print("\t\t 1) Bingo! %s is the secret number! \n\
             2) %s < the secret number.\n\
             3) %s > the secret number." % (m, m, m))
            userInput = input("Your option:")
            userInput = userInput.strip()
    
        if userInput == '1':
            print("Succeeded! The secret number is %s." % m )
            break
        else:
            if userInput == '2':
                low = m + 1
            else:
                high = m - 1            
    
    if low > high:
        print("Failed！")
    

这个时候，假设神秘数是732，则运行结果如下：

    
    
    My guess is 500 1) Bingo! 500 is the secret number! 2) 500 < the secret number. 3) 500 > the secret number. Your option:2 My guess is 750 1) Bingo! 750 is the secret number! 2) 750 < the secret number. 3) 750 > the secret number. Your option:3 My guess is 625 1) Bingo! 625 is the secret number! 2) 625 < the secret number. 3) 625 > the secret number. Your option:2 My guess is 687 1) Bingo! 687 is the secret number! 2) 687 < the secret number. 3) 687 > the secret number. Your option:2 My guess is 718 1) Bingo! 718 is the secret number! 2) 718 < the secret number. 3) 718 > the secret number. Your option:2 My guess is 734 1) Bingo! 734 is the secret number! 2) 734 < the secret number. 3) 734 > the secret number. Your option:3 My guess is 726 1) Bingo! 726 is the secret number! 2) 726 < the secret number. 3) 726 > the secret number. Your option:2 My guess is 730 1) Bingo! 730 is the secret number! 2) 730 < the secret number. 3) 730 > the secret number. Your option:2 My guess is 732 1) Bingo! 732 is the secret number! 2) 732 < the secret number. 3) 732 > the secret number. Your option:1 Succeeded! The secret number is 732.
    

辅助程序的改进

哇！已经很不错了，我们就可以直接辅助攻击者赢得游戏啦~~ 但是等一等，发现没有，现在还有点小问题：

1） 虽然这样做可以找到神秘数，但是却要自己数到底猜了几轮。

2） 如果使用程序的人每次输入的都不是1，2或者3，而是其他字符，则程序会无限等待下去，就没个完了。

针对这两个小问题，我们还要做一点修改，具体方法是：

1） 为了能够确切的知道，辅助程序找到神秘数用了多少轮，我们再加入一个变量loopNum用来记录运行过的循环论数——也就是用户猜测数字的次数。

每一次猜测都打印出当前的猜测次数，如果超过了十次还没有答对，则直接输出失败。

2）
加入变量inputNum用来记录用户的输入次数，每次用户输入只允许出错三次，一旦超过3次，则直接退出程序——这是一个有尊严的程序，对于这么不认真的用户，程序不跟你玩了！

    
    
    low = 1
    high = 1000
    
    loopNum = 0 #记录循环轮数
    
    while low <= high:
    
        m = int((high - low) / 2) + low
        print("My guess is", m)
    
        # userInput是循环条件中被判断的变量，因此需要在循环之前先有个值，否则循环会出错。
        userInput = ""
    
        inputNum = 0
    
        while userInput != '1' and userInput != '2' \
              and userInput != '3' and inputNum < 4:
    
            if inputNum == 3:
                print("You input too many invalid options. Game over!")
                exit(0)
    
            print("\t\t 1) Bingo! %s is the secret number! \n\
             2) %s < the secret number.\n\
             3) %s > the secret number." % (m, m, m))
            userInput = input("Your option:")
            userInput = userInput.strip()
            inputNum += 1
    
        loopNum += 1
    
        if userInput == '1':
            print("Succeeded! The secret number is %s.\n\
            It took %s round to locate the secret number. \n" % (m, loopNum))
            break
        else:
            if userInput == '2':
                low = m + 1
            else:
                high = m - 1
    
    if low > high:
        print("Failed！")
    

这样我们就有了一个完整的“猜数攻击者小助手”，用它来辅助我们击败防守者！

我们可以尝试分别用1000和1作为神秘数，用小助手来辅助“攻击”一下，看看结果：

经过9轮猜测，猜中732； 经过10轮猜测，猜中1000； 经过9轮猜测，猜中1。

在此，给大家留道思考题： 如果把猜数字的规模放大，仍然是从$1$开始，到整数$x（x >
1000）$结束，确保$10$轮之内一定要猜中，那么$x$最大可以是多少？提示，$2^{10} == 1024$，那么$x$是不是应该等于$1024$呢？

