## 1. Base 模型 vs 微调模型

| 特性 | Base 模型 | 微调模型 |
|------|----------|----------|
| **训练阶段** | 仅预训练（Pre-training） | 预训练 + 下游任务微调（Fine-tuning） |
| **训练数据** | 大规模无标签语料（如 Wikipedia） | 小规模标注数据（如 SST-2, SQuAD） |
| **是否含任务头** | ❌ 否（只有 Transformer 主干） | ✅ 是（如分类层、问答头） |
| **能否直接推理** | ❌ 需添加并训练任务头 | ✅ 可直接用于预测 |
| **泛化能力** | 强（通用语言理解） | 弱（专精特定任务） |
| **示例** | `bert-base-uncased` | `textattack/bert-base-uncased-SST-2` |

> 💡 **比喻**：  
> - Base 模型 = 上过大学但没实习的学生  
> - 微调模型 = 已在岗位熟练工作的员工

---

## 2. 编码器是训练出来的吗？输入维度如何确定？

###  编码器是端到端训练的神经网络，包含：
- **可学习的 Token Embedding**：`[vocab_size, hidden_size]`
- **可学习的位置/段落嵌入**
- **多层 Transformer 编码块**：
  - 多头自注意力（Q/K/V 投影矩阵可训练）
  - 前馈网络（FFN，全连接层可训练）
  - LayerNorm（含可学习 γ, β）

### 🔧 输入向量维度（`hidden_size` / `d_model`）：
- 由模型配置文件（如 `config.json`）**预先定义**的超参数
- 常见值：BERT-base=768，BERT-large=1024，Llama-3=4096
- **训练和推理必须一致**，不可动态改变

> ✅ 所有参数通过反向传播联合优化，非静态映射。

---

## 3. 预训练中的“标签”从哪来？Loss 如何计算？

### 自监督学习（Self-supervised Learning）：从文本自身构造标签

#### BERT（MLM）：
- **输入**：`"I love [MASK] processing."`
- **标签**：被 mask 的原始词（如 `"natural"`）
- **Label 张量**：非 mask 位置设为 `-100`（PyTorch 忽略）
- **Loss**：仅在 mask 位置计算交叉熵

#### GPT（CLM）：
- **输入**：`["I", "love", "NLP"]`
- **标签**：`["love", "NLP", "."]`（下一个词）
- **Loss**：所有位置的 next-token 预测交叉熵

> 🔑 **核心思想**：无需人工标注，利用语言结构自动构造监督信号。

---

## 4. BERT 有解码器吗？MLM 输出如何限定到 [MASK]？

### ❌ BERT 是 **Encoder-only** 模型
- **无 Transformer Decoder**
- **无自回归生成能力**
- **不适用于开放文本生成**

### ✅ MLM Loss 限定机制：
- 模型输出：`[B, L, vocab_size]` logits
- 构造 labels：仅 mask 位置为真实 token ID，其余为 `-100`
- Loss 计算时自动忽略 `-100` 位置

```python
loss = CrossEntropyLoss(ignore_index=-100)(logits.view(-1, V), labels.view(-1))
```

> 📌 BERT 的“预测”由 **MLM Head（分类头）** 完成，非 Decoder。

---

## 5. [MASK] 是固定向量吗？为何能适应不同上下文？

### ✅ 初始 embedding 是固定的（但可学习）
- `[MASK]` 在 embedding 表中有唯一一行（如第 103 行）
- 预训练中被优化为“中性占位符”

### ✅ 但最终表示是**上下文感知**的
- 通过 **Self-Attention**，`[MASK]` 位置聚合其他词的信息
- 例：
  - `"The cat sat on the [MASK]."` → 输出 ≈ `"mat"`
  - `"She works at a software [MASK]."` → 输出 ≈ `"company"`

> 💡 **[MASK] 是“空白画布”，BERT 用上下文在其上“作画”**

---

## 6. 相同输入向量是否总得相似输出？

### ❌ 不是！

- **相同初始向量 + 不同上下文 → 完全不同的输出**
  - 如 `[MASK]` 在不同句中输出差异巨大
- **不同初始向量 + 相同上下文 → 可能相似（若语义相近）**
  - 如 `"car"` 和 `"automobile"` 在 `"I drive a ___."` 中输出接近

> ⚠️ 若输入分布外向量（如随机噪声），输出将无意义。  
> BERT 对输入分布敏感，非“万能扭曲器”。

---

## 7. 为何输出维度=输入维度，却说无解码器？

### ✅ 维度一致 ≠ 有解码器
- **Transformer Encoder 设计要求**：每层输入/输出维度相同（便于残差连接）
- **输出用途不同**：
  - BERT：768 维向量 → 接分类头做判别任务
  - T5/GPT：Decoder 逐词生成新序列

### 🔑 解码器的定义（标准 Transformer）：
1. Masked Self-Attention（防信息泄露）
2. Encoder-Decoder Cross-Attention
3. 自回归生成机制

> ✅ BERT **不具备以上任何一项**，故无解码器。

---

## 8. 能否用 embedding 表直接“反查”输出词？

### ❌ 不能！原因如下：

| 问题 | 说明 |
|------|------|
| **不可逆映射** | embedding 是 `[vocab] → ℝ^d` 的压缩，无法唯一反推 |
| **空间不一致** | 输出 `h` 是上下文表示，`e_w` 是静态表示 |
| **需要非线性变换** | MLM Head 中的 LayerNorm + Linear 对性能至关重要 |

### ✅ 正确做法：使用 **MLM Head**
- 通常与 embedding 权重共享（weight tying）
- 公式：`logits = (transformed_h) @ embedding_weight.T`
- 再经 Softmax 得词概率

> 🔬 实验证明：省略 MLM Head 会导致性能显著下降。

---

## 9. QKV 如何体现语义？

### 语义在 **注意力交互中动态构建**

#### 步骤：
1. 每个词生成 Q, K, V
2. `QKᵀ` 计算词间**语义相关性分数**
   - 例：`"bank"` 的 Q 与 `"money"` 的 K 高度匹配
3. `softmax(QKᵀ)V` 加权聚合上下文信息
4. FFN 进行非线性提炼，存储高层概念

#### 多层效果：
- 底层 Head：关注语法（主谓、邻近词）
- 高层 Head：关注语义（指代、话题、情感）

> 💡 **QKV 本身是数学操作，但注意力分布 = 模型对“语义关联”的内部判断**

---

## 10. BERT 输出如何对应到具体单词？

### 通过 **MLM 分类头（MLM Head）** 完成映射

#### 流程：
1. 取 `[MASK]` 位置输出向量 `h ∈ ℝ^768`
2. 经 `LayerNorm → Linear` 变换
3. 投影到词表：
   ```python
   logits = transformed_h @ embedding_weight.T  # [vocab_size]
   ```
4. `Softmax → argmax` 得预测词

#### 实现（Hugging Face）：
```python
class BertForMaskedLM:
    def __init__(self):
        self.bert = BertModel(config)
        self.cls = BertOnlyMLMHead(config)  # 包含 tied weight
```

> ✅ 这是一个**可训练的分类器**，非简单查表。

---

## 11. 核心总结

### 🎯 BERT 的本质
> **一个上下文感知的语义编码器，用于理解，而非生成。**

### 🔑 三大机制
1. **自监督预训练（MLM）**  
   → 从无标签文本自动构造学习信号
2. **Transformer Encoder（QKV + FFN）**  
   → 动态融合上下文，构建语义表示
3. **MLM Head（带 weight tying）**  
   → 将语义向量映射回词汇空间

### ✅ 适用任务
- 文本分类、NER、问答、语义匹配等**理解型任务**

### ❌ 不适用任务
- 文本生成、机器翻译、摘要（需 Encoder-Decoder 架构）

> 💬 **最后一句**：  
> **BERT 不是字典，不是生成器，而是一个在对话中理解词语含义的智能体。**

---

*📝 整理完毕 | 共覆盖 10+ 个核心问题 | 适合 NLP 学习者反复研读*