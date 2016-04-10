---
layout: post
title: "实现短文本相似匹配的简单方式"
description: ""
category: "work"
tags: [simihash, edit-distance, ]
published: true
---

最近做的爬虫需要加一个功能，能够根据标题尽量的去除重复的新闻，新闻标题的长度大多在 10字 以内，一般最大不超过20字。


网上找了一下，找到一下几种比较方式。


1. [最短编辑路径](http://www.programcreek.com/2013/12/edit-distance-in-java/)

2. [基于词袋](http://jacoxu.com/?p=344)

3. [simi hash + 海明距离](http://www.lanceyan.com/tech/arch/simhash_hamming_distance_similarity.html)

4. [余弦相似](http://www.ruanyifeng.com/blog/2013/03/cosine_similarity.html)



其中 1 不需要分词，2，3，4 都需要分词的支持。以下逐一分析他们的好与不好

### 最短编辑路径

他的原理是: 计算从 `string1` 最少修改几个字符可以变成 `string2` 由于文本都比较短，所以当初最先想到的最简单的方式就是通过此方法。

然而这种方式有一定的不足。

举一个极端的例子

	[冠军欧洲]特别企划：逆转之态 欧冠小百科
	[冠军欧洲]特别企划：逆转之态 欧文小百科

上面的 距离只有1， 但是明显是2个不同的内容。所以这种情况会被误判。


### 基于词袋

词袋 的原理是将2句句子分词，变成 2个集合 A, B，然后分别算 $$ \frac { A } {A \bigcup B} $$ 和 $$ \frac { B } {A \bigcup B} $$ 的值。

这个的值 越接近1，越相似.

可以想象，只要 `欧冠` 和 `欧文` 都在字典内，则会发生和上面一样的问题。


**所以，光分词还是不够，我们需要某种方式，给词加些权重，让`欧冠` 和 `欧文` 权重不同 是不是可以解决了？**


### simi hash + 海明距离

网上都说 simi hash 是Google 用来对网页去重的算法。

他的原理是将字符串分词后，将每个词 变成 n 个长度的 二进制串 (1010101010)，然后将 **每个 0 变成 －1**， 然后根据每个词的权重，给每个词乘上权重，比如 (1 -1 1 -1)  => *4 =>  (4 -4 4 -4), 然后将每个词 叠加后降维重新变成 二进制串 (1010)...

如果没看懂的话可以看[这篇文章](http://www.lanceyan.com/tech/arch/simhash_hamming_distance_similarity.html) 有图，更直观.

所以可以想象，他的作用是将一个网页，变成了个 n个长度的二进制串，然后比较 二进制串 比较相似。这种做法是否适合我们的场景呢？

我做了如下测试

用了 jieba 分词，并基于他的默认的 idf 词库打分, 取得bit长度是128。


	String str2 = "[冠军欧洲]特别企划：逆转之态 欧冠小百科"; 
	String str3 = "[冠军欧洲]特别企划：逆转之态 姚明小百科"; 

2 和 3 的距离是 8

	String str5 = "篮球贡献获世界肯定 姚明将入选NBA名人堂 晚间体育新闻";
	String str6 = "姚明已确定入选2016届名人堂 午间体育新闻";
	String str7 = "篮球贡献获世界肯定 姚明将入选NBA名人堂";

5 和 6 的距离是36

5 和 7 的距离是0 

发现，结果有点走极端。。。只有0的才是对的，但有些确实是相似的，却无法识别。如果把 可接受距离设置成 10的话，那么 str2 和 str3 就会被误判。

由于我的业务，并不考虑任何性能因素。所以最后试了一下结果更平滑的 `余弦相似`

### 余弦相似

具体分析可以参考 [ruanyifeng 的文章](http://www.ruanyifeng.com/blog/2013/03/cosine_similarity.html)

简单来说就是 利用了余弦定理可以扩展到 `n维` 的特性，然后将加权后的分词列表 变成 2 个 n维向量代入计算。。

同 simi hash， 测试了一下


	String str1 = "[冠军欧洲]特别企划：狭路相逢 球队交锋史";
	String str2 = "[冠军欧洲]特别企划：逆转之态 欧冠小百科";
	String str3 = "[冠军欧洲]特别企划：逆转之态 姚明小百科";
  
	String str5 = "篮球贡献获世界肯定 姚明将入选NBA名人堂 晚间体育新闻";
	String str6 = "姚明已确定入选2016届名人堂 午间体育新闻";
	String str7 = "篮球贡献获世界肯定 姚明将入选NBA名人堂";
	String str8 = "美媒：篮球明星姚明将入选美国NBA名人堂";
	String str9 = "姚明已确定入选2016届名人堂";


 	这里我们和上面的 `simi hash` 得出的结果坐一下对比 (带 `=` 号的是余弦相似的结果)


	26
	=0.7036674410874154
	18
	=0.6121186320747724
	8
	=0.5967533823473924


	36
	=0.47578567188668835
	0
	=0.9858871667757847
	24
	=0.695671087790831
	36
	=0.4171801708243138


排错率提高了不少。实测发现，他还有一些不足，举个极端的例子，如果 有2个标题


	冠军/欧洲  [5,10]

	篮球/公园  [5,10]


由于标题短，所以容易发生权重相同的情况。。。


实际例子： ```[克洛普“回家” 多特蒙德战平利物浦] - [韩国：韩军举行飞行员救援演习]``` 这2个标题的相似度是 99%



### 解决方法

通过实际的结果发现，很多这种情况的标题大都不大相似，原因和基于 [TF-IDF](http://www.ruanyifeng.com/blog/2013/03/tf-idf.html) 词库有关. 

所以我们完全可以将 余弦算出的结果 再做一次 `最短编辑路径` 将误判结果排出就行了。

实际效果不错。。。


### TODO

构造自己的 [TF-IDF 词典](http://www.ruanyifeng.com/blog/2013/03/tf-idf.html).

[matrix67 的新词发现算法](http://www.matrix67.com/blog/archives/5044) 

实现 https://github.com/sing1ee/dict_build

不过实际测试时用了178M 300w 条新闻标题 stackoverflow了 ＝ ＝


## reference

http://www.lanceyan.com/tech/arch/simhash_hamming_distance_similarity.html

http://jacoxu.com/?p=344


[edit-distance 多种算法实现](https://github.com/rrice/java-string-similarity)

[edit-distance commons lang 实现] `StringUtils.getLevenshteinDistance`







