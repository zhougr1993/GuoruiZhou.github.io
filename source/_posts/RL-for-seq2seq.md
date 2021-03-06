---
title: Notes on Sequence Level Training with Recurrent Neural Networks
date: 2016-10-05 15:20:09
tags:
---
# Sequence级别的seq2seq训练过程，用强化学习来优化整个句子效果

[原论文连接](http://arxiv.org/abs/1511.06732)

###Summary

最近突然想把自己看过的东西记下来，决定开通博客啦，就以这篇seq2seq领域的论文开始吧。

先简单介绍一下seq2seq是什么。
processing of seq2seq：
![](https://www.tensorflow.org/versions/r0.11/images/basic_seq2seq.png)
借用一下tensorflow里Sequence-to-Sequence的图。这个任务的目标简单的说就是将一个序列翻译成另一序列。

具体到这张图里我们的输入是有序的A,B,C序列，目标序列是输出X,Y,Z。其中每一个白色的方框是RNN的一个Cell，输入部分和输出部分的RNN分别共享两套参数。整个过程可看做将输入的A,B,C通过输入RNN编码为一个Cell code，输出部分根据上一时态的输入以及Cell状态预估下一个输出词的概率分布即<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default"></script> \\(P(w_{t+1}^g|h_{t+1},w_t)\\) 根据此概率分布找出最适合的输出，可抽象为一个encode之后decode的过程。

此类过程可以做很多工作，比如机器翻译讲英文翻译为中文，摘要抽取将长文转换为单字信息量更大的短文，或者是人机对话的QA问答都可以抽象成类似的过程。

### Motivation
回到这篇论文来看，它主要目标是试图解决seq2seq领域的两个重要问题：

- Exposure Bias: seq2seq的Training阶段的decode过程，我们是有ground truth的，即训练decode过程的RNN时我们可以输入真实的序列上一个time step的词，这种训练过程叫XENT。但是在Testing阶段的decode过程我们并不知道上一个time step的词是什么，我们输入给model的是一个预测出来概率最大的词，如果这个词错了，误差会随后续过程传递并且模型没有机会纠正。

- Loss-Evaluation Mismatch: 我们训练模型的时候，采用的loss是word级别的交叉熵，但是我们最终评判模型的指标是sequence级别的指标，比如机器翻译的BLEU.

### Related Work
对应于Exposure Bias，该论文之前有两种解决方法：

- beam search
    算法原理：图路径搜索中，每一步深度扩展的时候，剪掉一些质量比较差的结点，    保留下一些质量较高的结点，这样就减少了空间消耗，并提高了时间效率，但缺点就是    有可能存在潜在的最佳方案被丢弃，其中保留的质量较高路径的个数称为beam size.（走n步吃到嘴多的果实）
    ![](http://7xp3xc.com1.z0.glb.clouddn.com/201601/1475738457157.png
)
应用到seq2seq里就是每一次解码过程看成是一步深度扩展，每一次解码预测概率最大的beam size个词就是候选结点，并累积概率选择概率和最高的beam size条路径，把整个解码过程看成是寻求最大联合概率的图搜索过程。该方法能解决Exposure bias的原因在于它使解码过程不仅仅依赖于前一个词输出，还要满足全局解码概率最大，因而原始模型前一个预测错误而带来的误差传递的可能性就降低了。

- DaD (Data as Demonstrator)
对于Exposure bias造成解码结果不好的原因，bengio解释为由于训练和预测过程在输入数据的分布不同，前者是样本的数据分布，后者则是decode模型的输出分布。因而解决办法就是保证两个流程在解码的时候输入参数服从相同分布，即都采用前一个词的预测结果作为当前词的输入。
为此提出了一种退火算法来解决这个问题，在Training过程中引入一个概率值参数，称其为温度，当温度值较大时高概率采用ground truth词\\(w_t\\)输入，当值较小时，则高概率采用预测输出词\\(w_t^g\\)作为下一个输入，随着迭代次数的增加，该参数趋近于0.即完全采用前一个词的预测\\(w_t^g\\)作为输入。
Training的decode部分如下图：\\(w_t^g\\)为模型预测词，\\(w_t\\)为ground truth 后文中所有的带上标g的都为模型预测输出。
![](http://7xp3xc.com1.z0.glb.clouddn.com/201601/1475739099129.png)
### Model
此文借鉴了DaD的训练过程，同时由于有些目标如(BLEU)不能被直接优化，作者在此文中提出了一种新的训练流程，能可观的提高此类目标的效果。
核心思想是利用强化学习，强化学习可以通过随机的递归生成结果来寻找优化那些复杂的目标。作者将seq2seq的训练过程对应于强化学习抽象成了这么几个部分：

- action: 每一个time step 的候选词
- state: 每一个time step RNN隐层的神经元状态
- reward: 整个后续sequence解码完后的bleu等指标

这里需要强调一点，reward是整个sequence解码完的指标，所以作者题目是sequence级别的训练，不是word级别，此文的目标也是优化最后整个生产sequence的效果。

因此作者将Reinforce Loss函数定义为整个生成出的sequence拿到的负reward:
![](http://7xp3xc.com1.z0.glb.clouddn.com/201601/1475740434191.png
)
对参数\\(\theta\\)求导:
![](http://7xp3xc.com1.z0.glb.clouddn.com/201601/1475740913152.png
)
其中\\(O_t\\)是softmax的输入。
![](http://7xp3xc.com1.z0.glb.clouddn.com/201601/1475740768186.png
)
这个公式的推导可以去看另一篇用RL的文章 https://arxiv.org/abs/1505.00521
需要注意的是 这个导数的右半部分其实就是交叉熵Loss(XENT指在decode部分输入为ground truth的训练方式)的求导结果:
![](http://7xp3xc.com1.z0.glb.clouddn.com/201601/1475741346812.png
)
而左半部分是整个被生成序列的reward减去t时刻状态\\(S_t\\)下所能得到的平均reward，这是强化学习比较常见的一个公式。
这个推导结果的物理意义可以很直观的解释，即在t+1时刻右端交叉熵Loss的限制下，左端要求我们选择被生成的词最后构成的reward \\(r(w_1^g,...,w_T^g)\\)要大于在t+1时刻时态的一个可获得的平均reward \\({\bar{r}}_{t+1}\\)。

前面提到是sequence级别的训练，所以最终每个时刻的loss要整个sequence全decode后才能得到，而目前我们唯一不知道的量就是reward \\({\bar{r}}_{t+1}\\)。在其他场景下，这个值可以用随机的生成结果来采样取得，本文作者用了非常简单的线性回归来预估这个值，输入为当前RNN模型的所有隐层节点状态，Loss为与真值reward二范数距离: \\({||\bar{r}-r||}^2\\)。

### 训练过程
偷个懒直接截原文图再来解释整个过程:
![](http://7xp3xc.com1.z0.glb.clouddn.com/201601/1475742637703.png)

这里面\\(N^{XENT}\\)和\\(N^{XE+R}\\)分别是两个可调的超参，分别代表两个训练过程的训练次数，XENT代表传统的交叉熵Loss训练并且decode时每一个时刻的输入为ground truth \\(w_t\\), XE+R代表的是Loss为前文提到的Reinforce Loss，T代表的是decode部分sequence长度，decode部分每一个时刻的输入为模型预测生成的输出\\(w_t^g\\)。

整个过程其实就分为两部分，一开始先训练\\(N^{XENT}\\)个epoch 用传统的XENT训练方式，之后呢在每一个sequence训练的decode部分，前s个step 用XENT交叉熵loss，后T-s个时态用Reinforce Loss训练。然后不断的将s从T开始减少，最后整个sequence都用Reinforce Loss。


这里就不贴实验结果图啦，总的来说在BLEU等整句指标上此文的方法都有显著的提高。

### 我的观点
- 本文的最大贡献其实不在于Exposure Bias部分而在于Loss-Evaluation Mismatch部分，提出了一个有效的训练方法，能优化那些复杂不直观的目标，同时近来reinforce learning在各领域发力，这种类似的训练方法可以值得大家在各个领域尝试一下。

- 训练过程设计得很巧妙，其实作者很聪明也了解reinforce learning的特性，强化学习有一个致命的缺点就是训练过程的不稳定性，作者让训练过程起始时刻用XENT来作为训练，是为了让模型到达一个较好又稳定的状态，之后再用强化学习来优化模型对于整句的目标。如果不这样安排，大可能模型无法收敛到一个好的解。

- 本文对Exposure Bias还是没有有效解决，这一点上我倒觉得还不如DaD，至少DaD在训练过程的后半程退火阶段，Training的decode过程和Testing的decode过程是一致的，而本文中XE+R部分虽然也是用模型生成的词\\(w_t^g\\)做输入但是Loss是RL Loss并不是完全依赖于交叉熵。这里是不对等的，所以预测过程用beam search依然会极大的提高本文提出的模型的效果。
