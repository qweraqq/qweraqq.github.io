---
layout: post
title: "Generative adversarial networks"
date: 2019-02-17 00:00:00 +0800
author: xiangxiang
categories: machine-learning deep-learning
tags: [GANs machine-learning deep-learning]
---
Dive into Generative adversarial networks
  <script type="text/x-mathjax-config">
    MathJax.Hub.Config({
      tex2jax: {
        skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
        inlineMath: [['$','$']]
      }
    });
  </script>
  <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>



## 0x00 核心思想(Likelihood-free learning)
- 从两个分布`$ P $`, `$ Q $`中分别采样得到样本`$ S_1 $`, `$ S_2 $`

`$ S_1 = \{ x \sim P \} $`

`$ S_2 = \{ x \sim Q \} $`

- 使用一种方法检验`$ S_1 $`, `$ S_2 $`给自代表的总体`$ P $`, `$ Q $`的差异是否显著

- 这里举例一种检验方法: two-sample t检验

零假设(null hypothesis):&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;$ H_0: P = Q $ 

备择假设(alternative hypothesis) $ H_1: P \neq Q $ 

双侧检验，检验水平(显著性水平)$ \alpha $

$ t = \frac { \overline { X _ { 1 } } - \overline { X _ { 2 } } } { \sqrt { \frac { \left( n _ { 1 } - 1 \right) S _ { 1 } ^ { 2 } + \left( n _ { 2 } - 1 \right) S _ { 2 } ^ { 2 } } { n _ { 1 } + n _ { 2 } - 2 } } \left( \frac { 1 } { n _ { 1 } } + \frac { 1 } { n _ { 2 } } \right) } $

根据实际得到的样本数据计算得到统计量`t`值，然后与查表得到的`t`界值对比，决定是否拒绝原假设

其中：
1. $ S _ { 1 } ^ { 2 } $和$ S _ { 2 } ^ { 2 } $ 表示样本方差

2. $ \overline { X _ { 1 } } $和$ \overline { X _ { 2 } } $ 表示样本均值

3. $  n _ { 1 } $和$ n _ { 2 } $ 表示样本容量

## 0x01 应用Likelihood-free learning思想到机器学习中

- `$ S _ {1} = D = \{ x \sim P _ {data} \} $` 表示我们实际获取的一个数据集

- `$ S _ {2} = \{ x \sim P _ { \theta } \} $` 表示我们想要训练得到的模型

- 目标: 我们训练得到的模型与实际数据模型一样，也就是说`$ S _ {1} = S _ {2} $`

- 应用Likelihood-free learning思想，既然我们的模型使得`$ S_1 $`, `$ S_2 $`的差异最小，我们就需要`minimizes a two-sample test objective`

- 总结一下新思路

`Optimize a surrogate objective that instead maximizes some distance between $ S _ {1} $ and $ S _ {2} $ `

## 0x02 GANs在machine learning体系的位置
- 这时候需要回顾一下应用Machine learning解决实际问题的时候需要考虑的三个要素:

{% highlight text %}
1. 使用何种模型结构(model)，比如是LR还是NN
2. 使用怎么样的正则化技巧(regularization)，比如是L1/L2/dropout 
3. 训练使用什么优化算法(optimization/training), 比如是Adam、SGD
{% endhighlight %}

- Likelihood-free, MLE(Maximum Likelihood Estimation), MAP(Maximum A Posterior)都是属于大的machine learning思想，GANs相当于在Likelihood-free这个大思想里提出了一个具体的可行方式

- GANs是一种思想，我觉得主要归属model+optimization/training, 比如看这个[链接](https://machinelearningmastery.com/what-are-generative-adversarial-networks-gans/)

{% highlight text %}
GANs are a model architecture for training a generative model
{% endhighlight %}

- 我们还需要选择更加具体的model以及optimization/training方式来实现这一种思想

## 0x03 GANs如何实现Likelihood-free learning

- GANs的模型包含两部分

1. $ G_\theta $:  a directed latent variable model that deterministically generates samples `x` from latent virable `z`
2.  $ D_\phi $: a function whose job is to distinguish samples from the real dataset and the generator

- A two player minimax game:

$ \min _ { \theta } \max _ { \phi } V \left( G _ { \theta } , D _ { \phi } \right) = \mathbb { E } _ {  x \sim p _ { data } } \left[ \log D _ { \phi } ( \mathbf { x } ) \right] + \mathbb { E } _ { z \sim p _ { z } } \left[ \log \left( 1 - D _ { \phi } \left( G _ { \theta } ( \mathbf { z } ) \right) \right) \right] $

1. The generator minimizes a two-sample test objective $ p_{data} = p_{\theta} $
2. The discriminator maximizes the objective $ p_{data} \neq p_{\theta} $

- 接下来从数学角度分析一下这个公式

- 展开上面的式子

$ \mathbb { E } _ {  x \sim p _ { data } } \left[ \log D _ { \phi } ( \mathbf { x } ) \right] + \mathbb { E } _ { z \sim p _ { z } } \left[ \log \left( 1 - D _ { \phi } \left( G _ { \theta } ( \mathbf { z } ) \right) \right) \right] $

` $ \Leftrightarrow $ `

$ \mathbb { E } _ {  x \sim p _ { data } } \left[ \log D _ { \phi } ( \mathbf { x } ) \right] + \mathbb { E } _ { x \sim { Generator } } \left[ \log \left( 1 - D _ { \phi } \left( x \right) \right) \right] $

` $ \Leftrightarrow $ `

$ \int _ { x } \left( p _ {data} \left( x \right) \log D _ { \phi }  \left( x \right) + p _ { generator } \left( x \right) \log \left( 1 - D _ { \phi } \left( x \right) \right) \right) d x $

- 如果我们取定一个Generator，通过对上面的式子求导为0就可以得到最优的判别模型`D`

接下来的内容后面补充

## 0x04 GAN traning
1. Sample minibatch of size m from data: `$ x ^ { (1) }, x ^ { (2) }, ..., x ^ { (m) }  \sim Data $ `

2. Sample minibatch of size m of noise:  `$ z ^ { (1) }, z ^ { (2) }, ..., z ^ { (m) }  \sim p _ { z } $ `

3. Take a gradient descent step on the generator parameters $ \theta $

4. Take a gradient ascent step on the discriminator parameters $ \phi $

5. Repeat n epochs

## 0x05 GAN training optimization challenges
- 上面的过程其实基于了一个不现实的假设, 也就是`GANs如何实现Likelihood-free learning`部分中假设我们取定的Generator就是真实数据来源的分布空间，我们只是不知道具体的参数而已
	- 举个例子：我们假设了数据就是一个正态分布，但是我们不知道正态分布的具体参数
{% highlight text %}
If the generator updates are made in function space and discriminator is optional at every step, 
then the generator is guaranteed to converge to the data distribution
{% endhighlight %}

- `In practice, the generator loss and discriminator loss keeps oscillating during GAN training` 

- No robust stopping criteria in practice

- Mode collapse: the generator of a GAN can often get stuck producing one of a few types of samples over and over again

## 0x06 常见的GANs模型
未完待续