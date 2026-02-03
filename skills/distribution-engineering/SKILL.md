---
name: distribution-engineering
description: |
  设计和管理 AI Agent 训练/评测数据的分布工程系统。用于：
  (1) 设计 dist_core schema：定义业务场景的 pools（实体池）、personas（用户画像）、scenarios（场景蓝图）
  (2) 数据挖掘：从原始对话日志中提取实体、意图、用户特征，填充 schema
  (3) 生成 recipe_spec：配置训练集/测试集/模拟评测的采样配方
  触发词：分布工程、dist_core、recipe、采样配方、数据合成、场景设计、persona设计、实体池、槽位定义
---

# Distribution Engineering

将业务场景结构化为可控的数据生产配置，实现"Schema → 合成 → 训练/评测"闭环。

## 核心概念

**双文件架构**：
- `dist_core.<project>.json` — 业务资产库（pools、personas、scenarios），长期维护
- `recipe_spec.*.json` — 采样配方，按需创建，通过 `dist_ref` 引用 core

## 工作流

### 阶段1：设计 dist_core

**触发**："帮我设计一个 XX 场景的 schema"、"定义 XX 的实体池和场景"

**输入**：业务场景描述、现有数据样例（可选）

**输出**：Markdown 说明 + JSON 代码块

**设计要点**：
1. **pools** — 实体池设计
   - 覆盖核心业务实体（目的地、产品、错误类型等）
   - 考虑长尾分布（`weight` 字段）
   - 区分来源：`MINED`（挖掘）vs `EXPERT`（专家补充）

2. **personas** — 用户画像
   - `tag`：一句话标签（如"技术小白+焦虑"）
   - `style`：语言风格（口语/书面、错别字倾向等）
   - `behavior_rules`：交互行为（挤牙膏、催促、跑题等）
   - `constraints`：情境约束（时间紧、不方便电话等）
   - `slot_bias`：偏好倾向

3. **scenarios** — 场景蓝图
   - `goal`：用户目标（一句话）
   - `context`：背景压力/限制
   - `slot_requirements`：槽位定义，包含 `ask_if_missing`、`extraction_hint`
   - `perturbations`：扰动类型列表
   - `agent_expectations`：Agent 应达成的行为

完整字段说明见 [references/schema-spec.md](references/schema-spec.md)

### 阶段2：数据挖掘

**触发**："基于这份对话日志，帮我挖掘实体/意图/用户特征"

**输入**：原始对话日志（CSV/JSON/文本）

**输出**：挖掘分析报告 + 填充建议（JSON 片段）

**挖掘维度**：
| 维度 | 产出 | 方法 |
|------|------|------|
| Entity Mining | pools 填充 | 名词扫描、聚类、频次统计 |
| Intent Mining | scenarios 骨架 | 对话链路提取、意图序列分析 |
| Trait Mining | personas | 发言特征（字数、频率、情感、用词） |
| Context Inpainting | context/perturbations | LLM 反推隐藏动机和背景 |

方法论详见 [references/mining-guide.md](references/mining-guide.md)

### 阶段3：生成 recipe_spec

**触发**："写一个训练配方"、"生成测试集采样配置"

**输入**：dist_core 引用 + 任务目标

**输出**：JSON 配方文件

**常见配方类型**：
- `train_sft` — SFT 训练数据
- `test_benchmark` — 评测基准集
- `sim_eval` — User Simulator 评测

**关键参数**：
- `sampling.mix` — 场景×Persona 配比
- `sampling.turns` — 对话轮数分布（建议 ladder 模式）
- `sampling.negatives` — 负样本配置（out_of_scope、missing_info 等）
- `sampling.slot_policy` — 槽位填充/隐瞒/矛盾概率

配方模板见 [assets/templates/](assets/templates/)

## 质量检查

生成配方后，使用 [references/distribution-checklist.md](references/distribution-checklist.md) 检查：
- 语义覆盖度
- 槽位分布均衡性
- 工作流完整性（正常/异常/边界全覆盖）

## 案例

旅行助手场景的完整示例见 [references/schema-spec.md](references/schema-spec.md) 末尾。
