---
layout: post
title: 如何在jekyll中引入Latex公式支持
category: python
tags: [python,latex]
---


直接在jekyll博客的head文件中加入：

```
<script type="text/x-mathjax-config">
        MathJax.Hub.Config({
          tex2jax: {
            skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
            inlineMath: [['$','$']]
          }
        });
      </script>
      <script src='https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/latest.js?config=TeX-MML-AM_CHTML' async></script>
```


行中公式可以用如下方法表示：

```
    $ 数学公式 $
```

独立公式可以用如下方法表示：

```
   $$ 数学公式 $$
```
   
效果如下：

$$
f(x) = \int_{-\infty}^\infty
\hat f(\xi)\,e^{2 \pi i \xi x}
\,d\xi
$$

  