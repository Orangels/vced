# CLIP 模型

CLIP模型将原有的图像标签替换为图像的文本描述信息，来监督视觉任务的训练，在下游任务中得到了较好的zero-shot的结果。

该模型将图像分类问题转换为图文匹配问题，在下游任务（以图像分类为例）的实现中，我们需要先基于图像标签集合，构造text prompt并将其通过clip的text encoder获得文本编码向量，之后，将图片通过image encoder获得图像编码向量。

对于每张图片，我们需要计算它与所有text prompt之间的距离（通过计算图像编码向量与文本编码向量之间的余弦相似度或点积得到），选择距离最近的text prompt所对应的标签作为该张图片的分类标签。这样的转换使得模型在zero-shot场景下可以获得与监督学习比肩的效果，除此之外，clip模型可广泛应用到图像检索，视频理解，图像生成等其他领域。接下来我们对CLIP模型进行简单的介绍。

## 原理介绍

### 前置原理介绍

先从简单的CV模型VGG 说起，VGG 模型将多个块进行堆叠，实现了当时人脸识别 state of the art 的结果。

后续研究神经网络模型层间特征图可视化的工作表明，模型最前端的神经网络层倾向于提取一些普遍的、共有的视觉特征，如纹理、边缘等信息。越往后则越倾向于任务相关的特征。涉及计算机视觉图像分类的模型，在特征提取功能上，更多的依赖模型的前端的神经层，而将特征映射到标签的功能则更多的依赖于模型末端的神经层。

也就是说，越靠近输出的神经网络层越具有任务相关性。NLP领域的第三范式(pre-trained + fine tune)便是采用了这种思想。语言模型在大规模语料上进行预训练之后，便具有了强大的语义表征能力。将PLM接上后续任务相关的神经网络层进行微调，便可以在下游任务中获得更好的效果。

一般情况下，倘若我们要实现一个猫狗图像二分类的任务，一个简单的想法就是对猫狗数据集分别打上对应的标签，进行监督学习。但同时，我们也可以通过对比学习的方法来实现：简单来说，就是不再将所有的猫（或狗）的图像通过神经网络分别映射到对应的标签，而是在经过神经网络处理之后，让原本属于同一类别的特征向量之间的距离尽可能接近，让原本不属于同一类别的特征向量之间的距离尽可能拉远。

以DPR(dense passage retrieval)为例，在 in-batch negatives 负采样中，对于单个问题，模型将其对应的标准答案作为正例（距离应尽可能接近），将其他问题的答案作为负例（距离应尽可能拉远）来优化dual encoder，取得了较好的效果。

BERT 模型的预训练任务主要为模拟人类的完形填空任务，在这种预训练方法下，模型需要同时关注上下文间的信息，从而得出当前位置的 token。另一种较强的 NLP模型GPT，则使用了自回归的方法来训练，也就是说，模型仅可通过当前位置之前的字段来推理当前位置的 token。在这之后，UniLM 模型将两种注意力矩阵机制联合起来进行训练。Encoder-Decoder 是 NLP 领域普遍使用的框架，但这块内容与 multimodal 部分关系不大，在此不过多赘述。但可以参考的是，如果我们能够将图像信息和语义信息（或者其他模态的信息）映射到同一空间后，再进行联合优化训练，那么从某种程度上来说来说，模型可以在多种模态之间建立联系。

### Clip原理介绍

Clip(Contrastive Language-Image Pretraining) 将语言信息和图像信息联合训练，实现了在下游任务上 zero-shot（Zero-shot learning 指的是在没有当前类别的训练样本的情况下，让模型学习到一个映射关系可以将这个样本映射到原有的向量空间，再通过向量空间中距离判断的方式推断出当前没有看过的样本可能会属于哪一类别） 的能力。具体来说，CLIP将图像的分类任务转化为了图文匹配的任务，将图像信息和语义信息映射到同一多模态语义空间下，再使用对比学习的方法进行训练。  

在预训练阶段，模型的数据来源为从互联网上搜集的4亿个（图像，文本）对。假设batch中有N个（图像，文本）对，那么我们便可以得到N个文本向量，记为：t1,t2,...,tn,以及N个图像向量，记为img1,img2,...,imgn。我们需要做的就是让t(i)与img(i)之间的语义距离尽可能接近，而与其他的img之间的距离尽可能拉远。如果我们将其看为一个矩阵，其中横轴方向为文本，纵轴方向为图像，且文本、图像的标号均按照次序排列，那么我们就可以得到一个N * N的方阵，方阵的每个元素(i,j)的位置为t(i)与img(j)之间的语义相似度，相似度可通过余弦或点积的方式获得。我们的优化目标就是让对角线上的元素的值尽可能的大，而其余部分的值尽可能地小。  

在zero shot阶段，我们可以根据上下文语义建立prompt模板，并将标签集合映射到该模板中，得到prompt text模板。比方说，现在我们需要做对(大象，冰箱，蚂蚁)的三分类任务，prompt模板为 "这是一张有关{类别}的照片"，将分类标签映射到prompt模板后可以得到集合：{“这是一张有关大象的照片”“这是一张有关冰箱的照片”“这是一张有关蚂蚁的照片”}。对集合中的文本，通过clip的text encoder之后，便可以得到三个类别对应的文本特征向量，之后，对于每一张需要分类的图片，我们只需要比较该图片的特征向量与三个类别对应的文本特征向量之间的语义距离，并选择最近的那一条文本所对应的标签作为图像的分类结果。
具体细节可参考论文《[Learning transferable visual models from natural language supervision](https://arxiv.org/pdf/2103.00020.pdf)》。

### 其他多模态模型介绍

限于篇幅原因，这里仅介绍目前较火的扩散模型(diffusion model)。

在图文生成领域，广为人知的模型有VAE，GAN等。最近大火的diffusion model则使用了一种非常有趣的思想来做生成任务。在此仅对其做简单介绍，不涉及理论公式部分。  

先假设这样一个场景，在咖啡拉花的时候，比如我们现在做了一个图案，假设我们将其放在那里静置，一段时间后图案就会慢慢扩散，最后变得一团糟。如果在扩散过程中，我们一直观察这这杯咖啡，就会发现：在静置的前几分钟，虽然图案已经有了一点点变化，但是我们还是可以很轻松的在大脑中从现有较为模糊的图案中恢复出最开始的图案。随着时间一点点的推移，图案变得越来越模糊，从中恢复出最开始的图案变得越来越困难，最后图案和咖啡混合在一起。

反过来想这个问题，我们能不能从最后的“图案和咖啡混合在一起”的状态，推出最初的图案呢？我们可以将拉花扩散的过程看成是“加噪”的过程，而将反向的恢复过程看成是“去噪”的过程，这样一来，只要我们能够从“加噪”过程中学习到一定的规律，那么我们就有可能在“去噪”过程中“逐步”恢复出最开始的图案。这是对扩散模型的一个非常浅显直观的认识，具体细节可参考相关系列论文。