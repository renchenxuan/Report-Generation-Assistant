# MVP实施基线

## 1. 目标与范围
- 目标：以“可跑通、可校验、可追溯、可联调”为优先，完成故障报告生成工作流 MVP。
- 范围：覆盖 10 节点编排、结构化输入输出、检索兜底、事实约束生成、审核字段、后端 API 联调。
- 非目标：不在 MVP 阶段引入复杂重排策略、多模型路由、深度自动评估平台。

## 2. 输入输出契约（字段列表 → 可执行 Schema）

### 2.1 入参契约（机器可校验）
后端传入固定 JSON，并以 JSON Schema 作为唯一字段契约来源。示例：

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "fault_report_input",
  "type": "object",
  "required": ["device_model", "fault_time", "fault_description"],
  "properties": {
    "report_id": {"type": "string", "minLength": 1},
    "customer_name": {"type": "string", "default": ""},
    "site_name": {"type": "string", "default": ""},
    "device_model": {"type": "string", "minLength": 1},
    "device_sn": {"type": "string", "default": ""},
    "fault_time": {"type": "string", "format": "date-time"},
    "fault_count": {"type": "integer", "minimum": 1, "default": 1},
    "fault_description": {"type": "string", "minLength": 1},
    "fault_cause": {"type": "string", "default": ""},
    "event_record_text": {"type": "string", "default": "无"},
    "waveform_analysis_text": {"type": "string", "default": "无"},
    "overall_appearance_text": {"type": "string", "default": "无"},
    "upper_board_text": {"type": "string", "default": "无"},
    "lower_board_text": {"type": "string", "default": "无"},
    "power_device_measure_text": {"type": "string", "default": "无"},
    "site_research_text": {"type": "string", "default": "无"}
  },
  "additionalProperties": false
}
```

### 2.2 assemble_final_json 校验规则
- 严格按 Schema 校验。
- `fault_time` 必须符合 ISO 8601（示例：`2026-04-09T07:44:08Z`）。
- 核心字段缺失（如 `device_model`、`fault_time`、`fault_description`）：
  - 返回异常（建议 400）
  - 返回明确提示（如“缺少设备型号，无法生成报告”）
- 非核心字段缺失：
  - 自动填默认值（如“无”）
  - 在 `ext_info.fallback_notes` 标注“字段为空，已兜底”

### 2.3 输出契约原则
- 最终输出必须为结构化 JSON。
- 必含：`report_id`、章节内容、`audit_metrics`、`ext_info`。
- `ext_info` 必须保留为空对象能力，作为后续扩展容器。

## 3. 节点依赖与字段传递规则

### 3.1 依赖链要求
10 个节点必须形成可追踪字段链，推荐维护“上游输出 → 下游输入”字段传递表。

| 上游节点 | 输出字段 | 下游节点 | 输入引用字段 |
|---|---|---|---|
| start_input | fault_description | preprocess_input | fault_description |
| preprocess_input | retrieval_keywords | retrieve_history_cases | retrieval_keywords |
| retrieve_history_cases | history_cases_snippets | generate_five_why | history_cases_snippets |
| generate_five_why | five_why_table | polish_report_sections | five_why_table |

### 3.2 字段最小化原则
- 每个节点仅输出下游必需字段。
- 禁止无目的透传大对象，避免 Dify 变量膨胀与调试复杂化。

### 3.3 失败兜底传播
- 检索失败或空结果时：输出 `retrieval_status: "failed"`。
- 下游生成节点 Prompt 必须消费该状态并显式说明“无历史案例证据”，禁止臆造外部事实。

## 4. 知识检索（MVP 最小可用）

### 4.1 初始化数据
- 历史案例：先上传 3~5 条典型故障案例（覆盖不同设备型号/故障类型）。
- 产品知识：至少 1 份手册核心片段（原理+常见排查）。
- 目标：保证检索在 MVP 阶段尽量非空。

### 4.2 检索兜底
- 若检索为空：
  - `history_cases_snippets: ["无匹配历史案例"]`
  - 生成内容中必须标注“无历史案例参考，仅基于用户输入事实分析”。

### 4.3 检索输出格式
统一使用“原文片段 + 来源标签”，例如：
- `【历史案例-逆变器过温】xxx`
- `【产品手册-功率器件】xxx`

## 5. 生成 Prompt 标准模板（统一复用）

```plaintext
【角色】设备故障报告专业撰写人
【输入依据】
- 用户输入事实：{{fact_summary}}
- 历史案例证据：{{history_cases_snippets}}
- 产品知识证据：{{product_knowledge_snippets}}
【任务】生成{{节点目标}}
【输出格式】{{结构化格式}}
【约束规则】
1. 仅基于上述依据生成，未提及的信息一律不新增；
2. 若依据不足，在内容中显式标注“【证据不足】”；
3. 责任归属仅给“建议方向”，标注“【待人工审核】”；
4. 语言正式、可审计。
```

### 5.1 节点输出强制项
- 所有生成节点必须输出 `evidence_citation`。
- 目的：保证每条关键结论可回溯到输入事实或检索证据。

## 6. 审核字段（结构化、可量化）

最终 JSON 中必须包含：

```json
{
  "audit_metrics": {
    "evidence_sufficiency": "高/中/低",
    "responsibility_confidence": 80,
    "risk_tips": ["【风险提示】5Y分析第3层无波形分析证据支撑"],
    "hallucination_check": "无/疑似/确认"
  }
}
```

口径约定：
- `responsibility_confidence` 取值范围固定为 0~100（整数）。
- 评分建议由责任归属节点基于证据完整性、事实一致性、历史案例匹配度三项综合给出。
- 若证据不足，分值应下调并同步写入 `risk_tips`。

### 6.1 产出归属规则
- 不在最终拼接节点“临时补写”审核字段。
- 由对应生成节点直接产出，再由 `assemble_final_json` 汇总。

## 7. 联调准备（离线样本 + 后端接口）

### 7.1 离线样本集
准备 3~5 份入参 JSON，覆盖：
- 必填字段齐全
- 非必填字段为空
- 检索有结果
- 检索无结果
- 单节点失败兜底场景

逐节点记录：
1. 是否符合字段契约
2. 是否有证据支撑
3. 审核字段是否合理

### 7.2 后端对接前置
- 生成 Dify 工作流 API Key，并限制后端服务 IP 调用。
- 安全加固建议（MVP 可先纳入规范）：API Key 定期轮换、最小权限、服务端安全存储，必要时叠加应用层鉴权（如 JWT/签名）。
- 约定异常码：
  - 400：入参不合法
  - 500：工作流节点执行失败
  - 504：检索超时
- 约定 `report_id` 贯穿：后端传入，最终 JSON 原样返回，便于落库关联。

## 8. 可扩展预留（低成本）
- 节点命名统一前缀：如 `fault_report_*`。
- 输出保留 `ext_info: {}`，用于后续附加软件版本、生产批次等字段。
- 在检索节点备注二期优化方向：Query 重写、TopK 动态、重排策略。

## 9. MVP 验收标准
- 合法入参可完整跑通 10 节点，输出符合 Schema。
- 人工抽检 3~5 份样本，生成内容无明显无证据臆造。
- 检索空/节点失败有明确兜底标注，链路不中断。
- 最终 JSON 含完整审核字段，支持人工快速风险判断。
- 后端可通过 API 调用 Dify，返回结果可映射 Word 模板核心字段。

## 10. 执行优先级（建议）
1. 先固化 Schema 与字段契约。
2. 再固化节点依赖表与兜底变量（`retrieval_status`）。
3. 再统一 Prompt 模板与 `evidence_citation`。
4. 最后进行离线样本联调并按验收标准打分。
