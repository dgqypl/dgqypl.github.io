---
layout: post
title:  "吴恩达 机器学习第9周 推荐系统 练习题答案"
author: mew151
image: assets/images/202007/accounting-administration-books-business-267582.jpg
---
<head>
    <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
    <script type="text/x-mathjax-config">
        MathJax.Hub.Config({
            tex2jax: {
            skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
            inlineMath: [['$','$']]
            }
        });
    </script>
</head>
1、Suppose you run a bookstore, and have ratings (1 to 5 stars) of books. Your collaborative filtering algorithm has learned a parameter vector $\theta^{(j)}$ for user $j$, and a feature vector $x^{(i)}$ for each book. You would like to compute the "training error", meaning the average squared error of your system's predictions on all the ratings that you have gotten from your users. Which of these are correct ways of doing so (check all that apply)? For this problem, let $m$ be the total number of ratings you have gotten from your users. (Another way of saying this is that $m=\sum_{i=1}^{n_{m}}\sum_{j=1}^{n_{u}}r(i,j)$). [Hint: Two of the four options below are correct.]

![](/assets/images/202007/RecommenderSystems_1.png){:height="67%" width="67%"}

解答：A选项$(\theta^{(j)})\_{i}$不应该有下标$i$，因为$\theta^{(j)}$是用户$j$的参数向量；同理，$x^{(i)}\_{j}$不应该有下标$j$，因为$x^{(i)}$是电影$i$的特征向量。C选项$\sum^{n}\_{k=1}$应该对$(\theta^{(j)})\_{k}x^{(i)}\_{k}$求和。

2、In which of the following situations will a collaborative filtering system be the most appropriate learning algorithm (compared to linear or logistic regression)?

![](/assets/images/202007/RecommenderSystems_2.png)

解答：A选项注意personally这个词。

3、You run a movie empire, and want to build a movie recommendation system based on collaborative filtering. There were three popular review websites (which we'll call A, B and C) which users to go to rate movies, and you have just acquired all three companies that run these websites. You'd like to merge the three companies' datasets together to build a single/unified system. On website A, users rank a movie as having 1 through 5 stars. On website B, users rank on a scale of 1 - 10, and decimal values (e.g., 7.5) are allowed. On website C, the ratings are from 1 to 100. You also have enough information to identify users/movies on one website with users/movies on a different website. Which of the following statements is true?

![](/assets/images/202007/RecommenderSystems_3.png)

4、Which of the following are true of collaborative filtering systems? Check all that apply.

![](/assets/images/202007/RecommenderSystems_4.png)

5、Suppose you have two matrices $A$ and $B$, where $A$ is 5x3 and $B$ is 3x5. Their product is $C = AB$, a 5x5 matrix. Furthermore, you have a 5x5 matrix $R$ where every entry is 0 or 1. You want to find the sum of all elements $C(i,j)$ for which the corresponding $R(i,j)$ is 1, and ignore all elements $C(i,j)$ where $R(i,j)=0$. One way to do so is the following code:

![](/assets/images/202007/RecommenderSystems_5_1.png)

Which of the following pieces of Octave code will also correctly compute this total? Check all that apply. Assume all options are in code.

![](/assets/images/202007/RecommenderSystems_5_2.png){:height="50%" width="50%"}