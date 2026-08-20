---
tags: [AI学习, 阶段2, RAG, Embedding, 选型]
created: 2026-08-07
---

# Embedding模型选型与本地部署

> ⚠️ **版本基线：2026-08-07 联网核对**。embedding 模型半年一换代，
> 你读到这篇时若已过去 2 个月以上，**先重新核一遍榜单再定型**（这是知识库 lint 的固定动作）。

## 这是什么/为什么重要

上一课说过：**换 embedding 模型 = 重建整个向量库**。所以选型是 RAG 项目里
**最早做、最难改**的决策——比选向量库、选生成模型都要紧。

Java 类比：这相当于选数据库的**主键生成策略**。上线前改是改配置，上线后改是数据迁移。

## 核心内容

### 1. 怎么看 MTEB 榜单（以及为什么不能只看榜）

**先收藏这几个地址**（2026-08-20 验证可访问）：

| 用途 | 地址 |
|---|---|
| **MTEB 官方榜**（含英文 / 多语言 MMTEB / 中文 C-MTEB 分栏） | https://huggingface.co/spaces/mteb/leaderboard |
| **Ollama 上能直接 pull 的 embedding 模型** | https://ollama.com/search?q=embedding |
| bge-m3 官方模型卡（前缀、用法、FAQ） | https://huggingface.co/BAAI/bge-m3 |

第一个是选型的**理论依据**，第二个是**你实际能用什么**——两者的交集才是你的候选池。
榜首模型在 Ollama 上拉不到（如 `gte-multilingual-base`），对本阶段就等于不存在。

MTEB（Massive Text Embedding Benchmark）是 embedding 领域的公认榜单，
但有三个坑，不知道就会选错：

**坑一：MTEB v2 的分数不能和 v1 比。**
2026 年 MTEB 出了 v2，题目集和计算方式都变了。你在老博客里看到的
"某模型 64.5 分"和新榜上的"70.58 分"**不是一回事**，别跨版本横比。

**坑二：榜分不同。** 至少有三个独立榜：英文 v2、多语言 MMTEB、中文 C-MTEB。
你做中文知识库，看 C-MTEB 和 MMTEB，看英文榜没意义。

**坑三：榜单测的是通用能力，不是你的场景。**
榜首模型在你的公司文档上未必最好。**最终必须用自己的数据做一次小评测**
（第 11 课 [[RAG评估体系]] 会教怎么建评测集）。

> [!tip] 这条和阶段 1 的教训同源
> [[错题与复盘#E43]]：json mode 的价值随模型能力变化——**别人的结论在你的配置下未必成立**。
> 榜单是筛掉明显不行的，不是替你做决定。

### 2. 2026-08 的候选清单

| 模型 | 参数量 | 维度 | 特点 | 适用 |
|---|---|---|---|---|
| **bge-m3** | 568M | 1024 | 多语言 100+；**同时输出 dense/sparse/多向量三种表示**；开源生产标准 | ⭐ 本阶段主力 |
| **Qwen3-Embedding** | 0.6B / 4B / 8B | 可变 | 100+ 语言，中英俱强；8B 在 MTEB 得 70.58 | 想要更强、显存够时 |
| KaLM-Embedding-Gemma3-12B | 12B | — | 2026-07 MMTEB 榜首（72.32） | 太大，本机不划算 |
| gte-multilingual-base | 305M | 768 | 阿里出品，小而快；**Ollama 上没有，需走 sentence-transformers** | 资源紧张时 |
| mxbai-embed-large / bge-large / nomic-embed-text | 小 | — | Ollama 可直接 pull，适合当对照组 | 做选型对比时 |
| NV-Embed-v2 | 7B | 4096 | 英文榜强 | 纯英文场景 |

**本阶段推荐 `bge-m3`**，理由不是它分最高，而是：

1. **体积小**（Ollama 上 1.2GB）。这一条在阶段 2 是硬约束而非偏好——见 [[03-RAG检索增强-MOC]] 的显存预算表：三个模型同驻时，可用的 19.78GB 里 embedding 每多占 1GB，KV cache 就少 16K 上下文（按 ex10 实测的 64 MB/1K 折算）
2. **它同时产出 dense + sparse 向量**——第 7 课 [[混合检索]] 要用 BM25 风格的稀疏检索，
   bge-m3 一个模型全包了，省一套依赖
3. Ollama 直接支持，`ollama pull bge-m3` 就完事
4. 中英文都稳，社区资料多，踩坑有人问过

等你做完项目②、有了自己的评测集，**再换 Qwen3-Embedding-4B 跑一次对比**——
那时候你能用数据说话，这就是一次很好的简历素材（"我做过 embedding 选型的 A/B 对照"）。

### 3. 本地部署：两条路

**路线 A：Ollama（推荐，和你现有环境一致）**

```bash
ollama pull bge-m3
```

```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")

resp = client.embeddings.create(model="bge-m3", input=["文本1", "文本2"])
vectors = [d.embedding for d in resp.data]
```

优点：接口和你阶段 1 用的一模一样，零学习成本；缺点：拿不到 sparse 向量（只给 dense）。

**路线 B：sentence-transformers（⚠️ 本课不需要，第 7 课再装）**

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("BAAI/bge-m3")
vectors = model.encode(["文本1", "文本2"], normalize_embeddings=True)
```

优点：能拿到全部三种表示、能精细控制 batch size 和 `normalize_embeddings`；
缺点：多一套依赖，模型要单独下载（注意 `HF_HOME` 已外置到 F 盘，见 [[硬件与环境]]）。

**建议**：前期用 Ollama 跑通，第 7 课做混合检索时再换 sentence-transformers 拿 sparse。

> [!warning] 现在别装 sentence-transformers
> 它不在 `phase2-rag` 的依赖里，而且要另外下载 HuggingFace 版权重（几个 GB）。
> 本课全程用 Ollama 就够——**等第 7 课真正需要 sparse 向量时再装**，否则只是白占磁盘。

### 4. 批量化：这是性能的分水岭

单条 embed 和批量 embed 的吞吐差**一个数量级**。

```python
# ❌ 一条一条来，GPU 大部分时间在空转等数据
vectors = [embed_one(t) for t in texts]

# ✅ 批量送，一次前向传播算完
resp = client.embeddings.create(model="bge-m3", input=texts)   # input 接受 list
```

建库时动辄几万个 chunk，这个差别就是"跑 5 分钟"和"跑 1 小时"。

**batch size 怎么定**：从 32 起步往上试，直到显存吃紧或吞吐不再提升。
这和阶段 1 ex10 找 `num_ctx` 上限是同一个套路——**测出拐点，别拍脑袋**。

### 5. 三个必须知道的限制

**① 输入长度上限。** embedding 模型有最大 token 数（`bge-m3` 是 8192，
多数模型只有 512）。**超出部分会被静默截断**——不报错，但后半段内容凭空消失。

这是 [[错题与复盘#E36]] 的又一个变体，而且更阴：你的 chunk 如果超长，
向量只代表前半段，检索时永远命中不到后半段的内容，**而你完全看不出来**。

第 5 课 [[分块策略]] 定 chunk 大小时，**必须把这个上限当硬约束**。

**② 归一化要统一。** 建库时归一化了，查询时也必须归一化，反之亦然。
混用会让相似度失真。sentence-transformers 用 `normalize_embeddings=True`，
Ollama 的返回是否已归一化**要自己验一下**（算一下模长是不是 1.0）。

**③ 前缀要统一。** 上一课讲的非对称检索前缀——建库用什么前缀，
查询就得用配套的那个。**建库时用错前缀 = 重建库**。

### 6. 一个容易忽略的成本项

云端 embedding API 是**按 token 计费**的。建库时把 10 万个 chunk 全部 embed 一遍，
这笔钱是一次性的但不小；而且**每次重建库都要再付一次**。

本地跑就没这个问题——这也是为什么本阶段全程用本地 embedding：
**你会反复重建库**（调 chunk 大小、换分块策略、试不同前缀），
每次都花钱的话，你会舍不得做实验，而做实验恰恰是这一阶段的全部意义。

## 动手任务

代码写到 `E:\workspace\AiStudy\phase2-rag\ex03_embedding_select.py`（骨架已建好）。

1. `ollama pull bge-m3`，跑通 embedding 调用，**验证返回向量是否已归一化**（算模长）
2. **批量 vs 单条对照实验**：100 条文本，分别用单条循环和一次批量，
   记录耗时——注意先预热（[[错题与复盘#E48]]：热身成本会污染首次测量）
3. **截断验证**：构造一段超过模型上限的长文本，故意把关键信息放在**最后**，
   然后检索它——看看是不是真的搜不到。**这是本课最重要的一个实验**，
   因为它演示了一种完全静默的失败
4. 再拉一个模型做对比：同一组句子，两个模型的相似度矩阵有什么不同？分数分布呢？

   **推荐 `ollama pull qwen3-embedding:0.6b`**（2026-08-20 核对规格）：

   | | bge-m3 | qwen3-embedding:0.6b |
   |---|---|---|
   | 体积 | 1.2 GB | **639 MB**（约一半） |
   | 上下文 | 8K | **32K**（4 倍） |
   | 参数量 | 566.7M | 0.6B |

   **更小、窗口更大——那代价在哪？** 这正是本题要测出来的。
   别只看"谁分高"，要看**分数分布**：有的模型任意两句都挤在 0.6 以上，
   它的 0.7 和另一个模型的 0.7 完全不是一回事（回顾上一课的阈值标定 [[错题与复盘#E55]]）。

   其他可直接 pull 的候选：`mxbai-embed-large` / `bge-large` / `nomic-embed-text`。
   （`gte-multilingual-base` **Ollama 上没有**，需走 sentence-transformers）
5. 记录你的选型决定和理由

**完成标准**：批量/单条的耗时数字；截断实验能复现出"后半段搜不到"；
两个模型的分数分布对比 + 一句选型结论。

## 相关笔记

> **本阶段第 3/13 课** | 上一篇：[[Embedding原理与相似度]] | **下一篇：[[文档加载与解析]]**
> 学习顺序总表见 [[03-RAG检索增强-MOC]]

- [[03-RAG检索增强-MOC]]
- [[Embedding原理与相似度]] · [[分块策略]] · [[混合检索]] · [[RAG评估体系]]
- [[Ollama本地模型]] · [[硬件与环境]]
- [[错题与复盘#E36]] · [[错题与复盘#E43]] · [[错题与复盘#E48]]
- [[AI学习地图]]
