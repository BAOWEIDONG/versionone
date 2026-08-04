# 代码 Review 报告 — 数据流转 & 前后端对接就绪度

> 审查范围: types.ts / store/app.ts / lib/api.ts / lib/scoring.ts / lib/campActivities.ts / mock/data.ts 及相关组件

---

## 一、严重问题 (必须修复)

### 1.1 审核创建 RewardClaim 时 campId 为空字符串

**文件**: `FulfillmentCenterView.vue` 第 243-251 行

```typescript
const claim: RewardClaim = {
  // ...
  campId: '',  // ← BUG: 空字符串
};
```

**影响**: `getCampRewardClaims(campId)` 的过滤逻辑是 `!c.campId || c.campId === campId`，空字符串是 falsy，导致该 claim 出现在所有营期中。学员端切换营期时会看到不该看到的记录。

**修复方向**: `crossCampPendingAudit` 已有 `camp.id`，需要将 campId 传入 `auditTarget`，再写入 claim。

### 1.2 积分商城完全没有 API 层

**文件**: `store/app.ts` 第 389-494 行 + `lib/api.ts`

以下 Store 方法只改本地状态，不调 API，`USE_MOCK=false` 后数据刷新即丢:

| Store 方法 | 缺失的 API |
|---|---|
| `addPointProduct` | POST /point-products |
| `updatePointProduct` | PATCH /point-products/:id |
| `deletePointProduct` | DELETE /point-products/:id |
| `exchangePointProduct` | POST /point-exchanges |
| `updateExchangeStatus` (取消) | PATCH /point-exchanges/:id |
| `shipExchange` | PATCH /point-exchanges/:id |

`init()` 也没有加载 `pointProducts` 和 `pointExchanges`。

### 1.3 活动配置 & 结营寄语无 API 持久化

**文件**: `store/app.ts` 第 89-122 行

| 数据 | 当前存储 | 缺失 API |
|---|---|---|
| `activityConfigByCamp` | 硬编码在 store 初始值 | GET/PUT /activity-config/:campId |
| `campMessages` | 硬编码在 store 初始值 | GET/PUT /camp-message/:campId/:studentId |
| `mealTimeConfigByCamp` | 仅内存（init 不加载） | GET /meal-time-config/:campId |

`updateMealTimeConfig(campId, config)` 调用 `api.updateMealTimeConfigApi(config)` 时没传 campId，后端无法知道更新哪个营期的配置。

### 1.4 日期格式不一致

**文件**: `store/app.ts` 第 436 行 vs `mock/data.ts` 第 19 行

```typescript
// store 中 exchangePointProduct 创建记录:
exchangeDate: new Date().toISOString(),  // "2026-08-01T12:00:00.000Z"

// mock 数据中:
exchangeDate: iso(-3, '14:30:00'),      // "2026-07-29 14:30:00"
```

两套格式混用。`new Date().toISOString()` 带 T 和 Z 后缀，与全站 `yyyy-MM-dd HH:mm:ss` 约定不一致。同样的问题出现在 `shipExchange` 的 `shipDate` 和 `deliveredAt`。

---

## 二、中等问题 (建议修复)

### 2.1 `reward-manage` 和 `point-exchange-history` 视图为死代码

**文件**: `store/app.ts` 第 50、60 行 + `App.vue` 第 24、40、67、82 行

这两个 View 在 `View` 类型中声明、在 `App.vue` 中注册了组件映射，但全局搜索 `setCurrentView('reward-manage')` 和 `setCurrentView('point-exchange-history')` 没有任何调用方。功能已被 `RewardConfigView` 和 `MyRewardsView` 替代。

**建议**: 删除 View 类型声明、App.vue 的 import 和 viewMap 注册。组件文件本身可暂保留但标记为未使用。

### 2.2 `getAllExchanges()` 已无调用方

**文件**: `store/app.ts` 第 457-460 行

`FulfillmentCenterView` 直接使用 `store.pointExchanges` 而非 `store.getAllExchanges()`。该函数仅在 store 中定义并 export，无任何组件调用。

### 2.3 `exchangePointProduct` 未校验 deliveryMethod 是否被商品支持

**文件**: `store/app.ts` 第 418-447 行

Store 层只检查积分和库存，不检查 `product.deliveryOptions` 是否包含传入的 `deliveryMethod`。当前依赖组件层校验，但 Store 作为数据层应该有防御性校验。

### 2.4 `getCampRewardTiers` 的 `!t.campId` 过滤可能泄露全局奖品

**文件**: `store/app.ts` 第 611-613 行

```typescript
function getCampRewardTiers(campId: string) {
  return rewardTiers.value.filter((t) => !t.campId || t.campId === campId);
}
```

没有 campId 的 tier 会出现在所有营期中。这在设计上可能是"全局共享奖品"，但也可能导致配置错误时奖品跨营期泄露。建议改为显式标记全局奖品（如 `campId: 'global'`），而非用空值隐式表示。

### 2.5 `api.ts` 中体重记录的 JSDoc 被截断

**文件**: `lib/api.ts` 第 240-243 行

```typescript
/**
 * 获取指定学员的体重打卡记录
 * GET /weight-records?studentId={studentId}
 *
// ============ 教练活动（图文/视频） ============
```

注释块未闭合就被下一个 section 截断，会导致 JSDoc 解析异常。

### 2.6 PointProduct 没有 campId 字段 — 商城商品全局共享

**文件**: `types.ts` 第 180-191 行 + `store/app.ts` 第 380 行

注释写明"全局，不按营期隔离"。如果这是设计决策，建议在类型定义中加注释说明。如果后续需要按营期配置商品，需要加 campId 字段。

---

## 三、数据流转链路审查

### 3.1 活动奖励领取链路 (有 bug)

```
学员达标(自动计算)
  → 营养师审核 (FulfillmentCenterView.submitAudit)
    → 创建 RewardClaim { status: 'confirmed', campId: '' }  ← BUG: campId 丢失
    → 扣减 RewardTier.stock
  → 学员领取 (CampActivitiesView.submitClaim)
    → 更新 RewardClaim { status: 'pending', deliveryMethod, recipientInfo }
  → 营养师发货 (FulfillmentCenterView.submitShipping/submitInPerson)
    → 更新 RewardClaim { status: 'shipped'/'in-person', trackingNumber }
```

**问题**: 第一步 campId 丢失导致后续链路中学员端按营期过滤看不到该 claim（或看到不该看到的）。

### 3.2 积分兑换链路 (逻辑正确，缺 API)

```
学员兑换 (PointsMallView.confirmExchange)
  → store.exchangePointProduct
    → 校验积分 (getStudentMallPoints = 总积分 - 非取消的已消耗)
    → 校验库存
    → 创建 PointExchangeRecord { status: 'pending' }
    → 扣减 PointProduct.stock
  → 营养师发货 (FulfillmentCenterView)
    → store.shipExchange { status: 'fulfilled', trackingNumber }
  / 学员取消
    → store.cancelExchange → updateExchangeStatus('cancelled')
      → 恢复 PointProduct.stock
      → 积分自动返还 (getStudentMallPoints 排除 cancelled)
```

**积分逻辑正确**: 取消后积分自动恢复，因为 `getStudentMallPoints` 只统计 `status !== 'cancelled'` 的消耗。
**库存逻辑正确**: 取消时 `updateExchangeStatus` 恢复库存，兑换时扣减库存。
**缺失**: 全程无 API 调用，`USE_MOCK=false` 后数据不持久。

### 3.3 积分计算链路 (正确)

```
饮食打卡 → dietRecords (含 dietitianScore)
运动打卡 → exerciseRecords (含 dietitianScore, duration)
  → calculateDietScore: 按日聚合, min(Σ餐分, 6), 未批注=0
  → calculateExerciseScore: duration≥40min=1分 + dietitianScore加分
  → calculateTotalScore = dietScore + exerciseScore
  → getStudentMallPoints = max(0, totalScore - 已消耗积分)
```

**逻辑正确**: 积分完全由打卡记录和营养师评分推导，无硬编码积分值。排名使用 dense ranking，同分同名次。

### 3.4 趣味活动链路 (正确，纯函数)

```
体重打卡 → computeWeightMilestones (历史最低体重，达标即锁定)
饮食+运动+体重打卡 → computeWeeklyChallenges (按周计算，8种主题循环)
饮食+运动+体重打卡 → computeLuckyDraw (完成率≥80%)
```

**逻辑正确**: 全部为纯函数，不产生副作用，不读写积分。活动进度实时计算，打卡后自动更新。

---

## 四、Mock 数据审查

### 4.1 硬编码的初始状态 (需迁移到后端)

| 数据 | 位置 | 问题 |
|---|---|---|
| `activityConfigByCamp` 初始值 | store/app.ts:89-93 | 三个营期的配置写死在代码里 |
| `campMessages` 初始值 | store/app.ts:83-85 | 寄语写死 |
| `mealTimeConfigByCamp` | store/app.ts:134 | 初始为空对象，不加载 |

### 4.2 Mock 数据本身 (可保留为开发种子)

`MOCK_STUDENTS`、`MOCK_DIET_RECORDS` 等 mock 数据使用相对日期 (`iso(-6, '10:00:00')`)，保证每次运行数据都在当前日期附近，这部分设计合理，可保留为开发种子数据。

---

## 五、清理建议汇总

| # | 操作 | 文件 | 优先级 |
|---|---|---|---|
| 1 | 修复 submitAudit 中 campId 为空 | FulfillmentCenterView.vue | P0 |
| 2 | 补充积分商城 API 层 (6个接口) | lib/api.ts + store/app.ts | P0 |
| 3 | 补充活动配置/寄语 API | lib/api.ts + store/app.ts | P1 |
| 4 | 修复日期格式统一为 yyyy-MM-dd HH:mm:ss | store/app.ts | P1 |
| 5 | init() 补充加载 pointProducts/exchanges/activityConfig | store/app.ts | P1 |
| 6 | 修复 mealTimeConfig API 传 campId | lib/api.ts + store/app.ts | P1 |
| 7 | 删除 reward-manage / point-exchange-history 死代码 | store/app.ts + App.vue | P2 |
| 8 | 删除 getAllExchanges 未使用方法 | store/app.ts | P2 |
| 9 | 修复 api.ts 注释截断 | lib/api.ts | P2 |
| 10 | Store 层加 deliveryMethod 防御校验 | store/app.ts | P2 |

---

*Report generated on 2026-08-01*
