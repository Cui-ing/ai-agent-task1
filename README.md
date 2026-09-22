# Task1 学习记录 — Chapter 2/3 · 上下文工程与用户记忆

《深入理解 AI Agent：设计原理与工程实践》第 2-3 章学习与实验记录。

## 1. 环境配置

| 项 | 说明 |
|:--|:--|
| 系统 | Linux (Ubuntu) |
| Python | conda 环境 `myenv`，Python 3.12 |
| 依赖 | 共享包 `agentbook` 已 editable 安装（`pip install -e ".[ch1]"` 时一并完成） |
| 实验仓库 | [bojieli/ai-agent-book](https://github.com/bojieli/ai-agent-book) → `chapter2/`、`chapter3/` |
| LLM Provider | DeepSeek 官方 API（`deepseek-v4-flash`） |

> 凭据通过 `.env` / 环境变量注入，**未包含在本仓库中**。

## 2. 实验清单与结果

| 实验 | 章节 | 内容 | 关键结果 |
|:--|:--:|:--|:--|
| 2-5 | Ch2 | 提示注入攻防：3 攻击 × 4 防御 × 3 次 | 矩阵全 0——强模型在无防御下即识破注入（文档已记录的现象） |
| 3-11 | Ch3 | 上下文化记忆块 vs 原始块（离线 BM25） | Recall@1 0.625 → 0.750，MRR 0.792 → 0.875 |
| 3-8 | Ch3 | 智能体化 RAG vs 非智能体化（DeepSeek） | 复杂问题：非智能体化答"知识库没有"，智能体化补充检索后给出"不构成累犯"的完整分析 |
| 扩展 | Ch3 | 检索参数对比：top-k 2 vs 5 | top-k=2：48%→81%；top-k=5：76%→100%（复杂题 58%→100%） |
| 3-5 | Ch3 | BM25 稀疏检索评测（离线） | recall@5=0.8；同义词查询漏召回，暴露稀疏检索短板 |

复现命令：

```bash
# 实验 2-5（DeepSeek）
cd chapter2/prompt-injection
LLM_PROVIDER=deepseek python demo.py -n 3 -m deepseek-v4-flash -t 0 -o result.json

# 实验 3-11（离线，秒级）
cd chapter3/contextual-retrieval-for-user-memory
python main.py --mode compare --output compare.json

# 实验 3-8（DeepSeek）
cd chapter3/agentic-rag
python main.py --provider deepseek --model deepseek-v4-flash --kb-type offline \
  --query "醉酒过失致人重伤且有盗窃前科如何量刑" --mode compare

# 检索参数对比（离线）
python compare_offline.py --top-k 2
python compare_offline.py --top-k 5

# 实验 3-5（离线）
cd chapter3/sparse-embedding && python cli.py --eval
```

## 3. 目录结构

```
.
├── README.md
├── notes/
│   ├── Chapter2-上下文工程.md          # 第 2 章笔记（含实验 2-5）
│   └── Chapter3-用户记忆与知识库.md     # 第 3 章笔记（含实验 3-8/3-11/3-5）
├── screenshots/                         # 实验截图（终端实测）
│   ├── 01_prompt_injection.png          # ① 提示注入攻防矩阵
│   ├── 02_agentic_rag_compare.png       # ② 智能体化 vs 非智能体化
│   ├── 03_contextual_retrieval.png      # ③ 上下文化检索对比
│   └── 04_topk_comparison.png           # ④ top-k 2 vs 5 检索参数对比
├── results/
│   ├── pi_deepseek.json                 # 实验 2-5 原始结果
│   └── contextual_compare.json          # 实验 3-11 原始结果
└── logs/
    ├── 01_prompt_injection.log          # 实验 2-5 完整日志
    ├── 02_agentic_rag_compare.log       # 实验 3-8 完整日志（含 ReAct 轨迹）
    ├── 03_agentic_rag_offline_topk2.log # top-k=2 离线对比
    ├── 03_agentic_rag_offline_topk5.log # top-k=5 离线对比
    ├── 04_contextual_retrieval.log      # 实验 3-11 日志
    ├── 05_sparse_bm25_default.log       # 实验 3-5 默认参数
    └── 06_sparse_bm25_k1_2_b_0.5.log    # 实验 3-5 调参对比
```

## 4. 结论与心得

见 [notes/Chapter2-上下文工程.md](notes/Chapter2-上下文工程.md) 与 [notes/Chapter3-用户记忆与知识库.md](notes/Chapter3-用户记忆与知识库.md)。

## 5. 总收获

**一、知识层面**

1. 上下文工程是 Harness 的核心：一次 LLM 调用的输入 = 静态前缀（system + tools）+ 动态轨迹（user / assistant / tool）；模型固定时，上下文质量比模型规模更决定 Agent 的上限。
2. 缓存友好是前置架构约束：系统提示词和工具定义一旦确定就不能改，动态信息一律追加到末尾——"改前缀"的代价会直接反映在延迟和账单上。
3. 短期上下文与长期记忆是两回事：轨迹 / 工作记忆管"这次任务"，用户记忆与知识库管"跨任务"；前者只增不改、任务结束即消失，后者需要提取、核验、更新、压缩，并承担隐私责任。
4. RAG 不是"给模型塞资料"：它是把知识变成可检索、可验证、可更新的管道；索引质量（分块策略、上下文前缀、结构化概览）决定检索上限，检索阶段的调参只能补救。
5. 检索没有银弹：稠密强在语义、稀疏强在精确匹配，生产要用"混合 + 重排序"；复杂问题还需要把检索工具交给模型（Agentic RAG）多轮迭代。

**二、实验层面**

| 观察 | 证据 |
|:--|:--|
| 强模型让"上下文层防御"看不出差异 | 提示注入矩阵全 0（3 攻击 × 4 防御 × 3 次），但 D4 运行时校验仍不可省 |
| 给记忆块补上下文前缀能提升召回 | Recall@1 0.625 → 0.750，孤立片段被"锚定"回情境 |
| 智能体化 RAG 的价值在复杂问题 | 单次检索答"知识库没有"，多轮检索补到《刑法》第 65 条，得出"不构成累犯" |
| 检索深度与检索轮数存在权衡 | top-k=2：48%→81%（复杂题 33%→92%）；top-k=5：76%→100% |
| 稀疏检索的结构性短板 | BM25 对同义词（cat）漏召回，调 k1/b 在该评测集上无改善 |

**三、方法与习惯**

- 每个实验保留三件套：原始日志、结构化结果 JSON、终端截图；参数对比（top-k 2/5、BM25 k1/b）比单次运行更能说明问题。
- 选实验先看 provider 兼容性：只有 DeepSeek key 时，优先挑支持 DeepSeek 的实验（prompt-injection、agentic-rag、contextual-retrieval），避免把时间花在环境不匹配上。
- 环境问题要记录：`ALL_PROXY=socks://...` 会让 httpx 报错，运行前 `unset ALL_PROXY all_proxy`；这类"环境噪音"不解决会被误判成实验失败。

**四、待继续**

- 第二章思考题 3（极端压缩是否存在不可逆信息损失）、第三章思考题 4（分块 / 索引管道是否会被"全量输入"取代）留待后续验证。
- 想亲手复现实验 2-10（上下文感知压缩省 >75% token），这次因单策略耗时较长未纳入。
