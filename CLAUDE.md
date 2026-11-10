# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目简介

waynboot-mall 是一个 Spring Boot 3.5.14 / JDK 17 的开源 H5 商城后端（Maven 多模块），包含管理端 API、移动端 API、消息消费服务三个可独立部署的应用。后台管理端保留 Spring Security + JWT，移动端已切换为 Sa-Token。当前重点围绕商品、订单、库存、支付、MQ、本地消息补偿和对账治理拆分，目标是让交易链路在高并发场景下更易维护、更可扩展。

## 常用命令

```bash
# 构建（跳过测试），jar 输出到 ./jars/
mvn clean package -DskipTests

# 运行所有测试
mvn test

# 按领域模块运行测试
mvn test -pl waynboot-domain-trade
mvn test -pl waynboot-domain-inventory

# 运行单个测试类
mvn test -pl waynboot-domain-trade -Dtest=PaymentCallbackSupportTest

# 运行单个测试方法
mvn test -pl waynboot-domain-trade -Dtest=PaymentCallbackSupportTest#markOrderPaid_alreadyPaid_skipsUpdate

# 按模块启动（也可用 IDE 直接启动各模块 *Application 主类）
mvn -pl waynboot-admin-api spring-boot:run
mvn -pl waynboot-mobile-api spring-boot:run
mvn -pl waynboot-message/waynboot-message-consumer spring-boot:run

# 使用本地 Maven 仓库目录
mvn "-Dmaven.repo.local=.m2/repository" test
```

## 本地启动

**前置中间件**：MySQL 8.0、Redis 3.0+、RabbitMQ 3.0+（需 delayed-message 插件）、Elasticsearch 7.x（需 IK 分词器和拼音插件）

1. 在 `wayn_shop` 数据库中按序导入 `db-init/` 下的 SQL：`wayn_shop_*.sql`（基础库表），增量治理脚本 `inventory_stock` / `local_message` / `order_status_log` / `payment_flow` / `seckill`
2. 编辑各模块的 `application-dev.yml`，填写本地中间件连接信息
3. 分别启动三个 Spring Boot 主类：
   - `AdminApplication`（waynboot-admin-api）→ 端口 81
   - `MobileApplication`（waynboot-mobile-api）→ 端口 82
   - `MessageApplication`（waynboot-message/waynboot-message-consumer）→ 端口 85

**Docker 部署**：
```bash
# 中间件
docker-compose -f docker-compose-es7-rabbitmq-redis-mysql.yml up -d
# 应用
docker-compose up -d
```

## AI 协作准则

本仓库的 Claude 协作规则吸收 `andrej-karpathy-skills` 的四个原则：编码前思考、简洁优先、精准修改、目标驱动验证。处理非平凡任务时，先说明假设、歧义点、影响范围和验证方式；如果需求可能有多种解释，不要静默选择复杂方案，应先收敛边界或提出更简单方案。

默认做最小可维护改动，不为单次需求新增通用框架、过度抽象或未要求的配置能力。每一行 diff 都应能追溯到用户目标；不要顺手重排无关代码、删除既有注释、重构未涉及链路或清理用户已有改动。发现无关坏味道时，先在反馈中说明，除非用户明确要求，否则不要纳入本次修改。

订单、库存、支付、秒杀、MQ、本地消息、认证等核心链路必须转成可验证目标：先读取现有实现和文档，明确不变量，再按模块边界小步修改，最后用对应模块测试、编译或文档 diff 自检闭环。文档更新同样需要对照代码，发现版本、认证模式、模块职责等描述与现状不一致时，应在本次相关文档中一并修正。

## 开发规范

（摘要自 `AGENTS.md`，编写代码前可查阅完整规则）

- **分层职责**：Controller 只做参数校验、权限、日志和 VO 返回封装，不写核心业务逻辑；业务逻辑放 Service / Support；MQ Consumer 只接收消息并调用 Service。入口模块（admin-api / mobile-api / message-consumer）不重复实现领域逻辑。
- **接口契约**：出参优先使用明确的 `VO` / `ResVO`，不要用裸 `Map<String, Object>` 承载复杂结构，也不直接暴露数据库实体。
- **代码风格**：统一 4 空格缩进、UTF-8（无 BOM）；类名 `UpperCamelCase`、方法和字段 `lowerCamelCase`、包名小写；禁止硬编码状态值、渠道值、业务常量，统一收敛到常量或枚举。
- **注释要求**：新增 support / helper / assembler 类必须有类级注释（职责边界及与编排层、领域层的关系）；公共方法补齐方法注释；关键分支（锁控制、条件更新、库存回滚、优惠券回退、支付回调防重等）写"为什么这样做"的注释。
- **模块协作**：新增公共能力前先确认是否确实被多个入口模块复用，避免继续堆入 `waynboot-common`；领域实现优先落对应 `waynboot-domain-*` 模块，跨域接口和实体才放 `waynboot-domain-api`；`waynboot-util` 保持轻量，不引入强业务依赖。
- **测试**：测试类命名 `*Test.java`，分布在各模块 `src/test/java`；涉及 MyBatis-Plus Lambda Wrapper（`Wrappers.lambdaQuery` 等）的纯单元测试必须先初始化实体 `TableInfo` 缓存（参考 `MybatisPlusTableInfoTestHelper`），否则会被 lambda cache 异常打断。
- **提交与配置**：commit message 沿用仓库风格，使用简短中文动宾短句（如 `修复用户密码更新问题`）；不要提交 `application*.yml` 等本地环境配置（`.githooks/pre-commit` 会拦截 staged 的 `application-dev` 配置）；涉及配置、数据库、中间件的改动需在 PR 中写明影响范围、依赖服务和验证方式。
- **交易链路改造**必须从完整链路评估，不要只做单点改动：梳理"同步主链路 + 异步补偿链路"两段，说明幂等键、事务边界、失败补偿方式和运维检查点；禁止新增 `shop_payment_profit_sharing_flow` 等资金拆分流水表。
- 阶段性重构说明文档统一放仓库根目录或 `docs/`，命名 `<域>_<阶段>_<主题>.md`。

## 模块结构

| 模块 | 职责 |
|---|---|
| `waynboot-common` | 通用配置、切面、策略接口、通用模型和基础设施 |
| `waynboot-domain-api` | 跨领域契约：entity / Mapper / Service 接口 / VO / 枚举 |
| `waynboot-domain-trade` | 订单、支付编排、状态机、本地消息、对账 |
| `waynboot-domain-inventory` | 库存冻结、库存流水、Redis 库存快照、库存对账 |
| `waynboot-domain-goods` | 商品、SKU、类目、搜索和 ES 同步 |
| `waynboot-domain-cart` | 购物车读写与选中商品聚合 |
| `waynboot-domain-promotion` | 优惠券、营销位和 Diamond 策略实现 |
| `waynboot-payment-channel` | 微信、支付宝、易支付的支付 / 退款渠道适配 |
| `waynboot-admin-api` | 管理端 REST 控制器 + Spring Security 配置 + Spring `@Scheduled` 治理任务 |
| `waynboot-mobile-api` | H5 移动端 REST 控制器 |
| `waynboot-message` | RabbitMQ 消息消费（core + consumer 子模块） |
| `waynboot-data` | Redis / Elasticsearch 基础设施封装 |
| `waynboot-util` | 常量、枚举、异常、工具类 |

## 核心架构

### 业务领域（按 `waynboot-domain-*` 拆分）

核心业务逻辑通过 ServiceImpl + Support 类组织，而非直接写在 Controller 中。Service 接口是对外暴露的能力边界，Support 类是同包内部的能力切片：

```
waynboot-domain-trade/src/main/java/com/wayn/domain/trade/support/order/        # 订单提交、取消、生命周期
waynboot-domain-trade/src/main/java/com/wayn/domain/trade/support/admin/order/  # 管理端发货、退款
waynboot-domain-trade/src/main/java/com/wayn/domain/trade/support/payment/      # 支付回调幂等处理
waynboot-domain-trade/src/main/java/com/wayn/domain/trade/outbox/               # 本地消息表（outbox 模式）中继与补偿
waynboot-domain-trade/src/main/java/com/wayn/domain/trade/support/seckill/      # 秒杀异步落单、超时释放、支付确认
waynboot-domain-inventory/src/main/java/com/wayn/domain/inventory/support/      # 库存冻结、Redis 预扣、库存快照
waynboot-domain-goods/src/main/java/com/wayn/domain/goods/support/              # 商品 ES 同步、变更
waynboot-domain-cart/src/main/java/com/wayn/domain/cart/support/                # 购物车读写
waynboot-domain-promotion/src/main/java/com/wayn/domain/promotion/support/      # 优惠券领取
waynboot-domain-promotion/src/main/java/com/wayn/domain/promotion/support/seckill/ # 秒杀活动、短期 token、漏斗过滤
```

### 关键设计模式

**订单提交链**：`OrderSubmitStockReduceStep` 等实现责任链，Redis 预扣库存作为并发门控，MySQL 为最终数据源，`finally` 块保证 Redis 释放。

**支付回调幂等**：`PaymentCallbackSupport` 使用条件更新（`applyExpectedStatus`）+ 二次查询区分"已被其他线程处理"与"真实失败"。`PaymentFlowSaveResult.DUPLICATE_CONFLICT` 通过抛出 `IllegalStateException` 触发 `TransactionTemplate` 回滚。

**本地消息表（outbox）**：`LocalMessageService` 以 `DuplicateKeyException` 实现幂等插入，指数退避重试（最多 5 次，间隔上限 60 秒），超限后转 `FAILED` 死信。`LocalMessageFailureClassifier` 分类失败原因，`LocalMessageCompensationLogService` 记录失败、死信和人工重投日志。

**秒杀漏斗模型**：秒杀下单先走活动状态、短期 token、用户频控、SKU 限流和 Redis Lua 库存原子扣减，再通过本地消息可靠投递异步落单，未支付订单由 60 秒延迟消息自动关单。订单仍统一写入 `shop_order` / `shop_order_goods`，活动上下文通过 `shop_order_activity_relation` 关联。

**状态机条件更新**：所有订单状态变更统一使用 `applyExpectedStatus(updateWrapper, sourceStatus)` 做乐观并发控制，`updated == 0` 时抛 `BusinessException`。

**Redis Lua 脚本**：`RedisCache` 中的库存预扣（`luaReserveStock`）和释放（`luaReleaseReservedStock`）均为原子 Lua 脚本，返回值语义：`1`=成功/幂等，`-1`=库存不足，`-2`=未初始化，`-3`=参数非法。

### 库存一致性

库存两态：`number`（可售库存）+ `lockedStock`（冻结库存）。变化规则：

```text
下单冻结：number - n，lockedStock + n
支付确认：lockedStock - n
超时取消：number + n，lockedStock - n
退款回补：number + n
```

原则：Redis Lua 只做入口削峰和热点 SKU 并发闸门；MySQL 条件更新负责最终一致性、避免超卖；库存流水通过唯一 `flowKey` 保证本地消息重试下的库存副作用幂等；`InventoryReconciliationService` 根据库存流水推导冻结库存，识别并修复 `lockedStock` 差异；热点 SKU 通过 Redis 库存快照和分桶 Key 降低单 Key 竞争。

### MQ 消费者（waynboot-message-consumer）

- `EmailSendConsumer`（EMAIL_DIRECT_QUEUE）
- `OrderPayConsumer`（ORDER_DIRECT_QUEUE）
- `OrderDelayConsumer`（ORDER_DELAY_QUEUE）
- `SeckillOrderConsumer`（SECKILL_ORDER_DIRECT_QUEUE）
- `SeckillOrderDelayConsumer`（SECKILL_ORDER_DELAY_QUEUE）

### 数据访问

- ORM：MyBatis-Plus，逻辑删除字段 `del_flag`（1=已删除）
- Mapper XML：`classpath*:mapper/**/*Mapper.xml`
- Redis：通过 `RedisCache` 统一封装，避免直接使用 `RedisTemplate`

### 认证

- 管理端（端口 81）：JWT，有效期 120 分钟，header `Authorization`
- 移动端（端口 82）：Sa-Token，有效期 30 天，header `Authorization`，兼容 `Authorization: <token>` 与 `Authorization: Bearer <token>` 两种格式
- 鉴权边界：后台继续使用 Spring Security + `@PreAuthorize`，移动端不要重新引入 Spring Security 过滤链，业务代码统一通过 `MobileSecurityUtils` 读取登录用户
- 移动端关键类（`com.wayn.mobile.framework`）：`MobileSaTokenConfiguration`（参数、白名单、跨域）、`MobileAuthorizationHeaderFilter`（请求头兼容）、`MobileSaTokenExceptionHandler`（鉴权失败统一错误码）

### 定时任务

Spring `@Scheduled` 调度，运行在 `AdminApplication` 进程内：

- 入口类：`waynboot-admin-api` → `com.wayn.admin.api.schedule.TradeGovernanceScheduledTask`
- 配置类：`com.wayn.admin.api.schedule.TradeScheduleProperties`，前缀 `wayn.schedule.trade.*`，cron 与批量参数均可覆盖默认值
- 三个治理任务：`refreshStockSnapshot` 刷新 Redis 库存快照（默认每 5 分钟）、`reconcileInventory` 库存流水对账（默认每小时整点，可受控自动修复 `lockedStock`）、`reconcilePaymentDaily` 支付日终对账（默认每日 02:00）
- 业务侧 relay：`LocalMessageRelayTask`（`waynboot-domain-trade`，每 5 秒一次）
- 失败处理由 Spring 默认 `TaskUtils.LoggingErrorHandler` 兜底，任务方法不需要 try/catch

### 治理接口

后台交易治理 REST 接口位于 `TradeOpsController`，路径前缀 `/ops/trade`，涵盖本地消息查询/重投、库存对账、Redis 库存快照刷新和支付日终对账。后台首页统计由 `DashboardController` 提供，挂载在 `/shop/dashboard`，统计口径集中在 `DashboardService`。

## 文档地图

- `ONBOARDING.md`：新手指南，含下单 / 支付回调链路追踪、订单状态流转图、库存模型、outbox、秒杀功能链路、测试规范与 FAQ —— 深入某条链路前先读对应章节
- `README.md` / `README_en.md`：对外项目说明与完整业务能力介绍
- `AGENTS.md`：完整的 AI 协作与 Java 开发规范（本文件"开发规范"一节为其摘要）
- `docs/MODULE_REFACTOR_PLAN.md`：模块拆分演进记录
- `docs/AI_GUIDE_AGENT_DESIGN.md`：AI 导购 Agent 技术方案（设计稿，尚未落地）
- `.claude/skills/waynboot-mall/SKILL.md`：仓库技能说明
