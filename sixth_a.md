**##CNN_DSSM##**

![](https://ai-studio-static-online.cdn.bcebos.com/2bca30ad06f048a5a8d9f4b475203a2115a317ca6a424fb7b6112fe247b58e76)

**##LSTM_DSSM##**

![](https://ai-studio-static-online.cdn.bcebos.com/862f60009b36460bbe1d8941460641a5d26f6e171524494190e3033d92f6f991)

**##MMoE##**
deep部分：存在多个Expert网络，每个Expert网络的输出最终会经过门网络进行加权平均（比较简单的线性加权，Attention的思想）
门网络通过softmax输出每个专家网络的可能性，线性加权相乘。然后进行分类或者回归任务
对于不同的任务通过相应的Gating Network来对不同的Expert赋予不同的权重，使得部分Expert“专注于各自擅长的任务”
![](https://ai-studio-static-online.cdn.bcebos.com/60ebe3e5bda041d7b4781b4a49601fafe5b9e4c8fdf743549d7303d936b7ba40)

Shallow Tower部分
position bias问题，文章中shallow Tower部分主要作用就是消除Position bias的影响。这部分模型的输入是与position bias相关的一些特征（如广告展现时的排序位置、用户机型等），文章提到了用户机型也算在这部分feature当中，比较直观的认知是不同机型的尺寸的差异可能会对这些position bias有所影响。实际上我们线上的PC搜索模型当中也是采用同样的架构，即Main Tower + Shallow Tower这种类似Wide&Deep的模型结构
![](https://ai-studio-static-online.cdn.bcebos.com/2823bf9a66604d049c70c4df76d9ac38a90924cc965944c5b813d1116ac819a6)


**##ShareBottom##**

Shared-Bottom的思路就是多个目标底层共用一套共享layer，在这之上基于不同的目标构建不同的Tower。这样的好处就是底层的layer复用，减少计算量，同时也可以防止过拟合的情况出现。

![](https://ai-studio-static-online.cdn.bcebos.com/372027f55c194f2b98c294f7a265bd3a12e675c38d2e47058a9c6f399a686787)

最终选用MOE结构的算法还是Shared-Bottom结构的呢？其实取决于业务效果。上面一张图介绍了Shared-Bottom以及OMOE、MMOE在不同目标相关性下的的效果比对。
不难发现，无论目标Correlation是什么数值，MOE结构的算法的loss永远低于Shared-Bottom类型的，显然MOE结构更优。
而OMOE在目标相关性最高的情况下（Correlation=1）和MMOE的效果相似，其它情况下不如MMOE。也就是说，目标相关性越低MMOE较其它二者的优势越明显，相关性非常高的情况下MMOE会近似于OMOE。
另外，解释下相关性Correlation的概念，可以理解为业务正相关性。比如点赞和踩，这两个行为肯定是相关性很低的，如果一个模型既要支持点赞率提升，也支持踩提升，一定要选MMOE。比如收藏和点赞，这两个目标就是相关性非常高的目标。

**##YouTube深度学习视频推荐系统##**

1. YouTube 推荐系统架构
为了对海量的视频进行快速、准确的排序，YouTube 也采用了经典的召回层 + 排序层的推荐系统架构。
![](https://ai-studio-static-online.cdn.bcebos.com/8f628bc26e364354a6b190b23f550a172bc8da8aabac462a86ab887bcde56e88)

其推荐过程可以分成二级。第一级是用候选集生成模型（Candidate Generation Model）完成候选视频的快速筛选，在这一步，候选视频集合由百万降低到几百量级，这就相当于经典推荐系统架构中的召回层。第二级是用排序模型（Ranking Model）完成几百个候选视频的精排，这相当于经典推荐系统架构中的排序层。
无论是候选集生成模型还是排序模型，YouTube 都采用了深度学习的解决方案。

2. 候选集生成模型
用于视频召回的候选集生成模型，架构如下图所示。
![](https://ai-studio-static-online.cdn.bcebos.com/f3d06fad0dca45f8bf5705c437afe586be61da06785a4abea1fc7d87c5d58765)
最底层是它的输入层，输入的特征包括用户历史观看视频的 Embedding 向量，以及搜索词的 Embedding 向量。对于这些 Embedding 特征，YouTube 是利用用户的观看序列和搜索序列，采用了类似 Item2vec 的预训练方式生成的。

除了视频和搜索词 Embedding 向量，特征向量中还包括用户的地理位置 Embedding、年龄、性别等特征。这里我们需要注意的是，对于样本年龄这个特征，YouTube 不仅使用了原始特征值，还把经过平方处理的特征值也作为一个新的特征输入模型。
这个操作其实是为了挖掘特征非线性的特性。

确定好了特征，这些特征会在 concat 层中连接起来，输入到上层的 ReLU 神经网络进行训练。

三层 ReLU 神经网络过后，YouTube 又使用了 softmax 函数作为输出层。值得一提的是，这里的输出层不是要预测用户会不会点击这个视频，而是要预测用户会点击哪个视频，这就跟一般深度推荐模型不一样。

总的来讲，YouTube 推荐系统的候选集生成模型，是一个标准的利用了 Embedding 预训练特征的深度推荐模型，它遵循Embedding MLP 模型的架构，只是在最后的输出层有所区别。

3. 候选集生成模型独特的线上服务方法
4. 排序模型
输入层，相比于候选集生成模型需要对几百万候选集进行粗筛，排序模型只需对几百个候选视频进行排序，因此可以引入更多特征进行精排。具体来说，YouTube 的输入层从左至右引入的特征依次是：
impression video ID embedding：当前候选视频的 Embedding；
watched video IDs average embedding：用户观看过的最后 N 个视频 Embedding 的平均值；
language embedding：用户语言的 Embedding 和当前候选视频语言的 Embedding；
time since last watch：表示用户上次观看同频道视频距今的时间；
#previous impressions：该视频已经被曝光给该用户的次数；
这 5 类特征连接起来之后，需要再经过三层 ReLU 网络进行充分的特征交叉，然后就到了输出层。这里重点注意，排序模型的输出层与候选集生成模型又有所不同。不同主要有两点：一是候选集生成模型选择了 softmax 作为其输出层，而排序模型选择了 weighted logistic regression（加权逻辑回归）作为模型输出层；二是候选集生成模型预测的是用户会点击“哪个视频”，排序模型预测的是用户“要不要点击当前视频”。

其实，排序模型采用不同输出层的根本原因就在于，YouTube 想要更精确地预测 用户的观看时长，因为观看时长才是 YouTube 最看中的商业指标，而使用 Weighted LR 作为输出层，就可以实现这样的目标。

在 Weighted LR 的训练中，我们需要为每个样本设置一个权重，权重的大小，代表了这个样本的重要程度。为了能够预估观看时长，YouTube 将正样本的权重设置为用户观看这个视频的时长，然后再用 Weighted LR 进行训练，就可以让模型学到用户观看时长的信息。

对于排序模型，必须使用 TensorFlow Serving 等模型服务平台，来进行模型的线上推断。

5. 训练和测试样本的处理
为了能够提高模型的训练效率和预测准确率，Youtube采取了诸多处理训练样本的工程措施，主要有3点：

候选集生成模型把推荐模型转换成 多分类问题，在预测下一次观看的场景中，每一个备选视频都会是一个分类，而如果采用softmax对其训练是很低效的。
Youtube采用word2vec中常用的 负采样训练方法减少每次预测的分类数量，从而加快整个模型的收敛速度。
在对训练集的预处理过程中，Youtube没有采用原始的用户日志，而是 对每个用户提取等数量的训练样本。
YouTube这样做的目的是减少高度活跃用户对模型损失的过度影响，使模型过于偏向活跃用户的行为模式，忽略数量更广大的长尾用户体验。
在处理测试集时，Youtube没有采用经典的随机留一法，而是一定要以用户最近一次观看的行为作为测试集。
只留最后一次观看行为做测试集主要是为了避免引入未来信息(future information)，产生于事实不符的数据穿越问题。
6. 处理用户对新视频的爱好
7. 总结
YouTube 推荐系统的架构是一个典型的召回层加排序层的架构，其中候选集生成模型负责从百万候选集中召回几百个候选视频，排序模型负责几百个候选视频的精排，最终选出几十个推荐给用户。

候选集生成模型是一个典型的 Embedding MLP 的架构，要注意的是它的输出层一个多分类的输出层，预测的是用户点击了“哪个”视频。在候选集生成模型的 serving 过程中，需要从输出层提取出视频 Embedding，从最后一层 ReLU 层得到用户 Embedding，然后利用 最近邻搜索快速 得到候选集。

排序模型同样是一个 Embedding MLP 的架构，不同的是，它的输入层包含了更多的用户和视频的特征，输出层采用了 Weighted LR 作为输出层，并且使用观看时长作为正样本权重，让模型能够预测出观看时长，这更接近 YouTube 要达成的商业目标。
