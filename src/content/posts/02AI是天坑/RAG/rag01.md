# RAG01

Retrieval-Augmented-Generation（检索增强生成）

---

**先从资料库里检索相关内容**

**再基于这些内容来生成答案**

---

## 实现原理

分片 → 索引 → 召回 → 重排 → 生成

&nbsp;

---

提问前链路

分片 → 索引

---

提问后链路

召回 → 重排 → 生成

&nbsp;

## 分片

把文档切割成多个片段，可以使用多种切分方式，如按页码、段落

&nbsp;

## 索引

1. 通过**Embedding**将片段文本转换为**向量**
2. 将片段文本和片段向量存入**向量数据库**中

&nbsp;

### 向量

<img src="../../assets/cf63513a-1a85-48f1-bdb8-5fc555629bb6.png" alt="image.png" width="517" height="70">

<img src="../../assets/a339868a-35d2-42a9-a3ac-a7c1d75c1e45.png" alt="image.png" width="405" height="212">

<img src="../../assets/e521dcde-59c5-4ef8-b4ff-841e8408eaf7.png" alt="image.png" width="365" height="215">

&nbsp;

### Embedding

<img src="../../assets/5f55f973-4245-40eb-945e-f2cb4b0604d3.png" alt="image.png" width="465" height="316">

[https://huggingface.co/spaces/mteb/](https://huggingface.co/spaces/mteb/)

&nbsp;

### 向量数据库

用于存储和查询**向量**的数据库

![image.png](../../assets/32e01820-7268-40cc-b100-1b50b70f2d38.png)

![image.png](../../assets/4bf195e9-bf6a-49f1-b2a5-0879802141a2.png)

&nbsp;

## 召回

搜索与用户问题相关的片段

![image.png](../../assets/010de908-163a-405e-b8fa-6266f274b817.png)

计算向量相似度

- 余弦相似度
- 欧氏距离
- 点积
- …

&nbsp;

## 重排

根据召回结果进行重拍，得到重排结果

&nbsp;

&nbsp;

## 生成过程

<img src="../../assets/c334ee95-9aed-4aca-b041-62fc3399de65.png" alt="image.png" width="655" height="213">

&nbsp;

## 整体流程

准备部分（提问前）

<img src="../../assets/6a186df0-d299-4696-96b7-c951056499bc.png" alt="image.png" width="812" height="147">

回答部分（提问后）

<img src="../../assets/3216f677-d9f3-4447-bf46-5e9ac9d6317e.png" alt="image.png" width="669" height="278">

&nbsp;

&nbsp;