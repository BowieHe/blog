---
title: 本地知识库：基于Qdrant的向量化和搜索
date: 2026-01-15 17:06:33
updated: 2026-01-15 17:06:33
categories:
    - 教程
tags:
    - LLM
    - 本地知识库
comments: true
description: 本地知识库搭建之向量化和搜索
keywords:
top_img: https://blog-1321748307.cos.ap-shanghai.myqcloud.com/note/%E6%9C%AC%E5%9C%B0%E6%95%B0%E6%8D%AE%E5%BA%93-cover.png
cover: https://blog-1321748307.cos.ap-shanghai.myqcloud.com/note/%E6%9C%AC%E5%9C%B0%E6%95%B0%E6%8D%AE%E5%BA%93-cover.png
---

上一篇文章[《本地知识库：关于切片（Chunking）的思考》](https://blog.bowie25.cn/2025/02/10/%E6%9C%AC%E5%9C%B0%E7%9F%A5%E8%AF%86%E5%BA%93%EF%BC%9A%E5%88%86%E5%9D%97/)里，我提到在做 RAG 时，应该保留上下文信息。

最近，我正在开发一个名为 Memo Echo 的 Obsidian 插件，旨在通过 AI 激活沉睡的笔记。在这个过程中，随着测试用的知识库逐渐扩大，我遇到了一个棘手的 RAG 难题：单纯把“标签”和“正文”拼在一起做向量化，往往会顾此失彼。 标签放进了正文里，要么干扰了正文的语义，要么被正文淹没搜不出来。

为了解决存储问题，我调研了市面上的主流向量数据库。相比于 Milvus 这种功能强大但部署维护相对复杂，Qdrant 非常的轻量，但在性能和并发表现上却毫不逊色。这种“小而美”的特质，非常适合作为 Obsidian 这类个人知识库的本地后端。

正是在深入挖掘 Qdrant 的文档时，我发现了它在 v1.7 版本引入的 Named Vectors（命名向量） 功能。这提供了一个非常优雅的解决方案，它的核心思想很简单：不再强求把所有鸡蛋放在一个篮子里，而是让一条数据拥有多个维度的索引。

今天，我们来看看这具体是怎么实现的，以及它能带来哪些有趣的搜索玩法。

## Named Vectors

为了方便理解这个概念，我们可以打个通俗的比方：这就像是一个人的生物识别档案。

在数据库里，每一个数据点（Point）就是 **“你”** 这个人。

-   Payload（本体）：这是你的个人信息，比如身份证号、名字、家庭住址。无论怎么查，最后要找的都是这个本体。
-   Vector（特征）：这是用来识别你的手段。

在传统单向量模式下，系统只能存一张你的全身生活照。 当你需要“刷脸支付”时，因为照片里脸部占比太小，可能识别不准；当你需要“指纹解锁”时，全身照里根本看不清指纹。所有特征混在一起，反而什么都不突出。

而在 Named Vectors 模式下，系统允许存储你的多个维度的特征：

-   Face Vector（人脸特征）：专门用于面部识别。
-   Voice Vector（声纹特征）：专门用于语音识别。
-   Fingerprint Vector（指纹特征）：专门用于指纹识别。

重点来了：无论你是通过刷脸、说话还是按指纹被系统识别出来，系统找到的永远是“你”这同一个人（同一个 ID 和 Payload）。

![](https://blog-1321748307.cos.ap-shanghai.myqcloud.com/note/%E6%9C%AC%E5%9C%B0%E6%95%B0%E6%8D%AE%E5%BA%93-namedVector.png)

回到代码世界，这就意味着我们可以把“正文”和“标题”拆开，分别生成向量，但共享同一个 Payload：

```JavaScript
// 多向量模式：拥有多张“脸”
{
  id: 1,
  payload: { text: "...", title: "Rust笔记" },
  vectors: {
    content: [0.1, 0.2, 0.9...], // 脸 A：根据正文生成（细节丰富）
    title:   [0.8, 0.1, 0.5...]  // 脸 B：根据标题生成（高度概括）
  }
}
```

这不仅让结构更清晰，更重要的是，它改变了我们写入和检索的方式。

## 写入：只需极少的代码改动

对于开发者来说，迁移成本很低。你不需要重写整个数据库逻辑，只需要在调用 upsert（插入/更新）接口时，把 vector 字段从一个数组变成一个对象。

来看一段伪代码，关键在于构造 vectors 对象：

```JavaScript
// 1. 准备文本
const contentText = "Rust 的所有权机制...";
const titleText = "Rust 核心原理";

// 2. 分别生成向量 (调用你的 Embedding 模型)
const contentVec = await embed(contentText);
const titleVec = await embed(titleText);

// 3. 入库 (Qdrant API)
await client.upsert("my_collection", {
  points: [
    {
      id: "uuid-1",
      payload: { ... }, // 原始数据
      // 关键变化在这里：
      vectors: {
        "content": contentVec,  // 存入内容向量
        "title":   titleVec     // 存入标题向量
      }
    }
  ]
});
```

你看，并没有复杂的逻辑，只是多存了一份索引数据。

![](https://blog-1321748307.cos.ap-shanghai.myqcloud.com/note/%E6%9C%AC%E5%9C%B0%E6%95%B0%E6%8D%AE%E5%BA%93-%E5%86%99%E5%85%A5.png)

## 搜索：三种全新的策略

既然有了两个向量（content 和 title），我们在搜索时就有了极大的自由度。这才是 Named Vectors 真正的威力所在。

我们可以设计出三种不同层级的搜索玩法。

### 玩法 A：单路切换（用户自主选择）

这是最直观的交互。你可以在搜索框旁边做一个下拉菜单：[ 搜索正文 ] 或 [ 搜索标题 ]。

如果用户记得文章的大概名字，选“搜索标题”，准确率会极高；如果用户只记得某个细节，选“搜索正文”。

代码逻辑：

```JavaScript
// 用户选择了 "Search by Title"
const searchTarget = userSelect === 'title' ? 'title' : 'content';

await client.search("my_collection", {
  // 告诉数据库：只去匹配这个名字的向量
  vector: {
    name: searchTarget,
    vector: queryEmbedding
  },
  limit: 10
});
```

### 玩法 B：双管齐下（并集搜索）

很多时候，用户并不知道该搜标题还是搜正文。比如搜“内存安全”，可能有的文章标题里有，有的文章写在正文里。

这时候，我们可以使用 Qdrant 的 Prefetch 功能，同时去两个池子里捞鱼，然后把结果合并。

逻辑流：

第一步：去 title 向量区，找前 10 个最像的（标题匹配最准）。

第二步：去 content 向量区，找前 10 个最像的（防止漏网之鱼）。

第三步：合并这两份名单，去重展示。

```JavaScript
await client.search("my_collection", {
  // 主查询：在 content 里找
  vector: {
    name: "content",
    vector: queryEmbedding
  },
  // 预取：同时在 title 里找，并把结果融合进来
  prefetch: [
    {
      vector: {
        name: "title",
        vector: queryEmbedding
      },
      limit: 10
    }
  ],
  limit: 10
});
```

### 玩法 C：加权融合（智能打分）

这是最高级的玩法，也就是简单的 Reranking（重排序）思想。

我们认为：标题匹配上的权重，应该高于正文匹配上的权重。 比如搜“Rust 内存”：

笔记 A：《Rust 内存管理》（标题直接命中，得分高）。

笔记 B：《学习周报》（正文提了一句“Rust 内存不错”，标题不相关）。

如果我们给标题设定 70% 的权重，正文设定 30% 的权重，笔记 A 就会稳稳地排在笔记 B 前面。

虽然 Qdrant 内部支持复杂的融合算法，但在应用层实现这个逻辑也很简单：

```JavaScript
// 伪代码：应用层加权
const results = allFoundPoints.map(point => {
  // 假设我们获取了两个维度的相似度分数
  const titleScore = point.vectors['title'].score;
  const contentScore = point.vectors['content'].score;
  // 自定义公式
  const finalScore = (titleScore * 0.7) + (contentScore * 0.3);
  return { ...point, score: finalScore };
});

// 按 finalScore 重新排序
results.sort((a, b) => b.score - a.score);
```

## 总结

技术架构的演进，往往是把“一团模糊”的东西拆解成“清晰独立”的模块。

Named Vectors 就是把“文章特征”拆解了。它不需要复杂的外部系统，仅仅通过简单的配置修改，就能让本地知识库的搜索体验提升一个台阶。

对于简单需求，用“玩法 A”，让用户自己选。

对于想要全面，用“玩法 B”，不错过任何线索。

对于追求精准，用“玩法 C”，让最相关的结果排在第一位。
