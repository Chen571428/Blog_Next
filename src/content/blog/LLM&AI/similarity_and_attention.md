---
title: '向量相似度与注意力机制'
description: '从初学者的视角看注意力机制'
publishDate: 2026-02-26 07:00:00
tags:
  - AI
  - LLM
  - NLP
  - Study
  - Note
---

考虑两个 $n$ 维的特征 （列向量）$\mathbf{a},\mathbf{b}\in \mathbb{R}^{n}$ ，他们之间的相似度是一个很有用的指标，可以用于注意力机制（计算 **权重/注意力分数** 并以内积的形式对上下文中的特征进行加权求和（行向量与矩阵相乘所得矩阵的每一个行向量都是矩阵的行向量组的线性组合），其中的注意力分数就是 **向量相似度**）等。

## Cosine Similarity 余弦相似度
其点积（内积）$\mathbf{a}^T\mathbf{b} = \lVert \mathbf{a}\rVert_{2}\lVert \mathbf{b}\rVert_{2}\cos{\theta} = \sum_{i=1}^na_{i}b_{i}$
很自然地，几何上，两个向量的夹角越小，其余弦值越大，内积也越大。
使用模长（$L_{2}$范数）把点积归一化到$[-1,1]$，也就是直接利用向量夹角的余弦值$\cos\langle\mathbf{a},\mathbf{b}\rangle =\frac{\mathbf{a}^T\mathbf{b}}{\lVert \mathbf{a}\rVert \lVert \mathbf{b}\rVert}$来衡量向量的相似度，就称为**余弦相似度**

## Scaled Dot-Product 缩放点积
向量的点积本身，已经在长度和方向两个层面衡量了两个向量的相似度。
特殊地，余弦相似度$\cos\langle\mathbf{a},\mathbf{b}\rangle =\frac{\mathbf{a}^T\mathbf{b}}{\lVert \mathbf{a}\rVert \lVert \mathbf{b}\rVert}=(\frac{\mathbf{a}^T}{\lVert \mathbf{a} \rVert})(\frac{\mathbf{b}}{\lVert \mathbf{b} \rVert})$ 把两个向量都投影到了高维**单位超球面**上再进行点积，虽然起到了归一化的效果，但损失了模长这一信息的表达能力。

某种意义上，向量的模长/范数可以理解为向量的**绝对重要性**。尽管有的任务中我们希望所有特征都是**平等**的，于是我们通过会初始化和归一化使得每一个特征的模长的期望都为$1$，此时使用余弦相似度非常合理且自然。

但在语言建模这种任务上，我们会希望模型能调节某些token的绝对重要性（直觉上，语言上下文中的某些部分（比如用户指令、句子中的关键词）也会比别的词具有更大的重要性），于是我们不是直接拿两个token的 Token Embedding向量直接做相似度计算，而是分别用可学习的$W^Q,W^K,W^V\in \mathbb{R}^{d_{model}}$（以单头注意力为例）做一次**线性变换**，这提供了模型调节单个token在上下文中的绝对重要性的能力。

若此时仍旧使用余弦相似度就掩盖了模长信息、QKV投影对模长的调节。

> 后话：
> 	23年LLaMA-3之后，$\text{QK Norm}$逐渐成为了大模型的常用组件（Qwen3系列、GLM系列、Deepseek的latent norm等），这一组件认为$\frac{QK^T}{\sqrt{ d_{k} }}$是无界的，而$\text{softmax}$只关注值之间的相对差，稍大的差距（>10）即会造成过饱和，导致数值波动太大，影响训练稳定性。
> 	
> 	$\text{QK norm}$通过在$Q,K$投影之后进行归一化（$\text{L2 Norm}$或$\text{RMS Norm}$等）来解决这个问题。显然，做$\text{L2 Norm}$之后直接就退化回了余弦相似度。做$\text{RMS Norm}$则稍好，因为$\text{RMS Norm}$中有 $\epsilon$ 和可学习的 $\mathbf{g}$ 缩放因子。不过，即使是$\text{RMS Norm}$，也导致此时的计算数学上基本等价于带缩放向量的余弦相似度。
> 	
> 	我们倾向于将这个事情理解为Trade-off，毕竟，在超大模型上，与训练稳定性相比，微小的性能损失是可接受的。
> 	
> 	那为啥不直接把点积改成余弦相似度，而是先norm再点积呢？原因在易于实现&工程效率


于是，我们转而直接对两个向量做点积，由于向量中各元素在初始化/归一化之后服从 $\mathcal{N}(0,1)$，故点积的值服从$\mathcal{N}(0,d_{\text{model}})$，于是用$\frac{1}{\sqrt{ d_{\text{model}} }}$进行缩放，将方差恢复为$1$
这就是所谓的**缩放点积**

## Attention 注意力
缩放点积计算了两个向量的相似度，注意力机制则希望利用这种相似度来计算**权重**，把其他向量的信息**加权**融合到这个向量的特征中去，这样，我们就在每一个向量中建模了上下文信息。

具体来说，我们直接使用矩阵乘法来实现批量的向量点积：
$$
(QK^T)_{i,j}=\sum_{k=1}^nq_{i,k}\cdot k_{j,k} =\mathbf{q}_{i}\cdot \mathbf{k}_{j} 
$$
$QK^T$矩阵的$i,j$元即为$\mathbf{h}_{i}$和$\mathbf{h_{j}}$ （h表示hidden state）的点积，于是$\frac{QK^T}{\sqrt{ d_{\text{model}} }}$的第$i$行表示了$\mathbf{h}_{i}$对所有$\mathbf{h}$的相似度。

如何利用这个相似度来把$\mathbf{h}$的上下文信息融合到$\mathbf{h_{i}}$中去呢？自然地想到直接进行线性组合：
$$\mathbf{h}_{i}^\prime = \sum_{j=1}^n w_{j}\mathbf{h}_{j}$$
于是用一个矩阵$W^{Attn}\in\mathbb{R}^{N\times N}$来乘我们的hidden state矩阵就好了，当然，实践中我们会给hidden state先做一次线性变换，得到$V=\mathbf{h}W^V$，再计算新的$\mathbf{h}^\prime=W^{Attn}V$。

如何从$\frac{QK^T}{\sqrt{ d_{\text{model}} }}$得到$W^{Attn}$呢？不能直接让$W^{Attn}$就等于$\frac{QK^T}{\sqrt{ d_{\text{model}} }}$吗？
首先后者肯定是不行的，最显然的一个原因和之前做**缩放**的原因相同：行向量元素的和服从$\mathcal{N}(0,N)$ $V$的每个元素服从$\mathcal{N}(0,1)$，乘起来后矩阵中每个元素服从$\mathcal{N}(0,N)$，**梯度爆炸**；其次，缺乏**非线性**

我们需要：$$W^{Attn}=\operatorname{Row-wise Normalization}(\frac{QK^T}{\sqrt{ d_{\text{model}} }})$$
用什么函数来做这个行归一化呢？直接除以$N$不行吗？现在我们知道，LLM中最常用的这个“**激活**”函数是：$$\operatorname{Row-wise Normalization}(\mathbf{x})\equiv\operatorname{Softmax}(\mathbf{x})=\frac{e^x_{i}}{\sum_{j=1}^N e^x_{j}}$$
非得用$\operatorname{Softmax}$不可吗？并不是这样，$\text{Softmax}$函数有各种各样的缺点，包括数值稳定性、$N^2$的计算时间复杂度，输出均值总为$1$以及由此带来的$\text{Attention Sink}$问题等等等等，很多模型也进行了探索和尝试，有的改造由此导出了所谓的$\text{Linear-attention}$，等等；

不过，在实践中，由于$\text{Flash-attention}$这样的技术把$\text{Softmax}$这个函数优化地非常快，且$\operatorname{Softmax}$的有的特性（初始化时行方差为$\frac{e-1}{N^2}$远小于One-Hot时的方差$\frac{1}{N}$，即初始化时注意力极度**涣散**；放大了元素的相对差，重要的token可以占据压倒性的优势等等）非常地好，在实践中的性能优势非常大，主流大模型不会直接把$\operatorname{Softmax}$直接换掉，而是利用其他方式来修复$\operatorname{Softmax}$的缺陷，例如Qwen的Gated Attention通过门控机制解决Attention Sink的问题。

最终我们得到：
$$
\operatorname{Attention}(\mathbf{h})=\operatorname{Softmax}\left( \frac{QK^T}{\sqrt{ d_{\text{model}} }} \right)V
$$
其中：
$$
Q=\mathbf{h}W^Q, K=\mathbf{h}W^K,V=\mathbf{h}W^V
$$
这就是所谓的**自注意力机制**

如果把产生$Q$的$\mathbf{h}$换成和产生$K,V$的$\mathbf{h}$不一样的序列得到的向量，就是所谓的**交叉注意力**。
