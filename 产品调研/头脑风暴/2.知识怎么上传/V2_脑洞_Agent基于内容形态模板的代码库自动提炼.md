# SkyBrain 解决方案 V2 — 脑洞探索：Agent 基于内容形态模板的代码库自动提炼

> 状态：讨论中
> 阶段：V2 解决方案与产品雏形
> 核心问题：确立了四大知识内容形态之后，能否让 Agent 自动扫描项目仓库代码，按形态模板生成对应的知识文件？

---

## 一、问题缘起：内容形态分类不只是"贴标签"

前置脑洞已确立了知识的四大内容形态：

```
操作型（Skill / Workflow / Reference）   → 告诉 AI "怎么做"
契约型（API Spec / Schema）             → 告诉 AI "是什么"
认知型（Architecture / Glossary）       → 告诉 AI "在哪、是什么项目"
约束型（Rule / Convention）             → 告诉 AI "绝不能做什么"
```

这些分类最初被设计来回答"知识库里存什么"。但如果把视角反转：

> **这些内容形态本质上定义了"Agent 需要从代码库中提取什么"。**

每一种形态对应了代码库中特定的**信号源**，Agent 只要能识别这些信号源，就能按模板提炼出结构化的知识文件。

---

## 二、核心设计理念：形态模板驱动的自动提炼

```
                     ┌─────────────────────┐
                     │   四大内容形态模板    │
                     │  (定义了要提取什么)   │
                     └──────────┬──────────┘
                                │ 驱动
                ┌───────────────┼───────────────┐
                ↓               ↓               ↓
          📋 操作型模板    📐 契约型模板    📖 认知型模板    ⚖️ 约束型模板
          (Skill提取器)  (API提取器)    (架构提取器)    (规范提取器)
                │               │               │               │
                ↓               ↓               ↓               ↓
          ┌─────────────────────────────────────────────────────────┐
          │              Agent 扫描项目仓库代码                       │
          │  读取目录结构 / 配置文件 / 核心代码 / 注释 / AST 分析     │
          └─────────────────────────────────────────────────────────┘
                                │
                                ↓
          ┌─────────────────────────────────────────────────────────┐
          │          按形态模板生成结构化知识文件                      │
          │  skills/payment_auth.md   api_specs/order_payment.md    │
          │  architecture/overview.md  rules/sql_safety.md          │
          └─────────────────────────────────────────────────────────┘
                                │
                                ↓
                    存入本地沙盒 → 开发者审核 → 提报团队
```

---

## 三、每种形态的提取策略

### 📋 操作型知识提取（Skill / Workflow / Reference）

**提取目标**：从代码中逆向工程出"在这个项目里怎么做某件事"

| 知识类型 | 信号源 | 提取策略 |
|----------|--------|----------|
| **Skill** | Service 层核心方法、工具类、中间件 | 定位关键业务类 → 提取方法签名 + 核心逻辑 → 大模型总结为"操作范例" |
| **Workflow** | 跨模块调用链路、事件处理链、状态机 | 分析模块间的 import 依赖 → 追踪调用链 → 大模型还原为"业务流程" |
| **Reference** | `package.json`、`pom.xml`、`go.mod`、import 语句 | 解析依赖声明 → 提取版本号 → 扫描项目中对该依赖的实际使用方式 → 生成"依赖用法" |

**示例提取流程（Skill）：**

```
Agent 扫描 src/main/java/com/xxx/service/OrderService.java
  → 识别到 PaymentService.createPayment() 方法
  → 读取方法体：签名验证 → 幂等检查 → 支付调用 → 异常回滚
  → 结合注释和异常处理逻辑
  → 生成 Skill 文件：
     "支付创建操作规范"
     前置条件：订单状态为 CONFIRMED
     核心流程：1. 验签 2. 幂等 3. 调用支付网关 4. 异常回滚
     关键代码：...（提取实际代码片段）
```

### 📐 契约型知识提取（API Spec / Schema）

**提取目标**：从接口定义和数据模型中提取精确的契约

| 知识类型 | 信号源 | 提取策略 |
|----------|--------|--------|
| **API Spec** | Controller 文件、路由定义、JSDoc/JavaDoc、OpenAPI 注解 | 解析路由注解 → 提取 HTTP 方法 + 路径 + 参数类型 → 提取鉴权注解 → 生成接口契约 |
| **Schema** | Entity/Model 文件、数据库迁移文件、ORM 定义 | 解析实体类字段 → 提取类型 + 约束 + 关系 → 生成表结构定义 |

**示例提取流程（API Spec）：**

```
Agent 扫描 src/main/java/com/xxx/controller/OrderController.java
  → 识别 @PostMapping("/order/payment") 注解
  → 读取方法签名：payment(@RequestBody OrderPaymentReq req)
  → 读取 JavaDoc: "处理订单支付请求，需要 JWT 鉴权"
  → 读取返回类型 PaymentResult 的结构
  → 生成 API Spec 文件：
     POST /order/payment
     入参：OrderPaymentReq {orderId, amount, channel}
     返回：PaymentResult {paymentId, status}
     鉴权：JWT
```

### 📖 认知型知识提取（Architecture / Glossary）

**提取目标**：从项目结构和技术选型中还原宏观视图

| 知识类型 | 信号源 | 提取策略 |
|----------|--------|--------|
| **Architecture** | 目录结构、`README.md`、`skybrain.yaml`、技术栈配置文件 | 解析目录树 → 识别分层模式（DDD/MVC/微服务） → 提取模块列表 → 大模型总结架构概述 |
| **Glossary** | 枚举类、常量类、领域对象类名、业务注释 | 扫描实体类和枚举 → 提取类名 + 字段说明 → 大模型推断业务术语语义 |

**示例提取流程（Architecture）：**

```
Agent 扫描项目根目录：
  → 识别 src/main/java/com/xxx/ 下：
     controller/、service/、domain/、infrastructure/
  → 识别到典型 DDD 分层模式
  → 扫描各子目录发现模块：order、payment、inventory
  → 读取 pom.xml 确认技术栈：Spring Boot 3.2 + MyBatis-Plus
  → 生成 Architecture 文件：
     "OrderSystem 采用 DDD 分层架构
      Interface 层：controller/ 处理 HTTP
      Application 层：service/ 编排用例
      Domain 层：domain/ 核心逻辑
      Infrastructure 层：infrastructure/ 持久化
      模块包含：order（订单核心）、payment（支付）、inventory（库存）"
```

### ⚖️ 约束型知识提取（Rule / Convention）

**提取目标**：从配置文件和代码模式中推断团队规范

| 知识类型 | 信号源 | 提取策略 |
|----------|--------|--------|
| **Rule** | ESLint/PMD/Checkstyle 规则、安全中间件、加密工具类 | 解析 lint 配置文件 → 提取强制规则 → 发现安全注解/过滤器 → 生成安全约束 |
| **Convention** | 代码命名模式统计、目录布局约定、Git 提交规范 | 统计分析类名模式 → 发现路径约定 → 提取编码风格 → 生成编码公约 |

**示例提取流程（Convention）：**

```
Agent 扫描所有 Controller 类：
  → 统计发现 100% 的 Controller 类名以 Controller 结尾
  → 100% 放在 src/.../controller/ 路径下
  → 所有 Service 类均无 HttpServletRequest 引用
  → 生成 Convention 文件：
     "Controller 类统一放在 .../controller/，类名以 Controller 结尾
      Service 层禁止直接操作 HttpServletRequest"
```

---

## 四、各形态提取的可靠性差异

不同形态的自动化提取可靠程度不同，需要标注"置信度"：

| 内容形态 | 提取方式 | 自动提取置信度 | 说明 |
|----------|----------|---------------|------|
| API Spec | 注解/路由解析 | 🟢 高 | 结构化程度高，可直接从注解提取 |
| Schema | 实体类解析 | 🟢 高 | 字段类型和约束明确可读 |
| Reference | 依赖声明解析 | 🟢 高 | `package.json`/`pom.xml` 结构化 |
| Architecture | 目录结构 + LLM 推断 | 🟡 中 | 目录可读，但分层模式需要推断 |
| Convention | 代码模式统计 | 🟡 中 | 概率统计，可能遗漏小众约定 |
| Workflow | 调用链推理 | 🟠 中低 | 跨模块链路推理依赖 LLM 理解 |
| Skill | 方法逻辑总结 | 🟠 中低 | 需要 LLM 深度理解业务语义 |
| Rule | 配置文件解析 | 🟢 高 | lint 配置可直接翻译 |
| Glossary | 语义推断 | 🔴 低 | 业务术语的真实含义难以自动获取 |

**关键洞察**：Rubye、Convention 等可以从配置文件直接翻译的形态，自动提取效果最好；而 Skill、Workflow、Glossary 等依赖业务理解的知识，自动提取只能产出"草案"，需要人工审核修正。

---

## 五、提取产物与现有流程的衔接

自动提取的知识文件直接对接已有的"知识三级生命周期"：

```
Agent 扫描仓库 → 按形态模板生成知识文件
                         │
                         ↓
                 存入本地沙盒（Draft 状态）
                         │
                         ↓
              开发者预览、编辑、修正
                         │
                         ↓
                MCP "提报给团队"（部分提报）
                         │
                         ↓
              云端 AI 预审（冲突检测）
                         │
                         ↓
              TL Web 端审批 → 并入中央大脑
```

**关键设计**：
- 自动提取不会直接进入中央大脑，必须是 Draft → 沙盒 → 提报 → 审核 的完整路径
- 每种形态的生成产物自带"提取置信度"标签，低置信度的自动标记为"需人工确认"
- 开发者可以选择"全部提报"或"逐条挑选"哪些自动生成的知识进入提案

---

## 六、方案面临的核心挑战与应对策略

### 挑战 1：不同项目的技术栈差异大

**问题**：Java Spring Boot 和 Go Gin 的注解体系完全不同，一套提取策略无法覆盖。

**应对**：
- 每种技术栈维护独立的"提取器插件"（如 SpringBootExtractor、GinExtractor）
- 框架无关的形态（如 Architecture、Convention）使用通用策略
- Agent 首先通过项目配置文件自动识别技术栈，再加载对应提取器

### 挑战 2：大模型推断的"幻觉"

**问题**：LLM 可能把一段废弃代码当作规范，或错误理解业务术语的含义。

**应对**：
- 所有自动提取的知识标记来源（源文件路径 + 行号），开发者可追溯验证
- 低置信度的知识自动标记为"需人工审核"，不会静默进入提案
- 结合 Git Blame 信息，给近期活跃代码更高权重

### 挑战 3：提取时机与触发策略

**问题**：什么时候触发扫描？每次提交都扫太频繁，只扫一次又会过时。

**应对**：
- 首次接入项目时做一次"全量冷启动扫描"
- 后续通过 Git Hook 监听变更，仅对有变更的模块做"增量扫描"
- 开发者也可以手动触发对特定模块的重新扫描

---

## 七、结论

本次脑洞提出：

> **四大内容形态分类不仅定义了"存什么"，更是 Agent 自动提炼知识的目标模板。** 每种形态映射到代码库中的特定信号源（注解、配置文件、目录结构、代码模式），Agent 通过识别信号源 + 大模型推断，可以自动生成相应形态的知识文件。

高置信度形态（API Spec、Schema、Reference、Rule、Convention）可以做到高度自动化；低置信度形态（Skill、Workflow、Glossary）产出草案后需人工修正。所有自动产物走完整的三级生命周期审核流，确保质量可控。
