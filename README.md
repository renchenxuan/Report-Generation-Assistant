# Report-Generation-Assistant

你是设备故障报告 AI 工具dify工作流搭建助手  
Dify 工作流详细设计、知识库详细方案与节点 Prompt 文档

## 1. 文档目标
本文档基于技术分析与解决方案报告，整理一套可落地的 AI 工具设计方案，你重点关注其中的AI部分覆盖以下三部分内容：
- Dify 工作流详细设计流程
- 知识库设计的详细方案
- Dify 每个关键节点的详细 Prompt

本文档目标是支持项目进行方案评审、Dify 工作流搭建、后端接口对接和后续实施。

## 2. 模板结构分析与自动化边界

### 2.1 模板核心结构
根据提供的 Word 模板，报告核心内容包括：
- 封面
- 致客户说明
- 问题描述
- 三、故障原因分析
- 四、解决方案
- 署名与日期

模板中已识别出的重点内容包括：
- 封面标题：XXX故障原因分析与解决方案报告
- 公司抬头：深圳市XX科技有限公司
- 致函内容：说明故障背景、调查过程、原因结论和措施方向
- 问题描述区：
  - 问题描述
  - 临时措施
  - 现场处理方案
  - 厂内处理方案
- 故障原因分析区：
  - 1、事件记录分析
  - 2、故障时刻波形分析
  - 3、整体外观分析
  - 4、开盖上层板分析
  - 5、开盖下层板分析
  - 6、功率器件测量分析
  - 7、现场调研分析
  - 8、5Y 根本原因分析
  - 9、原因总结
- 解决方案区：
  - 长期纠正措施
  - 预防 / 优化措施

模板表格中已识别出的基础字段包括：
- 故障机组条码
- 设备型号
- 发生时间
- 现场名称
- 故障数量
- 详细故障描述

模板中还包含 5Y 根本原因分析表，字段为：
- 提问顺序
- 问题
- 原因分析

### 2.2 自动化边界设计
建议按 “用户输入 + 系统接口补充 + AI 自动生成 + 人工审核” 划分：

#### 用户输入
- 客户名称
- 现场名称
- 故障名称
- 设备型号 #没有改自动
- 故障设备条码
- 故障时间
- 故障数量 #支持手动修改，默认 1
- 问题描述 #光伏设备数据来自：SE17 - 组串产品售后服务工单流程的：问题主题 + 问题描述 + 问题初步定位，其他设备来来自 SE03 电子流的：问题主题 + 问题描述 + 问题初步定位
- 故障原因导向（报告文档生成以此为根本依据，及文档中的原因总结）
- 时间记录图片和描述（分析）
- 波形分析图片和描述（分析）
- 整体外观图片和描述（分析）
- 开盖上板分析图片和描述（分析）
- 开盖下板分析图片和描述（分析）
- 功率器件测量图片和描述（分析）
- 现场调研图片和描述（分析）

#### AI 自动生成
- 致客户说明
- 问题描述补写
- 临时措施
- 5Y 根本原因分析
- 原因总结
- 责任归属建议
- 解决方案
- 现场处理方案
- 厂内处理方案

#### 人工审核
- 责任归属准确性
- 结论是否越权
- 是否存在证据不足却强下结论
- 解决方案是否可执行
- 是否允许该报告进入知识库

#### AI 编排层
- Dify Workflow

功能：
- 输入预处理
- 知识检索
- 外部接口调用
- 多阶段 LLM 生成
- 结构化 JSON 输出

## 4. 端到端业务流程

### 4.1 主流程
- 用户录入报告基础信息与故障描述
- 上传图片、波形、日志等附件
- Spring Boot 保存草稿并生成 report_id
- 用户点击 “生成报告”
- Spring Boot 调用 Dify Workflow
- Dify 进行知识检索与外部接口查询
- Dify 分章节生成结构化报告内容
- Dify 返回 JSON 给 Spring Boot
- Spring Boot 将 JSON 落库并映射到模板
- 用户在线预览并人工审核
- 审核通过后导出 Word/PDF
- 选中的正式报告回灌知识库

### 4.2 生成策略
不建议一次性长文生成，建议采用：
- 先抽取事实
- 再归纳证据
- 再做分析判断
- 再做根因追溯
- 最后生成解决方案和正式报告语句

优点：
- 降低幻觉
- 可解释性强
- 易于调优
- 更适合 Dify 节点编排

### 5.1 工作流目标
工作流目标是根据结构化输入、知识检索结果和外部系统证据，生成适配模板的结构化 JSON，供 Spring Boot 映射到 Word 模板。

推荐工作流名称：`fault_report_generate_workflow`

### 5.2 输入参数设计
推荐由 Spring Boot 向 Dify 传入以下结构化参数：

```json
{
  "customer_name": "客户名称",
  "site_name": "现场名称",
  "device_type": "设备类型",
  "device_model": "设备型号",
  "device_sn": "设备条码",
  "fault_time": "负责时间",
  "fault_count": 2,
  "fault_description": "故障描述",
  "fault_cause": "故障问题导向",
  "waveform_analysis_text": "故障录波分析文本",
  "event_record_text": "事件记录分析文本",
  "overall_appearance_text": "整体外观分析文本",
  "upper_board_text": "开盖上层板分析文本",
  "lower_board_text": "开盖下层板分析文本",
  "power_device_measure_text": "功率器件测量分析文本",
  "site_research_text": "现场调研分析文本"
}
```

建议扩展字段：
- 产品线
- 软件版本
- 硬件版本
- 部件名称
- 物料编码
- 生产批次
- 客户反馈原文
- 维修记录编号

### 5.3 节点设计总览
推荐节点顺序如下：
- start_input
- preprocess_input
- retrieve_history_cases
- retrieve_product_knowledge
- generate_customer_letter_problem_desc
- generate_five_why
- generate_root_cause_and_responsibility
- generate_solution
- polish_report_sections
- assemble_final_json

## 6. Dify 节点详细设计

### 6.1 start_input
- 节点类型：Start
- 职责：接收后端传入的结构化参数，校验关键输入项
- 关键必填字段：
  - device_model
  - fault_time
  - fault_description
- 输出：原始输入对象

### 6.2 preprocess_input
- 节点类型：Code 或 LLM
- 职责：清洗空字段、标准化故障摘要、提取检索关键词
- 输出：
  - normalized_fault_summary
  - retrieval_keywords
  - possible_parts
  - possible_environment_factors
  - fact_summary

### 6.3 retrieve_history_cases
- 节点类型：Knowledge Retrieval
- 职责：检索历史故障报告库
- 检索建议：
  - Query：设备型号 + 故障描述 + 告警码 + 部件关键词
  - TopK：3~5
  - 过滤条件：产品线、机型、文档已审核
- 输出：相似案例原始片段

### 6.4 retrieve_product_knowledge
- 节点类型：Knowledge Retrieval
- 职责：检索产品原理、维修手册、检测规范
- 输出：原理知识原始片段

### 6.5 generate_customer_letter_problem_desc
- 节点类型：LLM
- 职责：生成致客户说明、生成正式问题描述
- 输出：
  - customer_letter
  - problem_description_refined

### 6.6 generate_five_why
- 节点类型：LLM
- 职责：生成 5Y 根本原因分析表
- 输出：five_why_table

### 6.7 generate_root_cause_and_responsibility
- 节点类型：LLM
- 职责：生成原因总结，给出责任归属建议
- 输出：
  - root_cause_summary
  - responsibility_judgement

### 6.8 generate_solution
- 节点类型：LLM
- 职责：生成临时措施、纠正措施、预防措施、客户侧建议、内部质量动作
- 输出：
  - temporary_measures
  - corrective_actions
  - preventive_actions
  - customer_side_actions
  - internal_quality_actions

### 6.9 polish_report_sections
- 节点类型：LLM
- 职责：对章节语言做正式化润色，不新增事实
- 输出：各章节 final 版字段

### 6.10 assemble_final_json
- 节点类型：Code
- 职责：汇总最终 JSON，校验必填字段，输出给 Spring Boot
