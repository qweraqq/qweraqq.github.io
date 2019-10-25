## About Github Pages
based on jekyll

## DEBUG locally
docker run -it --rm -v "$PWD":/usr/src/app -p "4000:4000" starefossen/github-pages

## LaTex Support
- 直接在`_posts`目录中md文件的meta信息后面加入
```js
  <script type="text/x-mathjax-config">
    MathJax.Hub.Config({
      tex2jax: {
        skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
        inlineMath: [['$','$']]
      }
    });
  </script>
  <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
```

- 比如这样：
```
---
layout: post
title: "Title"
date: 2019-05-01 00:00:00 +0800
author: author
categories: machine-learning deep-learning
tags: [machine-learning deep-learning]
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
```

- 在markdown正文中就可以了
```
$ a^2 + b^2 = c^2 $
```

- 参考链接
1. [http://www.iangoodfellow.com/blog/jekyll/markdown/tex/2016/11/07/latex-in-markdown.html](http://www.iangoodfellow.com/blog/jekyll/markdown/tex/2016/11/07/latex-in-markdown.html)

2. [https://stackoverflow.com/questions/26275645/how-to-support-latex-in-github-pages](https://stackoverflow.com/questions/26275645/how-to-support-latex-in-github-pages)