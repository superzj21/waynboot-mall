# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this module.

## 模块定位

`waynboot-domain-trade` 是交易领域库模块（无独立 `*Application`、无 `application.yml`），承载订单、支付编排、状态机、本地消息和支付对账实现，由 `waynboot-admin-api`（端口 81）、`waynboot-mobile-api`（端口 82）、`waynboot-message-consumer`（端口 85）三个入口模块加载。

依赖关系（见 `pom.xml`）：领域能力依赖 `domain-cart` / `domain-promotion` / `domain-inventory` / `domain-goods`，渠道适配依赖 `payment-channel`，基础设施依赖 `data-redis`、`message-core`（排除 spring-boot-starter-web）。对外服务接口契约（`IOrderService`、`IPayService`、`PaymentReconciliationService` 等）定义在 `waynboot-domain-api`，本模块只提供 `ServiceImpl` + `Support` 实现，不新增对外接口副本。

## 常用命令

```bash
# 运行本模块全部测试（在仓库根目录执行；依赖模块未安装到本地仓库时加 -am）
mvn test -pl waynboot-domain-trade
mvn test -pl waynboot-domain-trade -am

# 运行单个测试类 / 单个测试方法
mvn test -pl waynboot-domain-trade -Dtest=PaymentCallbackSupportTest
mvn test -pl waynboot-domain-trade -Dtest=PaymentCallbackSupportTest#markOrderPaid_alreadyPaid_skipsUpdate
```

## 包结构与职责

```
service/impl/            外观层：PayServiceImpl、MobileOrderServiceImpl、OrderServiceImpl、
                         PaymentReconciliationServiceImpl、OrderStatusLogServiceImpl、
                         OrderUnpaidServiceImpl、PaymentFlowServiceImpl、OrderGoodsServiceImpl 等
                         只做请求分发，不堆交易编排细节
support/order/           下单编排、取消补偿、生命周期、状态机、查询、校验、组装
support/order/submit/chain/  单笔下单责任链（OrderSubmitChain + 步骤实现）
support/payment/         支付准备（PaymentPrepareSupport）、回调（PaymentCallbackSupport）、
                         支付成功后置动作（PaymentPostActionSupport）
support/seckill/         秒杀漏斗提交、异步落单、超时关单、活动库存确认/释放
support/admin/order/     管理端订单查询、发货、退款支撑
support/common/          RedisTradeLockSupport：交易链路分布式锁统一封装
outbox/                  本地消息表（outbox 模式）服务、relay、补偿、失败分类
resources/mapper/trade/  Mapper XML：Order、OrderGoods、AdminOrder、Address、Member
```

## 核心链路

### 单笔下单（support/order + submit/chain）

`OrderSubmitSupport` 负责编排，实际落单走 `OrderSubmitChain` 责任链。步骤顺序由 `OrderSubmitStep` 接口常量显式定义（Spring 注入顺序不作为业务顺序来源）：

```text
100  OrderSubmitDuplicateCheckStep      查重（可中断链条，防止重复扣库存/落库）
200  OrderSubmitContextBuildStep        上下文构建（金额计算、聚合数据，见 OrderSubmitContext）
300  OrderSubmitStockReduceStep         Redis Lua 库存预扣（并发门控，finally 保证释放）
400  OrderSubmitEntityBuildStep         订单实体组装
500  OrderSubmitSinglePersistStep       订单 + 明细落库
600  OrderSubmitSingleDelayMessageStep  未支付延迟关单 outbox 消息（与落库同事务，共同提交）
```

当前只有 `OrderSubmitMode.SINGLE` 一种模式。整条链在 `OrderSubmitSupport.submit()` 的 `@Transactional` 内执行。所有消息（异步下单、延迟关单）**不直接投递 RabbitMQ**，先由 `OrderSubmitMessageSupport` 写本地消息表，统一由 `outbox` relay 投递；延迟关单时长从 `WaynConfig.getUnpaidOrderCancelDelayTime()` 读取（分钟）。

### 订单状态机（OrderStateTransitionSupport）

```text
101 未付款 → 201 已付款 / 102 用户取消 / 103 系统取消
201 已付款 → 202 申请退款 / 301 已发货
202 申请退款 → 203 退款成功
301 已发货 → 401 用户收货 / 402 系统收货
```

- 状态流转必须经 `validateTransition` 校验，非法流转抛 `BusinessException`
- 所有状态变更必须走 `applyExpectedStatus` 条件更新（`updated == 0` 即抛异常），禁止裸 UPDATE
- `OrderStatusLogServiceImpl` 把每次流转落审计日志；日志失败只记错误，不反向打断支付/取消/退款主链路

### 支付（support/payment）

- `PaymentPrepareSupport`：支付前订单校验、商品摘要组装、支付方式原子更新
- `PaymentCallbackSupport`：渠道差异只体现在验签和参数解析，订单状态更新、幂等判断、后置动作走统一入口。条件更新 + 二次查询区分"已被其他线程处理"与"真实失败"；`PaymentFlowSaveResult.DUPLICATE_CONFLICT` 抛 `IllegalStateException` 触发 `TransactionTemplate` 回滚
- `PaymentPostActionSupport` 实现 `LocalMessageHandler`：支付回调事务内**只写本地消息**（topic `ORDER_PAID_POST_ACTION`），冻结库存确认、虚拟销量更新由 relay 重试执行，避免后置副作用丢失

### 本地消息表（outbox）

```text
INIT(0) → relay 扫描 → 投递 RabbitMQ 或调用 LocalMessageHandler → SENT(1)
                    → 失败指数退避重试（最多 5 次，间隔上限 60 秒）→ 超限转 FAILED(2)
```

- `LocalMessageService`：创建（`DuplicateKeyException` 幂等）、查询、状态流转；业务侧不直接操作状态字段
- `LocalMessageRelayTask`：`@Scheduled(fixedDelay = 5000L)`，每 5 秒批量扫描（默认 50 条）
- `LocalMessageRelaySupport`：扫描待投递消息，按消息自带 exchange/routingKey/delayMillis 投递，或按 topic 分发到本地处理器
- `LocalMessageHandler`：不需要 MQ、但仍需重试保障的业务副作用（如支付后置动作）实现此接口
- 补偿治理：`LocalMessageCompensationService`（失败查询/人工重投）、`LocalMessageCompensationLogService`（失败/死信/重投日志）、`LocalMessageFailureClassifier`（异常文本 → 稳定失败原因编码）

outbox 主题（`LocalMessageTopics`）：`ORDER_SUBMIT`、`ORDER_UNPAID_DELAY`、`ORDER_PAID_POST_ACTION`、`SECKILL_ORDER_SUBMIT`。

### 秒杀（support/seckill）

- `SeckillOrderSubmitSupport`：漏斗过滤 + 异步落单消息保存，**不直接创建订单**
- `SeckillOrderCreateSupport`：MQ 消费端调用，同一事务内完成普通 SKU 冻结、秒杀活动库存冻结、`shop_order_activity_relation` 写入
- `SeckillOrderMessageSupport`：落单消息（`SECKILL_ORDER_DIRECT_EXCHANGE/ROUTING`）；60 秒未支付延迟关单（`ORDER_DELAY_EXCHANGE` + `SECKILL_ORDER_DELAY_ROUTING`，`SECKILL_UNPAID_DELAY_MILLIS = 60_000`）
- `SeckillOrderTimeoutSupport`：**只有订单状态从待支付切到自动关闭成功后才释放活动库存**，避免与支付成功并发时误回补
- `SeckillInventoryConfirmSupport`：支付成功后活动库存冻结 → 已售，通过订单活动关联状态保证本地消息重试幂等
- `SeckillInventoryReleaseSupport`：只回补秒杀活动库存和 Redis 抢购资格；普通 SKU 冻结库存释放由 `OrderCancellationSupport` 负责，不要在此重复实现

### 取消与生命周期

- `OrderCancellationSupport`：分布式锁（`RedisTradeLockSupport`）内完成关单、冻结库存释放、优惠券回退；`TransactionTemplate` 在锁内提交，保证锁释放时数据已落库
- `OrderLifecycleSupport`：退款、取消、删除、确认收货的统一状态流转入口
- `OrderUnpaidServiceImpl`：未支付订单超时关闭入口（延迟消息消费），实际逻辑收敛到 `OrderCancellationSupport`

### 管理端支撑（support/admin/order）

- `AdminOrderQuerySupport`：列表/详情查询 + 状态、退款、支付方式展示文案（`AdminOrderMapper.selectOrderListPage`、热销商品聚合 `selectTopGoodsByPaidOrders`）
- `AdminOrderShipmentSupport`：发货前校验 + 带当前状态条件的发货更新
- `AdminOrderRefundSupport`：退款前校验、三方退款调用、退款结果落库、库存回补；条件更新避免重复退款覆盖状态

### 支付对账

`PaymentReconciliationServiceImpl` 基于支付流水、渠道账单、退款流水和订单表识别支付链路差异。日终对账调度入口在 `waynboot-admin-api` 的 `TradeGovernanceScheduledTask#reconcilePaymentDaily`（默认每日 02:00），本模块只提供对账能力，不新增调度。

## 关键不变量（修改前必须遵守）

1. 订单状态变更：先 `validateTransition` 校验，再 `applyExpectedStatus` 条件更新，禁止绕过状态机直接改状态字段
2. 支付回调事务内只写本地消息，后置副作用一律走 outbox relay（幂等键：outbox `message_key`、支付流水 `flow_key`、秒杀活动关联状态）
3. 所有 MQ 消息（含延迟消息）必须经本地消息表投递，禁止绕过 outbox 直接 `RabbitTemplate.convertAndSend`
4. 补偿/重试路径不能反向打断主链路（状态日志失败只记错误、消息缺失时跳过并告警日志）
5. 交易链路改造必须从"同步主链路 + 异步补偿链路"整体评估，说明幂等键、事务边界和失败补偿方式；禁止新增资金拆分流水表

## 测试

- 测试类 30 个 / 用例 113 个，位于 `src/test/java`，覆盖 outbox（6）、order 下单/取消/生命周期/状态机（10）、payment（2）、seckill（4）、admin order（3）、service（4）、common（1），新增逻辑应补齐对应 `*Test.java`
- 测试为 Mockito 纯单元测试（`@ExtendWith(MockitoExtension.class)`），不依赖真实 MySQL / Redis / RabbitMQ，可用 IDE 或 `mvn test -pl waynboot-domain-trade` 直接运行
- 涉及 MyBatis-Plus Lambda Wrapper 的测试必须先初始化实体 `TableInfo` 缓存（参考 `MybatisPlusTableInfoTestHelper`），否则会被 lambda cache 异常打断
