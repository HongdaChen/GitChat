上一篇我们学习了 BERT 的原理，这里我们将它应用在 NER 实战上，我们在前面课程中也学习过 NER，这里我们用同样的数据集来看如何应用
BERT，大家可以与前面的模型进行比较。

另外推荐大家使用 Google 的云端 Jupyter Notebook 开发环境
[Colab](https://colab.research.google.com/)，自带
TensorFlow、Matplotlib、NumPy、Pandas 等深度学习基础库，还可以免费使用 GPU、TPU。

![](https://images.gitbook.cn/8786b2d0-980d-11ea-8a85-89626584851d)

![](https://images.gitbook.cn/95a22610-980d-11ea-83bd-cd494846175a)

下面进入实战，本文所用的数据仍然是 [Kaggle 上的一个 NER
数据集](https://www.kaggle.com/abhinavwalia95/entity-annotated-
corpus#ner_dataset.csv)，可以将其中的ner_dataset.csv 下载到本地。

### 加载所需的库

首先我们需要安装 transformers，里面包括 NLP 中比较前沿的模型：BERT、GPT、XLNet 等等，支持 TensorFlow 和
PyTorch。

    
    
    !pip install transformers
    !pip install seqeval
    
    import pandas as pd
    import numpy as np
    from tqdm import tqdm, trange
    
    import torch
    from torch.utils.data import TensorDataset, DataLoader, RandomSampler, SequentialSampler
    from transformers import BertTokenizer, BertConfig
    
    from keras.preprocessing.sequence import pad_sequences
    from sklearn.model_selection import train_test_split
    
    torch.__version__
    

### 处理数据

**首先看一下数据长什么样子** ，ner_dataset.csv 这个数据中有 4 列，分别是句子序号，句子中的单词，单词的 POS 标签和 NER
标签，一共有 47959 个句子，包括 35178 个不同的单词，和 17 种 NER 标签。

    
    
    data = pd.read_csv("ner_dataset.csv", encoding="latin1").fillna(method="ffill")
    data.head(10)
    

![](https://images.gitbook.cn/28ce9810-9813-11ea-a037-85f3d67d7d8f)

    
    
    data['Sentence #'].nunique(), data.Word.nunique(), data.POS.nunique(), data.Tag.nunique()
    
    
    
    (47959, 35178, 42, 17)
    

17 种 NER 标签如下所示，包括人名、地名、机构名、专有名词等，其中 B 的意思是 beginning，比如一个人名的第一个字母，I 的意思是
inside，如一个人名中的其他字母，O 为非实体：

    
    
    data.Tag.value_counts()
    

![](https://images.gitbook.cn/3f8d3200-9813-11ea-b14e-fbca2675b1bb)

然后用 SentenceGetter 将数据构造成“单词-POS标签-NER标签”这样的格式：

    
    
    class SentenceGetter(object):
    
        def __init__(self, data):
            self.n_sent = 1
            self.data = data
            self.empty = False
            agg_func = lambda s: [(w, p, t) for w, p, t in zip(s["Word"].values.tolist(),
                                                               s["POS"].values.tolist(),
                                                               s["Tag"].values.tolist())]
            self.grouped = self.data.groupby("Sentence #").apply(agg_func)
            self.sentences = [s for s in self.grouped]
    
        def get_next(self):
            try:
                s = self.grouped["Sentence: {}".format(self.n_sent)]
                self.n_sent += 1
                return s
            except:
                return None
    
    getter = SentenceGetter(data)
    

我们可以看一下生成数据的结果：

    
    
    sent = getter.get_next()
    sentences = getter.sentences
    print(sent)
    

原来的 data 就变成了下面这种形式：

    
    
    [('Thousands', 'NNS', 'O'), ('of', 'IN', 'O'), ('demonstrators', 'NNS', 'O'), ('have', 'VBP', 'O'), ('marched', 'VBN', 'O'), ('through', 'IN', 'O'), ('London', 'NNP', 'B-geo'), ('to', 'TO', 'O'), ('protest', 'VB', 'O'), ('the', 'DT', 'O'), ('war', 'NN', 'O'), ('in', 'IN', 'O'), ('Iraq', 'NNP', 'B-geo'), ('and', 'CC', 'O'), ('demand', 'VB', 'O'), ('the', 'DT', 'O'), ('withdrawal', 'NN', 'O'), ('of', 'IN', 'O'), ('British', 'JJ', 'B-gpe'), ('troops', 'NNS', 'O'), ('from', 'IN', 'O'), ('that', 'DT', 'O'), ('country', 'NN', 'O'), ('.', '.', 'O')]
    

**后面的模型需要的输入数据为 sentences 和 labels** ，我们可以分别拿出第一个数据看看它们的样子：

    
    
    sentences = [[word[0] for word in sentence] for sentence in getter.sentences]
    sentences[0]
    
    
    
    ['Thousands',
     'of',
     'demonstrators',
     'have',
     'marched',
     'through',
     'London',
     'to',
     'protest',
     'the',
     'war',
     'in',
     'Iraq',
     'and',
     'demand',
     'the',
     'withdrawal',
     'of',
     'British',
     'troops',
     'from',
     'that',
     'country',
     '.']
    

labels[0] 就是 sentences[0] 相应的NER标签序列：

    
    
    labels = [[s[2] for s in sentence] for sentence in getter.sentences]
    print(labels[0])
    
    
    
    ['O', 'O', 'O', 'O', 'O', 'O', 'B-geo', 'O', 'O', 'O', 'O', 'O', 'B-geo', 'O', 'O', 'O', 'O', 'O', 'B-gpe', 'O', 'O', 'O', 'O', 'O']
    

建立 tag2idx 字典用来将 label 转化为数字序列：

    
    
    tag_values = list(set(data["Tag"].values))
    tag_values.append("PAD")
    tag2idx = {t: i for i, t in enumerate(tag_values)}
    

这里最长的序列长度为 75 个标记，在论文中 BERT 支持最大长度为 512 个标记的序列，批次大小和论文一样的设置为 32：

    
    
    MAX_LEN = 75
    bs = 32
    

我们在 colab 中设置了 GPU，可以用 get_device_name 查看：

    
    
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
    n_gpu = torch.cuda.device_count()
    
    torch.cuda.get_device_name(0)
    # 'Tesla P100-PCIE-16GB'
    

**我们要使用 transformers 中的预训练标记生成器 BertTokenizer 和定义好的词汇表**
，可以很方便地将文本转化为数字序列，预训练模型我们使用最小的 bert-base-cased 就可以比较好地处理当前这个任务了：

    
    
    tokenizer = BertTokenizer.from_pretrained('bert-base-cased', do_lower_case=False)
    

其中 BertTokenizer 是一个专门为 BERT 使用的类，它用贪心法创建了一个固定大小的词汇表，包括所有英语字符的词汇表和模型所训练的
30,000 个最常见的英文单词和子单词。我们会用到 tokenize() 函数，用来对文本进行标记化，还会用到 encode()
函数，对文本进行标记化并转化成相应的 id。

**下面要做的是将文本数据标记化。**

还是拿第一个句子为例，文本为：

    
    
    ['Thousands', 'of', 'demonstrators', 'have', 'marched', 'through', 'London', 'to', 'protest', 'the', 'war', 'in', 'Iraq', 'and', 'demand', 'the', 'withdrawal', 'of', 'British', 'troops', 'from', 'that', 'country', '.'] 
    

相应的标签为：

    
    
    ['O', 'O', 'O', 'O', 'O', 'O', 'O', 'O', 'B-geo', 'O', 'O', 'O', 'O', 'O', 'B-geo', 'O', 'O', 'O', 'O', 'O', 'B-gpe', 'O', 'O', 'O', 'O', 'O']
    

tokenize_and_preserve_labels 要做的是将句子中的每个单词做成标记，BERT 所使用的标记生成器是基于 Wordpiece
的，使用时标记器首先检查整个单词是否在词汇表中，如果没有就将单词分解为词汇表中包含的尽可能大的子单词，最后才将单词分解为单个字符，例如其中
demonstrators 这个单词会被分成 `['demons', '##tra', '##tors']` 三部分，即分成子单词级别的。

分解后同时要在标签上也做相应的调整，demonstrators 的 NER 标签为 O，因为分成了三个子单词，所以标签序列中这里也变成了 3 个 O。

    
    
    def tokenize_and_preserve_labels(sentence, text_labels):
        tokenized_sentence = []
        labels = []
    
        for word, label in zip(sentence, text_labels):
    
            # 给单词做标记解析，计算分解出的子单词数量
            tokenized_word = tokenizer.tokenize(word)
            n_subwords = len(tokenized_word)
    
            # 将单词标记添加到整个标记词列表中
            tokenized_sentence.extend(tokenized_word)
    
            # 将标签加到整个标签列表中
            labels.extend([label] * n_subwords)
    
        return tokenized_sentence, labels
    
    
    
    tokenized_texts_and_labels = [
        tokenize_and_preserve_labels(sent, labs)
        for sent, labs in zip(sentences, labels)
    ]
    
    tokenized_texts = [token_label_pair[0] for token_label_pair in tokenized_texts_and_labels]
    labels = [token_label_pair[1] for token_label_pair in tokenized_texts_and_labels]
    

最后得到的第一个句子的 tokenized_texts 和 labels 就是这样：

    
    
    ['Thousands', 'of', 'demons', '##tra', '##tors', 'have', 'marched', 'through', 'London', 'to', 'protest', 'the', 'war', 'in', 'Iraq', 'and', 'demand', 'the', 'withdrawal', 'of', 'British', 'troops', 'from', 'that', 'country', '.']
    
    ['O', 'O', 'O', 'O', 'O', 'O', 'O', 'O', 'O', 'O', 'B-geo', 'O', 'O', 'O', 'O', 'O', 'B-geo', 'O', 'O', 'O', 'O', 'O', 'B-gpe', 'O', 'O', 'O']
    

**然后需要将标记序列和标签序列剪切或填充到指定的长度。**

这一步是 token embedding，将单词标记映射到 id：

    
    
    input_ids = pad_sequences([tokenizer.convert_tokens_to_ids(txt) for txt in tokenized_texts],
                              maxlen=MAX_LEN, dtype="long", value=0.0,
                              truncating="post", padding="post")
    
    print(input_ids[0])
    
    
    
    [26159  1104  8568  4487  5067  1138  9639  1194  1498  1106  5641  1103
      1594  1107  5008  1105  4555  1103 10602  1104  1418  2830  1121  1115
      1583   119     0     0     0     0     0     0     0     0     0     0
         0     0     0     0     0     0     0     0     0     0     0     0
         0     0     0     0     0     0     0     0     0     0     0     0
         0     0     0     0     0     0     0     0     0     0     0     0
         0     0     0]
    
    
    
    tag2idx
    
    
    
    {'B-art': 10,
     'B-eve': 1,
     'B-geo': 0,
     'B-gpe': 2,
     'B-nat': 3,
     'B-org': 13,
     'B-per': 15,
     'B-tim': 14,
     'I-art': 9,
     'I-eve': 7,
     'I-geo': 16,
     'I-gpe': 5,
     'I-nat': 4,
     'I-org': 8,
     'I-per': 6,
     'I-tim': 12,
     'O': 11,
     'PAD': 17}
    
    
    
    tags = pad_sequences([[tag2idx.get(l) for l in lab] for lab in labels],
                         maxlen=MAX_LEN, value=tag2idx["PAD"], padding="post",
                         dtype="long", truncating="post")
    
    print(tags[0])
    
    
    
    [11 11 11 11 11 11 11 11  0 11 11 11 11 11  0 11 11 11 11 11  2 11 11 11
     11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11
     11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11
     11 11 11]
    

然后做 **Mask word embedding 用来掩盖序列中的填充数据** 。用 1 表示真正的标记，用 0 表示填充的标记，就是
input_ids[0] 中有单词的位置为 1，后面填充的位置为 0：

    
    
    attention_masks = [[float(i != 0.0) for i in ii] for ii in input_ids]
    attention_masks[0]
    

选择数据集中的 10% 来验证模型：

    
    
    tr_inputs, val_inputs, tr_tags, val_tags = train_test_split(input_ids, tags,
                                                                random_state=2018, test_size=0.1)
    tr_masks, val_masks, _, _ = train_test_split(attention_masks, input_ids,
                                                 random_state=2018, test_size=0.1)
    

将 inputs、tags、masks 数据转化成 torch 张量：

    
    
    tr_inputs = torch.tensor(tr_inputs)
    val_inputs = torch.tensor(val_inputs)
    tr_tags = torch.tensor(tr_tags)
    val_tags = torch.tensor(val_tags)
    tr_masks = torch.tensor(tr_masks)
    val_masks = torch.tensor(val_masks)
    

定义 dataloader，在模型训练时使用 RandomSampler 对数据进行混洗，在测试模型时，使用 SequentialSampler
按顺序传递数据：

    
    
    train_data = TensorDataset(tr_inputs, tr_masks, tr_tags)
    train_sampler = RandomSampler(train_data)
    train_dataloader = DataLoader(train_data, sampler=train_sampler, batch_size=bs)
    
    valid_data = TensorDataset(val_inputs, val_masks, val_tags)
    valid_sampler = SequentialSampler(valid_data)
    valid_dataloader = DataLoader(valid_data, sampler=valid_sampler, batch_size=bs)
    

### 建立模型

我们要用 transformer 中的 BertForTokenClassification 来做标记级别的预测，它也是一个预训练模型，里面包括了 BERT
模型，并在 BERT 的顶层加了标记级别的分类器：

    
    
    import transformers
    from transformers import BertForTokenClassification, AdamW
    
    transformers.__version__
    
    
    model = BertForTokenClassification.from_pretrained(
        "bert-base-cased",
        num_labels=len(tag2idx),
        output_attentions = False,
        output_hidden_states = False
    )
    

这里我们简单介绍一下 transformer 中 BERT
相关的模型，详细的大家感兴趣可以去看[文档](https://huggingface.co/transformers/v2.1.1/model_doc/bert.html#bertconfig)：

  1. BertModel：实现了基本 BERT 模型，包括 embeddings、encoder 和 pooler。
  2. BertPooler：Pool 的作用是输出的时候用一个全连接层将整个句子的信息用第一个 token 来表示。
  3. BertForSequenceClassification：一般用来进行文本分类任务。
  4. BertForTokenClassification：token 级别的分类模型，一般用来进行序列标注任务。
  5. BertForQuestionAnswering：用来做问答任务。

接着需要将模型参数传递给 GPU：

    
    
    model.cuda();
    

开始微调之前， **设置优化器还有要更新的参数** ，其中正则化中加入
weight_decay，即在梯度更新时需要增加一项，目的是想使模型权重接近于0，这样做可以让优化器 AdamW 的效果更好：

    
    
    FULL_FINETUNING = True
    if FULL_FINETUNING:
        param_optimizer = list(model.named_parameters())
        no_decay = ['bias', 'gamma', 'beta']
        optimizer_grouped_parameters = [
            {'params': [p for n, p in param_optimizer if not any(nd in n for nd in no_decay)],
             'weight_decay_rate': 0.01},
            {'params': [p for n, p in param_optimizer if any(nd in n for nd in no_decay)],
             'weight_decay_rate': 0.0}
        ]
    else:
        param_optimizer = list(model.classifier.named_parameters())
        optimizer_grouped_parameters = [{"params": [p for n, p in param_optimizer]}]
    
    optimizer = AdamW(
        optimizer_grouped_parameters,
        lr=3e-5,
        eps=1e-8
    )
    

训练过程中也给学习率加了一个线性下降的策略，让学习率随着训练过程逐渐地减少：

    
    
    from transformers import get_linear_schedule_with_warmup
    
    epochs = 3
    max_grad_norm = 1.0
    
    # 训练步数 = batches * epochs
    total_steps = len(train_dataloader) * epochs
    
    # 学习率策略
    scheduler = get_linear_schedule_with_warmup(
        optimizer,
        num_warmup_steps=0,
        num_training_steps=total_steps
    )
    

训练过程中的评估指标使用 f1_score，标记的预测使用准确率来评估：

    
    
    from seqeval.metrics import f1_score
    
    def flat_accuracy(preds, labels):
        pred_flat = np.argmax(preds, axis=2).flatten()
        labels_flat = labels.flatten()
        return np.sum(pred_flat == labels_flat) / len(labels_flat)
    
    
    ## 每个 epoch 存储一下平均损失用来绘图
    loss_values, validation_loss_values = [], []
    

### 模型训练

开始训练模型，epoch 我们只设置为 3，训练过程中计算损失和验证损失：

    
    
    for _ in trange(epochs, desc="Epoch"):
    
        # 1. 训练模型：
    
        # 模型的训练模式
        model.train()
    
        total_loss = 0
    
        for step, batch in enumerate(train_dataloader):
            # 将批次数据添加到GPU
            batch = tuple(t.to(device) for t in batch)
            b_input_ids, b_input_mask, b_labels = batch
    
            # 执行反向传播之前将之前计算的梯度清零
            model.zero_grad()
    
            # 前向传播
            outputs = model(b_input_ids, token_type_ids=None,
                            attention_mask=b_input_mask, labels=b_labels)
    
            loss = outputs[0]
            # 反向传播，计算梯度
            loss.backward()
    
            # 记录训练损失
            total_loss += loss.item()
    
            # 做梯度剪切用来应对梯度爆炸问题
            torch.nn.utils.clip_grad_norm_(parameters=model.parameters(), max_norm=max_grad_norm)
    
            # 更新参数
            optimizer.step()
            # 更新学习率
            scheduler.step()
    
        # 计算平均损失
        avg_train_loss = total_loss / len(train_dataloader)
        print("Average train loss: {}".format(avg_train_loss))
    
        # 这个数据用来画学习曲线
        loss_values.append(avg_train_loss)
    
    
        # 2. 验证模型：
    
        # 每轮训练后在验证集衡量模型表现
    
        # 模型的验证模式
        model.eval()
    
        eval_loss, eval_accuracy = 0, 0
        nb_eval_steps, nb_eval_examples = 0, 0
        predictions , true_labels = [], []
        for batch in valid_dataloader:
            batch = tuple(t.to(device) for t in batch)
            b_input_ids, b_input_mask, b_labels = batch
    
            with torch.no_grad():
                # 前向计算
                # 因为没有标签所以返回 logits 而不是损失
                outputs = model(b_input_ids, token_type_ids=None,
                                attention_mask=b_input_mask, labels=b_labels)
            # 将 logits 和标签移到 CPU
            logits = outputs[1].detach().cpu().numpy()
            label_ids = b_labels.to('cpu').numpy()
    
            # 计算这批测试句子的准确度
            eval_loss += outputs[0].mean().item()
            eval_accuracy += flat_accuracy(logits, label_ids)
            predictions.extend([list(p) for p in np.argmax(logits, axis=2)])
            true_labels.extend(label_ids)
    
            nb_eval_examples += b_input_ids.size(0)
            nb_eval_steps += 1
    
        eval_loss = eval_loss / nb_eval_steps
        validation_loss_values.append(eval_loss)
        print("Validation loss: {}".format(eval_loss))
        print("Validation Accuracy: {}".format(eval_accuracy/nb_eval_steps))
        pred_tags = [tag_values[p_i] for p, l in zip(predictions, true_labels)
                                     for p_i, l_i in zip(p, l) if tag_values[l_i] != "PAD"]
        valid_tags = [tag_values[l_i] for l in true_labels
                                      for l_i in l if tag_values[l_i] != "PAD"]
        print("Validation F1-Score: {}".format(f1_score(pred_tags, valid_tags)))
        print()
    

![](https://images.gitbook.cn/27b45900-9814-11ea-bcac-1f458c1b219e)

**画出训练损失和验证损失的学习曲线** ，这个例子中 epoch = 1 是比较好的：

    
    
    import matplotlib.pyplot as plt
    %matplotlib inline
    
    import seaborn as sns
    
    sns.set(style='darkgrid')
    
    sns.set(font_scale=1.5)
    plt.rcParams["figure.figsize"] = (12,6)
    
    # 画学习曲线
    plt.plot(loss_values, 'b-o', label="training loss")
    plt.plot(validation_loss_values, 'r-o', label="validation loss")
    
    plt.title("Learning curve")
    plt.xlabel("Epoch")
    plt.ylabel("Loss")
    plt.legend()
    
    plt.show()
    

![](https://images.gitbook.cn/393da2d0-9814-11ea-b0a6-ebd9ebfac77b)

### 模型预测

训练好的模型可以应用于全新的句子用来预测，例如我们有下面这个句子：

    
    
    test_sentence = """
    Mr. Trump’s tweets began just moments after a Fox News report by Mike Tobin, a 
    reporter for the network, about protests in Minnesota and elsewhere. 
    """
    

首先将文本数据进行标记解析，然后转化成 torch 张量：

    
    
    tokenized_sentence = tokenizer.encode(test_sentence)
    input_ids = torch.tensor([tokenized_sentence]).cuda()
    

将这个句子输入给模型，得到 output，选择最大值作为每个单词的预测标签结果，然后将 token 和 label 由数字序列转化成实际的单词和标签：

    
    
    with torch.no_grad():
        output = model(input_ids)
    
    label_indices = np.argmax(output[0].to('cpu').numpy(), axis=2)
    
    
    tokens = tokenizer.convert_ids_to_tokens(input_ids.to('cpu').numpy()[0])
    
    new_tokens, new_labels = [], []
    for token, label_idx in zip(tokens, label_indices[0]):
        if token.startswith("##"):
            new_tokens[-1] = new_tokens[-1] + token[2:]
        else:
            new_labels.append(tag_values[label_idx])
            new_tokens.append(token)
    
    for token, label in zip(new_tokens, new_labels):
        print("{}\t{}".format(label, token))
    

从下面的结果可以看出，模型成功地预测出了人名，机构名，地名，包括 B 和 I 的位置也是正确的：

![](https://images.gitbook.cn/53eed3b0-9814-11ea-bcac-1f458c1b219e)

* * *

我们通过这篇文章知道了如何使用微调 BERT 模型，再加上前面 Transformer 和 BERT 的理论学习，大家可以思考下面两个面试题：

  * Transformer 在 BERT 中的应用？
  * Transformer 和 BERT 的区别是什么？

