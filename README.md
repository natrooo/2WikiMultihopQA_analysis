# 2WikiMultihopQA 多跳问答探索器

基于 [2WikiMultihopQA](https://huggingface.co/datasets/voidful/2WikiMultihopQA) 的多跳问答数据管理系统与可视化网站，可部署到 GitHub Pages。

## 数据集介绍

2WikiMultihopQA 是一个面向多跳推理的大规模问答数据集，由 Ho 等人在 2021 年 COLING 会议上提出。与 HotpotQA 类似，该数据集要求模型跨越两篇维基百科文章进行推理才能得出正确答案。

### 核心特征

- **多跳推理需求**：每个问题需要从两篇不同的维基百科文章中检索信息并进行推理，单篇文章不足以提供完整答案
- **知识图谱路径**：问题源于维基百科知识图谱中的三元组推理链，保证了问题的高质量和逻辑性
- **四种推理类型**：
  - `comparison`（比较推理）：比较两个实体的属性，如年龄、国籍等
  - `bridge`（桥接推理）：通过一个实体链接到另一个实体，链式传递信息
  - `inference`（推断推理）：从多篇文章中综合信息进行逻辑推断
  - `composition`（组合推理）：组合多个事实片段得到最终答案
- **干扰设置**：每个问题提供 10 篇文档（2 篇支持文档 + 8 篇干扰文档），模拟真实检索场景
- **句子级支持事实标注**：精确标注支持答案的文档标题和具体句子编号

### 与 HotpotQA 的对比

| 特性 | HotpotQA | 2WikiMultihopQA |
|------|----------|-----------------|
| 知识源 | 维基百科 | 维基百科 |
| 推理跳数 | 2-hop | 2-hop |
| 推理类型 | bridge / comparison | bridge / comparison / inference / composition |
| 问题构造 | 众包 | 知识图谱自动生成 |
| 数据规模 | ~113K | ~100K+ |
| 数据库 | SQLite + FTS5 | Redis + RediSearch |

## 数据库选型

| 组件 | 技术 | 用途 |
|------|------|------|
| 主库 | Redis + RediSearch | 高性能内存存储、全文检索、多跳事实与段落关联 |
| 发布层 | JSON 静态文件 | GitHub Pages 无后端，浏览器端检索与可视化 |
| 可选扩展 | RedisJSON | 文档模型存储，原生 JSON 查询支持 |

选择 Redis 的原因：内存数据库毫秒级响应、RediSearch 模块提供全文检索能力、丰富的数据结构（Hash / Set / Sorted Set）天然适合 HotpotQA 的「问题—段落—支撑事实」层级关联，无需 ORM 即可用键值结构表达复杂关系。

## 功能

- **多跳检索**：Fuse.js 模糊搜索 + 按推理类型 / 聚类过滤
- **多跳可视化**：根据 supporting_facts 构建 Question → Hop1 → Hop2 → Answer 推理图（vis-network）
- **聚类分析**：对问题文本做 TF-IDF + K-Means 聚类，ECharts 散点图与柱状图
- **上下文浏览**：展示 distractor 设置下的段落与支持句
- **统计分析**：类型分布饼图、聚类大小柱状图、推理类型组成分析
- **四种推理类型支持**：comparison、bridge、inference、composition 全覆盖

## Redis 数据模型

```
# 问题主数据（Hash）
question:<id>  →  {question, answer, type, level}

# 上下文文档（Hash）
context:<qid>:<doc_idx>  →  {title, is_supporting, sentences[]}

# 支持事实索引（Set）
supporting:<qid>  →  {(title, sent_id), ...}

# 全文搜索索引（RediSearch）
FT.SEARCH idx:questions  →  关键词匹配

# 聚类结果（Hash）
cluster:<qid>  →  {cluster_id, coord_x, coord_y}
```

## 数据导入说明

### 环境要求

- Python 3.8+
- Redis 7.0+（需启用 RediSearch 模块）
- redis-py 客户端库

### 从 Hugging Face 导入

```bash
# 1. 启动 Redis 服务（确保已加载 RediSearch 模块）
redis-server --loadmodule /path/to/redisearch.so

# 2. 导入验证集（distractor 配置）
python scripts/import_data.py --dataset voidful/2WikiMultihopQA --config distractor --split validation

# 若出现 504/429 超时，用断点续传：
python scripts/import_data.py --resume

# 限制导入条数
python scripts/import_data.py --limit 2000
```

### 数据预处理流程

1. 从 HuggingFace API 下载 Parquet 文件
2. 使用 pandas + pyarrow 读取 Parquet 格式
3. 遍历嵌套结构，展开 context 数组和 sentences 数组
4. 使用 Redis Pipeline 批量写入（每 2,000 条提交一次）
5. 通过 RediSearch 为问题文本建立全文索引 `FT.CREATE`
6. 导出 JSON 静态文件供 Web 前端使用

## 本地运行

```bash
# 1. 确保 Redis 服务已启动
redis-cli ping   # 应返回 PONG

# 2. 导入数据
python scripts/import_data.py

# 3. 导出 Web 数据（Redis → JSON 静态文件）
python scripts/export_web_data.py

# 4. 本地预览
python -m http.server 8080
# 打开 http://localhost:8080
```

## 部署到 GitHub Pages

1. 将本仓库推送到 GitHub
2. Settings → Pages → Build and deployment 选择分支部署
3. 站点地址：`https://<username>.github.io/2WikiMultihopQA_analysis/`

在线演示：**[https://natrooo.github.io/2WikiMultihopQA_analysis/](https://natrooo.github.io/2WikiMultihopQA_analysis/)**

## 技术栈

| 层面 | 技术 |
|------|------|
| 前端 | HTML5 / CSS3 / ECharts / Chart.js |
| 数据处理 | Python / pandas / scikit-learn |
| 数据库 | Redis + RediSearch（全文搜索） |
| 搜索 | RediSearch / Fuse.js（前端模糊匹配） |
| 部署 | GitHub Pages |

## 目录结构

```
2WikiMultihopQA_analysis/
├── scripts/
│   ├── import_data.py           # HF API → Redis
│   └── export_web_data.py       # Redis → JSON 静态文件
├── data/                        # JSON 数据文件（Web 前端用）
│   ├── questions.json           # 问题摘要
│   ├── cluster_data.json        # 聚类结果 + PCA 坐标
│   └── cluster_terms.json       # 聚类关键词
├── index.html                   # 主页面
├── css/                         # 样式文件
├── js/                          # JavaScript 脚本
└── .github/workflows/
    └── pages.yml                # GitHub Pages 自动部署
```

## 引用

```bibtex
@inproceedings{ho2021constructing,
  title={Constructing A Multi-hop QA Dataset for Comprehensive Evaluation of Reasoning Steps},
  author={Ho, Xanh and Nguyen, Anh-Khoa Duong and Sugawara, Saku and Aizawa, Akiko},
  booktitle={Proceedings of the 28th International Conference on Computational Linguistics (COLING)},
  year={2021}
}
```
