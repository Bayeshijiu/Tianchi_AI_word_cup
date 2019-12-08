# Tianchi_AI_word_cup
天池/AI Word Cup-2018世界杯新闻智能创作极限挑战赛/ 冠军

## 赛题分析

  比赛提供历史足球比赛新闻相关数据作为训练写作模型的依据，要求在复赛阶段实时获取新闻事件及图集数据，并在极短时间内产出新闻。开放性赛题不限定新闻的形式，只要产生符合事实的世界杯比赛相关新闻。

  历史新闻数据的格式为新闻标题、文本、实体名称、关键字等构成的非结构化文本，实时新闻事件的格式为动作、时间、人物等关键信息构成的结构化文本。需要利用历史新闻数据和实时新闻事件，产生符合事实的世界杯足球比赛新闻。

  根据对比赛方提供的数据进行分析，我们认为本次比赛有三个难以解决的问题：

1. 本赛题要解决的问题是基于结构化数据的文本生成，即测试时输入结构化的数据，模型输出描述和解释结构化数据的文本。

2. 训练提供的历史新闻数据为非结构化文本，并没有与之对应的历史新闻的结构化文本数据。

3. 生成新闻需基于数据和事实。

## 算法思路

  新闻生成主要有抽取式和生成式两种方式，抽取式是从历史新闻（非结构化数据）中抽取文本作为文档摘要，构建新的稿件。生成式需基于结构化数据或语义表达生成新的语句，产生实时的新闻报道。

  本赛题我们团队采用了模板、摘要提取、seq2seq三种方法实现新闻生成。


### 一、模板生成

  这是最容易想到的方法，实现起来也最简单，针对数据表格每一行，提取描述事件的主要字段，类似填空一样，填入构造的模板中，如：时间+人物+事件，即可生成最简单的实时战报。同时，这样生成的新闻看起来也更单调，缺乏变化。在此不再对基于规则的方法进行赘述。

### 二：摘要提取

  根据历史新闻数据信息，直接从非结构化文本中提取相关已有的历史新闻信息生成新闻。

1. 算法思路：
    历史新闻文本里边有大量的数据，包括各队的一些球队新闻、历史比赛信息，也包括一些球员的个人新闻等等。我们希望能够利用这些历史数据，使得产生的新闻能够更加丰富，而不是仅仅局限于对比赛的实时报道。

    所以我们通过提取与实时比赛相关的历史文本，提取原历史新闻里的关键句，作为对整篇新闻的总结摘要。最后将此摘要作为新产生的一整篇新闻或者新闻里的一部分。

2. 算法描述
    由于数据的局限性，我们决定采用无监督算法TextRank，一种用于文本的基于图的排序算法，其基本思想来源于PageRank，具体流程如下。

textrank

3. 算法难点
(1) 关键词提取：
  容易想到将国家队名、重要球员名作为关键词，然后去历史新闻里（标题/内容/实体/关键词）去匹配。这样确实匹配到很多的新闻，但是质量往往很差。 

  在此基础上，我们进一步筛选出一些关键词，如[‘点评’、 ‘看点’、 ‘攻略’、 ‘历史’、 ‘简介’、 ‘巡礼’…….]，将其与之前的关键词结合匹配，可匹配到质量不错的历史文章。

(2) 数据清洗：
  原历史新闻文本里有大量的噪声数据，如果直接用TextRank提取摘要时，很容易将一些噪声干扰句作为关键句，所以有必要在使用算法前先对数据进行清洗。数据清洗我们主要通过正则匹配、PPL进行。

4. 结果分析
- 产生新闻的形式较为灵活，可以产生独立的文章作为赛前导读，也可以是其中的一个段落作为对比赛的分析。

- 可以事先定义句数，对新闻篇幅有一定的控制。

- 历史新闻中提取的语句，产生的新闻真实度、可靠性高。

### 三、新闻补全模型

结合新闻事件信息和历史新闻数据生成新闻，根据给定的事件数据生成基本的事实文本，用训练出的神经网络模型基于事实生成对事件的描述。

1. 算法思路：
  由于训练数据与测试数据的类型不统一，使用历史新闻训练时面对的是非结构化文本，而测试时的事件信息为结构化数据，无法让模型学习直接通过结构化数据生成新闻。同时，由于新闻的生成需要基于新闻事件提供的真实信息，而基于神经网络的文本生成算法缺乏对新闻可靠性的约束。

  为了让两个数据集的格式保持一致，同时充分利用新闻事件提供的信息，我们对训练数据和测试数据分别进行处理，使它们都转化为缺乏具体描述的非结构化文本。从而将新闻生成问题视为对新闻中基本信息进行自动补全，使整个问题转化为一个序列到序列（seq2seq）问题。

model

2. 数据处理：
  要将两个数据集都转化为缺乏具体描述的非结构化文本，需要对历史新闻数据进行删减，实时新闻事件的数据经过变换之后拼接成非结构化文本。

图片1

  对历史新闻数据，我们采用的方法是按照词性来筛选，将其中的描述性词语替换成一个固定的tag，作为训练模型的源端数据，而未经过处理的原始新闻作为训练的目标端数据。对实时新闻事件数据，需要将具体的数据按照语言描述的顺序拼接起来，同时加入代表占位符的tag，作为最终的测试数据。
  
3. 算法模型：
  基于注意力机制的Seq2seq模型成功应用到了自然语言处理的多个领域，在机器翻译、文本摘要、对话生成等任务中大放异彩，模型主要有三大模块，编码器、解码器和注意力模块。我们采用的模型正是RNNSearch模型，采用两层RNN作为解码器，在注意力模块生成语义编码的时候可以参考更加丰富的目标端信息，在生成新闻时可以对模型有进一步的约束。

4. 关键技术：
  在新闻数据中，有大量的命名实体（球员、球队名称、地名），UNK问题严重，命名实体的生成不准确。为了解决这个问题，我们采用了BPE编码来达到减小词汇表的目的。

  在数据处理时有两种方法，第一种是将历史原文中的描述性语句替换为tag，另一种是直接删掉，测试数据也保持同样的形式。加入占位符，产生新闻的质量较高，但是测试时从结构化数据转换的时候可能有多种形式，需要通过对比筛选出效果较好的一种。去掉占位符，生成新闻的质量会受到一定影响，但在测试的时候将测试数据转化为非结构化数据比较容易。在本次大赛中我们采用的模型是加入占位符的方法，为了避免模型学习到一个占位符对应一个单词的模式，训练数据中对相邻占位符合并。

5. 结果分析：
- 产生的新闻灵活、丰富，变化性高。

- 模型自动学习到对新闻事件的具体描述，不需要人工提取特征。

- 复用性强，不仅可以用作从结构化文本中生成对应的描述新闻，也可以对非结构化数据进行文本复述、改写。

- 训练的成本较高，训练时间长。

## 比赛感想

  这次比赛我们最大的收获就是科研和实际应用问题有大不同，如何把实际问题转化为一个可解决的问题是关键。另外，一定要深入理解问题，保持概念清晰，灵活运用大数据，模型只是一种工具，如何将模型与数据结合产生更好的结果才是最终的问题。最重要的是，要多尝试解决问题的不同方法，思考的方向会不同，每个方向的深入挖掘，会对问题产生更深度的理解。

  感谢主办方给予一次难得的学习、爱好、实战共存的机会，也希望天池大数据竞赛能够越办越好！
