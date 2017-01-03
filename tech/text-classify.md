# 文本分类问题解决步骤

python版使用svm算法对短文本进行分类的样例：https://github.com/IBBD/short-text-svm


## STEP0: 了解业务背景，数据产生逻辑

这是问题解决的基础步骤，不详述。

## STEP1: 获得训练集

如果已经有一个质量好的样本训练集，可跳过这个步骤。

事实上，当我们需要对一大堆文本进行分类的时候，通常是没有样本数据的，只有一大堆文本。要想继续，则必须先得到一个质量比较好的样本数据集。

这里有两个方法： 

1. 直接人工标注的方式 
2. 人工提取明显的分类特征，使用正则进行批量标注

第一种纯人工标注，这是一种笨方法，效率很低，而且效果不见得好，因为不同的人，会有不同的判别标准，就算同一个人，前后的标准也比较难保持一致的，当要标注大量文本的时候。

第二种属于半人工半自动的方式，边阅读文本边提取特征，并对提取的在全局的范围内进行校验（使用正则就能将满足该特征的文本全部找出来），如果该特征下的文本都是ok的，则将该特征和该分类进行关联。例如对于客户的投诉文本，得到的特征的格式可能如下：

```python
{
    "xiaotou": u"小偷|治安|被盗",
    "jiakao": u"驾考|驾照考试|驾照约考|考.{0,5}驾照|驾校",
    "xiaofei": u"退货|退钱|退款|退还定金|不打表|出租车拼客|退还货款|退订金|霸王条款|欺诈消费者|欺詐消費者|欺诈销售|强买强卖",
    "wangluo": u"电信.{0,3}收费",
}
```

说明：例如对于`xiaotou`这个分类，只要文本中出现`小偷`，`治安`，`被盗`这三个关键词中的一个，我们就认为该文本属于`xiaotou`这个分类。

而这个正则`电信.{0,3}收费`在匹配时电信和收费这两个关键词，并且中间可以存在0-3个字，例如可以匹配`电信收费`，`电信乱收费`，`电信胡乱收费`等。

python实现正则匹配的方法如下：

```python
import re

comp = re.compile(r"(%s)" % pattern)
groups = re.findall(comp, text)
if groups and len(groups) > 0:
    # 满足条件，可以给该文本划到相应的分类
```

注意：

- 特征的提取必须是比较明显的，宁愿少取特征，也要避免加入错误的分类数据（虽然这个很可能是避免不小的，就算纯人工识别，也很难避免）
- 对于提取的分类特征，校验是非常重要的
- 对于同一个文本，通过特征匹配之后，我们只取只满足单个分类的文本（正则匹配的时候，同一个文本可能会被匹配到多个分类，这类文本不要作为样本，因为特征不明显）

## STEP2: 选择分类算法，对样本进行训练

分类的算法有很多，大多也是比较成熟的，例如贝叶斯算法，SVM算法等，不过为了得到更有优化的效果，通常需要不断的对算法参数进行微调，对特定领域进行不断适应，才能得到一个比较的训练结果。

训练最后得到的训练模型通常保存在文件中，后面做预测时使用。

## STEP3: 利用训练好的模型，对样本数据进行预测，判断正确率

用训练好的模型对样本进行预测，看和已经标注的分类是否吻合，对于不吻合的数据，提取出来，判断是标注的分类错了，还是预测的分类错了。

- 如果是标注的分类错了，可能就得调整特征，重新标注文档。
- 如果是预测的分类错了，这是允许的，只要错误率保持在一个比较低的水平。

不断迭代step1到step3，以提升样本的质量，和预测的效果。

预测时，如果有一些文本虽然预测值和标注值一致，但是预测值的置信度并不高，这类数据也可以列入观察的范围，也可能是样本原因导致的。

注：这里并不是追求预测准确100%，因为那样可能会导致过拟合。

## STEP4: 对剩余未标注的数据进行预测

终于到了最后的分类预测，不过这也是一个不断迭代优化的过程。对于预测结果中的一些置信度不高的数据进行观察，看看是否可以提取相应的特征（对于提取的特征需要进行校验），如果可以，则加入相应的特征。

不断迭代step1到step4，直到达到一个比较好的效果。

## STEP5: 评估分类的效果

这里评估，只能对数据进行抽样分析。。。

## 附录：对于区分标题和内容的文本分类

在实际应用中，很多分类都是包含标题和内容这两种类型的，例如新闻或者文章等，就很明显有标题和内容这两个字段。

这时，分类过程中，可能会出现这样的情况：根据标题预测的分类和根据内容预测的分类不同，这时怎么平衡？

对于新闻或者文章，通常比较容易取舍，因为通常以内容字段为主，因为内容的信息量比标题大很大。这时标题可以用作校验。

但是对于短文本，例如投诉或者评论，有标题和内容两个字段，但是内容字段的信息可能未必比标题大，这时就要考虑了。通常可以采取这样的策略（简单做法）：

- 默认以标题的分类为主，因为包含的信息通常是比较明确的，而内容的信息反而可能噪音比较多。当然这个实际问题还得实际分析。
- 但是标题有时也是基本没有意义的（置信度比较低），这时应该改用内容字段的分类，如果内容分类的置信度比标题的置信度高（这里可以设置阀值，高出百分之多少才算有效）
- 标题的分类和内容分类不一致的，可以人工进行检验，看看分类的效果哪个好。

