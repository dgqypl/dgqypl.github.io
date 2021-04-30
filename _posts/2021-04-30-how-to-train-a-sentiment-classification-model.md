---
layout: post
title: 如何训练一个情感分类模型？
subtitle: 
tags: [机器学习,NLP]
---

在自然语言处理（Natural Language Processing）中，情感分类（Sentiment Classification）一般是指判断一段文本所表达的情绪状态，它可以是两类，如正面/负面，也可以是三类，如积极/消极/中性等。情感分类的应用场景十分广泛，如可以抓取用户在电商网站（淘宝/京东/亚马逊）、美食网站、电影网站上发表的评论并进行情感分类来获得用户对于某一产品的整体使用感受，进而通过调整产品的功能或销售策略等以提升产品的价值。

那如何搭建这样一个分类识别的应用系统，帮助我们实现上述的功能呢？本文以一个电影评论的例子，带你了解如何通过使用DL4J框架来训练一个情感分类模型。

<br>

本文涉及到深度学习中关于自然语言处理的相关理论知识，包括但不限于[循环神经网络](https://zh.wikipedia.org/wiki/%E5%BE%AA%E7%8E%AF%E7%A5%9E%E7%BB%8F%E7%BD%91%E7%BB%9C)（Recurrent neural network）、[LSTM](https://zh.wikipedia.org/wiki/%E9%95%B7%E7%9F%AD%E6%9C%9F%E8%A8%98%E6%86%B6)（Long Short-Term Memory）、[词嵌入](https://zh.wikipedia.org/wiki/%E8%AF%8D%E5%B5%8C%E5%85%A5)、[Word2vec](https://zh.wikipedia.org/wiki/Word2vec)，因此你需要对以上的这些有一定的了解，才能更好的理解本文接下来所讲的内容。

由于我们会使用DL4J来训练一个情感分类模型，那么首先我会先来介绍一下它是什么。

[DL4J](https://deeplearning4j.org/)（Deeplearning4j）是一个基于Java语言的开源的、分布式的深度学习框架。它利用最新的分布式计算框架Spark和Hadoop来加速对模型的训练。在多GPU环境下，其性能媲美[Caffe](https://caffe.berkeleyvision.org/)。另外，该框架是完全开源的（遵循了[Apache License 2.0](https://github.com/eclipse/deeplearning4j/blob/master/LICENSE)），并由开发者社区和Konduit团队共同维护。

框架本身也兼容任何JVM语言，如Scala、Clojure或Kotlin。其底层计算是用C、C++和Cuda编写的。在对其他语言的支持方面，DL4J可以通过Keras来读取其他开源框架（例如TensorFlow、Theano）生成的模型，而无需浪费精力重写模型。

接下来，本问以使用Maven管理的项目为例（使用其他包依赖管理工具的可以自行类比），来讲一下如何在项目中引入DL4J，为后面的使用做准备。

由于DL4J支持使用CPU/GPU进行模型的训练，因此对于不同的机器，需要在pom.xml中引入的依赖也有所不同。

1、当使用CPU时，需要在pom.xml中引入以下部分：
```xml
<dependency>
      <groupId>org.nd4j</groupId>
      <artifactId>nd4j-native</artifactId>
      <version>1.0.0-beta7</version>
</dependency>
<dependency>
      <groupId>org.nd4j</groupId>
      <artifactId>nd4j-native</artifactId>
      <version>1.0.0-beta7</version>
      <classifier>macosx-x86_64-avx2</classifier>
</dependency>
```
> \<classifier\>标签中的内容，根据操作系统对AVX（Advanced Vector Extensions）的支持，可用值如下：<br>
> Generic x86 (no AVX): linux-x86_64, windows-x86_64, macosx-x86_64<br>
> AVX2: linux-x86_64-avx2, windows-x86_64-avx2, macosx-x86_64-avx2<br>
> AVX512: linux-x86_64-avx512<br><br>
> 注：AVX是指一组能够加速数值计算的CPU指令集。

2、当使用GPU时（需要在装有CUDA v9.2+和兼容NVIDIA的硬件上），pom.xml中引入以下部分：
```xml
<dependency>
        <groupId>org.nd4j</groupId>
        <artifactId>nd4j-cuda-10.1-platform</artifactId>
        <version>1.0.0-beta7</version>
</dependency>
<dependency>
        <groupId>org.deeplearning4j</groupId>
        <artifactId>deeplearning4j-cuda-10.1</artifactId>
        <version>1.0.0-beta7</version>
</dependency>
<dependency>
        <groupId>org.bytedeco</groupId>
        <artifactId>cuda-platform-redist</artifactId>
        <version>10.1-7.6-1.5.2</version>
</dependency>
```
> 关于artifactId可用的CUDA版本有：nd4j-cuda-9.2-platform, nd4j-cuda-10.0-platform, nd4j-cuda-10.1-platform, nd4j-cuda-10.2-platform<br>
> 关于cuda-platform-redist[可用的版本](https://repo1.maven.org/maven2/org/bytedeco/cuda-platform-redist/)有：10.1-7.6-1.5.2, 10.2-7.6-1.5.2, 10.2-7.6-1.5.3, 11.0-8.0-1.5.4

除了上面引入的（不同）依赖外，（无论是CPU还是GPU）还需要引入一些通用的依赖：
```xml
<dependency>
    <groupId>org.deeplearning4j</groupId>
    <artifactId>deeplearning4j-core</artifactId>
    <version>1.0.0-beta7</version>
</dependency>
<dependency>
    <groupId>org.deeplearning4j</groupId>
    <artifactId>deeplearning4j-ui</artifactId>
    <version>1.0.0-beta7</version>
</dependency>
```

至此，我们已经成功的把DL4J框架引入我们的项目中。接下来，我会以一个Imdb电影网站的例子，讲一下如何使用DL4J来训练一个情感分类模型。最终要达成的结果是，给定一段电影评论文本，例如：
```text
Hated it with all my being. Worst movie ever. Mentally- scarred. Help me. It was that bad.TRUST ME!!!
```

模型将识别出它是一个负面评论：
```text
p(positive): 0.11769601702690125
p(negative): 0.8823040127754211
```
> 即正（positive）样本的概率约为0.12，负（negative）样本的概率约为0.88。因此模型将认为它是一个负面评论。

<br>

我们将使用RNN的方式来训练，训练该模型的输入是**词向量文件**+**电影评论文本**，输出是上面的关于正样本/负样本的概率。

为了训练模型，首先我们来进行词向量的训练。我们可以使用爬虫或者其他方式从网站上收集到数据（做为训练集），然后使用以下4步来训练词向量：

1、对训练集中的每行文本进行预处理（在这个例子中就是指一行电影评论文本），定义分词工厂及预处理器：
```java
TokenizerFactory t = new DefaultTokenizerFactory(); // 使用默认的分词工厂，通过分隔空格来获得单词
t.setTokenPreProcessor(new CommonPreprocessor()); // 设置token的预处理器，CommonPreprocessor()会去掉语句中所有的数字、标点符号和一些特殊符号，并将所有字母变为小写
```
过程如下图所示：
![](../assets/images/2021-04-30-how-to-train-a-sentiment-classification-model/%E9%A2%84%E5%A4%84%E7%90%86.png)
2、定义数据迭代器，在训练过程中会不断的通过此迭代器从训练集中读取数据：
```java
SentenceIterator iter = new BasicLineIterator(filePath); // 使用DL4J框架内置的基于行读取的迭代器
```
3、定义Word2Vec对象并开始训练词向量：
```java
Word2Vec vec = new Word2Vec.Builder()
	.minWordFrequency(5) // 单词在训练集中最少出现的次数，低于此值的单词在训练开始之前就会被移除
	.iterations(1) // 每个mini-batch在训练时的迭代次数
	.layerSize(300) // （输出的）词向量的维度数
	.windowSize(5) // skip-Gram的上下文大小
	.iterate(iter)
	.tokenizerFactory(t)
	.build();

vec.fit();
```
4、保存词向量文件，为后续训练情感分类模型所用：
```java
WordVectorSerializer.writeWord2VecModel(vec, "pathToSaveModel.txt");
```

至此，词向量就训练完毕了。接下来，我们来训练模型，这里的**关键是要自定义数据迭代器**，**将训练集结合前一步生成的词向量构建模型的特征**。

我们将训练词向量时预处理后的文本，对其中的每个单词判断其是否有词向量。通过这一步拿到所有有词向量的单词，用于构建特征：
![](../assets/images/2021-04-30-how-to-train-a-sentiment-classification-model/%E6%9E%84%E5%BB%BAfeatures.png)

实现上述过程的代码需要自定义一个SentimentIterator，实现DataSetIterator接口并覆写next(int num)方法，这个方法的入参是mini-batch的电影评论文本，返回值是一个DataSet对象，它有4个参数：features，labels，featuresMask，labelsMask。接下来我仅展示代码逻辑的核心部分（不必要的代码以...省略）：

1、预处理文本并找到文本中有词向量的单词：
```java
List<List<String>> allTokens = new ArrayList<>(mini-batch.size());
int maxLength = 0;

for(int i = 0; i < mini-batch.size(); i++){
    String text = mini-batch.get(i); // 原始文本
    List<String> tokens = tokenizerFactory.create(text).getTokens(); // 预处理之后的单词List
    List<String> tokensFiltered = new ArrayList<>();
    for(String t : tokens ){
        if (word2vec.hasWord(t)) tokensFiltered.add(t); // 对每个单词判断其是否有词向量
    }
    allTokens.add(tokensFiltered);
    maxLength = Math.max(maxLength,tokensFiltered.size());
}

if(maxLength > truncateLength) maxLength = truncateLength; // truncateLength代表文本的最大长度限制，如果超过则会被截断
```

2、初始化返回值参数对象：
```java
INDArray features = Nd4j.create(new int[]{mini-batch.size(), word2vecOutputSize, maxLength}, ...);
INDArray labels = Nd4j.create(new int[]{mini-batch.size(), 2, maxLength}, ...); // 两类标签，正面评论/负面评论
INDArray featuresMask = Nd4j.zeros(mini-batch.size(), maxLength); // 对features做pad，初始化值全为0
INDArray labelsMask = Nd4j.zeros(mini-batch.size(), maxLength); // 对labels做pad，初始化值全为0
```

3、填充features，labels，featuresMask，labelsMask的值：
```java
for( int i=0; i<mini-batch.size(); i++ ){
    List<String> tokens = allTokens.get(i); // 文本所有有词向量的单词
    int seqLength = Math.min(tokens.size(), maxLength);
    final INDArray vectors = word2vec.getWordVectors(tokens.subList(0, seqLength)).transpose(); // 构建features

    features.put(new INDArrayIndex[] {
            NDArrayIndex.point(i), NDArrayIndex.all(), NDArrayIndex.interval(0, seqLength)}, vectors);
    featuresMask.get(new INDArrayIndex[] {NDArrayIndex.point(i), NDArrayIndex.interval(0, seqLength)}).assign(1); // 将每个feature(即单词)对应的位置都置为1

    int idx = (positive[i] ? 0 : 1); // 将文本对应的标签值赋值给idx，如果是正面评论，将idx设置为0；如果是负面评论，将idx设置为1
    int lastIdx = Math.min(tokens.size(),maxLength);
    labels.putScalar(new int[]{i,idx,lastIdx-1},1.0); // 设置labels：将最后一个feature(即单词)对应的输出置为idx，并使用1.0来标记该位置
    labelsMask.putScalar(new int[]{i,lastIdx-1},1.0);
}
```

4、最后返回DataSet：
```java
return new DataSet(features,labels,featuresMask,labelsMask);
```

至此，我们就把含有特征的数据迭代器SentimentIterator实现完毕了。接下来，我们使用一个两层的RNN模型，将SentimentIterator做为模型的输入，来开始训练：

1、初始化模型：
```java
int batchSize = 64;
int vectorSize = 300; // 即为词向量的维度数
int nEpochs = 1;
int truncateLength = 256; // 文本的最大长度限制，如果超过则会被截断

MultiLayerConfiguration conf = new NeuralNetConfiguration.Builder()
    .updater(new Adam(5e-3)) // 使用Adam算法做为所有层的默认梯度更新器
    .l2(1e-5) // l2正则系数
    .weightInit(WeightInit.XAVIER) // 使用Xavier方法初始化权重
    .gradientNormalization(GradientNormalization.ClipElementWiseAbsoluteValue).gradientNormalizationThreshold(1.0) // 梯度归一化策略，ClipElementWiseAbsoluteValue表示当梯度值在threshold之外时，会被裁剪为threshold值
    .list()
    .layer(new LSTM.Builder().nIn(vectorSize).nOut(256) // 定义模型的第一层，使用LSTM构建
        .activation(Activation.TANH).build())
    .layer(new RnnOutputLayer.Builder().activation(Activation.SOFTMAX) // 定义模型的第二层（输出层）
        .lossFunction(LossFunctions.LossFunction.MCXENT).nIn(256).nOut(2).build())
    .build();

MultiLayerNetwork net = new MultiLayerNetwork(conf);
net.init();
```

2、读取词向量并构建训练集和测试集的数据迭代器：
```java
WordVectors word2vec = WordVectorSerializer.loadStaticModel(new File(wordVectorsPath));
SentimentIterator train = new SentimentIterator(DATA_PATH, word2vec, batchSize, truncateLength, ...);
SentimentIterator test = new SentimentIterator(DATA_PATH, word2vec, batchSize, truncateLength, ...);
```

3、开始训练：
```
net.setListeners(new ScoreIterationListener(1), new EvaluativeListener(test, 1, InvocationType.EPOCH_END)); // 输出每次迭代后的Loss值，并在epoch最后评估模型在测试集上的表现
net.fit(train, nEpochs); // 开始训练模型
```

模型训练完毕后，我们可以在控制台上看到模型在测试集上的表现：
```text
========================Evaluation Metrics========================
 # of classes:    2
 Accuracy:        0.8608
 Precision:       0.8657
 Recall:          0.8542
 F1 Score:        0.8599
Precision, recall & F1: reported for positive class (class 1 - "1") only


=========================Confusion Matrix=========================
     0     1
-------------
 10843  1657 | 0 = 0
  1823 10677 | 1 = 1

Confusion matrix format: Actual (rowClass) predicted as (columnClass) N times
==================================================================
```

> 可以看到模型的F1值约为0.86，如果要提升模型的性能，可以对参数进行调试、训练更长时间或者使用更大的训练集。

模型训练完毕后，我们可以使用如下的方法来对电影评论文本进行预测：
```java
INDArray features = test.loadFeaturesFromString(reviewText, truncateLength); // 该方法的逻辑也是将文本转换成features
INDArray networkOutput = net.output(features); // 根据features进行预测
long timeSeriesLength = networkOutput.size(2); // 获取time step长度
INDArray probabilitiesAtLastWord = networkOutput.get(NDArrayIndex.point(0), NDArrayIndex.all(), NDArrayIndex.point(timeSeriesLength - 1)); // 获取最后一个time step的输出，也即为模型的预测值

System.out.println("Short negative review: \n" + reviewText);
System.out.println("\n\nProbabilities at last time step:");
System.out.println("p(positive): " + probabilitiesAtLastWord.getDouble(0));
System.out.println("p(negative): " + probabilitiesAtLastWord.getDouble(1));
```

最终你将看到如本文一开始的输出结果：
```text
Short negative review:
Hated it with all my being. Worst movie ever. Mentally- scarred. Help me. It was that bad.TRUST ME!!!


Probabilities at last time step:
p(positive): 0.11769601702690125
p(negative): 0.8823040127754211
```

<br>

总结

<br>
到此，我们使用DL4J框架完成了一个情感分类模型的训练，你可以根据业务的需要使用它来快速在你的项目中训练一个情感分类模型。最后我来给你总结一下：这篇文章我先通过介绍了DL4J的概念，让你对它有了一定的认识，接下来我向你讲述了在项目中引入DL4J的方法。最后，我以一个电影评论的例子，详细为你讲述了使用DL4J训练一个情感分类的模型。我把文中的知识点汇成下面一张图，你可以参考一下：

![](../assets/images/2021-04-30-how-to-train-a-sentiment-classification-model/%E6%80%9D%E7%BB%B4%E5%AF%BC%E5%9B%BE.png)

另外，本文所展示的代码完整版你可以到[这里](https://github.com/dgqypl/dl4jSentimentClassification)去查看和下载。

最后，本文的讲解对于深度学习框架DL4J仅是一个抛砖引玉，它除了可以自定义CNN、RNN等网络，也内置了许多诸如[LeNet](https://github.com/eclipse/deeplearning4j/blob/master/deeplearning4j/deeplearning4j-zoo/src/main/java/org/deeplearning4j/zoo/model/LeNet.java)、[AlexNet](https://github.com/eclipse/deeplearning4j/blob/master/deeplearning4j/deeplearning4j-zoo/src/main/java/org/deeplearning4j/zoo/model/AlexNet.java)、[VGG-16](https://github.com/eclipse/deeplearning4j/blob/master/deeplearning4j/deeplearning4j-zoo/src/main/java/org/deeplearning4j/zoo/model/VGG16.java)、[ResNet](https://github.com/eclipse/deeplearning4j/blob/master/deeplearning4j/deeplearning4j-zoo/src/main/java/org/deeplearning4j/zoo/model/ResNet50.java)等经典的网络实现，如果有兴趣，你可以自己去研究查看。