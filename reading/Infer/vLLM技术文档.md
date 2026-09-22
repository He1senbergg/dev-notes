
这篇文章本质上不是“提出一个新模型”，而是**一篇工程系统优化论文**。它讲的是：在 vLLM 前面放一个 **Semantic Router 语义路由器**，这个路由器要做安全检测、PII 检测、领域分类、模型选择，但它不能太慢，也最好不要单独占一张 GPU。论文的目标是把这个路由器做得足够快、足够省显存，让它能和 vLLM 推理服务共用同一张 GPU。论文标题里的 **98× Faster** 指的是在 8K token 输入场景下，端到端路由延迟从 **4918 ms 降到 50 ms**。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

## 1. 先说它解决的是什么问题

在大模型服务里，请求通常不是直接打到模型上，而是会先经过一层网关或路由层。这个路由层可能要判断：

1. 这个请求是不是越狱攻击；
2. 里面有没有隐私信息，比如邮箱、身份证号、信用卡号；
3. 这个请求属于什么领域，比如代码、法律、医疗、数学；
4. 应该把请求发给哪个后端模型；
5. 是否要改写 header，或者把 `model=auto` 改成具体模型名。

论文里的系统叫 **vLLM Semantic Router**。它作为 **Envoy ext_proc 外部处理过滤器**工作。也就是说，请求先进入 Envoy，然后 Envoy 通过 gRPC 把请求头、请求体交给 Semantic Router 检查，Router 再返回处理结果，比如修改 header 或者决定路由目标。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

这个设计的问题是：**路由器处在请求链路最前面，它每慢 1 ms，用户感知的 TTFT 就多 1 ms。** 所以路由器必须极快。

但现在实际请求越来越长，比如 RAG、代码审查、文档总结，很容易有 8K、16K、32K token。长上下文一来，路由器里的分类器也得看长文本，这就带来三个瓶颈。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

# 2. 三个核心瓶颈

## 瓶颈一：标准 Attention 显存爆炸

Router 里用的是类似 BERT 的分类器，论文中是 **mmBERT-32K**，基于 ModernBERT 扩展到 32K token。它要做多个分类任务，比如：

| 分类器               | 作用                       |
| -------------------- | -------------------------- |
| Domain classifier    | 判断请求领域，用于模型路由 |
| Jailbreak classifier | 检测越狱攻击               |
| PII classifier       | 检测隐私信息               |

问题是，标准 self-attention 的显存复杂度大致是：

[
O(n^2)
]

其中 (n) 是 token 长度。比如 8K token 时，一个分类器光 attention mask 就要大约 **1.5 GB**；16K 时变成 **6 GB**；32K 时变成 **24 GB**。如果三个分类器并发跑，8K 就要 **4.5 GB** attention mask 显存。论文说，Router 和 vLLM 共用 GPU 时，vLLM 已经占用了大部分 HBM，给每个 classifier session 剩下的显存只有大约 **718 MB**，所以标准 SDPA 在 8K 以上会 OOM。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

你可以简单理解为：

> vLLM 本身已经把 GPU 显存吃得差不多了，Router 如果再用标准 attention 处理长文本，显存就炸了。

---

## 瓶颈二：CPU 跑分类太慢

不用 GPU，改用 CPU 跑 ONNX 或 Candle，可以避免 GPU 显存压力，但延迟太高。论文给的数据是，8K token 的完整路由链路：

| 后端                  | 8K token 端到端延迟 |
| --------------------- | ------------------: |
| ONNX CPU              |             4918 ms |
| Candle CPU            |             1818 ms |
| GPU + Flash Attention |              127 ms |

也就是说，CPU 虽然不占 GPU，但作为在线路由器太慢了。用户每个请求先等 1.8 秒甚至 4.9 秒再开始模型推理，基本不可接受。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

## 瓶颈三：Envoy 请求体处理有序列化开销

Envoy 的 ext_proc 有不同处理模式。很多情况下会用 **BUFFERED mode**，也就是先把完整 HTTP body 收齐，再一次性交给外部处理服务。

这会导致：

1. 请求体要被完整缓存；
2. gRPC protobuf 要序列化/反序列化；
3. Router 里还要对 OpenAI 格式 JSON 做 `json.Unmarshal`；
4. 修改模型名后还要 `json.Marshal` 回去。

长 prompt 的 JSON body 很大，这些处理会随 body 大小线性增加。论文说，即使前面已经用了 GPU 加速和 prompt compression，16K token 请求在 BUFFERED 模式下端到端还有 **142 ms**，其中很多时间花在 JSON 解析和序列化上。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

# 3. 论文的总体方案：三阶段优化

论文的方案分成三层：

| 阶段    | 优化点                                      | 主要解决什么                                                     |
| ------- | ------------------------------------------- | ---------------------------------------------------------------- |
| Stage 1 | CK Flash Attention for ONNX Runtime on ROCm | 解决 AMD GPU 上 ONNX Runtime 没有 Flash Attention，导致 SDPA OOM |
| Stage 2 | Prompt Compression                          | 把长 prompt 压缩到 512 token，让分类器永远只看短文本             |
| Stage 3 | Near-Streaming Body Processing              | 减少 Envoy/JSON/request body 处理开销                            |

最终在 8K token 下：

| 配置                              | 端到端延迟 |
| --------------------------------- | ---------: |
| ONNX CPU baseline                 |    4918 ms |
| GPU + CK Flash Attention          |     127 ms |
| + Prompt Compression              |      62 ms |
| + Near-Streaming + Zero-Copy JSON |      50 ms |

累计约 **98.4 倍加速**。16K token 下，最终方案是 **108 ms**。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

# 4. Stage 1：给 AMD + ONNX Runtime 补 Flash Attention

## 4.1 为什么要专门做这个？

NVIDIA 上很多框架已经有 Flash Attention，例如 cuDNN 或 flash-attention 库。但论文说，ONNX Runtime 的 ROCm Execution Provider 在 AMD GPU 上没有对应的 Flash Attention 集成，所以会退化成标准 SDPA。标准 SDPA 要显式生成完整 attention matrix / mask，长文本就 OOM。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

他们的硬件是：

| 组件         | 配置                        |
| ------------ | --------------------------- |
| GPU          | AMD Instinct MI300X         |
| 显存         | 192 GB HBM3                 |
| ROCm         | 7.0                         |
| ONNX Runtime | 1.22.1                      |
| 模型         | mmBERT-32K，270M 参数，FP16 |
| Envoy        | v1.33                       |

实验是在一张 MI300X 上同时跑两个 vLLM serving instance 和 semantic router。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

## 4.2 Flash Attention 的直观理解

普通 Attention 做的是：

[
QK^T \rightarrow \text{Softmax} \rightarrow \text{乘以 } V
]

问题是 (QK^T) 会生成一个 (n \times n) 的矩阵。token 越长，矩阵越大。8K token 时矩阵已经非常大，16K、32K 更夸张。

Flash Attention 的思想是：

> 不把完整 (n \times n) attention 矩阵一次性放进显存，而是分块计算，边算边融合 Softmax 和后续乘法。

所以它不是近似算法，结果理论上仍然是精确 attention，只是计算方式更省显存。论文引用 FlashAttention 相关工作，说它通过 tiling 和 kernel fusion 把 attention 的内存占用从平方级降低。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

## 4.3 他们具体怎么做？

他们做了一个 **ONNX Runtime custom operator**，名字叫：

```text
com.ck::CKFlashAttention
```

大致流程是：

1. 先把 mmBERT-32K 从 PyTorch 导出成 ONNX；
2. 用一个 Python 脚本 `rewrite_graph.py` 扫描 ONNX graph；
3. 找到标准 SDPA 子图；
4. 把这一串节点替换成一个 `CKFlashAttention` 节点；
5. 运行时加载 `libort_ck_flash_attn.so`；
6. 通过 HIP kernel 调用 AMD Composable Kernel 的 FMHA 实现。

原来的 ONNX 子图类似：

```text
Q * scale
MatMul(Q, K^T)
Add(mask)
Softmax
MatMul(attn, V)
```

替换后变成一个节点：

```text
CKFlashAttention(Q, K, V, pad_bias)
```

这样就不再生成完整 2D attention mask，显存占用明显下降。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

## 4.4 ModernBERT 的特殊点

ModernBERT 不是每一层都做全局 attention。它有交替的 local/global attention：

| 层类型       | attention 范围           |
| ------------ | ------------------------ |
| Local layer  | 128-token sliding window |
| Global layer | 全序列                   |

论文里说，local 层用 `window_left=63, window_right=64`，global 层则对应 full attention。它们在图重写时根据 mask 节点名识别 local/global 层，再设置不同的 window 参数。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

这里你可以理解为：

> 不是简单地把所有 attention 都替换成同一种 Flash Attention，而是要保留 ModernBERT 原来的局部/全局 attention 行为。

---

## 4.5 Stage 1 的效果

论文的 GPU 加速数据：

|   输入长度 | ONNX CPU E2E | GPU + FA E2E |   加速 |
| ---------: | -----------: | -----------: | -----: |
|  500 token |       803 ms |        22 ms | 36.5× |
| 2000 token |      1773 ms |        31 ms | 57.2× |
| 8000 token |      4918 ms |       127 ms | 38.7× |

另外，SDPA 和 Flash Attention 对比：

| 序列长度 |   SDPA |     FA |
| -------: | -----: | -----: |
|      512 |  19 ms |  19 ms |
|     1024 |  26 ms |  23 ms |
|     2048 |  51 ms |  32 ms |
|     4096 | 167 ms |  51 ms |
|     8192 |    OOM | 105 ms |
|    16384 |    OOM | 259 ms |
|    32768 |    OOM | 756 ms |

这说明 Stage 1 主要解决了两个问题：

1. **CPU 太慢**；
2. **标准 SDPA 长文本 OOM**。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

但它还不够。因为虽然 FA 不 OOM，16K 还是 259 ms，32K 是 756 ms。分类器有三个，并发请求一多，延迟还是会上去。

---

# 5. Stage 2：Prompt Compression，把长文本压到 512 token

这是这篇文章对你工作可能更有参考价值的部分。

它不是对上游 LLM 的 prompt 做压缩，而是：

> **只压缩给 Router 分类器看的文本。真正发给大模型的原始 prompt 不变。**

这个点非常重要。用户请求的完整内容仍然原封不动发给后端 LLM；压缩只用于 domain / PII / jailbreak 等分类判断。论文明确说，原始未压缩 prompt 总是被发送给上游 LLM，压缩只用于 router classifier 的 evaluation text。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

## 5.1 为什么要压缩？

因为 Router 做分类其实不一定需要看完整 16K 或 32K token。

例如用户发来一个很长的 RAG 文档，里面可能有大量背景材料。但真正决定路由的内容可能只是：

```text
请帮我分析这个 Python 报错……
```

或者：

```text
这是一份医学报告……
```

或者：

```text
Ignore all previous instructions...
```

所以论文认为，分类信号通常集中在 prompt 的局部区域，而不是均匀分布在所有 token 里。于是可以先选出最关键的句子，压到 512 token，再给分类器。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

## 5.2 压缩方法不是 LLM，而是传统 NLP

这篇论文没有用 LLMLingua 这种神经网络压缩器，因为神经压缩本身也要跑模型，会增加延迟。它用的是传统 NLP 方法：

| 信号               | 作用                                                   |
| ------------------ | ------------------------------------------------------ |
| TextRank           | 找到和其他句子相似、代表性强的句子                     |
| Position Weighting | 保留开头和结尾，因为长上下文有 U-shaped attention 现象 |
| TF-IDF             | 找到包含稀有、领域关键词的句子                         |
| Novelty            | 找到和全文中心不一样的异常句子，比如 PII、越狱攻击     |

论文说，对 16K token 输入，其他神经压缩方法可能要 0.3 秒到数秒，而他们这个传统 NLP 压缩流程测得约 **19 ms**，而且不需要神经网络推理。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

## 5.3 压缩流程

大致流程如下：

### 第一步：句子切分

先把输入文本切成句子。它支持 Latin scripts、CJK、Arabic、Devanagari。也就是英文、中文、阿拉伯文、印地语这类文本都尽量支持。为了避免大 prompt 造成 GC 压力，最多保留 500 个句子参与排序。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

### 第二步：计算 TextRank

TextRank 类似 PageRank。把每个句子看成一个节点，如果两个句子的词频向量相似，就连边。和很多重要句子都相似的句子，会得到更高分。

这个信号偏向“代表全文主题”的句子。比如一篇关于 Kubernetes 调度的长文里，多次出现：

```text
pod scheduling
node resource
GPU allocation
```

这些相关句子会有较高 TextRank。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

### 第三步：位置权重

论文基于 “Lost in the middle” 现象：长上下文里，模型往往更关注开头和结尾，中间的信息容易被忽略。所以它给开头和结尾更高权重，中间更低权重。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

直观上就是：

```text
[开头]     权重大
[中间]     权重小
[结尾]     权重大
```

这对实际 LLM 请求很合理。因为：

1. system prompt 常在开头；
2. 用户最终问题常在结尾；
3. 中间可能是大量参考文档。

---

### 第四步：TF-IDF 信息密度

TF-IDF 用来找“更有信息量”的句子。

比如一堆通用句子：

```text
This document explains the background.
This section introduces the method.
```

信息量不高。

但下面这种句子：

```text
The request contains CUDA OOM during vLLM prefix cache allocation.
```

里面有 `CUDA`、`OOM`、`vLLM`、`prefix cache` 这种领域词，更能说明请求属于大模型推理 / 系统工程领域。TF-IDF 会让这种句子得分更高。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

### 第五步：Novelty / 反中心性

TextRank 找的是“代表性强”的句子，但越狱攻击、PII 往往不是代表性内容，而是异常内容。

比如一个很长文档里突然出现：

```text
My credit card number is ...
```

这句话和全文主题可能不太相似，但它对 PII 检测非常关键。

所以论文加入 Novelty 信号：计算句子和全文中心向量的距离，越不像全文平均内容，Novelty 越高。这样可以更容易保留异常句子。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

### 第六步：加权求总分，然后选句子

每个句子最终分数是四个分数的加权和：

```text
最终分数 = TextRank 分数 + 位置分数 + TF-IDF 分数 + Novelty 分数的加权组合
```

论文说权重是通过 384 个 Wikipedia 测试样本 grid search 调出来的。位置权重最高，因为 domain question 或 system prompt 很多时候在开头或结尾；TF-IDF 是主要内容信号；TextRank 保留代表性句子；Novelty 权重最低，因为过高 Novelty 会伤害领域分类准确率。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

另外，它会强制保留：

```text
PreserveFirstN = 3
PreserveLastN = 2
```

也就是前 3 句和后 2 句优先保留。这非常工程化，因为很多请求的重要信息确实在头尾。

最后按分数选句子，直到达到 512 token 预算。选出来的句子按原始顺序重新拼回去，尽量保留上下文连贯性。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

## 5.4 Prompt Compression 的性能效果

压缩前后单个信号分类延迟：

|  输入 | 信号      |   不压缩 | 压缩到 512 token |
| ----: | --------- | -------: | ---------------: |
|   500 | Jailbreak |  10.1 ms |           9.3 ms |
|  2000 | Jailbreak |  18.0 ms |          11.1 ms |
|  8000 | Jailbreak |  45.3 ms |          10.5 ms |
| 16000 | Jailbreak | 126.6 ms |          10.4 ms |
| 16000 | Domain    |  85.3 ms |           7.1 ms |
| 16000 | PII       |  84.6 ms |           6.2 ms |

可以看到，只要压到 512 token，分类器延迟基本就固定了。16K token 的 Jailbreak 分类从 **126.6 ms 降到 10.4 ms**。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

压缩流程本身也有开销：

| 原始 token | 输出 token | 压缩比例 | 压缩耗时 |
| ---------: | ---------: | -------: | -------: |
|       2000 |        510 |    25.1% |     2 ms |
|       4000 |        511 |    12.7% |     4 ms |
|       8000 |        512 |     6.4% |     9 ms |
|      16000 |        512 |     3.2% |    19 ms |

也就是说，16K 输入压缩要 19 ms，但它节省的 GPU 分类时间远大于 19 ms，所以是值得的。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

# 6. Stage 3：Near-Streaming Body Processing

Stage 1 和 Stage 2 解决的是分类器计算问题，但请求链路还有 JSON 和 Envoy body 处理开销。Stage 3 就是优化这块。

---

## 6.1 原来的问题

在 BUFFERED 模式下，Envoy 要等整个 HTTP body 收完，再发给 ext_proc 服务。Router 拿到完整 JSON 后，要做：

```text
json.Unmarshal
读取 model/messages/stream
分类/路由
修改 model 字段
json.Marshal
返回修改后的 body
```

长 prompt 时 body 可能几十 KB 到几百 KB，这些 JSON 操作会明显拖慢路由。论文说，16K token 即使用 GPU + FA + compression，BUFFERED 还是有 142 ms E2E，其中 JSON 解析和序列化占了很大部分。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

## 6.2 第一层优化：Zero-Copy JSON

他们不再完整反序列化 OpenAI 请求体，而是用 Go 里的：

```text
gjson
sjson
```

做局部字段读取和修改。

例如：

| 操作          | 老方式                       | 新方式               |
| ------------- | ---------------------------- | -------------------- |
| 读取 model    | 完整 json.Unmarshal          | gjson 直接查路径     |
| 读取 messages | 完整构造 OpenAI SDK struct   | 必要时才解析         |
| 修改 model    | unmarshal 后改字段再 marshal | sjson 直接字节级替换 |

这样减少了全量 JSON 对象构造和内存复制。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

## 6.3 第二层优化：StreamedBodyHandler

它设计了一个三状态机：

```text
init -> passthrough
     -> accumulate
```

含义如下：

### 状态一：init

收到第一个 chunk。Router 尝试从第一个 chunk 中读取 `model` 字段。

### 状态二：passthrough

如果请求里明确写了模型名，比如：

```json
{
  "model": "qwen-72b",
  "messages": [...]
}
```

那就不需要 domain routing。Router 可以直接把 body chunk 透传给上游，只改 header 或做轻量路由。

这个路径几乎不复制 body。

### 状态三：accumulate

如果请求是：

```json
{
  "model": "auto",
  "messages": [...]
}
```

说明需要 Router 判断应该走哪个模型。于是它会累积 body，同时在 chunk 到达过程中做一些增量 NLP 预处理，比如句子切分、TF-IDF 统计。等 end-of-stream 到了，再做完整分类。

论文强调，这个设计能把 I/O 和 NLP 预处理重叠起来，减少端到端等待。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

## 6.4 Near-Streaming 和其他策略的区别

| 策略                | 能检查 body 吗 | 能改 body 吗  | 拷贝开销    |
| ------------------- | -------------- | ------------- | ----------- |
| Pure passthrough    | 不能           | 不能          | 0           |
| Header-only routing | 不能看 prompt  | 只能改 header | 0           |
| Full buffering      | 能             | 能            | 2\~3 次拷贝 |
| 本文 Near-streaming | 自适应         | 自适应        | 0 或 1 次   |

这个策略的关键是：**不是所有请求都需要完整检查 body。**

如果用户已经指定了模型，就走 passthrough；如果是 `model=auto`，才进入 accumulate 检查。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

# 7. 整体实验结果

## 7.1 分阶段加速

论文最核心的表是这个：

| 配置                          | token | E2E 延迟 | 累计加速 |
| ----------------------------- | ----: | -------: | -------: |
| ONNX CPU, BUFFERED baseline   |    8K |  4918 ms |    1.0× |
| Candle CPU, BUFFERED          |    8K |  1818 ms |    2.7× |
| ONNX GPU SDPA                 |    8K |      OOM |        - |
| Stage 1: GPU + CK FA          |    8K |   127 ms |   38.7× |
| Stage 2: + Prompt Compression |    8K |    62 ms |   79.3× |
| Stage 3: + Near-Streaming     |    8K |    50 ms |   98.4× |
| Extended                      |   16K |   108 ms |        - |

所以标题里的 **98×** 是这样来的：

```text
4918 ms / 50 ms ≈ 98.36
```

它不是说模型推理快了 98 倍，而是说 **Router 路由链路** 从 CPU baseline 到完整优化方案快了 98 倍。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

## 7.2 并发下的效果

没有 compression 时，Flash Attention 虽然省显存，但长文本 + 并发仍然会慢。

论文给的数据：

| 原始 token | Raw FA C=1 | Raw FA C=10 | Raw FA C=20 |
| ---------: | ---------: | ----------: | ----------: |
|        512 |      19 ms |       77 ms |      142 ms |
|       2048 |      32 ms |      141 ms |      275 ms |
|       8192 |     105 ms |      601 ms |     1058 ms |
|      16384 |     259 ms |     1567 ms |     3089 ms |
|      32768 |     756 ms |     5406 ms |     9872 ms |

加上 compression + streaming 后：

| 原始 token |    C=1 |   C=10 |   C=20 |
| ---------: | -----: | -----: | -----: |
|        512 |  17 ms |  75 ms | 140 ms |
|       2048 |  24 ms |  92 ms | 157 ms |
|       8192 |  50 ms | 118 ms | 183 ms |
|      16384 | 108 ms | 166 ms | 231 ms |
|      32768 | 125 ms | 183 ms | 248 ms |

这个差别很大。核心原因是：压缩后，GPU 永远只处理 512 token，原始 prompt 是 8K、16K、32K 都不影响分类器输入长度。论文说，未压缩 FA 在 32K、C=20 时达到 9.9 秒；压缩和 streaming 后，32K、C=20 估算低于 250 ms。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

## 7.3 吞吐

单请求串行吞吐大致是延迟倒数：

| 配置                         |    延迟 | req/s |
| ---------------------------- | ------: | ----: |
| ONNX CPU 8K baseline         | 4918 ms |   0.2 |
| GPU+FA 500 token             |   22 ms |  45.5 |
| GPU+FA 8K token              |  127 ms |   7.9 |
| GPU+FA+comp+stream 500 token |   17 ms |  58.8 |
| GPU+FA+comp+stream 8K token  |   50 ms |  20.0 |
| GPU+FA+comp+stream 16K token |  108 ms |   9.3 |

并发情况下，论文说 16K token、C=10 时，系统可以达到大约 **60 requests/s**。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

# 8. 准确率结果：有好有坏

这部分要特别注意。论文不是简单说“压缩不会影响准确率”，实际结果更复杂。

他们做了 384 个测试样本，来自 8 篇 Wikipedia 文章、8 个领域、4 种 prompt 长度、12 种信号位置组合。每个 prompt 嵌入：

1. jailbreak prefix；
2. PII token；
3. domain-specific technical question。

然后比较原始 prompt 和压缩到 512 token 后的分类结果。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

结果：

| 指标                  | 原始 prompt | 压缩 prompt |
| --------------------- | ----------: | ----------: |
| Domain classification |       53.1% |       61.2% |
| PII detection         |       78.5% |       92.4% |
| Jailbreak detection   |       70.8% |       56.6% |

可以看到：

1. **领域分类变好**：53.1% → 61.2%；
2. **PII 检测变好**：78.5% → 92.4%；
3. **越狱检测变差**：70.8% → 56.6%。

论文解释说，压缩像一个去噪过程，把大量无关 token 去掉，让分类信号更集中，所以 domain 和 PII 更好。但 jailbreak 是更依赖上下文的异常意图，压缩可能删掉一些对越狱判断有帮助的上下文，所以会变差。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

这里有一个工程上的处理：

> 论文说生产中 **jailbreak 和 PII 可以跑原始未压缩 prompt**，而 compression 主要用于 domain routing。

这个说法稍微有点复杂，因为前面又展示了 PII 压缩后更好。但它的核心意思是：不要盲目把所有分类任务都压缩，要按任务选择是否压缩。对于 domain routing，压缩是明显有利的；对于 jailbreak，要谨慎。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

# 9. 它和普通 vLLM serving 优化有什么区别？

这篇文章优化的不是：

```text
大模型生成 token 的速度
```

而是：

```text
请求进入大模型之前，Router 做分类和路由决策的速度
```

也就是说，它不是在优化 vLLM 的 KV cache、PagedAttention、batching、prefill/decode 本身，而是在优化 vLLM 前面的 Semantic Router。

一个请求链路可以理解成：

```text
用户请求
  ↓
Envoy
  ↓
Semantic Router
  ↓
分类：domain / jailbreak / PII
  ↓
决定路由到哪个模型
  ↓
vLLM 真正执行推理
  ↓
返回结果
```

这篇论文主要优化中间这段：

```text
Envoy + Semantic Router + 分类器
```

所以你不能把它理解成：

> vLLM 推理整体快了 98 倍。

更准确是：

> vLLM 前面的语义路由器，在长 prompt 分类路由场景下，端到端路由延迟比 CPU baseline 快了约 98 倍。

---

# 10. 这篇文章对你当前工作的参考价值

结合你之前做的推理调度、OpenAI 接口适配、模型路由、SLO 调度，这篇文章有几个很直接的参考点。

---

## 10.1 请求路由可以不只是“规则路由”

你现在的调度/路由可能更多是：

```text
模型名 -> 实例列表
任务类型 -> 队列
负载 -> 选择实例
健康状态 -> 过滤实例
```

这篇论文强调另一种路由：

```text
根据 prompt 内容语义决定模型
```

例如：

| prompt 内容 | 路由目标           |
| ----------- | ------------------ |
| 代码问题    | code model         |
| 数学问题    | math model         |
| 医疗问题    | safe/medical model |
| 普通聊天    | general model      |
| 可疑越狱    | 拦截或走安全模型   |
| 含 PII      | 打标或特殊处理     |

这和你现在的调度系统不是冲突关系，而是上下游关系：

```text
Semantic Router 决定“该用哪个模型”
你的 Scheduler 决定“这个模型的哪个实例来处理”
```

---

## 10.2 “分类器延迟”本身要纳入 TTFT

这篇文章反复强调：Router 的每一毫秒都会加到 TTFT 上。你做推理服务时，如果前置流程有：

```text
鉴权
参数校验
模型路由
安全检测
工具调用解析
prompt 预处理
调度排队
```

这些都不是免费的。它们会直接影响用户感知延迟。

所以你的系统里如果以后加语义路由，不能只说“加个分类器”，还要问：

```text
分类器 p50 / p90 / p99 延迟是多少？
长 prompt 下会不会慢？
并发下是否会抢推理 GPU？
显存是否会影响 vLLM KV cache？
```

这篇论文正是在回答这些工程问题。

---

## 10.3 Prompt Compression 可以用于“路由判断”，不一定用于“模型输入”

这一点非常有用。

很多人一听 prompt compression，会以为是把用户输入压缩后给大模型。但这会改变模型回答，风险比较大。

这篇文章的做法更稳：

```text
原始 prompt：给后端 LLM
压缩 prompt：只给分类器/路由器
```

这个思路你可以迁移到很多场景：

| 场景                 | 是否需要完整 prompt  |
| -------------------- | -------------------- |
| 真正模型回答         | 需要                 |
| 判断领域             | 不一定               |
| 判断是否含 PII       | 不一定，但要小心召回 |
| 判断任务类型         | 不一定               |
| 判断 prompt 长度类别 | 不需要               |
| 选择模型             | 不一定               |
| 安全拦截             | 可能需要完整输入     |

也就是说，**路由器不一定要看完整请求，只要看足够判断路由的摘要/关键句即可。**

---

## 10.4 如果你们不是 AMD，这篇文章仍然有价值

Stage 1 的 CK Flash Attention 是 AMD ROCm 特定的。论文也说，NVIDIA 上已经有较成熟的 FlashAttention 支持，所以 Stage 1 的直接价值主要在 AMD + ONNX Runtime 场景。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

但 Stage 2 和 Stage 3 是硬件无关的：

1. Prompt Compression 在 CPU 上做；
2. Near-streaming body processing 是 Go/Envoy/request body 处理；
3. Zero-copy JSON 思路也不依赖 GPU。

所以如果你们的推理服务是昇腾/NPU、NVIDIA GPU、CPU，都可以借鉴后两部分。

---

# 11. 论文的局限和需要警惕的地方

## 11.1 它是 arXiv 技术报告，不是严格 peer-reviewed 期刊

这意味着它更偏工程报告。里面的结果可以参考，但要注意实验设置是否和你的生产环境一致。

---

## 11.2 Baseline 选择会影响 98× 的观感

98× 是相对 **ONNX CPU 8K baseline 4918 ms** 得来的。如果你拿一个更强的 GPU baseline、或者你的 CPU 分类器不是这么慢，那么倍数会变小。

所以正确读法是：

```text
在他们的 Envoy + ONNX CPU baseline + MI300X + mmBERT-32K 设置下，
完整优化链路达到了约 98× 加速。
```

不是普遍保证任何环境都快 98 倍。

---

## 11.3 Accuracy 不是全面提升

Domain 和 PII 变好，但 Jailbreak 变差。尤其安全检测场景，对 false negative 很敏感。压缩可能删掉关键上下文，这在安全任务里很危险。

所以如果你要借鉴，建议区分：

```text
Domain routing：可以压缩
PII detection：可以尝试压缩，但要重视召回率
Jailbreak/safety：最好保留全文检测，或者做双路径检测
```

比如：

```text
压缩文本跑 domain classifier
原始文本跑 jailbreak keyword/rule/light classifier
高风险请求再跑重模型
```

---

## 11.4 Streaming 的 accumulate 路径仍然要缓存完整 body

论文自己也承认，streaming handler 的 accumulate 路径在分类前仍然会 buffer 整个 body。所以它主要减少的是 JSON 处理和序列化开销，不是彻底消除 body 内存占用。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

## 11.5 压缩 token 预算用的是近似 tokenization

论文限制里提到，目前 prompt compression 用的是字符长度启发式，不是模型真实 tokenizer。未来应该接入模型特定 tokenizer，才能精确控制 token budget。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

这点在中文场景尤其要小心。英文字符数和 token 数的关系还相对稳定，中文、代码、URL、JSON 混合输入就不一定。

---

# 12. 用一句话总结这篇论文

这篇论文讲的是：

> 在 vLLM 前面做一个语义路由器，让它用多个 BERT 类分类器做领域分类、安全检测和 PII 检测；为了让这个路由器不拖慢 TTFT、不单独占 GPU，作者做了三层优化：AMD ONNX Runtime 的 CK Flash Attention、传统 NLP prompt compression、Envoy near-streaming/zero-copy JSON，最终把 8K token 路由延迟从 4918 ms 降到 50 ms，并让 Router 可以和 vLLM 共用一张 GPU。([arXiv](https://arxiv.org/html/2603.12646?_immersive_translate_auto_translate=1 "98× Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

# 13. 对你来说最该记住的三个点

第一，**它不是 vLLM 推理加速论文，而是 vLLM 前置语义路由器加速论文。**

第二，**Prompt Compression 的正确用法可以是“只给分类器压缩，不改真正发给 LLM 的 prompt”。** 这个思路很适合工程系统。

第三，**长 prompt 下，路由器本身也会成为系统瓶颈。** 如果你们以后做基于内容的模型路由、安全检测、PII 检测，必须把 Router 的 TTFT、显存、并发、p99 延迟一起设计，而不是只看后端模型推理性能。

---

# Q&A

```
接下来，回答我的以下问题：

1. 什么是Envoy ext_proc？
2. 什么是mmBERT-32K？什么是ModernBERT？它们两者，与Bert的关系是什么？是基于Bert-large做的吗？
3. CPU 跑 ONNX 或 Candle，这句怎么理解？是CPU模型的格式一般是这两种吗？
4. SDPA OOM是什么？什么是SDPA？
5. 什么是FlashAttention？什么是CK FlashAttention？
6. 什么是BUFFERED mode？怎么理解收齐Http Body以后再交由外部处理服务？是类似于，一个服务里有http组件、解析组件，http组件完成了body获取以后，把东西交由解析组件去处理吗？以及，其中的这四步，我也不理解含义：

- 请求体要被完整缓存；
- gRPC protobuf 要序列化/反序列化；
- Router 里还要对 OpenAI 格式 JSON 做 json.Unmarshal；
- 修改模型名后还要 json.Marshal 回去。
```

下面我按你的 6 个问题逐个解释。先给一个总图，方便你把这些词放到同一条链路里：

```text
用户请求
  ↓
Envoy 网关 / 代理
  ↓
Envoy ext_proc 过滤器
  ↓ 通过 gRPC 把请求头/请求体发给外部处理服务
Semantic Router 外部服务
  ↓
解析 OpenAI JSON 请求
  ↓
拿 prompt 跑 mmBERT-32K 分类器
  ↓
判断 domain / jailbreak / PII
  ↓
决定真实模型名，比如 auto -> llama-70b
  ↓
把修改后的请求交给 vLLM
```

---

# 1. 什么是 Envoy ext_proc？

**Envoy** 可以先理解成一个高性能代理 / 网关，类似：

```text
用户请求 -> Envoy -> 后端服务
```

它可以做路由、负载均衡、鉴权、限流、TLS、日志等事情。

**ext_proc** 全称是 **External Processing Filter**，中文可以叫：

> Envoy 外部处理过滤器。

它的作用是：**Envoy 自己不处理复杂业务逻辑，而是把 HTTP 请求/响应的一部分发给一个外部服务，让外部服务判断该怎么处理。**

Envoy 官方文档说，ext_proc 会把一个叫做 external processor 的外部 gRPC 服务接入 HTTP filter chain；这个外部服务可以检查和修改 headers、body、trailers，也可以直接返回一个新的 HTTP 响应。([Envoy Proxy](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_proc_filter "External Processing — envoy 1.39.0-dev-6a45c7 documentation"))

你可以把它理解成：

```text
Envoy：我负责接请求、转发请求
外部处理服务：我负责判断这个请求能不能过、该怎么改、该路由到哪里
```

在这篇论文里：

```text
Envoy ext_proc
  ↓
vLLM Semantic Router
```

Semantic Router 就是那个 **external processor**。

---

## 一个具体例子

用户发请求：

```json
{
  "model": "auto",
  "messages": [
    {
      "role": "user",
      "content": "帮我写一个 Python 快排"
    }
  ]
}
```

Envoy 收到请求后，不知道 `model=auto` 应该路由到哪个真实模型。于是它通过 ext_proc 把请求交给 Semantic Router。

Semantic Router 分析后觉得这是代码类请求，于是改成：

```json
{
  "model": "code-model",
  "messages": [
    {
      "role": "user",
      "content": "帮我写一个 Python 快排"
    }
  ]
}
```

然后 Envoy 再把修改后的请求转发给后端 vLLM。

所以 ext_proc 不是一个模型，也不是一个路由算法。它是 Envoy 提供的一种机制：

> 允许 Envoy 在请求链路中间调用外部服务，让外部服务检查和修改 HTTP 请求/响应。

---

# 2. 什么是 mmBERT-32K？什么是 ModernBERT？它们和 BERT 的关系是什么？是基于 BERT-large 做的吗？

先说结论：

> **ModernBERT 是 BERT 风格的新一代 encoder-only Transformer。它不是简单基于原始 BERT-large 改出来的。**
> **mmBERT-32K 是这篇 vLLM Semantic Router 论文里用的长上下文分类模型，基于 ModernBERT 思路/架构扩展到 32K token，用于 domain / jailbreak / PII 等分类。**

---

## 2.1 BERT 是什么？

BERT 是一种 **encoder-only Transformer**。

Transformer 大致有三类：

| 类型            | 代表                      | 适合任务                        |
| --------------- | ------------------------- | ------------------------------- |
| Encoder-only    | BERT、RoBERTa、ModernBERT | 分类、检索、embedding、文本匹配 |
| Decoder-only    | GPT、LLaMA、Qwen          | 文本生成、聊天                  |
| Encoder-decoder | T5、BART                  | 翻译、摘要、生成                |

BERT 的特点是：

```text
双向理解文本
不主要用于生成
更适合分类、匹配、检索
```

比如判断一句话是正面还是负面、判断两个句子是否相似、判断 prompt 属于什么领域，这些都适合 BERT 类模型。

---

## 2.2 ModernBERT 是什么？

ModernBERT 可以理解成：

> 把 BERT 这种 encoder-only 架构，用现代 Transformer 技术重新做了一遍。

Hugging Face 页面写得比较清楚：ModernBERT 是一个 **modernized bidirectional encoder-only Transformer model，BERT-style**，在 2 trillion 英文和代码 token 上预训练，原生上下文长度到 8192 token。它用了 RoPE、local-global alternating attention、unpadding、Flash Attention 等现代优化。([Hugging Face](https://huggingface.co/answerdotai/ModernBERT-large "answerdotai/ModernBERT-large · Hugging Face"))

它和老 BERT 的关系可以这么看：

```text
BERT：老一代 encoder-only Transformer
ModernBERT：现代化后的 BERT 风格 encoder-only Transformer
```

它不是 GPT 那种生成模型，而是仍然偏向：

```text
分类
检索
语义匹配
embedding
rerank
```

---

## 2.3 ModernBERT 相比原始 BERT 改了什么？

原始 BERT 的上下文长度通常是 **512 token**。ModernBERT 原生支持 **8192 token**，这对长文档分类、RAG 检索、代码检索更友好。([Hugging Face](https://huggingface.co/answerdotai/ModernBERT-large "answerdotai/ModernBERT-large · Hugging Face"))

它主要现代化在几个方面：

| 点               | BERT                 | ModernBERT                  |
| ---------------- | -------------------- | --------------------------- |
| 位置编码         | 绝对位置编码         | RoPE                        |
| 上下文长度       | 通常 512             | 原生 8192                   |
| Attention        | 全局 attention       | local/global 交替 attention |
| padding 处理     | 经常算 padding token | unpadding 减少无效计算      |
| attention kernel | 传统实现             | 使用 Flash Attention        |
| 激活函数         | GeLU                 | GeGLU 等现代结构            |

ModernBERT 论文里说，每三层中有一层使用 global attention，其余层使用 128-token local sliding window attention。([arXiv](https://arxiv.org/html/2412.13663v2 "Smarter, Better, Faster, Longer: A Modern Bidirectional Encoder for Fast, Memory Efficient, and Long Context Finetuning and Inference"))

这句话什么意思？

假设输入有 8192 个 token。

**global attention** 是：

```text
每个 token 都能看见所有 token
```

**local attention** 是：

```text
每个 token 只看附近一小段 token
```

比如窗口大小 128，那么第 1000 个 token 主要只看 1000 附近的 token，而不是看完整 8192 个 token。

这样可以降低长文本 attention 的计算量和显存。

---

## 2.4 mmBERT-32K 是什么？

这篇论文里的 **mmBERT-32K** 可以理解成：

> 一个用于 Semantic Router 的长上下文 BERT 类分类器，能够处理最长 32K token 的输入。

它的作用不是聊天，不是生成回答，而是做分类。

在论文里，它被用于多个 signal classifier：

```text
domain classifier：判断请求领域
jailbreak classifier：判断是否越狱
PII classifier：判断是否含隐私信息
modality classifier：判断模态类型
```

论文摘要里明确说，router 要拦截 LLM 请求做 safety classification、domain routing、PII detection，而且长上下文分类在 8K-32K token 下会遇到标准 attention 的 O(n²) 显存问题。([arXiv](https://arxiv.org/abs/2603.12646?utm_source=chatgpt.com "98$\times$ Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

## 2.5 mmBERT-32K、ModernBERT、BERT 的关系

可以画成这样：

```text
Transformer
  ↓
Encoder-only Transformer
  ↓
BERT
  ↓
ModernBERT：现代化 BERT 风格 encoder-only 模型
  ↓
mmBERT-32K：这篇论文里用于 router 分类的长上下文 ModernBERT 类模型
```

更准确地说：

```text
BERT 是祖先类型
ModernBERT 是现代化 BERT-style encoder
mmBERT-32K 是用于本文路由分类的长上下文模型
```

---

## 2.6 它们是基于 BERT-large 做的吗？

**不是简单基于 BERT-large 做的。**

原始 BERT-large 大概是：

```text
24 层
hidden size 1024
约 340M 参数
最大长度通常 512 token
```

ModernBERT-large 是：

```text
28 层
约 395M 参数
原生 8192 token
RoPE
local/global attention
Flash Attention
GeGLU
unpadding
```

Hugging Face 页面列出 ModernBERT-base 是 149M 参数，ModernBERT-large 是 395M 参数。([Hugging Face](https://huggingface.co/answerdotai/ModernBERT-large "answerdotai/ModernBERT-large · Hugging Face"))

所以你可以说它是 **BERT-style**，但不能说它是“拿 BERT-large 继续训一下”这么简单。

更合适的表述是：

> ModernBERT 是重新设计和重新预训练的现代 encoder-only 模型，继承了 BERT 的“双向 encoder”思想，但架构和训练方式都已经现代化。mmBERT-32K 则是这篇论文里基于这种长上下文 encoder 思路，用于路由分类的模型。

---

# 3. “CPU 跑 ONNX 或 Candle”怎么理解？CPU 模型的格式一般是这两种吗？

这句话要拆开。

**ONNX** 和 **Candle** 不是“CPU 模型格式”的同一类东西。

它们分别是：

| 名称         | 本质              | 你可以怎么理解               |
| ------------ | ----------------- | ---------------------------- |
| ONNX         | 模型交换格式      | 一种保存模型计算图的标准格式 |
| ONNX Runtime | 推理引擎          | 用来运行 ONNX 模型           |
| Candle       | Rust 机器学习框架 | 类似轻量 PyTorch / 推理框架  |

---

## 3.1 ONNX 是什么？

ONNX 全称是 **Open Neural Network Exchange**。

它可以理解成一种模型中间格式。

比如你在 PyTorch 里训练了一个模型：

```python
model = MyClassifier()
```

训练好以后，你可以导出成：

```text
model.onnx
```

然后在部署服务里用 ONNX Runtime 加载它。

大概流程：

```text
PyTorch 训练模型
  ↓ export
ONNX 文件
  ↓
ONNX Runtime 推理
```

ONNX Runtime 可以跑在 CPU 上，也可以跑在 GPU 上。不是说 ONNX 只能 CPU。

这篇论文里说的 **ONNX CPU baseline**，意思是：

```text
用 ONNX Runtime 加载模型，但是执行设备是 CPU
```

---

## 3.2 Candle 是什么？

Candle 是 Hugging Face 做的一个 Rust 机器学习框架。它是一个 minimalist ML framework for Rust，重点是性能和易用性，也支持 GPU。([GitHub](https://github.com/huggingface/candle?utm_source=chatgpt.com "huggingface/candle: Minimalist ML framework for Rust"))

你可以把 Candle 理解成：

```text
Rust 版轻量深度学习/推理框架
```

为什么这篇论文会提 Candle？

因为 vLLM Semantic Router 很可能是 Go/Rust/C++ 这类工程服务体系，作者想比较不同推理后端：

```text
ONNX Runtime CPU 跑分类器
Candle CPU 跑分类器
ONNX Runtime GPU + FlashAttention 跑分类器
```

这里的重点不是模型格式，而是：

> 分类器在不同推理后端、不同执行设备上的延迟。

---

## 3.3 CPU 模型格式一般是 ONNX 或 Candle 吗？

不是。

更准确地说：

> CPU 只是执行设备；模型格式和推理框架有很多种。

常见组合是：

| 模型格式 / 框架            |            可以 CPU 吗 |  可以 GPU 吗 |
| -------------------------- | ---------------------: | -----------: |
| PyTorch`.pt`/`.bin`    |                   可以 |         可以 |
| ONNX`.onnx`              |                   可以 |         可以 |
| TensorFlow SavedModel      |                   可以 |         可以 |
| TensorRT engine            |               主要 GPU |           是 |
| GGUF                       | 常用于 CPU / llama.cpp | 也可部分 GPU |
| Candle native weights      |                   可以 |         可以 |
| Safetensors + Transformers |                   可以 |         可以 |

所以“CPU 跑 ONNX 或 Candle”不是说 CPU 模型只有两种格式，而是说：

> 这篇论文比较了两种 CPU 推理实现：ONNX Runtime CPU 和 Candle CPU。

---

## 3.4 为什么 CPU 跑会慢？

因为 mmBERT-32K 这类 Transformer 分类器里有大量矩阵乘法和 attention 计算。

CPU 擅长通用逻辑，但不如 GPU/NPU 擅长大规模并行矩阵计算。

所以 8K token 时：

```text
ONNX CPU：4918 ms
Candle CPU：1818 ms
GPU + Flash Attention：127 ms
```

论文摘要也说，完整优化把 8K token 路由从 4918 ms 降到 50 ms。([arXiv](https://arxiv.org/abs/2603.12646?utm_source=chatgpt.com "98$\times$ Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

---

# 4. SDPA OOM 是什么？什么是 SDPA？

先说结论：

> **SDPA** 是 Scaled Dot-Product Attention，也就是 Transformer attention 的标准计算形式。
> **SDPA OOM** 是说用这种标准 attention 算长文本时，显存不够，Out Of Memory。

---

## 4.1 SDPA 全称

SDPA：

```text
Scaled Dot-Product Attention
```

中文可以叫：

```text
缩放点积注意力
```

它是 Transformer 里最核心的一步。

公式大概是：

```text
Attention(Q, K, V) = softmax(QK^T / sqrt(d)) V
```

你不需要先管公式，先看直觉。

每个 token 都会生成三个向量：

```text
Q：Query，我要找什么
K：Key，我是什么
V：Value，我携带的信息
```

然后每个 token 的 Q 会和所有 token 的 K 做相似度计算。

例如有 4 个 token：

```text
A B C D
```

那么 attention 会算：

```text
A 看 A/B/C/D 的权重
B 看 A/B/C/D 的权重
C 看 A/B/C/D 的权重
D 看 A/B/C/D 的权重
```

这会形成一个矩阵：

```text
A   B   C   D
A    .   .   .   .
B    .   .   .   .
C    .   .   .   .
D    .   .   .   .
```

如果有 n 个 token，这个矩阵就是：

```text
n × n
```

---

## 4.2 为什么会 OOM？

因为 token 数一长，矩阵会平方级增长。

| token 数 | attention 矩阵规模 |
| -------: | -----------------: |
|      512 |         512 × 512 |
|     8192 |       8192 × 8192 |
|    16384 |     16384 × 16384 |
|    32768 |     32768 × 32768 |

8192 × 8192 大约是 6700 万个元素。

如果还有：

```text
batch size
num heads
多层 Transformer
mask
中间激活
并发请求
多个 classifier
```

显存就会很快爆掉。

论文里说，在 Router 和 vLLM 共用 GPU 时，标准 attention 的 O(n²) memory 会让 8K-32K 长上下文分类不可行；8K token 时，三个并发分类器仅 attention mask 就需要约 4.5GB，超过 vLLM 留给 Router 的显存。([arXiv](https://arxiv.org/abs/2603.12646?utm_source=chatgpt.com "98$\times$ Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

所以 **SDPA OOM** 的意思是：

```text
用标准 Scaled Dot-Product Attention 算长文本时，
中间 attention 矩阵 / mask / 激活占用太大，
导致 GPU 显存不够。
```

---

## 4.3 SDPA 是算法，还是某个库？

两者都可能被这么叫，但语境不同。

在理论上：

```text
SDPA = Transformer 标准 attention 公式
```

在工程里：

```text
SDPA = 某个框架里实现标准 attention 的算子
```

比如 PyTorch 里有：

```python
torch.nn.functional.scaled_dot_product_attention
```

在这篇论文里，说 “SDPA OOM” 更接近工程含义：

> ONNX Runtime ROCm 后端里使用标准 SDPA 实现，长序列下显存占用太大，导致 OOM。

---

# 5. 什么是 FlashAttention？什么是 CK FlashAttention？

---

## 5.1 FlashAttention 是什么？

FlashAttention 是一种更省显存、更高效的 attention 实现。

它不是改 Transformer 的数学结果，而是改计算方式。

普通 SDPA 的粗略流程是：

```text
1. 计算 QK^T，得到完整 n×n attention 分数矩阵
2. 对这个矩阵做 softmax
3. 再乘以 V
```

问题是第一步会产生巨大的中间矩阵。

FlashAttention 的思路是：

```text
不要把完整 n×n 矩阵一次性存到显存里
而是分块计算
边算边做 softmax
边算边和 V 融合
尽量使用 GPU SRAM / shared memory
减少 HBM 读写
```

你可以用做饭类比：

**普通 SDPA：**

```text
先把所有菜全切好，铺满整个厨房台面
再统一炒
```

**FlashAttention：**

```text
一小批一小批处理
处理完马上进入下一步
不把所有中间结果堆在台面上
```

ModernBERT 论文也说，Flash Attention 是现代 Transformer 模型中的核心组件，提供 memory and compute efficient attention kernels。([arXiv](https://arxiv.org/html/2412.13663v2 "Smarter, Better, Faster, Longer: A Modern Bidirectional Encoder for Fast, Memory Efficient, and Long Context Finetuning and Inference"))

---

## 5.2 FlashAttention 解决什么？

主要解决两个问题：

### 第一，省显存

普通 attention 需要显式存较大的中间 attention matrix。

FlashAttention 避免完整保存这个矩阵，所以显存占用更低。

### 第二，减少显存读写

GPU 很多时候不是算力不够，而是数据搬运慢。

FlashAttention 通过 kernel fusion 和 tiling，让数据尽量留在更快的片上存储里，减少读写 HBM 的次数。

---

## 5.3 FlashAttention 是近似的吗？

不是。

正常情况下，FlashAttention 是 **exact attention**，不是近似 attention。

它计算的数学结果应该和标准 attention 基本一致，只是浮点误差层面可能有细微差异。

所以它不是：

```text
少看一些 token
降低精度来换速度
```

而是：

```text
同样的 attention，用更高效的方式算
```

---

## 5.4 什么是 CK FlashAttention？

CK 是 **Composable Kernel**，是 AMD ROCm 生态里的高性能 kernel 库。

所以 **CK FlashAttention** 可以理解成：

> 基于 AMD Composable Kernel 实现的 FlashAttention。

这篇论文的 Stage 1 就是在 ONNX Runtime 的 ROCm 后端里接入一个自定义算子：

```text
com.ck::CKFlashAttention
```

为什么要这么做？

因为 NVIDIA 生态里 FlashAttention 支持比较成熟，但论文说 AMD ROCm + ONNX Runtime 的场景下，缺少对应的 FlashAttention 集成。论文摘要也明确说，Stage 1 目标是 AMD ROCm，而 NVIDIA GPU 已经可以通过 cuDNN 获得 FlashAttention；他们做的 CK Flash Attention operator for ONNX Runtime on ROCm，把 attention memory 从 O(n²) 降到 O(n)，并让 SDPA OOM 的 8K-32K token 场景可以跑起来。([arXiv](https://arxiv.org/abs/2603.12646?utm_source=chatgpt.com "98$\times$ Faster LLM Routing Without a Dedicated GPU: Flash Attention, Prompt Compression, and Near-Streaming for the vLLM Semantic Router"))

所以：

```text
FlashAttention：一类高效 attention 算法/实现思想
CK FlashAttention：AMD CK 版本的 FlashAttention 实现
```

类似于：

```text
排序算法：sort
C++ 实现：std::sort
Python 实现：list.sort
```

FlashAttention 是大类，CK FlashAttention 是 AMD ROCm 场景下的一个具体实现。

---

# 6. 什么是 BUFFERED mode？

这是 Envoy ext_proc 里的 body 处理模式。

Envoy 官方 proto 文档说，request body 的行为取决于 body send mode：在 `BUFFERED` 或 `BUFFERED_PARTIAL` 模式下，body 会以单个 message 发送给 external processor；在 `STREAMED` 或 `FULL_DUPLEX_STREAMED` 模式下，body 会拆成多个 message 发给 external processor；`NONE` 模式则不发送 body。([Envoy Proxy](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/ext_proc/v3/ext_proc.proto "External Processing Filter (proto) — envoy 1.39.0-dev-6a45c7 documentation"))

简单说：

```text
BUFFERED mode = 先把完整 body 收齐，再一次性交给外部处理服务
STREAMED mode = body 来一块，就发一块给外部处理服务
NONE mode = 不把 body 给外部处理服务
```

---

## 6.1 什么是 HTTP Body？

一个 HTTP 请求大致分三部分：

```text
请求行
请求头 Headers
请求体 Body
```

比如 OpenAI Chat Completions 请求：

```http
POST /v1/chat/completions HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: Bearer xxx

{
  "model": "auto",
  "messages": [
    {
      "role": "user",
      "content": "帮我解释 TCP ACK"
    }
  ],
  "stream": true
}
```

上面的：

```text
POST /v1/chat/completions HTTP/1.1
```

是请求行。

下面这些是 headers：

```text
Host
Content-Type
Authorization
```

最后这个 JSON 就是 body：

```json
{
  "model": "auto",
  "messages": [
    {
      "role": "user",
      "content": "帮我解释 TCP ACK"
    }
  ],
  "stream": true
}
```

---

## 6.2 “收齐 HTTP Body 以后再交由外部处理服务”怎么理解？

你的类比基本是对的，但要稍微修正一下。

你说：

> 是类似于，一个服务里有 http 组件、解析组件，http 组件完成了 body 获取以后，把东西交由解析组件去处理吗？

这个理解在单进程内部是类似的。

但是在 Envoy ext_proc 里，它不是同一个服务里的两个组件，而是两个服务：

```text
Envoy 进程
  ↓ gRPC
External Processor 进程，也就是 Semantic Router
```

更准确是：

```text
Envoy 负责接收 HTTP 请求
Envoy 先把 body 完整读完并缓存起来
Envoy 把完整 body 封装成 gRPC protobuf 消息
Envoy 发给 Semantic Router
Semantic Router 解析 body，做分类和路由
Semantic Router 返回修改结果
Envoy 继续把请求发给 vLLM
```

所以它确实类似：

```text
HTTP 组件收完 body -> 解析组件处理
```

但中间多了一层：

```text
跨进程 / 跨服务的 gRPC 通信
```

---

# 6.3 BUFFERED mode 下的完整链路

假设用户请求是：

```json
{
  "model": "auto",
  "messages": [
    {
      "role": "user",
      "content": "这里是一段 16K token 的长文档……"
    }
  ],
  "stream": true
}
```

在 BUFFERED mode 下，大致是：

```text
1. 用户开始上传 HTTP body
2. Envoy 暂时不把请求继续转给 vLLM
3. Envoy 把整个 body 收完
4. Envoy 把完整 body 存在内存里
5. Envoy 把 body 包装成 protobuf message
6. Envoy 通过 gRPC 发给 Semantic Router
7. Semantic Router 收到 protobuf message
8. Semantic Router 从里面取出原始 JSON body
9. Semantic Router json.Unmarshal 成结构体
10. Semantic Router 取出 model/messages
11. Semantic Router 跑 prompt compression / classifier
12. Semantic Router 判断 auto 应该改成哪个模型
13. Semantic Router 修改结构体里的 model 字段
14. Semantic Router json.Marshal 回 JSON 字节串
15. Semantic Router 把修改后的 body 返回给 Envoy
16. Envoy 用新 body 继续请求 vLLM
```

这就是为什么它慢：长 body 在这个链路里被读、存、编码、解码、解析、再编码。

---

# 6.4 解释你不理解的四步

## 第一步：请求体要被完整缓存

原句：

> 请求体要被完整缓存。

意思是：

```text
Envoy 必须先把整个 HTTP body 收到内存里，不能来一点处理一点。
```

比如 body 是 10MB 的 JSON：

```text
用户上传第 1MB
Envoy 暂存
用户上传第 2MB
Envoy 暂存
...
用户上传第 10MB
Envoy 暂存
```

等 10MB 全部收完，Envoy 才把它交给 Semantic Router。

这会有两个问题：

### 问题一：内存占用

如果一个请求 body 是 10MB，100 个并发请求就是：

```text
10MB × 100 = 1000MB
```

这还只是 body 缓存，不包括 protobuf、JSON 解析后的结构体、classifier 输入等。

### 问题二：延迟

Semantic Router 必须等 body 全部收完后才能开始处理。

如果用户上传很慢，或者 body 很大，就会增加等待时间。

---

## 第二步：gRPC protobuf 要序列化/反序列化

原句：

> gRPC protobuf 要序列化/反序列化。

这句话说的是 Envoy 和 Semantic Router 之间通信的成本。

Envoy ext_proc 不是直接把内存指针交给 Semantic Router。它们之间通过 gRPC 通信，而 gRPC 默认使用 protobuf 编码消息。

### 什么是 protobuf？

protobuf 可以理解成一种二进制消息格式。

比如原始 JSON body 是：

```json
{
  "model": "auto",
  "messages": [
    {
      "role": "user",
      "content": "hello"
    }
  ]
}
```

Envoy 要把它放进 ext_proc 的请求消息里，大概像：

```text
ProcessingRequest {
  request_body {
    body: "{...原始 JSON bytes...}"
    end_of_stream: true
  }
}
```

这个 `ProcessingRequest` 不是直接以文本形式发送，而是要编码成 protobuf 二进制。

### 序列化是什么意思？

序列化就是：

```text
内存里的对象 -> 网络上传输的字节
```

比如 Envoy 内部有一个 C++ 对象：

```cpp
ProcessingRequest req;
req.body = json_body;
```

要通过 gRPC 发出去，就要变成一串 bytes：

```text
08 96 01 12 8A 01 ...
```

这一步叫 **serialize / marshal / encode**。

### 反序列化是什么意思？

反序列化就是反过来：

```text
网络收到的字节 -> 内存里的对象
```

Semantic Router 收到 bytes 后，要还原成：

```go
ProcessingRequest
```

这一步叫 **deserialize / unmarshal / decode**。

所以一次 ext_proc body 处理至少有：

```text
Envoy：protobuf 序列化
网络/gRPC：传输
Semantic Router：protobuf 反序列化
Semantic Router：protobuf 序列化响应
Envoy：protobuf 反序列化响应
```

这就是额外开销。

---

## 第三步：Router 里还要对 OpenAI 格式 JSON 做 json.Unmarshal

原句：

> Router 里还要对 OpenAI 格式 JSON 做 json.Unmarshal。

这个和上一步不是同一个东西。

上一步是：

```text
gRPC protobuf 的解析
```

这一步是：

```text
HTTP body 里面 JSON 的解析
```

也就是说，Semantic Router 收到 protobuf 后，只是拿到了 body bytes：

```text
body = b'{"model":"auto","messages":[...]}'
```

但它还不知道 JSON 里面的字段含义。

它要读取：

```text
model 是什么？
messages 在哪里？
用户 prompt 是什么？
stream 是 true 还是 false？
```

所以它要把 JSON 字符串解析成程序里的结构体。

在 Go 里，常见写法是：

```go
var req ChatCompletionRequest
json.Unmarshal(bodyBytes, &req)
```

逐行解释一下：

```go
var req ChatCompletionRequest
```

定义一个变量 `req`，类型是 `ChatCompletionRequest`。这个结构体里面可能有：

```go
Model    string
Messages []Message
Stream   bool
```

然后：

```go
json.Unmarshal(bodyBytes, &req)
```

含义是：

```text
把 bodyBytes 里的 JSON 字节解析出来，填充到 req 这个结构体里。
```

举例：

输入 JSON：

```json
{
  "model": "auto",
  "stream": true
}
```

解析后：

```go
req.Model  == "auto"
req.Stream == true
```

这一步的成本是：JSON 越大，解析越慢，内存分配越多。

---

## 第四步：修改模型名后还要 json.Marshal 回去

原句：

> 修改模型名后还要 json.Marshal 回去。

Router 判断完之后，可能要把：

```json
"model": "auto"
```

改成：

```json
"model": "qwen-code"
```

如果它已经把 JSON 解析成 Go 结构体，那么修改很简单：

```go
req.Model = "qwen-code"
```

但问题是，Envoy 和 vLLM 需要的是 HTTP body，也就是 JSON 字节串，不是 Go 结构体。

所以还要做：

```go
newBodyBytes, err := json.Marshal(req)
```

含义是：

```text
把 Go 结构体重新编码成 JSON 字节。
```

比如内存里的结构体：

```go
ChatCompletionRequest{
    Model: "qwen-code",
    Stream: true,
}
```

重新变成：

```json
{
  "model": "qwen-code",
  "stream": true
}
```

这一步叫 **json.Marshal**。

它的问题是：

1. 要重新遍历整个结构体；
2. 要重新生成一份 JSON 字节；
3. 长 prompt 会被重新复制；
4. body 很大时，耗时和内存都会增加。

---

# 6.5 这四步为什么会慢？

把四步连起来看：

```text
原始 HTTP body
  ↓ Envoy 缓存完整 body
Envoy 内存中的 body bytes
  ↓ protobuf 序列化
gRPC 二进制消息
  ↓ Semantic Router 反序列化 protobuf
Router 拿到 body bytes
  ↓ json.Unmarshal
Go 结构体
  ↓ 修改 model 字段
Go 结构体
  ↓ json.Marshal
新的 JSON body bytes
  ↓ protobuf 序列化响应
gRPC 响应
  ↓ Envoy 反序列化 protobuf
Envoy 得到新 body
  ↓
转发给 vLLM
```

这里至少有两套编码/解码：

```text
protobuf 编解码
JSON 编解码
```

还有多次内存复制：

```text
HTTP body buffer
protobuf message buffer
JSON parse 后的结构体
marshal 出来的新 JSON buffer
```

所以论文才说要做：

```text
near-streaming body processing
zero-copy JSON
```

目标是减少：

```text
完整缓存
完整 JSON 解析
完整 JSON 重写
多次内存复制
```

---

# 6.6 Zero-copy JSON 是怎么减少开销的？

传统做法：

```go
var req ChatCompletionRequest
json.Unmarshal(body, &req)

req.Model = "qwen-code"

newBody, _ := json.Marshal(req)
```

这会完整解析和完整重写。

论文里提到的 zero-copy JSON 思路更像：

```text
不用完整解析整个 JSON
只直接找到 model 字段
只替换 model 字段
其他 body 内容尽量不动
```

比如原始 body：

```json
{
  "model": "auto",
  "messages": [
    {
      "role": "user",
      "content": "很长很长的 prompt ..."
    }
  ]
}
```

其实 Router 如果只想改 `model`，没必要把整个 `messages` 完整解析成结构体，再重新生成。

它可以做：

```text
找到 $.model
把 auto 替换成 qwen-code
其他字段保持原样
```

这样可以减少大量 JSON 解析和重写成本。

---

# 6.7 BUFFERED mode 和 STREAMED mode 的区别，用水管理解

## BUFFERED mode

像先拿一个大桶把水接满：

```text
水管来水
  ↓
桶里攒满
  ↓
一次性交给处理器
```

对应 HTTP：

```text
body chunk 1
body chunk 2
body chunk 3
...
全部收完
  ↓
一次性发给 Semantic Router
```

优点：

```text
处理逻辑简单
外部服务一次就能看到完整 body
```

缺点：

```text
必须等待完整 body
内存占用高
长 prompt 延迟高
```

---

## STREAMED mode

像水一来就边接边处理：

```text
水管来一点
  ↓
处理一点
水管再来一点
  ↓
再处理一点
```

对应 HTTP：

```text
body chunk 1 -> 发给 Semantic Router
body chunk 2 -> 发给 Semantic Router
body chunk 3 -> 发给 Semantic Router
```

优点：

```text
不用等完整 body
可以边收边处理
降低首段处理延迟
```

缺点：

```text
逻辑复杂
如果必须修改完整 JSON，还是可能需要攒起来
```

---

## 这篇论文说的 Near-Streaming

它不是完全 streaming，而是接近 streaming。

它的思想是：

```text
如果不需要看完整 body，就直接 passthrough
如果需要做 auto routing，就边收 body 边做一些预处理
最后必要时再汇总判断
```

比如：

### 情况一：用户已经指定模型

```json
{
  "model": "qwen-72b",
  "messages": [...]
}
```

Router 不需要 domain routing，可以直接放行。

```text
不需要完整缓存 body
不需要完整解析 messages
```

### 情况二：用户写的是 model=auto

```json
{
  "model": "auto",
  "messages": [...]
}
```

这时需要看 prompt 内容，判断该走哪个模型。

所以它仍然要处理 body，但可以在 body chunk 到达时做一些预处理，比如句子切分、TF-IDF 统计，而不是等全部收完后才开始所有工作。

---

# 最后把 6 个问题串起来

这篇论文里的链路可以这么理解：

```text
Envoy ext_proc：
    让 Envoy 可以调用外部 Semantic Router 处理请求

mmBERT-32K：
    Semantic Router 里面用于分类的长上下文 BERT 类模型

ModernBERT：
    mmBERT-32K 背后的现代 BERT-style encoder 架构思想

ONNX / Candle：
    跑分类模型的不同推理后端

SDPA：
    标准 scaled dot-product attention，长文本时显存 O(n²)

SDPA OOM：
    标准 attention 在 8K/16K/32K token 下显存爆掉

FlashAttention：
    更省显存、更高效的 attention 实现

CK FlashAttention：
    AMD ROCm / Composable Kernel 版本的 FlashAttention

BUFFERED mode：
    Envoy 先收完整 HTTP body，再通过 gRPC 一次性交给 Router

protobuf 序列化/反序列化：
    Envoy 和 Router 之间 gRPC 通信的编码解码成本

json.Unmarshal：
    Router 把 OpenAI JSON body 解析成程序结构体

json.Marshal：
    Router 修改 model 后，把结构体重新转回 JSON body
```

你可以把这篇论文的核心问题压缩成一句话：

> Semantic Router 想看 prompt 内容来决定路由，但长 prompt 会让 body 处理、JSON 解析、BERT 分类、attention 显存全部变成瓶颈，所以作者分别从 Envoy body 处理、JSON 零拷贝、Prompt Compression、FlashAttention 四个层面把这些瓶颈压下去。
