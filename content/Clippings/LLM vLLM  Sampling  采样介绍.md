---
title: "LLM / vLLM : Sampling / 采样介绍"
source: "https://zhuanlan.zhihu.com/p/1911546380444496008"
author:
  - "[[hcxx]]"
published:
created: 2025-09-05
description: "什么是sampler ？LLM生成token的宏观过程可以见下图. 其中Sampler是将模型输出的hidden state转化为token id的过程. 在生成token的过程中，首先用户输入的prompt通过tokenization转换为token id list，例如： from…"
tags:
  - "clippings"
---
## 什么是Sampler？

LLM生成token的宏观过程可以见下图. 其中Sampler是将模型输出的hidden state转化为token id的过程.

![](https://pica.zhimg.com/v2-b5ebab649fef6a45f59e78208d57e0e8_1440w.jpg)

在生成token的过程中，首先用户输入的prompt通过[tokenization](https://zhida.zhihu.com/search?content_id=258383047&content_type=Article&match_order=1&q=tokenization&zhida_source=entity)转换为token id list，例如：

```
from transformers import BertTokenizer

tokenizer = BertTokenizer.from_pretrained("bert-base-uncased")
text = "Tell me about what is the meaning of life?"
encoded_input = tokenizer.encode(text, add_special_tokens=True)

print("Token IDs:", encoded_input)
```

输出：

```
Token IDs: [101, 2425, 2033, 2055, 2054, 2003, 1996, 3574, 1997, 2166, 1029, 102]
```

之后再通过[token embeddings](https://zhida.zhihu.com/search?content_id=258383047&content_type=Article&match_order=1&q=token+embeddings&zhida_source=entity) 将token转换为对应向量. 一般来说，token embeddings包含一个`V x H`维度的权重矩阵，其中`V`是vocabulary size，即单词集合的数量，在英文模型中可能是30000~50000这个范围. `H`是token对应向量的维度，一般和Attention计算中输入和输出的hidden state维度相同(不一定要相同，但是这样做可以直接将embedding的权重用在sampling过程中来计算相似度，这会在后面介绍).  
token embedding的权重一般由训练学习得到，推理过程中直接加载权重即可. Tokenization的过程类似于查表的过程，如：

```
import torch
import torch.nn as nn

torch.set_printoptions(precision=1)

vocab_size = 8
embedding_dim = 4
input_ids = torch.tensor([3,4,1],dtype=torch.int)

custom_weights_list = [
    [0.1, 0.2, 0.3, 0.4], 
    [1.1, 1.2, 1.3, 1.4], 
    [2.1, 2.2, 2.3, 2.4],  
    [3.1, 3.2, 3.3, 3.4],  
    [4.1, 4.2, 4.3, 4.4],  
    [5.1, 5.2, 5.3, 5.4],  
    [6.1, 6.2, 6.3, 6.4],  
    [7.1, 7.2, 7.3, 7.4]   
]
custom_weights_tensor = torch.FloatTensor(custom_weights_list)
custom_embeddings = nn.Embedding.from_pretrained(custom_weights_tensor)

with torch.no_grad():
    token_embeddings = custom_embeddings(input_ids)  

print(token_embeddings)
```

输出：

```
tensor([[3.1, 3.2, 3.3, 3.4],
        [4.1, 4.2, 4.3, 4.4],
        [1.1, 1.2, 1.3, 1.4]])
```

将token embeddings和position embeddings相加，即可得到输入到模型中的初始hidden state.  
LLM的主要推理都花费在hidden state在一层层的[Attention Layer](https://zhida.zhihu.com/search?content_id=258383047&content_type=Article&match_order=1&q=Attention+Layer&zhida_source=entity)的计算中. 当拿到最后一层Attention Layer输出的hidden state时，接下来就是我们介绍的sampling操作，它就是通过hidden state生成next token的过程，主要过程如下所示:

![](https://pic2.zhimg.com/v2-8e87a4db25ef3b2ec375d551a1d425a1_1440w.jpg)

下面主要介绍Sampling中的各个计算阶段.

## Presence and Frequency Penalties

拿到从hidden state计算得到的logits之后，我们还需要对logits作一些前置处理操作，这些操作能过调整模型的输出特性.  
Presence Penalties (存在惩罚）和 Frequency Penalties (频率惩罚)用于调节和优化生成文本的质量与多样性，就像写作一样，反复表达同一个意思时，我们会尽可能地更换词汇来提升文章的流畅度.  
通过计算已经生成的tokens的出现和频率，Sampler 将这两种惩罚作用于logits上，其中Presence Penalties只关注某个token是否已经存在，而 Frequency Penalties 更进一步地考虑token的出现频率，这两者的作用可以见下图所示.

  

![](https://pic3.zhimg.com/v2-1f6ef4c435a1b626a4d5526314bf5600_1440w.jpg)

## [Temperature Scaling](https://zhida.zhihu.com/search?content_id=258383047&content_type=Article&match_order=1&q=Temperature+Scaling&zhida_source=entity)

在某些时候，我们希望LLM的输出更具有创造性和多样性，而有时则要求它更加确定以及精准，这种特性可以通过temperature参数去控制  
例如，用户输入"你觉得今天应该穿什么?". 在低温时(low temperature)，LLM可能会输出"根据天气预报，今天气温适中，建议穿一件外套". 而在高温(high temperature)时，LLM则会更加具有创造性："今天你应该穿一件由阳光编织的斗篷，搭配一朵会说话的花".  
temperature同样是作用于logits上来实现的，即`logits = logits / temperature`.  
下图展示了不同的temperature 参数作用于logits之后计算出的概率结果. temperature越大时，概率分布就越平缓，在后续的采样过程中，得到的token就越多样.

![](https://pic3.zhimg.com/v2-11dafc2c718321dbe935a46518701712_1440w.jpg)

## Top-p and Top-k Truncation

经过temperature scaling之后，我们会将logits通过softmax转换为概率. 一般来说，下一步就是从tokens的概率分布中去采样出next token. 但是简单依赖于全体token的概率分布去采样时，会有小概率出现前言不搭后语的问题.  
例如在"The weather today is very"之后，候选词的概率分布为：

```
pleasant: 0.15
warm: 0.10
sunny: 0.30
banana: 0.01
purple: 0.05
cloudy: 0.25
running: 0.02
terrible: 0.08
unpredictable: 0.03
apple: 0.01
```

那么会有一定的随机概率选中`apple`或者`banana`，并输出"The weather today is very apple".  
为了避免这样的问题，在sampling之前，一般还会有top-p & top-k truncation操作.  
顾名思义，该操作就是将低概率的tokens从采样集合中丢弃，从而避免模型输出荒谬的结果. 具体来说：

- top-p truncation: 以概率p作为阈值. 将token按照概率从大到小排序取. 找到第N个token并满足前N-1个token的概率之和小于等于p，而前N个token的概率之和大于p. 将N之后的token全部丢弃.
- top-k truncation: 以数量k作为阈值. 将token按照概率从大到小排序取，将第k个token之后的tokens全部丢弃.

top-p & top-k truncation的具体操作流程可以见下图. 其中`p = 0.95, k = 5`.

![](https://pic2.zhimg.com/v2-0338eaf9e590e62c3c6165f06aebab11_1440w.jpg)

经过top-p & top-k truncation后，可选的token集合`[sunny, cloudy, pleasant, warm, terrible]`全部都是合理的结果.

## Sampling

经过一系列对logits以及prob的前置处理后，现在开始真正的sampling操作. 这里可以选择多种策略，一般包括greedy, random, 以及beam search.

### Greedy / [Random Sampling](https://zhida.zhihu.com/search?content_id=258383047&content_type=Article&match_order=1&q=Random+Sampling&zhida_source=entity)

Greedy sampling 和 Random sampling是最简单以及最直观的策略，假设经过一系列上述的前置操作后，有4个待选token以及对应概率如下：

```
Token A: 0.6 
Token B: 0.25
Token C: 0.1
Token D: 0.05
```

- greedy sampling：从候选token中选择概率最大的token，所以token A一定会是最终采样的next token (确定过程，deterministic)
- random sampling: 将上述的概率分布作为多项分布的权重进行采样，其中token A被采样的概率是0.6，其余同理. (随机过程，stochastic)
![](https://pic1.zhimg.com/v2-a540755046c54f9e3b3f23e55581acd0_1440w.jpg)

### [Beam Search](https://zhida.zhihu.com/search?content_id=258383047&content_type=Article&match_order=1&q=Beam+Search&zhida_source=entity)

Beam Search是一种更复杂的sampling方法，它是一种多路径搜索的sampling策略.  
无论是greedy sampling还是 random sampling，最终只会保留一个采样的token用于下一次迭代的token生成. 但是，我们能否也试试其他可能呢？就像小说中经常提到的"莫欺少年穷"，可能token B只是在当前的step中概率分布比较低，但是基于它的后续生成token更具有潜力呢？  
假设有一台算力无限的计算机，以及一个只有两种token (A 和 B)的语言，对于用户的某个输入prompt，我们可以计算出所有可能回复，直至到达停止符.

![](https://pic3.zhimg.com/v2-051738990e0c6930891c6819b1b6dc46_1440w.jpg)

如上图所示，在遇到停止符号之后，我们可以通过最后一个token的概率或者其路径上节点的概率总和来决定最后的回复. 例如，假设`p(A-B-A)`的概率最大，那么我们最后的回复为`ABA`.  
但是现实中没有这样算力无限的计算机，且token集合要大得多. 假设我们使用的token集合大小是30000，那么生成一段100词回复的可能路径个数为`30000^100`，这是完全不现实的.  
Beam Search 是一种在Greedy Sampling / Random Sampling和穷举之间的折中方法. 相对于列出所有可能，它每次只会保留前W个概率最大的节点，其中W被称之为beam search的宽度(width). 当某一条路径对应的token无一存在于这W个token之中时，这条路径会被直接抛弃. 下图展示了当token set为(A,B,C,D), W =3时，应用beam search的执行路径：

![](https://pic1.zhimg.com/v2-50c04ad937b746d962d75b90b978272e_1440w.jpg)

Beam Search 一般需要消耗更高的算力. 同时注意到多个选中节点可能会来自于同一个父节点，例子上图step3中的 AAA 和AAB都来自于同一个父节点AA，这表示它们共享一个context，其k-v cache也可以共享. vLLM的scheduler针对beam search这种采样方式也作了相应的优化，这将在后续探讨.

## Sampler的细节问题

Sampling的过程涉及到一些可能会被忽视的细节，在最后这里补充.

### Sampler用哪一个Hidden State 作为输入？

对于自回归的LLM模型来说，它的推理过程包括两阶段——[prefill阶段](https://zhida.zhihu.com/search?content_id=258383047&content_type=Article&match_order=1&q=prefill%E9%98%B6%E6%AE%B5&zhida_source=entity)以及[decoding阶段](https://zhida.zhihu.com/search?content_id=258383047&content_type=Article&match_order=1&q=decoding%E9%98%B6%E6%AE%B5&zhida_source=entity).

在prefill阶段，我们可以一次性获得用户输入的prompt，它对应N个tokens，我们可以通过batch推理的方式，一次性获得这些token的k-v ，并cache下来. 当模型推理结束时，我们获得N个hidden state，但是只有最后一个token的hidden state才会输入到Sampler中，用来生成第一个回复token.

既然这样，为什么在prefill阶段还需要在每一层去计算N个token的hidden states 呢？因为计算最后一个token的输出hidden state依赖于所有 token 的 key 和 value，而这些 key/value 又是由前一层的 hidden states 转换来的. 因此，即使最终只需要最后一个 token 的 hidden state，在计算时仍然需要前面所有 token 的中间表示.

![](https://pic3.zhimg.com/v2-21d97192ec3a6c29a30435c8d52b2c5e_1440w.jpg)

而在decoding阶段，我们每次只需要将最后一个生成的token的hidden state送到模型中推理，然后得到输出的hidden state，这个输出的hidden state将输入到sampling中去生成下一个token.

### Sampler 用什么权重去计算Logits？

Sampling的过程和分类过程类似，都是通过hidden state(特征向量)计算得到token id (分类类别). 所以第一步是将hidden state(维度为`H`) 转换为logits(维度为`V`). 这个过程有两种做法：

- 方法一：类似于图像分类，训练一个全连接层(FC)用于分类，这个FC的权重是`H x V`的矩阵，其中`V`是vocabulary size. `H`是hidden state的维度. 这个权重由训练过程中的反向传播学习得到. DeepSeek-VL2采用的就是这种方式.
- 方法二 ：输入到模型的hidden state是从token embedding中查表得到的，那么经过模型的多层attention layer计算后，输出的hidden state是否可以直接和token embedding计算相似度得到logits ？这是可以的， hidden state的维度是`H`，token embedding的维度是`V x H`. 这样将hidden state和token embedding中的`V`个向量分别计算内积，即可得到维度为`V`的logits. GPT-2模型采用的就是这种方式.

实际上，方法一和方法二的计算过程是一致的(矩阵相乘)，换句话来说，FC的计算也可以理解为一种相似度的计算. FC的权重可以理解为token (分类类别)的hidden state / (特征向量)的聚类中心. 所以方法一和方法二的区别在于我们是否重新学习这个聚类中心，还是直接复用token embeddings的权重来作为这个聚类中心.  
`vllm`或者`transformers`中一般通过`tie_word_embeddings` 这个参数来控制使用哪种权重.