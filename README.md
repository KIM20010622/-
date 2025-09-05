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
  "accu": 罪名编号,
  "law": 法条编号,
  "time": 36,
  "term_cate": 2,
  "term": 6
}
