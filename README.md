# tianchi_multimodal

在 BLIP2 基础上进行改造，将原始「图-文」检索改为「文→图+文」检索。

数据集基于天池竞赛 [【天池经典打榜赛】赛道五-多模态图文检索赛](https://tianchi.aliyun.com/competition/entrance/532420/information) 的训练集与验证集构造。可用于电商领域「文->图文融合模态」的训练，以适配通过关键词查询商品图片与商品文本描述的需求。

---

## 数据集处理

- **文本**：原数据中的中文描述均通过 `qwen-mt-flash` 进行翻译。
- **图像**：图片内容通过 `qwen3-vl-32b-instruct` 补充对应描述。
  - **验证集**：每张图片 1 条描述。
  - **训练集**：每张图片 3 条描述，训练时会随机选择一条以增强文本多样性并提升 query 对语义变化的鲁棒性。

注：在使用大模型生成caption和翻译文本后，对数据进行了统一的清洗与校验，包括语言层面上去除中文及非英文字符，结构层面上校验 JSON 字段完整性与格式一致性，以及样本层面上过滤重复样本、空值样本和无法生成有效 caption 的图片，并同步删除其对应的 annotation，以保证数据集整体质量与可用性。

### 验证集格式示例

```json
{
  "query_id": 248816,
  "query_text": "Christmas pillow",
  "item_ids": {
    "1006938": "Green Christmas tree shaped cushion with festive embroidery and star patterns.",
    "561749": "Christmas table runner with snowman and trees.",
    "936929": "Green velvet cushion with white embroidery, festive design.",
    "286314": "Festive gingerbread man plush toy.",
    "141999": "Cushion with Santa face design in cream and red.",
    "183846": "Red Christmas pillow with Santa and reindeer design."
  }
}
```

### 训练集格式示例

```json
{
  "query_id": 1,
  "query_text": "Phone case clay",
  "image_id": 97905,
  "text_input": [
    "Blue glitter phone case with moon and clouds design",
    "Transparent TPU case with blue celestial pattern",
    "Dreamy night sky themed iPhone cover with sparkles"
  ]
}
```


### 数据文件说明

| 文件名 | 数量 | 说明 |
|--------|-------|-------|
| `MR_train_queries_with_captions.json` | ~25w数据，~13w图像 | 训练集查询及每图 3 条 caption |
| `MR_valid_queries_with_captions_standard_simple.jsonl` |~4.5k数据，~3w图像 | 验证集查询，每个查询对应多张图，每图1条caption |

路径：`data/en_aliyun_dataset/`

---

## 结果

图像特征提取选用的是**eva_vit_g.pth (vit)**，检索指标为 Recall@1/5/10 及均值。

| 任务  | BLIP2 配置 | 训练集|  R@1 | R@5 | R@10 | R_mean | 备注 |
|----------|----------|--|-----|-----|------|--------|------|
| 图-文 | pretrain | - | 35.93 | 57.60 | 66.62 | 53.38 | 原始图-文检索，电商图-文验证集，未融合模态 zero-shot |
| 文→(图+文) | pretrain | - | 25.92 | 43.21 | 50.43 | 39.85 | 改造后的文→图+文，zero-shot |
| 文→(图+文) |  pretrain| COCO训练集 | 33.24 | 53.75 | 61.57 | 49.52 | 改造后的文→图+文，COCO训练集上微调 |
| 文→(图+文)  | pretrain | 电商图文训练集| 43.66 | 65.73 | 73.73 | 61.04 | 改造后的文→图+文，电商图文数据集上微调 |
| 文→(图+文) | COCO_ft|电商图文训练集| 43.31 | 65.83 | 75.13 | 61.42 | 在 COCO 微调模型上再用电商数据微调 |

在 zero-shot 设置下，「文→图+文」检索相比原始「图-文」检索存在一定性能下降。一方面，基于大模型生成的 caption 在重构数据过程中不可避免地引入噪声；另一方面，原始 BLIP2 主要建模的是 query 与单一图像表示之间的直接对齐关系，而改造后的任务需要在 query 与融合后的图像及文本联合表示之间进行匹配，融合后的 embedding 在语义空间中的判别性有所减弱，从而降低了相似度匹配的敏感度。

当在电商领域图文数据集上进行针对性微调后，模型能够逐步适应融合模态下的表示分布，检索性能显著提升，并整体优于未引入文本融合的纯图文检索设置。同时可以观察到，COCO 数据集作为中间阶段预训练能够提供较为稳定的初始化，但模型在电商检索任务上的最终性能上限仍主要由领域内数据决定。