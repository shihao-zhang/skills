# Schema 规格说明

## dist_core 结构

文件命名：`dist_core.<project>.v<version>.json`

```json
{
  "meta": { ... },
  "pools": { ... },
  "personas": [ ... ],
  "scenarios": [ ... ]
}
```

### meta

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| project | string | ✓ | 项目标识 |
| version | string | ✓ | 语义化版本号 |
| domain | string | ✓ | 业务领域 |
| notes | string | | 备注 |
| tags | string[] | | 标签 |

### pools

实体池，key 为池名称。

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| type | string | ✓ | 值类型：`text` / `number` / `enum` |
| values | any[] | ✓ | 候选值列表 |
| sampling_hint | string | | 采样提示（如"高频优先"） |
| weight | number[] | | 与 values 对应的采样权重 |

### personas

用户画像数组。

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | string | ✓ | 唯一标识，建议 `p_<描述>` |
| tag | string | ✓ | 一句话标签 |
| style | string | ✓ | 语言风格描述 |
| behavior_rules | string[] | ✓ | 交互行为规则列表 |
| constraints | string[] | | 情境约束 |
| slot_bias | object | | 槽位偏好，key 为槽位名，value 为偏好描述 |

### scenarios

场景蓝图数组。

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | string | ✓ | 唯一标识，建议 `scn_<描述>` |
| stage | string | ✓ | 业务阶段（如 booking/payment/support） |
| goal | string | ✓ | 用户目标（一句话） |
| context | string | | 背景/压力/限制条件 |
| slot_requirements | array | ✓ | 槽位定义列表 |
| perturbations | string[] | | 扰动类型 |
| agent_expectations | string[] | ✓ | Agent 应达成的行为 |
| eval_policy_ref | string | | 关联的评测策略 ID |

#### slot_requirements 子结构

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| name | string | ✓ | 槽位名称 |
| required | boolean | ✓ | 是否必填 |
| value_from | object | ✓ | 取值来源：`{ "pool": "<pool_name>" }` 或 `{ "free_text": true }` |
| format_hint | string | | 格式提示 |
| ask_if_missing | string | ✓ | 缺失时 Agent 追问话术 |
| extraction_hint | string | | 抽取提示 |
| examples | string[] | | 示例值 |

---

## recipe_spec 结构

文件命名：`recipe.<type>.v<version>.json`

```json
{
  "meta": { ... },
  "dist_ref": { ... },
  "target": { ... },
  "sampling": { ... },
  "constraints": { ... }
}
```

### meta

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| name | string | ✓ | 配方名称 |
| version | string | ✓ | 版本号 |
| notes | string | | 备注 |

### dist_ref

引用的 dist_core。

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| project | string | ✓ | 项目名 |
| version | string | ✓ | 版本号 |
| checksum | string | | 校验值 |

### target

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| kind | string | ✓ | `train_sft` / `test_benchmark` / `sim_eval` |
| count | number | ✓ | 目标样本数 |
| output_format | string | ✓ | `jsonl` / `csv` / `parquet` |

### sampling

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| seed | number | | 随机种子 |
| mix | array | ✓ | 场景×Persona 配比 |
| turns | object | | 对话轮数分布 |
| negatives | object | | 负样本配置 |
| slot_policy | object | | 槽位策略 |

#### mix 子结构

| 字段 | 类型 | 说明 |
|------|------|------|
| scenario | string | 场景 ID |
| persona | string | Persona ID，可为 `"RANDOM"` |
| ratio | number | 占比（0-1） |

#### turns 子结构

| 字段 | 类型 | 说明 |
|------|------|------|
| mode | string | `ladder`（阶梯）/ `uniform`（均匀）/ `custom` |
| p1 | number | 单轮占比 |
| p2 | number | 两轮占比 |
| p3plus | number | 三轮及以上占比 |

#### negatives 子结构

| 字段 | 类型 | 说明 |
|------|------|------|
| ratio | number | 负样本总占比 |
| types | string[] | 类型：`out_of_scope`、`missing_info`、`contradiction`、`tool_fail` |
| injection_hint | string | 注入方式描述 |

#### slot_policy 子结构

| 字段 | 类型 | 说明 |
|------|------|------|
| fill_required_prob | number | 必填槽位填充概率 |
| withhold_info_prob | number | 信息隐瞒概率（模拟挤牙膏） |
| contradiction_prob | number | 前后矛盾概率 |

### constraints

| 字段 | 类型 | 说明 |
|------|------|------|
| frozen | boolean | 是否冻结（禁止修改） |
| coverage_min | object | 最低覆盖要求 |
| cost_budget | object | 成本预算 |

---

## 完整案例：旅行助手

### dist_core.travel_agent.v0.1.0.json

```json
{
  "meta": {
    "project": "travel_agent",
    "version": "0.1.0",
    "domain": "OTA_customer_service",
    "notes": "旅行助手客服场景 - MVP版本",
    "tags": ["travel", "booking", "payment"]
  },
  "pools": {
    "destination": {
      "type": "text",
      "values": ["新加坡", "普吉岛", "首尔", "亚庇", "曼谷", "东京"],
      "sampling_hint": "前4个为高频目的地",
      "weight": [0.25, 0.20, 0.20, 0.15, 0.10, 0.10]
    },
    "hotel_star": {
      "type": "enum",
      "values": ["三星", "四星", "五星", "民宿"],
      "weight": [0.2, 0.4, 0.3, 0.1]
    },
    "payment_error": {
      "type": "text",
      "values": ["支付宝限额", "二维码识别失败", "系统繁忙请稍后", "银行卡余额不足", "网络超时"],
      "sampling_hint": "长尾分布，覆盖常见支付问题"
    },
    "budget_range": {
      "type": "text",
      "values": ["1000以内", "1000-3000", "3000-5000", "5000以上", "不限"],
      "weight": [0.1, 0.3, 0.35, 0.15, 0.1]
    }
  },
  "personas": [
    {
      "id": "p_tech_weak_anxious",
      "tag": "技术小白+焦虑型",
      "style": "短句口语、偶有错别字、倾向发截图而非文字描述。",
      "behavior_rules": [
        "信息挤牙膏：被追问才补充关键信息",
        "遇到失败会催促、重复询问",
        "可能插入无关小问题（如'那边热不热'）"
      ],
      "constraints": [
        "时间紧：3小时后出发",
        "不方便电话，只能打字"
      ],
      "slot_bias": {
        "budget": "prefer_low",
        "hotel_star": "prefer_4star"
      }
    },
    {
      "id": "p_experienced_decisive",
      "tag": "老手+决策果断型",
      "style": "信息完整、表达清晰、直接说需求。",
      "behavior_rules": [
        "一次性给出多个关键信息",
        "对方案快速做决定",
        "不耐烦冗余确认"
      ],
      "constraints": [],
      "slot_bias": {
        "budget": "prefer_high",
        "hotel_star": "prefer_5star"
      }
    },
    {
      "id": "p_price_sensitive",
      "tag": "价格敏感+比价型",
      "style": "反复询问价格、要求对比多个选项。",
      "behavior_rules": [
        "每个方案都问'还有更便宜的吗'",
        "会拿其他平台价格来比较",
        "决策周期长，需要多轮确认"
      ],
      "constraints": [
        "预算有限但不明说具体数字"
      ],
      "slot_bias": {
        "budget": "prefer_low",
        "hotel_star": "no_preference"
      }
    }
  ],
  "scenarios": [
    {
      "id": "scn_hotel_booking_normal",
      "stage": "booking",
      "goal": "用户要预订指定目的地的酒店。",
      "context": "用户已确定出行日期，需要找到合适的住宿。",
      "slot_requirements": [
        {
          "name": "destination",
          "required": true,
          "value_from": { "pool": "destination" },
          "format_hint": "城市或地区名称",
          "ask_if_missing": "请问您想去哪个城市呢？",
          "extraction_hint": "从用户表达中提取地名",
          "examples": ["新加坡", "普吉岛"]
        },
        {
          "name": "check_in_date",
          "required": true,
          "value_from": { "free_text": true },
          "format_hint": "YYYY-MM-DD 或自然语言",
          "ask_if_missing": "入住日期是哪天呢？",
          "examples": ["下周五", "3月15号"]
        },
        {
          "name": "hotel_star",
          "required": false,
          "value_from": { "pool": "hotel_star" },
          "ask_if_missing": "对酒店星级有要求吗？",
          "examples": ["四星", "五星"]
        },
        {
          "name": "budget",
          "required": false,
          "value_from": { "pool": "budget_range" },
          "ask_if_missing": "预算大概是多少呢？",
          "examples": ["3000左右", "不限"]
        }
      ],
      "perturbations": [
        "信息不全：只说'订酒店'不给目的地",
        "模糊表达：'便宜点的'、'好一点的'"
      ],
      "agent_expectations": [
        "收集完必填信息后再推荐",
        "推荐时说明价格区间和特色",
        "提供2-3个选项供选择"
      ],
      "eval_policy_ref": "rubric_booking_basic"
    },
    {
      "id": "scn_payment_fail",
      "stage": "payment",
      "goal": "用户支付遇到问题，需要排查解决。",
      "context": "用户已选好酒店，在支付环节卡住。可能很着急。",
      "slot_requirements": [
        {
          "name": "order_id",
          "required": false,
          "value_from": { "free_text": true },
          "ask_if_missing": "方便告诉我订单号吗？我帮您查一下。",
          "examples": ["OD20240315001"]
        },
        {
          "name": "payment_error",
          "required": true,
          "value_from": { "pool": "payment_error" },
          "format_hint": "错误提示文案或截图描述",
          "ask_if_missing": "您看到的报错提示是什么？能发截图吗？",
          "extraction_hint": "从用户描述或截图OCR中提取",
          "examples": ["系统繁忙", "限额了"]
        },
        {
          "name": "payment_method",
          "required": false,
          "value_from": { "free_text": true },
          "ask_if_missing": "您用的是什么支付方式？",
          "examples": ["支付宝", "微信", "银行卡"]
        }
      ],
      "perturbations": [
        "信息模糊：只说'付不了'不说具体报错",
        "情绪激动：反复催促'快帮我解决'",
        "突然发截图：不配文字说明"
      ],
      "agent_expectations": [
        "先安抚情绪，再收集信息",
        "根据错误类型给出针对性方案",
        "提供兜底方案：换支付方式/人工处理",
        "如需等待，明确告知预计时间"
      ],
      "eval_policy_ref": "rubric_troubleshooting"
    },
    {
      "id": "scn_change_booking",
      "stage": "post_booking",
      "goal": "用户要修改已预订的酒店（改日期/换房型/取消）。",
      "context": "用户行程有变，需要调整订单。可能涉及退改政策。",
      "slot_requirements": [
        {
          "name": "order_id",
          "required": true,
          "value_from": { "free_text": true },
          "ask_if_missing": "请提供您的订单号，我帮您查询。",
          "examples": ["OD20240315001"]
        },
        {
          "name": "change_type",
          "required": true,
          "value_from": { "free_text": true },
          "format_hint": "改期/换房/取消",
          "ask_if_missing": "您是想改日期、换房型还是取消订单呢？",
          "examples": ["改到下周", "想换大床房", "不去了想退"]
        },
        {
          "name": "new_date",
          "required": false,
          "value_from": { "free_text": true },
          "ask_if_missing": "想改到哪天呢？",
          "examples": ["3月20号", "推迟两天"]
        }
      ],
      "perturbations": [
        "信息矛盾：先说改期，后又说想退",
        "对政策不满：'凭什么要收手续费'"
      ],
      "agent_expectations": [
        "先查询订单状态和退改政策",
        "清晰说明费用影响",
        "提供操作指引或代为处理",
        "遇到不满情绪要解释政策合理性"
      ],
      "eval_policy_ref": "rubric_change_order"
    }
  ]
}
```

### recipe.train_sft.v0.1.0.json

```json
{
  "meta": {
    "name": "train_sft_travel_v1",
    "version": "0.1.0",
    "notes": "旅行助手 SFT 训练集 - 重点覆盖支付异常场景"
  },
  "dist_ref": {
    "project": "travel_agent",
    "version": "0.1.0"
  },
  "target": {
    "kind": "train_sft",
    "count": 5000,
    "output_format": "jsonl"
  },
  "sampling": {
    "seed": 42,
    "mix": [
      { "scenario": "scn_hotel_booking_normal", "persona": "RANDOM", "ratio": 0.4 },
      { "scenario": "scn_payment_fail", "persona": "p_tech_weak_anxious", "ratio": 0.3 },
      { "scenario": "scn_payment_fail", "persona": "p_experienced_decisive", "ratio": 0.1 },
      { "scenario": "scn_change_booking", "persona": "p_price_sensitive", "ratio": 0.2 }
    ],
    "turns": {
      "mode": "ladder",
      "p1": 0.2,
      "p2": 0.4,
      "p3plus": 0.4
    },
    "negatives": {
      "ratio": 0.15,
      "types": ["out_of_scope", "missing_info", "tool_fail"],
      "injection_hint": "out_of_scope用于拒答训练，missing_info用于追问训练"
    },
    "slot_policy": {
      "fill_required_prob": 0.85,
      "withhold_info_prob": 0.4,
      "contradiction_prob": 0.05
    }
  },
  "constraints": {
    "frozen": false,
    "coverage_min": {
      "per_scenario": 100,
      "required_slots_coverage": 0.95
    },
    "cost_budget": {
      "max_tokens_per_sample": 1500,
      "max_usd_total": 100
    }
  }
}
```
