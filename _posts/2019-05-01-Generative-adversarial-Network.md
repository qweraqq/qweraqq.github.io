---
layout: post
title: "Generative adversarial networks"
date: 2019-05-01 00:00:00 +0800
author: xiangxiang
categories: machine-learning deep-learning
tags: [GANs machine-learning deep-learning]
---
  <script type="text/x-mathjax-config">
    MathJax.Hub.Config({
      tex2jax: {
        skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
        inlineMath: [['$','$']]
      }
    });
  </script>
  <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>

Dive into Generative adversarial networks

## 核心思想(Likelihood-free learning)
- 从两个分布`$ P $`, `$ Q $`中分别采样得到样本`$ S_1 $`, `$ S_2 $`

`$ S_1 = \{ x \sim P \} $`

`$ S_2 = \{ x \sim Q \} $`

- 使用一种方法检验`$ S_1 $`, `$ S_2 $`给自代表的总体`$ P $`, `$ Q $`的差异是否显著

- 这里举例一种检验方法: t检验

零假设(null hypothesis):&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;$ H_0: P = Q $ 

备择假设(alternative hypothesis) $ H_1: P \neq Q $ 

双侧检验，检验水平(显著性水平)$ \alpha $

$ t = \frac { \overline { X _ { 1 } } - \overline { X _ { 2 } } } { \sqrt { \frac { \left( n _ { 1 } - 1 \right) S _ { 1 } ^ { 2 } + \left( n _ { 2 } - 1 \right) S _ { 2 } ^ { 2 } } { n _ { 1 } + n _ { 2 } - 2 } } \left( \frac { 1 } { n _ { 1 } } + \frac { 1 } { n _ { 2 } } \right) } $

根据实际得到的样本数据计算得到统计量`t`值，然后与查表得到的t界值对比，决定是否拒绝原假设

其中：
1. $ S _ { 1 } ^ { 2 } $和$ S _ { 2 } ^ { 2 } $ 表示样本方法

2. $ \overline { X _ { 1 } } $和$ \overline { X _ { 2 } } $ 表示样本均值

3. $  n _ { 1 } $和$ n _ { 2 } $ 表示样本容量
