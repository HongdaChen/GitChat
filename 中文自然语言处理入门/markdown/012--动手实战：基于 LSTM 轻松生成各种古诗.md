目前循环神经网络（RNN）已经广泛用于自然语言处理中，可以处理大量的序列数据，可以说是最强大的神经网络模型之一。人们已经给 RNN
找到了越来越多的事情做，比如画画和写诗，微软的小冰都已经出版了一本诗集了。

而其实训练一个能写诗的神经网络并不难，下面我们就介绍如何简单快捷地建立一个会写诗的网络模型。

![enter image description
here](https://images.gitbook.cn/0fdb0170-8db8-11e8-80d1-2d51ff7e1c55)

本次开发环境如下：

  * Python 3.6
  * Keras 环境
  * Jupyter Notebook

整个过程分为以下步骤完成：

  1. 语料准备
  2. 语料预处理
  3. 模型参数配置
  4. 构建模型
  5. 训练模型
  6. 模型作诗
  7. 绘制模型网络结构图

下面一步步来构建和训练一个会写诗的模型。

**第一** ，语料准备。一共四万多首古诗，每行一首诗，标题在预处理的时候已经去掉了。

**第二** ，文件预处理。首先，机器并不懂每个中文汉字代表的是什么，所以要将文字转换为机器能理解的形式，这里我们采用 One-Hot
的形式，这样诗句中的每个字都能用向量来表示，下面定义函数 `preprocess_file()` 来处理。

    
    
        puncs = [']', '[', '（', '）', '{', '}', '：', '《', '》']
        def preprocess_file(Config):
            # 语料文本内容
            files_content = ''
            with open(Config.poetry_file, 'r', encoding='utf-8') as f:
                for line in f:
                    # 每行的末尾加上"]"符号代表一首诗结束
                    for char in puncs:
                        line = line.replace(char, "")
                    files_content += line.strip() + "]"
    
            words = sorted(list(files_content))
            words.remove(']')
            counted_words = {}
            for word in words:
                if word in counted_words:
                    counted_words[word] += 1
                else:
                    counted_words[word] = 1
    
            # 去掉低频的字
            erase = []
            for key in counted_words:
                if counted_words[key] <= 2:
                    erase.append(key)
            for key in erase:
                del counted_words[key]
            del counted_words[']']
            wordPairs = sorted(counted_words.items(), key=lambda x: -x[1])
    
            words, _ = zip(*wordPairs)
            # word到id的映射
            word2num = dict((c, i + 1) for i, c in enumerate(words))
            num2word = dict((i, c) for i, c in enumerate(words))
            word2numF = lambda x: word2num.get(x, 0)
            return word2numF, num2word, words, files_content
    

在每行末尾加上 `]`
符号是为了标识这首诗已经结束了。我们给模型学习的方法是，给定前六个字，生成第七个字，所以在后面生成训练数据的时候，会以6的跨度，1的步长截取文字，生成语料。如果出现了
`]` 符号，说明 `]` 符号之前的语句和之后的语句是两首诗里面的内容，两首诗之间是没有关联关系的，所以我们后面会舍弃掉包含 `]` 符号的训练数据。

**第三** ，模型参数配置。预先定义模型参数和加载语料以及模型保存名称，通过类 Config 实现。

    
    
    class Config(object):
        poetry_file = 'poetry.txt'
        weight_file = 'poetry_model.h5'
        # 根据前六个字预测第七个字
        max_len = 6
        batch_size = 512
        learning_rate = 0.001
    

**第四** ，构建模型，通过 PoetryModel 类实现，类的代码结构如下：

    
    
        class PoetryModel(object):
            def __init__(self, config):
                pass
    
            def build_model(self):
                pass
    
            def sample(self, preds, temperature=1.0):
                pass
    
            def generate_sample_result(self, epoch, logs):
                pass
    
            def predict(self, text):
                pass
    
            def data_generator(self):
                pass
            def train(self):
                pass
    

类中定义的方法具体实现功能如下：

（1）init 函数定义，通过加载 Config 配置信息，进行语料预处理和模型加载，如果模型文件存在则直接加载模型，否则开始训练。

    
    
        def __init__(self, config):
                self.model = None
                self.do_train = True
                self.loaded_model = False
                self.config = config
    
                # 文件预处理
                self.word2numF, self.num2word, self.words, self.files_content = preprocess_file(self.config)
                if os.path.exists(self.config.weight_file):
                    self.model = load_model(self.config.weight_file)
                    self.model.summary()
                else:
                    self.train()
                self.do_train = False
                self.loaded_model = True
    

（2）`build_model` 函数主要用 Keras 来构建网络模型，这里使用 LSTM 的 GRU 来实现，当然直接使用 LSTM 也没问题。

    
    
        def build_model(self):
                '''建立模型'''
                input_tensor = Input(shape=(self.config.max_len,))
                embedd = Embedding(len(self.num2word)+1, 300, input_length=self.config.max_len)(input_tensor)
                lstm = Bidirectional(GRU(128, return_sequences=True))(embedd)
                dropout = Dropout(0.6)(lstm)
                lstm = Bidirectional(GRU(128, return_sequences=True))(embedd)
                dropout = Dropout(0.6)(lstm)
                flatten = Flatten()(lstm)
                dense = Dense(len(self.words), activation='softmax')(flatten)
                self.model = Model(inputs=input_tensor, outputs=dense)
                optimizer = Adam(lr=self.config.learning_rate)
                self.model.compile(loss='categorical_crossentropy', optimizer=optimizer, metrics=['accuracy'])
    

（3）sample 函数，在训练过程的每个 epoch 迭代中采样。

    
    
        def sample(self, preds, temperature=1.0):
            '''
            当temperature=1.0时，模型输出正常
            当temperature=0.5时，模型输出比较open
            当temperature=1.5时，模型输出比较保守
            在训练的过程中可以看到temperature不同，结果也不同
            '''
            preds = np.asarray(preds).astype('float64')
            preds = np.log(preds) / temperature
            exp_preds = np.exp(preds)
            preds = exp_preds / np.sum(exp_preds)
            probas = np.random.multinomial(1, preds, 1)
            return np.argmax(probas)
    

（4）训练过程中，每个 epoch 打印出当前的学习情况。

    
    
        def generate_sample_result(self, epoch, logs):  
                print("\n==================Epoch {}=====================".format(epoch))
                for diversity in [0.5, 1.0, 1.5]:
                    print("------------Diversity {}--------------".format(diversity))
                    start_index = random.randint(0, len(self.files_content) - self.config.max_len - 1)
                    generated = ''
                    sentence = self.files_content[start_index: start_index + self.config.max_len]
                    generated += sentence
                    for i in range(20):
                        x_pred = np.zeros((1, self.config.max_len))
                        for t, char in enumerate(sentence[-6:]):
                            x_pred[0, t] = self.word2numF(char)
    
                        preds = self.model.predict(x_pred, verbose=0)[0]
                        next_index = self.sample(preds, diversity)
                        next_char = self.num2word[next_index]
                        generated += next_char
                        sentence = sentence + next_char
                    print(sentence)
    

（5）predict 函数，用于根据给定的提示，来进行预测。

根据给出的文字，生成诗句，如果给的 text 不到四个字，则随机补全。

    
    
        def predict(self, text):
                if not self.loaded_model:
                    return
                with open(self.config.poetry_file, 'r', encoding='utf-8') as f:
                    file_list = f.readlines()
                random_line = random.choice(file_list)
                # 如果给的text不到四个字，则随机补全
                if not text or len(text) != 4:
                    for _ in range(4 - len(text)):
                        random_str_index = random.randrange(0, len(self.words))
                        text += self.num2word.get(random_str_index) if self.num2word.get(random_str_index) not in [',', '。',
                                                                                                                   '，'] else self.num2word.get(
                            random_str_index + 1)
                seed = random_line[-(self.config.max_len):-1]
                res = ''
                seed = 'c' + seed
                for c in text:
                    seed = seed[1:] + c
                    for j in range(5):
                        x_pred = np.zeros((1, self.config.max_len))
                        for t, char in enumerate(seed):
                            x_pred[0, t] = self.word2numF(char)
                        preds = self.model.predict(x_pred, verbose=0)[0]
                        next_index = self.sample(preds, 1.0)
                        next_char = self.num2word[next_index]
                        seed = seed[1:] + next_char
                    res += seed
                return res
    

（6） `data_generator` 函数，用于生成数据，提供给模型训练时使用。

    
    
         def data_generator(self):
                i = 0
                while 1:
                    x = self.files_content[i: i + self.config.max_len]
                    y = self.files_content[i + self.config.max_len]
                    puncs = [']', '[', '（', '）', '{', '}', '：', '《', '》', ':']
                    if len([i for i in puncs if i in x]) != 0:
                        i += 1
                        continue
                    if len([i for i in puncs if i in y]) != 0:
                        i += 1
                        continue
                    y_vec = np.zeros(
                        shape=(1, len(self.words)),
                        dtype=np.bool
                    )
                    y_vec[0, self.word2numF(y)] = 1.0
                    x_vec = np.zeros(
                        shape=(1, self.config.max_len),
                        dtype=np.int32
                    )
                    for t, char in enumerate(x):
                        x_vec[0, t] = self.word2numF(char)
                    yield x_vec, y_vec
                    i += 1
    

（7）train 函数，用来进行模型训练，其中迭代次数 `number_of_epoch` ，是根据训练语料长度除以 `batch_size`
计算的，如果在调试中，想用更小一点的 `number_of_epoch` ，可以自定义大小，把 train 函数的第一行代码注释即可。

    
    
        def train(self):
                #number_of_epoch = len(self.files_content) // self.config.batch_size
                number_of_epoch = 10
                if not self.model:
                    self.build_model()
                self.model.summary()
                self.model.fit_generator(
                    generator=self.data_generator(),
                    verbose=True,
                    steps_per_epoch=self.config.batch_size,
                    epochs=number_of_epoch,
                    callbacks=[
                        keras.callbacks.ModelCheckpoint(self.config.weight_file, save_weights_only=False),
                        LambdaCallback(on_epoch_end=self.generate_sample_result)
                    ]
                )
    

**第五** ，整个模型构建好以后，接下来进行模型训练。

    
    
        model = PoetryModel(Config)
    

训练过程中的第1-2轮迭代：

![enter image description
here](https://images.gitbook.cn/2a5e35c0-8dbe-11e8-80d1-2d51ff7e1c55)

训练过程中的第9-10轮迭代：

![enter image description
here](https://images.gitbook.cn/30ce9df0-8dbe-11e8-aa21-25f031a4e022)

虽然训练过程写出的诗句不怎么能看得懂，但是可以看到模型从一开始标点符号都不会用 ，到最后写出了有一点点模样的诗句，能看到模型变得越来越聪明了。

**第六** ，模型作诗，模型迭代10次之后的测试，首先输入几个字，模型根据输入的提示，做出诗句。

    
    
        text = input("text:")
        sentence = model.predict(text)
        print(sentence)
    

比如输入：小雨，模型做出的诗句为：

> 输入：text：小雨
>
> 结果：小妃侯里守。雨封即客寥。俘剪舟过槽。傲老槟冬绛。

**第七** ，绘制网络结构图。

模型结构绘图，采用 Keras自带的功能实现：

    
    
        plot_model(model.model, to_file='model.png')
    

得到的模型结构图如下：

![enter image description
here](https://images.gitbook.cn/d431b450-8dbe-11e8-80d1-2d51ff7e1c55)

本节使用 LSTM 的变形 GRU 训练出一个能作诗的模型，当然大家可以替换训练语料为歌词或者小说，让机器人自动创作不同风格的歌曲或者小说。

**参考文献以及推荐阅读：**

  1. [基于 Keras 和 LSTM 的文本生成](https://blog.csdn.net/qiansg123/article/details/80131355)

### [示例数据下载](https://github.com/sujeek/chinese_nlp)

