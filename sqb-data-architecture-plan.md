# 收钱吧 OpenClaw 数据底座 — 完整方案

> 版本：v2.0 | 日期：2026-05-21 | 作者：@Cindy（数据架构师），协作：@Alina（数仓）、@Livia（数据开发）、@Angel（基础设施）| 更新：纳入 7 个业务域全景数据

---

## 目录

1. [背景与问题](#1-背景与问题)
2. [核心结论](#2-核心结论)
3. [目标架构](#3-目标架构)
4. [技术栈选择](#4-技术栈选择)
5. [各层详细设计](#5-各层详细设计)
6. [数据流与关键链路](#6-数据流与关键链路)
7. [查询治理规则](#7-查询治理规则)
8. [权限模型](#8-权限模型)
9. [三条底线（不可突破）](#9-三条底线不可突破)
10. [分阶段实施计划](#10-分阶段实施计划)
11. [组织分工模型](#11-组织分工模型)
12. [协作机制：YAML 指标模板](#12-协作机制yaml-指标模板)
13. [Phase 1 详细排期与资源](#13-phase-1-详细排期与资源)
14. [风险与缓解措施](#14-风险与缓解措施)
15. [附录](#15-附录)

---

## 1. 背景与问题

### 1.1 公司背景

收钱吧是国内最大的聚合支付公司，为商户提供数字化解决方案。每天生成 TB 级数据，已建设了 OpenClaw 智能体体系（400+ 实例），为所有管理者和员工提供 AI 助手，目标是**让员工可以自主分析数据、优化工作**。

### 1.2 业务域与数据资产全景

收钱吧数据仓库已覆盖 **7 个核心业务域**：

#### 一、支付交易域
交易流水、收支明细、订单支付、对账清算、资金流向、手续费、退款核销、渠道收付数据。
→ 支撑资金核算、交易风控、营收统计、对账复盘。

#### 二、广告营销域
广告投放流量、曝光点击、转化成交、渠道投产（ROI）、用户触达、素材效果、人群投放数据。
→ 用于投放优化、ROI 核算、营销复盘、精准获客。

#### 三、CRM 客户域
用户基础档案、会员等级、行为轨迹、互动记录、客户分层、留存活跃、生命周期数据。
→ 实现客户画像、客群分群、私域运营、用户维系。

#### 四、信贷金融域
授信额度、借贷记录、还款履约、风控评分、逾期台账、资质审核、借贷场景行为数据。
→ 服务信贷风控、资质核验、贷后管理、额度运营。

#### 五、校园业务域
校区维度信息、外卖配送、餐饮行业分布、校内交易数据。
→ 适配校园运营管理、校内业态经营分析。

#### 六、智慧门店域
门店客流、到店消费、商户运营、门店营收、线下动线、场地运营数据。
→ 支撑线下门店经营管控、消费分析、门店运营优化。

#### 七、全域来店客流域
全域到店客流、经营数据分析、跨渠道来客溯源、仓储管理、来客属性数据。
→ 实现客流溯源、引流效果评估、线下流量盘活。

### 1.3 当前数据架构

#### 1.3.1 数据存储与计算

```
业务数据(RDS MySQL) → DataWorks(MaxCompute 数仓) → BI 报表
```

- **RDS**：阿里云 RDS MySQL，承载在线业务
- **DataWorks**：阿里云 MaxCompute 离线数仓 + ETL
- **BI**：基于数仓 ADS 层的固定报表

#### 1.3.2 数仓分层结构（现有）

| 分层 | 全称 | 内容 | 应用场景 |
|---|---|---|---|
| **ODS** | 原始数据层 | 全业务原始日志、业务系统增量数据、第三方对接原始报文 | 数据同步、原始留痕、问题溯源、底层数据校验 |
| **DWD** | 明细数据层 | 各业务清洗后明细事实数据，统一字段口径、剔除脏数据 | 明细查询、业务明细对账、一线业务报表 |
| **DWS** | 汇总聚合层 | 按时间/区域/渠道/客群/门店聚合汇总 | 多维指标汇总、批量数据统计、中端业务分析 |
| **ADS** | 应用服务层 | 定制化宽表、主题指标、标签数据，直接对接业务系统 | 可视化大屏、业务看板、自助分析、系统数据调用 |

#### 1.3.3 全域数据应用场景（现有）

| 场景类别 | 具体应用 |
|---|---|
| **经营决策类** | 全业态营收汇总、多业务线利润核算、整体经营态势研判、季度/月度经营数据复盘 |
| **用户运营类** | 全域用户统一画像、跨业务用户行为打通、公私域联动运营、用户复购与转化提升 |
| **风控合规类** | 交易资金风控、信贷风险预警、异常客流/异常交易识别、业务数据合规审计 |
| **线下实体运营类** | 智慧门店全域统筹、校园业态经营、线下客流转化效率、实体网点效能优化 |
| **营销投放类** | 广告投放全链路归因、线上引流线下转化、跨业务营销活动效果复盘、精准人群定向推送 |
| **数据服务类** | 业务系统数据接口输出、部门自助取数、数据报表自动化、对外合规数据报送 |

### 1.4 核心矛盾

| 维度 | 现有架构能力 | OpenClaw 需求 | 匹配？ |
|---|---|---|---|---|
| 查询模式 | 预定义 SQL / 固定报表 | 自然语言 → 即席查询 | ❌ |
| 查询延迟 | 分钟 ~ 小时级（批处理） | 秒级（对话式交互） | ❌ |
| 并发模型 | 少量分析师提交 SQL | 400+ 用户随时提问 | ❌ |
| 语义理解 | 口径在 SQL 和人脑中 | 需要机器可读的指标定义 | ❌ |
| 数据入口 | RDS 不能分析、DataWorks 不适合即席 | 需要安全的、低延迟的查询 API | ❌ |
| 权限控制 | 表级权限（SRE 管理） | 用户级 + 商户级精细权限 | ❌ |

**结论：当前架构不适合 OpenClaw 直接访问数据。**

### 1.5 业务域对 OpenClaw 的开放可行性评估

| 业务域 | 数据成熟度 | 口径标准化程度 | 敏感度 | Phase 1 接入建议 |
|---|---|---|---|---|
| 支付交易域 | 高 | 高 | 中 | ✅ 首批（Phase 1） |
| 广告营销域 | 中-高 | 中 | 低 | ✅ 第二批（Phase 2） |
| CRM 客户域 | 中 | 中 | 高（PII） | ⚠️ 第二批，需脱敏 |
| 信贷金融域 | 中-高 | 高 | 极高 | ⚠️ 第三批，高合规要求 |
| 智慧门店域 | 中 | 中 | 低 | ✅ 第三批 |
| 校园业务域 | 中-低 | 低 | 中 | 🔜 第四批，需先治理口径 |
| 全域来店客流域 | 中-低 | 低 | 低 | 🔜 第四批，跨域依赖多 |
|---|---|---|---|
| 查询模式 | 预定义 SQL / 固定报表 | 自然语言 → 即席查询 | ❌ |
| 查询延迟 | 分钟 ~ 小时级（批处理） | 秒级（对话式交互） | ❌ |
| 并发模型 | 少量分析师提交 SQL | 400+ 用户随时提问 | ❌ |
| 语义理解 | 口径在 SQL 和人脑中 | 需要机器可读的指标定义 | ❌ |
| 数据入口 | RDS 不能分析、DataWorks 不适合即席 | 需要安全的、低延迟的查询 API | ❌ |
| 权限控制 | 表级权限（SRE 管理） | 用户级 + 商户级精细权限 | ❌ |

**结论：当前架构不适合 OpenClaw 直接访问数据。**

---

## 2. 核心结论

**不需要推翻现有架构。需要在现有 RDS + DataWorks + BI 之上，新增三层。**

### 2.1 四层根因分析

| 角度 | 根因 | 分析人 |
|---|---|---|
| 架构 | 缺 OLAP 即席查询引擎 + 缺语义层 | @Cindy |
| 数仓 | 缺面向 AI 的数仓出口（`ADS-AI`），指标口径散落 | @Alina |
| 基础设施 | 访问平面错位，无 AI 可消费的查询服务面 | @Angel |
| 数据接口 | 无 MCP Tool 接口，权限无法透明注入 | @Livia |

---

## 3. 目标架构

### 3.1 架构总图

```
──── 保留不动 ──────────────────────────────
RDS（业务库）→ DataWorks（离线 ETL）→ BI（固定报表）
                     │
                     │ 数据同步（T+1 / 小时级）
                     ▼
──── 新增三层 ──────────────────────────────

┌──────────────────────────────────────────────┐
│  Layer 1: OLAP 分析引擎                      │
│  引擎：StarRocks（推荐）或 ClickHouse         │
│  数据：从 DataWorks 同步核心表                │
│  能力：TB 级即席聚合查询，秒级返回             │
│  运维：数据平台团队                           │
└──────────────────┬───────────────────────────┘
                   │
┌──────────────────▼───────────────────────────┐
│  Layer 2: 语义层 + ADS-AI                    │
│  数仓层：交易/商品/营销 ADS-AI 汇总表          │
│  语义层：YAML 指标定义（机器可读）             │
│  Owner：数据产品/数据架构（横向）              │
│  实现：数据开发团队                           │
│  验证：业务域 Owner                          │
└──────────────────┬───────────────────────────┘
                   │
┌──────────────────▼───────────────────────────┐
│  Layer 3: 数据 API 层（MCP Server）          │
│  接口：5 个 MCP Tool                         │
│  鉴权：open_id → employee_id → 权限注入       │
│  治理：限流、超时、成本感知、审计              │
│  运维：平台团队                               │
└──────────────────┬───────────────────────────┘
                   │
          OpenClaw (400+ 实例)
```

### 3.2 关键设计原则

1. **不直连 RDS** — 保护在线业务
2. **不替代 DataWorks** — 继续做离线 ETL
3. **不替代 BI** — 继续出固定报表
4. **OLAP 引擎是增量** — 不对现有系统做破坏性改动
5. **语义层是灵魂** — 把散落 SQL 变成结构化指标
6. **MCP Tool 不是自由 SQL** — OpenClaw 通过受限 Tool 访问数据

---

## 4. 技术栈选择

| 层 | 推荐方案 | 备选 | 选择理由 |
|---|---|---|---|
| OLAP 引擎 | **StarRocks** | ClickHouse | 多表 JOIN 友好、物化视图强、MySQL 兼容协议、阿里云有托管 |
| 语义层定义 | **YAML + Git** | Cube.js, dbt semantic layer | 轻量、团队已熟悉 Git、可审计 |
| 数仓 AI 出口 | **DataWorks 新增 `ADS-AI` Schema** | StarRocks 独立建 Schema | 复用现有数仓资产，在 StarRocks 内建 AI 专用 schema |
| MCP API | **Spring Boot 3.3** | FastAPI (Python) | 团队 Java 为主，已有 Spring Boot 基础设施 |
| 异步通知 | **飞书 API** | — | 已有 App ID + Secret，已验证可发消息 |
| 权限存储 | **MySQL** | — | 复用现有员工映射表 |

---

## 5. 各层详细设计

### 5.1 Layer 1: OLAP 引擎（StarRocks）

**部署方式（推荐）**：阿里云 EMR StarRocks 托管版
- 无需自运维 StarRocks 集群
- 与阿里云 VPC 内 DataWorks 同步链路最短
- 最小规格：3 FE + 3 BE（可根据数据量调整）

**数据同步策略**：
| 数据表类型 | 同步方式 | 频率 | 延迟 |
|---|---|---|---|
| 交易明细事实表 | DataWorks → StarRocks Broker Load | T+1 日批 | D+1 08:00 |
| 交易汇总表（商户日） | DataWorks → StarRocks | T+1 日批 | D+1 08:00 |
| 维表（商户、行业、地区） | 全量同步 | 每日 | D+1 08:00 |
| 实时指标（后续） | CDC → Kafka → StarRocks Routine Load | 近实时 | 分钟级 |

---

### 5.2 Layer 2: 语义层 + ADS-AI

#### 5.2.1 数仓分层（在 StarRocks 内）

```
ods_trade_detail       — 交易明细 ODS
ods_merchant           — 商户维表
ods_product            — 商品维表
  │
dwd_trade_detail      — 交易明细宽表（含商户、商品、渠道维度）
dwd_refund_detail     — 退款明细宽表
  │
dws_trade_merchant_day — 商户日交易汇总
dws_trade_region_day   — 地区日交易汇总
dws_trade_industry_day — 行业日交易汇总
dws_refund_day         — 退款日汇总
  │
ads_ai_trade_overview     — 交易概览（给 OpenClaw）
ads_ai_merchant_ranking   — 商户排行（给 OpenClaw）
ads_ai_trade_trend        — 交易趋势（给 OpenClaw）
ads_ai_industry_breakdown — 行业分布（给 OpenClaw）
```

#### 5.2.2 Phase 1 首次开放的表（交易主题域）

| 表名 | 用途 | 对应 Tool |
|---|---|---|
| `ads_ai_trade_overview` | 交易总览：金额、笔数、客单价、退款率 | query_metrics |
| `ads_ai_trade_trend` | 按天/周/月交易趋势 | query_trend |
| `ads_ai_merchant_ranking` | 商户交易 TOP N | query_top_n |
| `ads_ai_industry_breakdown` | 行业分布 | query_metrics |
| `dwd_trade_detail` | 交易明细（限制查询） | query_detail |

#### 5.2.3 核心指标定义（Phase 1，15 个指标）

**交易域指标**：

| # | 指标名 | 口径 | 维度 | 刷新 |
|---|---|---|---|---|
| 1 | 交易金额 | SUM(success_amount) | 时间/商户/行业/地区/渠道 | T+1 |
| 2 | 交易笔数 | COUNT(success_orders) | 同上 | T+1 |
| 3 | 退款率 | 退款笔数/交易笔数 | 时间/商户/行业/渠道 | T+1 |
| 4 | 退款金额 | SUM(refund_amount) | 同上 | T+1 |
| 5 | 客单价 | 交易金额/交易笔数 | 时间/商户/行业/地区 | T+1 |
| 6 | 活跃商户数 | COUNT(DISTINCT active_merchant) | 时间/行业/地区 | T+1 |
| 7 | 笔均金额 | 交易金额/交易笔数 | 时间/渠道 | T+1 |
| 8 | 新商户数 | COUNT(new_merchants) | 时间/行业/地区 | T+1 |
| 9 | 交易成功率 | 成功笔数/总笔数 | 时间/渠道 | T+1 |
| 10 | 商户日均交易额 | AVG(merchant_daily_amount) | 时间/行业 | T+1 |
| 11 | TOP 10 商户交易额 | RANK merchant BY amount | 时间 | T+1 |
| 12 | 行业交易占比 | SUM(amount) GROUP BY industry | 时间 | T+1 |
| 13 | 支付渠道分布 | SUM(amount) GROUP BY channel | 时间 | T+1 |
| 14 | 环比增长率 | (本期-上期)/上期 | 时间/商户/行业 | T+1 |
| 15 | 周同比 | (本周-上周)/上周 | 时间/商户/行业 | T+1 |

---

### 5.3 Layer 3: MCP 数据 API 层

#### 5.3.1 五个 MCP Tool

| Tool | 用途 | 返回 | 查询目标 |
|---|---|---|---|
| `query_metrics` | 查指标在某时间范围的值 | 聚合值 + 维度拆分 | `ads_ai_*` / `dws_*` |
| `query_trend` | 查指标的趋势数据 | 时序数据 | `ads_ai_trade_trend` |
| `query_top_n` | 排名查询（TOP/BOTTOM N） | 排行列表 | `ads_ai_merchant_ranking` |
| `query_detail` | 查明细（限制 1000 行） | 明细列表 | `dwd_trade_detail` |
| `query_explain` | 查指标口径定义 | 指标定义文档 | 语义层 YAML |

#### 5.3.2 query_metrics 接口设计示例

```json
// 请求（OpenClaw 发起）
{
  "metric": "交易金额",
  "time_range": "2026-05-01~2026-05-20",
  "dimensions": ["industry"],
  "filters": ["region=华东"],
  "request_id": "req_xxx",
  "actor_identity": {
    "open_id": "ou_xxx",
    "employee_id": "E1000"
  }
}

// 响应
{
  "metric": "交易金额",
  "definition": "所有状态为成功的支付订单金额之和，不含退款",
  "data_freshness": "T+1 08:00",
  "source_layer": "StarRocks ads_ai_industry_breakdown",
  "data": [
    {"industry": "餐饮", "value": 123456789.00, "unit": "元"},
    {"industry": "零售", "value": 98765432.00, "unit": "元"}
  ],
  "request_id": "req_xxx"
}
```

#### 5.3.3 OpenClaw System Prompt 对接口径

在 OKR 项目的 `FeishuEventController` 基础上，OpenClaw 的 system prompt 不再写表名/SQL，而是写指标名/Tool：

```
你是收钱吧数据助手。你可以使用以下工具帮员工分析数据：

可用工具：
- query_metrics：查询指标，支持维度和过滤
- query_trend：查询趋势（按天/周/月）
- query_top_n：查询排名
- query_detail：查询明细（最多1000行）
- query_explain：查询指标的口径定义和可用维度

可用指标：交易金额、交易笔数、退款率、退款金额、客单价、
活跃商户数、笔均金额、新商户数、交易成功率、商户日均交易额
可用维度：时间、商户、行业、地区、支付渠道
```

---

## 6. 数据流与关键链路

### 6.1 核心查询链路（用户问 → 结果返回）

```
用户（飞书）
  │  "昨天我的交易额是多少？"
  ▼
OpenClaw
  │  自然语言理解 → 识别指标"交易金额"+维度"我的商户"
  ▼
MCP Tool: query_metrics
  │  参数：metric=交易金额, time_range=2026-05-20, filters=[我的商户]
  ▼
API 层权限注入
  │  SELECT authorized_merchant_ids WHERE employee_open_id='ou_xxx'
  │  → WHERE merchant_id IN (m1, m2, m3...)
  ▼
StarRocks 执行查询
  │  SELECT SUM(amount) FROM ads_ai_trade_overview
  │  WHERE date='2026-05-20' AND merchant_id IN (m1, m2, m3...)
  │  耗时：< 2 秒
  ▼
API 层返回结果
  │  {"metric":"交易金额","value": 345678.90, "data_freshness":"T+1 08:00"}
  ▼
OpenClaw 生成自然语言回复
  │  "你的商户昨天交易额为 ¥345,678.90（T+1 数据，更新于今早 8:00）"
  ▼
飞书消息推送给用户
```

### 6.2 异步大查询链路

```
用户 → OpenClaw → query_metrics（预估扫描 > 1000 万行）
  │
  ▼
API 层：降级为异步任务
  │  返回："查询数据量较大，已转为后台任务，完成后通过飞书通知你"
  ▼
后台执行查询 → 结果存入临时表
  │
  ▼
飞书 API 推送通知
  │  "你的查询已完成：2026年 Q1 餐饮行业交易趋势，点击查看：[链接]"
```

---

## 7. 查询治理规则

| 规则 | 阈值 | 违规处理 |
|---|---|---|
| 结果集上限 | `LIMIT 1000` 强制注入 | 截断 + 提示缩小范围 |
| 单次查询超时 | 5 秒 | Kill + 返回"请缩小范围或使用异步查询" |
| 时间窗口上限 | 最大 180 天 | 超限自动截断 + 提示 |
| 并发限制 | 每用户最多 3 并发 | 第 4 个排队，返回"请等待" |
| 大查询降级 | 预估扫描 > 1000 万行 | 转为异步任务，飞书通知结果 |
| 重复查询 | 相同参数 60 秒内 | 返回缓存结果 |
| 成本感知 | 预估扫描 > 10GB | 返回预估扫描量 + 要求用户确认 |
| 禁止操作 | DROP / ALTER / DELETE / 子查询嵌套 > 2 层 | 直接拒绝 |
| 禁止对象 | 不在白名单的表 | 返回"该数据暂不支持查询" |

---

## 8. 权限模型

### 8.1 三层权限体系

```
Level 1: 身份认证
  feishu_open_id → employee_id（查 employee_identity 表）

Level 2: 数据范围
  employee_id → authorized_merchant_ids（查数据权限表）
  
Level 3: 字段级控制
  敏感字段白名单（如身份证号、银行卡号 → MCP API 不暴露）
```

### 8.2 SQL 权限注入（透明给 OpenClaw）

```sql
-- 用户查询"昨天交易额"时，API 层自动改写为：
SELECT SUM(amount) AS total
FROM ads_ai_trade_overview
WHERE date = '2026-05-20'
  AND merchant_id IN (
    SELECT merchant_id FROM data_access_permission
    WHERE employee_id = 'E1000' AND access_granted = 1
  )
  AND region IN (
    SELECT region_code FROM data_access_permission
    WHERE employee_id = 'E1000' AND access_granted = 1
  );
```

OpenClaw 不需要知道权限规则，API 层透明注入。

---

## 9. 三条底线（不可突破）

| # | 底线 | 原因 | 违反后果 |
|---|---|---|---|
| 1 | **OpenClaw 不直连 RDS** | 保护在线业务性能和数据安全 | 生产事故 |
| 2 | **OpenClaw 不自由写 SQL** | 防止低效/有害查询、越权访问 | 安全漏洞 + 性能灾难 |
| 3 | **逐域开放，不一刀切全上** | 7 个业务域口径成熟度不同，按评估分批上线 | 项目失控 |

---

## 10. 分阶段实施计划

### Phase 1：支付交易域 MVP + 底座搭建（Week 1-3）

**目标**：OLAP 引擎 + MCP API 底座就绪，交易域 Top 15 指标 OpenClaw 可查

| 周 | 工作内容 | 产出 | 团队 |
|---|---|---|---|
| W1 | StarRocks 部署 + DataWorks 数据同步管道搭建 | 引擎就绪，交易核心表同步 | 平台 + 数据开发 |
| W1 | 交易域 `dws_trade_*` + `ads_ai_trade_*` 建表 | AI 专用汇总层初版 | 数据开发 |
| W2 | 15 个交易指标 YAML 定义 + 业务 Owner 验收 | 指标语义层就绪 | 业务 Owner + 数据开发 |
| W2 | MCP 5 Tool 接口开发 + 权限注入 | API 层开发完成 | 平台 |
| W3 | StarRocks 查询性能调优 | 95% 查询 < 3 秒 | 平台 |
| W3 | 端到端联调（飞书 → OpenClaw → MCP → StarRocks → 回复） | 全链路跑通 | 全体 |
| W3 | 灰度 5-10 人（交易/支付团队）试用 + 修 bug | 反馈收集 | 全体 |

**Phase 1 交付标准**：
- [ ] 员工在飞书问"昨天我的交易额"，OpenClaw 能在 5 秒内返回正确结果
- [ ] 权限正确：员工 A 查不到员工 B 的商户数据
- [ ] 查询过载保护生效（超时、结果集限制、并发限制）
- [ ] 查不到的数据给出明确提示（而不是编造）

### Phase 2：广告营销域 + CRM 客户域（Week 4-6）

**目标**：新增 2 个业务域，总指标 35-45 个，灰度扩大至 20-50 人

| 周 | 工作内容 | 产出 |
|---|---|---|
| W4 | 广告营销域 ADS-AI 建表 + 10-12 个指标 YAML 定义 | 营销域就绪 |
| W4-5 | CRM 客户域 ADS-AI 建表（脱敏处理）+ 8-10 个客户指标 | 客户域就绪（PII 脱敏） |
| W5 | NL2SQL 准确率优化（基于 Phase 1 的 bad case） | 准确率 > 85% |
| W5 | 异步大查询 + 飞书通知链路 | 长查询体验优化 |
| W6 | 灰度扩大至 50 人（跨支付、营销、运营团队） | 规模化验证 |

### Phase 3：信贷金融域 + 智慧门店域（Week 7-9）

**目标**：新增 2 个高价值域，总指标 55-65 个，灰度 100+ 人

| 周 | 工作内容 | 产出 |
|---|---|---|
| W7 | 信贷金融域 ADS-AI 建表（高合规隔离）+ 8-10 个信贷指标 | 信贷域就绪（独立权限） |
| W8 | 智慧门店域 ADS-AI 建表 + 8-10 个门店指标 | 门店域就绪 |
| W8-9 | 查询成本看板、审计日志完整化、数据血缘 | 治理闭环 |
| W9 | 灰度 100+ 人 + 稳定性观察 | 规模化就绪 |

### Phase 4：校园业务域 + 全域来店客流域（远期，Week 10+）

**目标**：完成 7 个业务域全覆盖

| 工作内容 | 说明 |
|---|---|
| 校园业务域口径治理 + ADS-AI 建表 | 先治理再开放，不直接暴露原始口径 |
| 全域来店客流域跨域数据整合 | 依赖前 6 个域的底座稳定 |

---

## 11. 组织分工模型

### 11.1 分工矩阵（RACI）

| 工作项 | 数据平台团队 | 数据开发团队 | 业务域 Owner | 数据产品 Owner | OpenClaw 团队 |
|---|---|---|---|---|---|
| OLAP 引擎部署运维 | **R/A** | C | | I | |
| 数据同步管道 | C | **R/A** | | I | |
| ADS-AI 表开发 | | **R/A** | C | C | |
| 指标 YAML 定义 | | C | **R/A** | **A** | |
| 指标口径仲裁 | | I | C | **R/A** | |
| MCP API 开发 | **R/A** | C | | I | |
| 权限模型与实现 | **R/A** | C | I | C | |
| 查询治理 | **R/A** | | | C | |
| OpenClaw Prompt/Tool 编排 | | | I | C | **R/A** |
| 飞书通知集成 | | C | | | **R/A** |

> R=Responsible（执行）, A=Accountable（批准）, C=Consulted（咨询）, I=Informed（知晓）

### 11.2 Phase 1 最小项目班子（4-5 人）

| 角色 | 人数 | 来源 | 关键技能 | 职责 |
|---|---|---|---|---|
| **数据产品 Owner（横向）** | 1 | 数据架构组 或 指定 PM | 业务理解 + 项目管理 | 跨域协调、口径仲裁、优先级决策 |
| **交易域业务 Owner** | 1 | 交易/支付业务线 | 交易业务口径（不需要 SQL） | 定义 15 个交易指标的"什么、怎么算" |
| **数据开发工程师** | 1-2 | 数据开发团队 | 数仓建模、SQL、DataWorks 调度 | 建表、ETL、ADS-AI 实现 |
| **平台工程师** | 1 | 基础设施/平台团队 | StarRocks、Spring Boot、权限 | OLAP 部署、MCP API 开发、查询治理 |

**总投入**：4-5 人 × 3 周 = Phase 1 交付

### 11.3 团队协作节奏

| 节奏 | 内容 | 参与者 | 时长 |
|---|---|---|---|
| 每日（前 2 周） | 站会：昨天的进度、今天的计划、阻塞点 | 全体 | 15 分钟 |
| 每周一 | 指标口径评审会（新指标上会） | 数据产品 + 业务 Owner + 数据开发 | 30 分钟 |
| 每周五 | 周度 Review：进展、风险、下周计划 | 全体 + @aaronchang | 30 分钟 |
| 灰度期 | 每日看反馈，P0/P1 bug 当天修 | 数据开发 + 平台 | 按需 |

---

## 12. 协作机制：YAML 指标模板

### 12.1 为什么用 YAML

- 业务团队不需要懂 SQL，只需要结构化描述"指标是什么"
- 数据团队不需要理解业务深层含义，只需要翻译成 SQL
- Git 管理：有修改历史，有 owner 归属，可审计
- 机器可读：可以被 MCP API 层直接解析，给 `query_explain` 返回

### 12.2 模板格式

```yaml
# 文件：metrics/trade.yaml
# 主题域：交易
# Owner：张三（收单业务线）
# 审批人：李四（数据产品 Owner）
# 最后修改：2026-05-21

metrics:
  - id: trade_amount
    name: 交易金额
    definition: 所有状态为"成功"的支付订单金额之和，不含退款
    formula: SUM(order_amount) WHERE order_status = 'SUCCESS'
    unit: 元
    refresh: T+1 08:00
    time_granularity: [day, week, month]
    dimensions:
      - name: 时间
        field: trade_date
      - name: 商户
        field: merchant_id
        sensitive: false
      - name: 行业
        field: industry_code
      - name: 地区
        field: region_code
      - name: 支付渠道
        field: payment_channel
    sensitive: false
    query_example: "昨天的交易金额是多少？按行业拆分"
    business_owner: 张三
    data_engineer: 王五
    status: approved  # draft | in_review | approved | deprecated
```

### 12.3 协作流程

```
业务 Owner 填 YAML（定义口径）
         ↓
数据开发翻译成 SQL + 建 ADS-AI 表
         ↓  (口径有歧义时提 issue)
业务 Owner 验收数据正确性
         ↓
数据产品 Owner 标记 approved
         ↓
MCP API 层加载指标到 Tool 白名单
         ↓
OpenClaw 可查询
```

---

## 13. Phase 1 详细排期与资源

### 13.1 甘特图概要

```
Week 1              Week 2              Week 3
[StarRocks 部署]    [MCP API 开发]      [性能调优]
[数据同步管道]       [权限注入]          [端到端联调]
[ADS-AI 建表]       [指标 YAML 验收]    [灰度 5-10 人]
[维表导入]           [Tool Prompt 对齐]  [Bug 修复]
```

### 13.2 各角色工作量估算

| 角色 | W1（人天） | W2（人天） | W3（人天） | 合计 |
|---|---|---|---|---|
| 数据产品 Owner | 1 | 2 | 2 | 5 |
| 交易域业务 Owner | 1 | 1.5 | 0.5 | 3 |
| 数据开发（每人） | 4 | 3 | 2 | 9 |
| 平台工程师 | 4 | 4 | 3 | 11 |

> 注：数据产品 Owner 和业务 Owner 为兼职投入，数据开发和平台工程师为专职投入。

### 13.3 前置依赖

| 依赖项 | 状态 | 负责人 |
|---|---|---|
| 阿里云 EMR StarRocks 开通 | ⬜ 待申请 | 平台工程师 |
| DataWorks → StarRocks 网络打通 | ⬜ 待确认 | 平台工程师 |
| 飞书 App 凭证（已有，但需轮换 secret） | ⬜ 待轮换 | @aaronchang |
| 开放 API 权限（发送消息） | ⬜ 待确认 | @aaronchang |
| 员工飞书 open_id 映射表 | ⬜ 待整理 | 数据开发 |

---

## 14. 风险与缓解措施

| 风险 | 概率 | 影响 | 缓解 |
|---|---|---|---|
| StarRocks 同步管道延迟，数据不准 | 中 | 中 | T+1 数据标注 `data_freshness`，用户知情 |
| 业务 Owner 无法及时验收指标口径 | 中 | 高 | 前 15 个指标选最成熟的，不需要从头定义 |
| OpenClaw 翻译准确率不足 | 中 | 中 | W3 灰度期重点收集 bad case，Phase 2 优化 |
| StarRocks 查询量超预期，成本过高 | 低 | 高 | 查询治理规则生效（限流/缓存/成本感知） |
| 400 人同时查询打挂 StarRocks | 低 | 高 | 灰度 5 → 20 → 50 → 全量，逐步放量 |
| 团队对 StarRocks 运维不熟悉 | 中 | 中 | 用阿里云托管版，降低运维负担 |

---

## 15. 附录

### A. StarRocks vs ClickHouse 对比

| 维度 | StarRocks | ClickHouse | 本次倾向 |
|---|---|---|---|
| 多表 JOIN | 较好（MPP 架构） | 弱（建议宽表） | StarRocks |
| MySQL 兼容 | 原生 MySQL 协议 | 需适配 | StarRocks |
| 物化视图 | 强（异步/同步） | 有 | StarRocks |
| 阿里云托管 | EMR StarRocks | 阿里云 ClickHouse | 两者都有 |
| 社区活跃度 | 高（2023 年后快速增长） | 高（成熟） | — |
| 运维复杂度 | 中 | 中-高 | StarRocks |
| 写入性能 | 中 | 极高 | — |

### B. 关键参考文档

- StarRocks 阿里云 EMR：<https://help.aliyun.com/product/435992.html>
- 飞书开放 API（发消息）：<https://open.feishu.cn/document/server-docs/im-v1/message/create>
- MCP 协议规范：<https://modelcontextprotocol.io>

### C. 术语表

| 缩写 | 全称 | 说明 |
|---|---|---|
| OLTP | Online Transaction Processing | 在线事务处理（如 RDS MySQL） |
| OLAP | Online Analytical Processing | 在线分析处理（如 StarRocks） |
| ODS | Operational Data Store | 贴源层 |
| DWD | Data Warehouse Detail | 数据仓库明细层 |
| DWS | Data Warehouse Summary | 数据仓库汇总层 |
| ADS | Application Data Store | 应用数据层（AI / BI 出口） |
| MCP | Model Context Protocol | AI 模型上下文协议（Anthropic） |
| RACI | Responsible/Accountable/Consulted/Informed | 责任分配矩阵 |
