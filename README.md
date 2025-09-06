# LADAN复现

首先运行LADAN-master/data_and_config/data/tongji_3.py用于对法律判决文书数据进行预处理与清洗，生成可用于 **罪名预测、法条预测、刑期预测** 的标准化数据集。

原始代码的路径有问题，应改成如下：
```python
flaw = open("../../law.txt",'r')
flaw = open("../../accu.txt",'r',encoding='utf-8')
```


## 功能说明
该脚本主要完成以下工作：
1. **读取映射表**：加载 `law.txt` 和 `accu.txt`，建立法条与罪名的编号映射。建立两个映射字典：  
- `law2num` / `num2law` ：法律到编号、编号到法律  
- `accu2num` / `num2accu` ：罪名到编号、编号到罪名  

这样可以方便后续将文字类别转换成数字索引。  
2. **加载数据集**：读取 `data_train.json`、`data_valid.json`、`data_test.json`。

每一行是一个案例（JSON 对象），包含以下字段：  

- `"fact"`：案件事实描述（文本）  
- `"meta"`：标签信息，主要包括：  
  - `"accusation"`：罪名
  - `"relevant_articles"`：相关法律条文
  - `"term_of_imprisonment"`：刑期信息（字典），包含：  
    - `"imprisonment"`：有期徒刑时长（单位：月）  
    - `"death_penalty"`：是否判处死刑（布尔值）  
    - `"life_imprisonment"`：是否判处无期徒刑（布尔值）  

#### 示例

```json
{
  "fact": "被告人张三因盗窃财物被公安机关抓获，经审理查明……",
  "meta": {
    "accusation": ["盗窃罪"],
    "relevant_articles": ["第264条"],
    "term_of_imprisonment": {
      "imprisonment": 24,
      "death_penalty": false,
      "life_imprisonment": false
    }
  }
}
```
3. **样本过滤**：
   - 删除二审案件`strpass in dic["fact"] != -1`；
   - 删除涉及多个罪名或多个法条的案件`len(dic["meta"]["accusation"]) > 1`、`len(dic["meta"]["relevant_articles"]) > 1`。
4. **类别筛选**：
   - 统计各罪名/法条的样本数量；
   - 只保留样本数 ≥ 100 的类别；
   - 生成新的 `new_law.txt` 和 `new_accu.txt`。
5. **案情处理**：
   - 使用正则表达式提取主要案情事实部分；
   - 使用 [THULAC](http://thulac.thunlp.org/) 进行中文分词。
6. **标签构造**：
   - `accu`: 罪名编号；
   - `law`: 法条编号；
   - `time`: 刑期（月数）；
   - `term_cate`: 刑罚类别（0=死刑，1=无期，2=有期徒刑）；
   - `term`: 刑期区间（共 11 档，从 >10年 到 0 月）。
7. **输出数据集**：
   - `train_cs.json`
   - `valid_cs.json`
   - `test_cs.json`

最后每个样本的格式如下：
```json
{
  "fact_cut": "分词后的案情文本",
  "accu": "罪名编号",
  "law": "法条编号",
  "time": 36,
  "term_cate": 2,
  "term": 6
}
```
# LegalΔ: Enhancing Legal Reasoning in LLMs via Reinforcement Learning with Chain-of-Thought Guided Information Gain

## 📌 简介
**LegalΔ** 是一个用于提升法律大模型（Legal LLMs）推理能力的全新框架。  
它结合了 **强化学习（RL）**、**链式思维（Chain-of-Thought, CoT）** 与 **信息增益奖励机制**，能够让模型不仅给出正确结论，还能生成更可靠、可解释的法律推理过程。

---

## 🔍 背景
- 现有法律大模型往往 **直接输出答案**，缺乏中间推理链；
- 可解释性与稳定性不足，难以满足 **复杂法律场景** 的需求；
- 现有强化学习方法主要奖励“结果正确”，但忽略了 **推理过程的质量**。

**目标**：构建一个能够 **准确、稳定、可解释** 地进行法律推理的框架。

---

## ⚙️ 方法：LegalΔ 框架
### 1. 双模式输入
- **Direct Answer**：直接回答
- **Reasoning-augmented Answer**：带推理链的回答  
➡️ 模型通过比较两者，学习信息差。

### 2. 信息增益奖励（Information-Gain Reward）
- 基于 Q 值（RL）与 logit 的数学等价性；
- 计算有无推理链时的置信度差异（ΔQ）；
- 作为奖励函数，引导模型生成 **高质量推理链**。

### 3. 两阶段训练
1. **蒸馏阶段**：利用 DeepSeek-R1 的高质量推理链做监督微调（SFT）；  
2. **强化阶段**：基于 GRPO（Group Relative Policy Optimization），结合信息增益奖励进一步优化推理能力。

---

## 📊 实验设置
- **数据集**：CAIL2018、JEC-QA、LAIC2021（域内任务）；DiscLaw（跨域任务）。  
- **任务**：
  - 法条预测（SPP-F）
  - 罪名预测（CCP）
  - 量刑预测（SLP）
  - 案例分析（CAS）
  - 金额计算（CAC）
- **对比方法**：HanFei、Fuzi、LexiLaw、ChatLaw、DiscLaw，以及基于 Qwen2.5 的 Zero-shot / SFT / DPO / R1-Distill。

---

## 🚀 主要结果
- **性能提升**：域内任务平均提升约 **10%**；跨域任务（司法考试题）保持强竞争力。  
- **稳定性增强**：信息增益奖励减少训练批次的方差，提高训练效率。  
- **推理质量提升**：
  - Token 层面：更关注法律相关词汇；
  - 链路层面：推理链困惑度更低；
  - 长场景任务中改进尤为明显。

---

# 📘 In-domain vs Out-of-domain

## 📌 简介
在机器学习与大模型研究中，**In-domain（域内）** 与 **Out-of-domain（跨域）** 是评估模型泛化能力的两个重要概念。  

---

## 🔍 In-domain（域内）
- **定义**：训练数据与测试数据来自 **相同或相似的任务和分布**。  
- **作用**：衡量模型在“熟悉场景”下的性能。  
- **在本文中的任务**：
  - 法条预测（SPP-F）
  - 罪名预测（CCP）
  - 量刑预测（SLP）
  - 案例分析（CAS）
  - 金额计算（CAC）  
➡️ 数据主要来自 **CAIL2018、JEC-QA、LAIC2021**。

---

## 🌍 Out-of-domain（跨域）
- **定义**：测试任务与训练数据分布 **不一致**，模型未见过这些任务。  
- **作用**：评估模型的 **泛化能力**，即能否在“陌生场景”中保持良好表现。  
- **在本文中的任务**：
  - 国家司法考试（NJE）
  - 专利代理人考试（PAE）
  - 公务员/事业单位法律知识考试（PFE、UNGEE、LBK）  
➡️ 数据来自 **DiscLaw 基准**。

---

## 📖 对比总结
| 评测类别       | 数据来源 | 作用 | 在本文的任务例子 |
|----------------|----------|------|-----------------|
| **In-domain**  | 训练集中出现过的任务或相似分布 | 测试模型在“熟悉场景”下的性能 | 法条预测、罪名预测、量刑预测、案例分析、金额计算 |
| **Out-of-domain** | 训练集中没有出现的新任务或不同分布 | 测试模型的 **泛化能力** | 司法考试：NJE、PAE、PFE、UNGEE、LBK |

---

<img width="681" height="418" alt="image" src="https://github.com/user-attachments/assets/9d1ac125-ec02-4bbd-8acb-421ae61a9b31" />



