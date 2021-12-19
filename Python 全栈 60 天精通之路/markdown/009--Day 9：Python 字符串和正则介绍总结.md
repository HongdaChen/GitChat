字符串无所不在，字符串的处理也是最常见的操作。

今天，一起学习字符串处理相关的操作。主要包括：

  * 基本的字符串操作
  * 高级字符串操作之正则部分

### 基本操作

#### **反转字符串**

    
    
    In [1]: s = "python"
    
    # 方法1    
    In [6]: rs = ''.join(reversed(s))
    
    In [7]: rs
    Out[7]: 'nohtyp'
    
    
    
    #方法2
    In [5]: s[::-1]
    Out[5]: 'nohtyp'
    

#### **字符串切片操作**

生成 1 到 15 的序列，并在满足条件的索引处，替换为 `java` 或 `python`。

    
    
    In [15]: java,python = "java", "python"
    
    In [16]: jl,pl=len(java),len(python)
    
    # 索引 2、5、8 处替换为 Java 字符
    # 索引 4 处替换为 Python 字符
    
    In [17]: [str(java[i%3*jl:] + python[i%5*pl:] or i) for i in range(1,10)]
    

输出结果，如下图：

![image-20200219223256071](https://images.gitbook.cn/0c756540-53f3-11ea-
ab27-11d05933a5c6)

#### **join 串联字符串**

用下划线 `_` 连接字符串 mystr：

    
    
    n [8]: mystr = ['I','love','Python']
    
    In [9]: res = '_'.join(mystr)
    
    In [10]: res
    Out[10]: 'I_love_Python'
    

![image-20200219223504920](https://images.gitbook.cn/3a291270-53f3-11ea-97af-0fc725ffa08c)

#### **分割字符串**

根据指定字符或字符串，分割一个字符串时，使用方法 split。

join 和 split 可看做一对互逆操作：

    
    
    In [23]: 'I_love_python'.split('_')
    Out[23]: ['I', 'love', 'python']
    

#### **替换**

字符串替换，使用 replace 方法。

如下字符串，小写的 o 全部替换为大写的 O：

    
    
    In [14]: s = 'i love python'.replace('o', 'O')
    
    In [15]: s
    Out[15]: 'i lOve pythOn'
    

#### **子串判断**

判断 a 串是否为 b 串的子串。

方法 1，使用 in：

    
    
    In [16]: a = 'our'
    In [17]: b = 'flourish'
    
    In [18]: r = True if a in b else False
    
    In [19]: r
    Out[19]: True
    

方法 2，使用方法 find，返回字符串 b 中匹配子串 a 的最小索引：

    
    
    In [16]: a = 'our'
    In [17]: b = 'flourish'
    
    In [21]: b.find(a)
    Out[21]: 2
    

#### **去空格**

清洗字符串时，位于字符串开始和结尾的空格，有时需要去掉，strip 方法能实现。

如下字符串，使用 strip，清理字符串开头和结尾的空格和制表符。

    
    
    In [24]: a = '  \tI love python  \b\n'
    
    In [25]: a
    Out[25]: '  \tI love python  \x08\n'
    
    In [26]: a.strip()
    Out[26]: 'I love python  \x08'
    

#### **字符串的字节长度**

encode 方法对字符串编码后：

    
    
    In [12]: def str_byte_len(mystr):
        ...:     mystr_bytes = mystr.encode('utf-8')
        ...:     return (len(mystr_bytes))
    
    
    In [13]: str_byte_len('i love python') 
    Out[13]: 13
    

![image-20200219232221426](https://images.gitbook.cn/56c34ea0-53f3-11ea-97af-0fc725ffa08c)

### 正则表达式

字符串封装的方法，处理一般的字符串操作，还能应付。但是，稍微复杂点的字符串处理任务，需要靠正则表达式，简洁且强大。

首先，导入所需要的模块 re：

    
    
    import re
    

首先，认识常用的元字符：

  * `.` 匹配除 "\n" 和 "\r" 之外的任何单个字符。 
  * `^` 匹配字符串开始位置 
  * `$` 匹配字符串中结束的位置 
  * `*` 前面的原子重复 0 次、1 次、多次 
  * `?` 前面的原子重复 0 次或者 1 次 
  * `+` 前面的原子重复 1 次或多次
  * `{n}` 前面的原子出现了 n 次
  * `{n,}` 前面的原子至少出现 n 次
  * `{n,m}` 前面的原子出现次数介于 n-m 之间
  * `( )` 分组，输出需要的部分

再认识常用的通用字符：

  * `\s` 匹配空白字符 
  * `\w` 匹配任意字母/数字/下划线 
  * `\W` 和小写 w 相反，匹配任意字母/数字/下划线以外的字符
  * `\d` 匹配十进制数字
  * `\D` 匹配除了十进制数以外的值 
  * `[0-9]` 匹配一个 0~9 之间的数字
  * `[a-z]` 匹配小写英文字母
  * `[A-Z]` 匹配大写英文字母

正则表达式，常会涉及到以上这些元字符或通用字符，下面通过 14 个细分的与正则相关的小功能，讨论正则表达式。

#### **search 第一个匹配串**

使用正则模块，search 方法，找出子串第一个匹配位置。

    
    
    In [31]: s = 'i love python very much'
    
    In [32]: pat = 'python'
    
    In [33]: r = re.search(pat,s)
    
    In [34]: r.span()
    Out[34]: (7, 13)
    

#### **match 与 search 不同**

正则模块中，match、search 方法匹配字符串不同

具体不同：

  * match 在原字符串的开始位置匹配
  * search 在字符串的任意位置匹配

原字符串：

    
    
    In [105]: s = 'flourish'
    

寻找模式串 our，使用 match 方法：

    
    
    In [106]: recom = re.compile('our')
    
    In [107]: recom.match(s) # 返回 None，找不到匹配
    

使用 search 方法：

    
    
    In [109]: res = recom.search(s)
    
    In [110]: res.span()
    Out[110]: (2, 5) # OK, 匹配成功，our 在原字符串的起始索引为 2
    

那么，什么字符串才能使用 match 方法匹配到 our？

比如，字符串 ourselves，ours 才能 match 到 our。

#### **finditer 匹配迭代器**

使用正则模块，finditer 方法，返回所有子串匹配位置的迭代器。

通过返回的对象 re.Match，使用它的方法 span 找出匹配位置。

    
    
    In [35]: s = '山东省潍坊市青州第1中学高三1班'
    
    In [36]: pat = '1'
    
    In [37]: r = re.finditer(pat,s)
    
    In [38]: for i in r:
        ...:     print(i)
    
    <re.Match object; span=(9, 10), match='1'>
    <re.Match object; span=(14, 15), match='1'>
    

#### **findall 所有匹配**

正则模块，findall 方法能查找出子串的所有匹配。

原字符串 s：

    
    
    In [39]: s = '一共20行代码运行时间13.59s'
    

目标查找出所有所有数字：通用字符 `\d` 匹配一位数字 [0-9]，`+` 表示匹配数字前面的一个字符 1 次或多次。

    
    
    In [40]: pat = r'\d+'
    
    In [41]: r = re.findall(pat,s)
    
    In [42]: print(r)
    ['20', '13', '59']
    

返回一个列表，找到三个数字 20、13、59，没有达到预期，期望找到 20、13.59。

因此，需要修改正则表达式。

#### **案例：匹配浮点数和整数**

  * `?` 表示前一个字符匹配 0 或 1 次
  * `.?` 表示匹配小数点（`.`）0 次或 1 次。

匹配浮点数和整数，第一版正则表达式：`r'\d+\.?\d+'`，图形化演示，此正则表达式的分解演示：

![image-20200220115900387](https://images.gitbook.cn/862a7830-53f3-11ea-
bb62-297df070c489)

    
    
    In [43]: s = '一共20行代码运行时间13.59s'
    
    In [44]: pat = r'\d+\.?\d+'
    
    In [45]: r = re.findall(pat,s)
    
    In [46]: r
    Out[46]: ['20', '13.59']
    

上面的正则表达式 `r'\d+\.?\d+'`，能适配所有情况吗？

    
    
    In [47]: s = '一共2行代码运行时间1.66s'
    
    In [48]: pat = r'\d+\.?\d+'
    
    In [49]: r = re.findall(pat,s)
    
    In [50]: r
    Out[50]: ['1.66']
    

观察结果，没有匹配到数字 2。

正则难点之一，需要考虑全面、足够细心，才可能写出准确无误的正则表达式。

出现问题原因：`r'\d+\.?\d+'`，后面的 `\d+` 表示至少有一位数字，因此，整个表达式至少会匹配两位数。

修复问题，重新优化正则表达式，将最后的 `+` 后修改为 `*`，表示匹配前面字符 0 次、1 次或多次。

最终正则表达式为：`r'\d+\.?\d*'`，正则分解图：

![image-20200220121007756](https://images.gitbook.cn/a18815b0-53f3-11ea-
ab27-11d05933a5c6)

    
    
    In [51]: pat = r'\d+\.?\d*'
    
    In [52]: r = re.findall(pat,s)
    
    In [53]: r
    Out[53]: ['2', '1.66']
    

#### **案例：匹配正整数**

案例：写出匹配所有 **正整数** 的正则表达式。

如果这样写：`^\d*$`，会匹配到 0，所以不准确。

如果这样写：`^[1-9]*`，会匹配 `1.` 串中 `1`，不是完全匹配，体会 `$` 的作用。

正确写法：`^[1-9]\d*$`，正则分解图：

![image-20200220122531707](https://images.gitbook.cn/b608a680-53f3-11ea-b360-6f8d231378f8)

    
    
    In [62]: s = [-16, 1.5, 11.43, 10, 5]
    In [63]: pat = r'^[1-9]\d*$'
    In [70]: [i for i in s if re.match(pat,str(i))]
    Out[70]: [10, 5]
    

#### **re.I 忽略大小写**

re.I 是方法的可选参数，表示忽略大小写。

如下，找出字符串中所有字符 t 或 T 的位置，不区分大小写。

    
    
    In [71]: s = 'That'
    
    In [72]: pat = r't'
    
    In [73]: r = re.finditer(pat,s,re.I)
    
    In [79]: for i in r:
        ...:     print(i.span())
        ...:
    (0, 1)
    (3, 4)
    

#### **split 分割单词**

正则模块中 split 函数强大，能够处理复杂的字符串分割任务。

如果一个规则简单的字符串，直接使用字符串，split 函数。

如下字符串，根据分割符 `\t` 分割：

    
    
    In [91]: s = 'id\tname\taddress'
    
    In [92]: s.split('\t')
    Out[92]: ['id', 'name', 'address']
    

但是，对于分隔符复杂的字符串，split 函数就无能为力。

如下字符串，可能的分隔符有 `,`、`;`、`\t`、`|`。

    
    
    In [100]: s = 'This,,,   module ; \t   provides|| regular ; '
    

正则字符串为：`[,\s;|]+`，`\s` 匹配空白字符，正则分解图，如下：

![image-20200220201747943](https://images.gitbook.cn/cd6925c0-53f3-11ea-
ab27-11d05933a5c6)

    
    
    In [102]: words = re.split('[,\s;|]+',s)
    
    In [103]: words
    Out[103]: ['This', 'module', 'provides', 'regular', '']
    

#### **sub 替换匹配串**

正则模块，sub 方法，替换匹配到的子串：

    
    
    In [113]: content="hello 12345, hello 456321"
    
    In [114]: pat=re.compile(r'\d+') #要替换的部分
    
    In [115]: m=pat.sub("666",content)
    
    In [116]: m
    Out[116]: 'hello 666, hello 666'
    

#### **compile 预编译**

如果要用同一匹配模式，做很多次匹配，可以使用 compile 预先编译串。

案例：从一系列字符串中，挑选出所有正浮点数。

正则表达式为：`^[1-9]\d*\.\d*|0\.\d*[1-9]\d*$`，字符 `a|b` 表示 `a` 串匹配失败后，才执行 `b`
串，正则分解图见下：

![image-20200220123743586](https://images.gitbook.cn/e5bb30f0-53f3-11ea-
abf1-59cee134db3d)

首先，生成预编译对象 rec：

    
    
    In [80]: s = [-16,'good',1.5, 0.2, -0.1, '11.43', 10, '5e10']
    
    In [81]: rec = re.compile(r'^[1-9]\d*\.\d*|0\.\d*[1-9]\d*$')
    

下面直接使用 rec，匹配列表中的每个元素，不用每次都预编译正则表达式，效率更高。

    
    
    In [84]: [ i for i in s if rec.match(str(i))]
    Out[84]: [1.5, 0.2, '11.43']
    

#### **贪心捕获**

正则模块中，根据某个模式串，匹配到结果。

待爬取网页的部分内容如下所示，现在想要提取 `<div>` 标签中的内容。

    
    
    In [125]: content = '''
         ...:     <h>ddedadsad</h>
         ...:     <div>graph</div>
         ...:    <div>math</div>'''
    

如果正则匹配串写做 `<div>.*</div>`：

    
    
    In [118]: result = re.findall(r'<div>.*</div>',content)
    
    In [119]: result
    Out[119]: ['<div>graph</div>bb<div>math</div>']
    

返回的结果：

    
    
    '<div>graph</div>bb<div>math</div>' 
    

如果我们不想保留字符串最开始的 `<div>` 和结尾的 `</div>`，那么，就需要使用一对 `()` 去捕获。

正则匹配串修改为：`<div>(.*)</div>`，只添加一对括号。

    
    
    In [120]: result = re.findall(r'<div>(.*)</div>',content)
    
    In [121]: result
    Out[121]: ['graph</div>bb<div>math']
    

  * 看到结果中已经没有开始的 `<div>`，结尾的 `</div>` 仅使用一对括号，便成功捕获到我们想要的部分。
  * `(.*)` 表示捕获任意多个字符，尽可能多地匹配字符，也被称为`贪心捕获`
  * `(.*)` 的正则分解图如下所示，`.` 表示匹配除换行符外的任意字符。 

![image-20200220213002588](https://images.gitbook.cn/fc5b6d70-53f3-11ea-
ab27-11d05933a5c6)

#### 非贪心捕获

观察上面返回的结果 `['graph</div>bb<div>math']`，如果只想要得到两个 `<div></div>` 间的内容，该怎么写正则表达式？

相比 `(.*)`，仅多添加一个 `?`，匹配串为 `(.*?)`。

    
    
    In [125]: content = '''
         ...:     <h>ddedadsad</h>
         ...:     <div>graph</div>
         ...:    <div>math</div>'''
    
    In [126]: result = re.findall(r'<div>(.*?)</div>',content)
    
    In [127]: result
    Out[127]: ['graph', 'math']
    

终于得到 2 个 `<div>` 对间的内容。

这种匹配模式串 `(.*?)`，被称为非贪心捕获。正则图中，红色虚线表示非贪心匹配。

![image-20200220214643717](https://images.gitbook.cn/0f1484b0-53f4-11ea-
abf1-59cee134db3d)

通过上述例子和分析，体会贪心和非贪心匹配的不同之处。

### 小结

今天学习：

  * 8 个字符串相关功能，如 join 连接串、split 分割串、replace 做简单替换。
  * 正则表达式中常见的元字符和通用字符。
  * 10 个正则表达式核心要点。
  * 2 个正则案例。

