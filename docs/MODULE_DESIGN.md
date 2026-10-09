# Farm Match — 前后端模块技术说明

> 对齐架构方案 v1.1.1（免登录本机进度；Flutter；腾讯云国际站；后台仅 config + IAP 验单）。  
> 2026-10-09｜开发  
> Notion：[前后端模块技术说明](https://app.notion.com/p/3f429db626158135995ee327fef1fae0)

---

## 0. 设计原则

1. **功能内聚、依赖单向**：UI → 应用用例 → 领域 → 数据/基础设施；禁止反向依赖  
2. **接口隔离**：商店、网络、本地存储都走抽象，便于单测与换实现  
3. **后台薄、客户端厚**：玩法与进度在客户端；服务端只做可信任边界（验单、配置）  
4. **配置驱动扩展**：新商品/关卡段优先走远程 config + 本地 entitlements，少改发版逻辑  
5. **失败可降级**：config / 验单失败有明确本地默认与重试，不堵死开玩  

---

## 1. 总览

```
farm_match_app/          # Flutter
  lib/
    app/                 # 组装、路由、DI
    core/                # 横切：错误、日志、主题、常量
    features/            # 按功能竖切
    shared/              # 跨 feature 小组件/工具

farm_match_api/          # NestJS
  src/
    main.ts
    app.module.ts
    config/              # 远程配置模块
    iap/                 # 验单模块
    common/              # 过滤器、守卫、限流、日志
    infrastructure/      # DB、HTTP 出网（Apple/Google）
```

运行时：`Flutter features` ←HTTPS→ `API config | iap` → `Postgres` + `Apple/Google`

---

## 2. 客户端（Flutter）架构

### 2.1 分层（每个 feature 内部）

| 层 | 目录约定 | 职责 |
|----|----------|------|
| Presentation | `presentation/` | 页面、Widget、Riverpod/Bloc；只渲染与派发意图 |
| Application | `application/` | 用例（UseCase）：改昵称、过关、购买、Restore |
| Domain | `domain/` | 实体、值对象、仓库接口；无 Flutter/HTTP 依赖 |
| Data | `data/` | 仓库实现、DTO、本地源、远程源 |

状态管理建议：**Riverpod**（或 Bloc，全项目统一一种）。导航：`go_router`。

### 2.2 Feature 模块清单

#### `identity`

- **域**：`LocalUser`（id、nickname、createdAt）  
- **用例**：`BootstrapIdentity`（首次生成）、`UpdateNickname`  
- **存储**：`flutter_secure_storage` / Hive / Isar；昵称可普通本地 DB  
- **扩展点**：日后若加绑定登录，只新增 `AuthProvider` 适配器，不改玩法  

#### `progress`（本机进度）

- **域**：`clearedLevel`、关卡统计、settings（muted 等）  
- **用例**：`LoadProgress`、`SaveProgress`、`ClearLevel`  
- **规则**：仅写本机；**不做**云同步；卸载即无（产品已定）  

#### `inventory`

- **域**：`move` / `undo` / `shuffle` 次数  
- **用例**：`ConsumeTool`、`GrantTools`（购后发放）  
- **规则**：消耗型；Restore **不**恢复次数  

#### `entitlements`

- **域**：`removeAds`、`unlockedLevelMax`（或 `ownedBundles: {barn, harvest}`）  
- **用例**：`ApplyPurchase`、`RestoreEntitlements`  
- **规则**：非消耗；Restore 只刷新本模块  
- **门闸**：`LevelGate.canPlay(level)` 供 game/home 查询  

#### `iap`

- **依赖**：商店 SDK 抽象 `StoreClient`（StoreKit2 / Play Billing 实现）  
- **流程**：`PurchaseProduct` → 拿凭证 → `IapApi.verify` → 成功则调 `entitlements` / `inventory`  
- **错误**：用户取消、待处理、验单失败分错误码；UI 不直接碰 SDK  

#### `config`

- **远程**：`GET /v1/config`  
- **本地**：内置 `default_config.json` 兜底 + 缓存上次成功响应  
- **用例**：`RefreshConfig`（启动、进前台）  

#### `game`

- **域**：牌面、托盘、匹配规则、关卡生成（可由现有 HTML 逻辑迁移）  
- **依赖**：`progress`、`inventory`、`entitlements`（只读门闸）、`config`  
- **禁止**：直接调商店或 HTTP；购相关走 iap/entitlements  

#### `home` / `store_ui` / `settings`

- 首页方钮未解锁 → 拉起 store 购买提示  
- Settings：改昵称、英文按钮 `Restore Purchases`、静音等  

### 2.3 客户端目录示例

```
lib/features/iap/
  domain/entities/product.dart
  domain/repositories/iap_repository.dart
  application/purchase_product.dart
  application/restore_purchases.dart
  data/store_client.dart
  data/iap_api.dart
  data/iap_repository_impl.dart
  presentation/store_page.dart
```

### 2.4 客户端扩展指南

- **新道具**：config 商品表 + inventory 字段 + UI；验单发放表驱动  
- **新关卡段包**：config + entitlements 映射表；门闸自动生效  
- **换状态库 / 换本地 DB**：只改 data 与 DI 绑定  

---

## 3. 服务端（NestJS）架构

### 3.1 模块化 + 六边形思路

每个业务模块：`Controller`（入站）→ `Service/UseCase` → `Port`（出站接口）→ `Adapter`（Postgres、Apple、Google）。

`common`：全局异常过滤、请求日志、限流、DTO 校验。

### 3.2 模块清单

#### `ConfigModule`

- `GET /v1/config?appVersion&platform`  
- 读配置 + version；可按 platform/appVersion 过滤；短缓存；必须限流  

#### `IapModule`

- `POST /v1/iap/verify`  
- Body：`platform`, `productId`, `receipt` | `purchaseToken`, 可选 `packageName`  
- 流程：校验 → Apple/Google Verifier → 查重 `store_txn_id` → 写 `orders` → 返回结果  
- **不**写用户进度；幂等：同一 txn 重复请求返回首次成功结果  

#### `InfrastructureModule`

- TypeORM/Prisma + Postgres  
- 出网 HTTP（超时、重试；普通日志不落全量敏感收据）  

#### `HealthModule`

- `GET /health` 探活  

### 3.3 服务端目录示例

```
src/iap/
  iap.module.ts
  iap.controller.ts
  application/verify-purchase.usecase.ts
  domain/order.ts
  domain/ports/store-verifier.port.ts
  domain/ports/order-repository.port.ts
  infrastructure/apple-verifier.adapter.ts
  infrastructure/google-verifier.adapter.ts
  infrastructure/order.repository.ts
  dto/verify-purchase.dto.ts
```

### 3.4 数据表（最小）

- `orders(id, platform, product_id, store_txn_id UNIQUE, status, raw jsonb, created_at)`  
- `config_entries(id, version, payload jsonb, updated_at)`  

### 3.5 服务端扩展指南

- **新 SKU**：商店建商品 + config 表；一般不改表结构  
- **换云厂商**：只换部署与密钥；代码不绑腾讯 SDK  
- **日后若加账号**：新建 `AuthModule`，勿把进度塞进 IapModule  

---

## 4. 模块依赖图（允许的方向）

**客户端**

- `presentation → application → domain ← data`  
- `game → progress | inventory | entitlements | config`  
- `store_ui / settings → iap → entitlements | inventory`  
- `iap → config`  

**服务端**

- `iap.controller → verify.usecase → StoreVerifierPort / OrderRepositoryPort`  
- `config.controller → config.service → ConfigRepository`  

禁止：`game` 直接依赖 Store SDK；禁止 API 模块循环引用。

---

## 5. 关键契约（摘要）

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/v1/config` | 远程配置；失败客户端用默认 |
| POST | `/v1/iap/verify` | 验单；成功后客户端写本机 |
| GET | `/health` | 探活 |

错误体：`{ "code": "IAP_VERIFY_FAILED", "message": "..." }`

---

## 6. 测试与质量

- 客户端：domain/application 纯单测；iap 用 Fake StoreClient  
- 服务端：verifier 夹具；订单幂等；OpenAPI 契约  
- CI：analyzer + test；禁止跨层乱引用  

---

## 7. 与架构方案的关系

总架构：[docs/TECH_ARCHITECTURE.md](./TECH_ARCHITECTURE.md)  
商品与产品规则以架构方案 + 策划功能说明为准；本文只描述代码怎么切、怎么依赖、怎么扩展。

### 增长模块（v1.2+｜仅 App）

- `invite`：邀请码生成、归因、双向发奖、日/终身上限。
- `deeplink`：Universal/App Links + scheme；deferred install；路由表 invite/gift/challenge/shop。
- `rate_prompt`：连胜与关卡钩子；In-App Review；冷却与弹窗互斥。
- `gift_remind`：收礼推送、晚间未送出提醒、红点、通知权限时机。
