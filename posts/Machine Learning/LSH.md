---
date: 2014-10-08
layout:  post
title:  K近邻算法之局部敏感哈希
tagline:  ""
categories:
-  MachineLearning
tags:
- LSH
---

问题背景
===
&emsp;&emsp;KNN 算法,即 K 最近邻算法,是模式识别中一种最基本的用于分类与回归的非参方法。

&emsp;&emsp;KNN 算法的核心思想是如果一个样本在特征空间中的 K 个最相邻的样本中的大多数属于某一个类别,则该样本也属于这个类别,并具有这个类别上样本的特性。该方法在确定分类决策上只依据最邻近的一个或者几个样本的类别来决定待分样本所属的类别。 KNN 方法在类别决策时,只与极少量的相邻样本有关。由于 KNN 方法主要靠周围有限的邻近的样本,而不是靠判别类域的方法来确定所属类别的,因此对于类域的交叉或重叠较多的待分样本集
来说,KNN 方法较其他方法更为适合。

&emsp;&emsp;KNN 问题最主要的难点在于,在输入数据规模较大(N 较大)的情况下,朴素的线性查找算法变得难以接受。为此,学界主要提出两种不同的改进思路:空间划分和哈希。

&emsp;&emsp;这里我们重点了解下局部敏感哈希。

局部敏感哈希
====
&emsp;&emsp;Indyk(1998) 最早提出了局部敏感哈希算法,为了解决所谓的“高维诅咒”给空间划分方法带来的优化难题。

&emsp;&emsp;1.基本原理
---
&emsp;&emsp;我们知道,通过建立 hash table 的方式我们能够得到 O(1)的查找时间性能,其中关键在于选取一个 hash function,将原始数据映射到相对应的桶内(bucket, hash bin),例如对数据求模:h = x mod w,w 通常为一个素数。在对数据集进行 hash 的过程中,会发生不同的数据被映射到了同一个桶中(即发生了冲突 collision),这一般通过再次哈希将数据映射到其他空桶内来解决。这是普通 hash 方法或者叫传统 hash 方法,其与 LSH 有些不同之处。

&emsp;&emsp;我们的目的是相近的向量对几乎相同的输入内容，产生相同或者相近的hashcode。也就是说，hashcode本身的相近程度就可以反应输入的相似程度。换句话说，如果两个向量相似，则它们的哈希值有较高的概率是相同的。因此，我们的hash fuctions应该反映空间的相对关系。

&emsp;&emsp;那具有怎样特点的hash functions才能够使得原本相邻的两个数据点经过hash变换后会落入相同的桶内？这些hash function需要满足以下两个条件：

&emsp;&emsp;1）如果d(x,y) ≤ d1， 则h(x) = h(y)的概率至少为p1；

&emsp;&emsp;2）如果d(x,y) ≥ d2， 则h(x) = h(y)的概率至多为p2；

&emsp;&emsp;其中d(x,y)表示x和y之间的距离，d1 < d2， h(x)和h(y)分别表示对x和y进行hash变换。

&emsp;&emsp;满足以上两个条件的hash functions称为(d1,d2,p1,p2)-sensitive。而通过一个或多个(d1,d2,p1,p2)-sensitive的hash function对原始数据集合进行hashing生成一个或多个hash table的过程称为Locality-sensitive Hashing。

&emsp;&emsp;2.哈希函数
---
&emsp;&emsp;LSH的实现主要依赖于局部敏感哈希函数族。
    
&emsp;&emsp;设计针对不同距离度量的局部敏感哈希函数族是设计局部敏感哈希算法的关键，下面介绍几种常用的距离度量。 

&emsp;&emsp;&emsp;&emsp;*余弦距离：两个点之间的余弦距离被视为由这两个点形成的向量之间的角度。不管这个空间是几维的，这个角度在 0 度于 180 度之间。 

&emsp;&emsp;&emsp;&emsp;LSH hash function为：H(V) = sign(V·R)，R是一个随机向量。V·R可以看做是V向R上进行投影操作。其是(d1,d2,(180-d1)180,(180-d2)/180)-sensitive的。

&emsp;&emsp;&emsp;&emsp;*Jaccard距离：又叫Jaccard系数或Jaccard相似性系数，用来比较样本集中的相似性和分散性的一个概率。Jaccard系数等于样本集交集与样本集合集的比值，即J=|A∩B|/|A∪B|J(V1, V2)=| V1∩V2 | / | V1∪V2 |

&emsp;&emsp;&emsp;&emsp;LSH hash function为：minhash，其是(d1,d2,1-d1,1-d2)-sensitive的。

&emsp;&emsp;&emsp;&emsp;*汉明距离：给定一个向量空间，汉明距离被定义为两个向量之间它们坐标不同的数目。

&emsp;&emsp;&emsp;&emsp;LSH hash function为：H(V) = 向量V的第i位上的值，其(d1,d2,1-d1/d,1-d2/d)-sensitive的。

&emsp;&emsp;3.LSH 应用于 KNN
---
&emsp;&emsp;我们只需在候选的table中逐个比较就可以了。LSH解决KNN给出的结果是非准确的。但是有较高的概率保证。

&emsp;&emsp;4.伪代码
---
<pre><code>
function preprocessin(Es)
{
    for each i in [1..l] do
        generate a random hash function Hi;
    done;
    for each i in [1..l] do
        for each j in [1..|E|] do
            store pj on bucket Hi(pj) of hash table Ti;
        done;
    done;
    return All hash table T;
}
</code></pre>

&emsp;&emsp;哈希表建立好之后就可以进行查询。对于某个待查询的点q：

<pre><code>
function approximate_knn_query(q)
{
    S <- null;
    for each i in [1..l] do
        S <- S add {points found in Hi(q) bucket of talbe Ti};
    done;
    return the K nearest neighbors of q in S;
}
</code></pre>

总结
===
&emsp;&emsp;本文简单介绍了LSH算法的原理和实现，对于高维下的近似查找，LSH有很高的准确性，而且空间复杂度和时间复杂度都相对较低，适用于数据规模很大而对不准确性有着较好容忍的情况。在图像检索，音乐检索，分类聚类等方面有着广泛应用。

参考文献
===
*[局部敏感哈希(Locality-Sensitive Hashing, LSH)方法介绍]("http://blog.csdn.net/icvpr/article/details/12342159")

*史世泽，局部敏感哈希的研究，西安电子科技大学，2013年3月

*Gionis, A., Indyk, P., & Motwani, R. (1999, September). Similarity search in high dimensions via hashing. In VLDB (Vol. 99, pp. 518-529).

*[LSH算法原理]("http://blog.csdn.net/fuyangchang/article/details/5631547")