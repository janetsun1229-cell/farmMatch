# Farm Match — 前后端模块技术说明

> 对齐架构方案 v1.1.1（免登录本机进度；Flutter；腾讯云国际站；后台仅 config + IAP 验单）。2026-10-09｜开发  
> Notion 原文：https://app.notion.com/p/3f429db626158135995ee327fef1fae0

## 0. 设计原则

1. **功能内聚、依赖单向**：UI → 应用用例 → 领域 → 数据/基础设施；禁止反向依赖
2. **接口隔离**：商店、网络、本地存储都走抽象，便于单测与换实现
3. **后台薄、客户端厚**：玩法与进度在客户端；服务端只做可信任边界（验单、配置）
4. **配置驱动扩展**：新商品/关卡段优先走远程 config + 本地 entitlements
5. **失败可降级**：config / 验单失败有本地默认与重试，不堵死开玩

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

## 2. 客户端（Flutter）

### 2.1 分层（每个 feature 内部）

| 层 | 目录约定 | 职责 |
|----|----------|------|
| Presentation | `presentation/` | 页面、Widget、Riverpod/Bloc；只渲染与派发意图 |
| Application | `application/` | 用例：改昵称、过关、购买、Restore |
| Domain | `domain/` | 实体、值对象、仓库接口；无 Flutter/HTTP 依赖 |
| Data | `data/` | 仓库实现、DTO、本地源、远程源 |

状态管理建议：**Riverpod**（或 Bloc，全项目统一）。导航：`go_router`。

### 2.2 Feature 模块

#### identity
- 域：`LocalUser`（id、nickname、createdAt）
- 用例：`BootstrapIdentity`、`UpdateNickname`
- 存储：secure storage / Hive / Isar

#### progress
- 域：clearedLevel、settings（muted 等）
- 用例：`LoadProgress`、`SaveProgress`、`ClearLevel`
- 仅本机；不做云同步；卸载即无

#### inventory
- 域：move / undo / shuffle 次数
- 用例：`ConsumeTool`、`GrantTools`
- Restore **不**恢复次数

#### entitlements
- 域：removeAds、ownedBundles `{barn, harvest}`
- 用例：`ApplyPurchase`、`RestoreEntitlements`
- 门闸：`LevelGate.canPlay(level)`

#### iap
- 抽象 `StoreClient`（StoreKit2 / Play Billing）
- 流程：Purchase → verify API → entitlements / inventory
- UI 不直接碰 SDK

#### config
- `GET /v1/config`；本地 `default_config.json` 兜底

#### game
- 牌面、托盘、匹配、关卡生成（可由 HTML 逻辑迁移）
- 依赖 progress / inventory / entitlements / config
- 禁止直接调商店或 HTTP

#### home / store_ui / settings
- 首页未解锁 → 购买提示
- Settings：改昵称、`Restore Purchases`、静音

### 2.3 扩展
- 新道具：config 商品表 + inventory + UI
- 新关卡段包：config + entitlements 映射；门闸自动生效

## 3. 服务端（NestJS）

### 3.1 模块
- **ConfigModule**：`GET /v1/config`；限流；短缓存
- **IapModule**：`POST /v1/iap/verify`；Apple/Google verifier；`store_txn_id` 幂等；不写用户进度
- **InfrastructureModule**：Postgres；出网 HTTP
- **HealthModule**：`GET /health`

### 3.2 数据表（最小）
- `orders(id, platform, product_id, store_txn_id UNIQUE, status, raw jsonb, created_at)`
- `config_entries(id, version, payload jsonb, updated_at)`

## 4. 依赖方向

客户端：`presentation → application → domain ← data`  
`game → progress | inventory | entitlements | config`  
`store_ui / settings → iap → entitlements | inventory`

服务端：`iap.controller → verify.usecase → StoreVerifierPort / OrderRepositoryPort`

禁止：game 依赖 Store SDK；API 模块循环引用。

## 5. 接口契约

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/v1/config` | 远程配置 |
| POST | `/v1/iap/verify` | 验单 |
| GET | `/health` | 探活 |

错误体：`{ "code": "IAP_VERIFY_FAILED", "message": "..." }`

## 6. 测试
- 客户端：domain/application 纯单测；iap 用 Fake StoreClient
- 服务端：verifier 夹具；订单幂等；OpenAPI 契约

## 7. 相关文档
- 架构方案：`docs/TECH_ARCHITECTURE.md` / Notion 架构页
- 产品功能与数值：`docs/GAME_FEATURES.md`

## 8. 账号、云存档与好友互赠（产品 v1.1）

客户端模块 `auth`、`cloud_save`、`gift`：默认游客本机档；第 5 关过关后可跳过绑定 X/Facebook；第 8 关撤回解锁后再推绑定并邀请好友。登录后云同步进度、道具与权益。规则与 Param ID 见 `GAME_FEATURES.md` §3.7.1、§3.7.1b、§5.14；后台 API 见 `TECH_ARCHITECTURE.md` 修订附记 v1.2。Restore Purchases 仍只恢复去广告与关卡包。
