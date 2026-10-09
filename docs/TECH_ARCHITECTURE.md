# Farm Match — 上架技术架构方案

> 目标：上架 Apple App Store 与 Google Play，供用户下载使用。  
> 客户端：Flutter 一套代码出 iOS / Android。  
> 后台：一套共用 API（账号、进度、端内支付、远程配置）。  
> 文档版本：v1.0｜2026-10-09｜作者：开发

---

## 1. 目标与边界

| 要做 | 不做（本版） |
|------|----------------|
| 双端商店上架安装包 | 网页版作为主交付（Pages 可继续作演示） |
| 游客可玩 + 云存档 | 邮箱登录 |
| Sign in with Apple / Google 绑定找回进度 | 订阅、体力系统 |
| 内购道具包 + 去广告 SKU | 卖通关 / 卖战力 |
| 远程配置（关卡开关、商品表等） | 服务端强校验每一局玩法（首版客户端权威） |

玩法与关卡逻辑仍在客户端执行；服务端管身份、存档、发货、配置。

---

## 2. 总体架构

```
┌─────────────────┐     HTTPS/JSON      ┌──────────────────────────┐
│  Flutter App    │ ◄─────────────────► │  Backend API (单体)      │
│  iOS / Android  │                     │  auth / progress / iap /  │
│                 │                     │  config                  │
└────────┬────────┘                     └────────────┬─────────────┘
         │                                           │
         │ StoreKit 2 / Play Billing                 │
         ▼                                           ▼
┌─────────────────┐                     ┌──────────────────────────┐
│ App Store /     │ ──验单回调/查询──► │ Postgres (+ Redis 可选)  │
│ Google Play     │                     │ 用户 / 进度 / 订单 / 配置 │
└─────────────────┘                     └──────────────────────────┘
```

**推荐默认栈（已锁定，可后续替换实现细节）**

| 层 | 选型 |
|----|------|
| 客户端 | Flutter 3.x + 官方 IAP 插件 |
| 后台 | Node.js + NestJS（或同等 Express；语言可换，接口契约不变） |
| 数据库 | PostgreSQL |
| 缓存/限流 | Redis（可选，首版可无） |
| 鉴权 | JWT access + refresh；游客 deviceId |
| 托管 | GCP 或 AWS（Cloud Run / ECS + RDS/Cloud SQL） |
| 观测 | Firebase Crashlytics + Analytics；服务端日志 + 错误追踪 |
| 现有 HTML 演示 | 继续维护于 GitHub Pages，与商店包解耦 |

---

## 3. 客户端模块（Flutter）

| 模块 | 职责 |
|------|------|
| `game` | 三消玩法、关卡、道具（移出/撤回/洗牌），由现有 HTML 逻辑迁移或重写 |
| `auth` | 游客启动、Apple/Google 绑定、token 刷新 |
| `progress` | 本地缓存 + 与云端同步（`updatedAt` 较新优先） |
| `iap` | 拉商品、发起购买、把凭证交服务端验单、刷新权益 |
| `config` | 启动拉取远程配置并缓存；失败用本地默认 |
| `store_ui` | 商店页：三档道具包 + 去广告 |

**端能力**

- 竖屏锁定、刘海 `safe-area`、禁过度滚动
- 音频：iOS 需用户手势后解锁
- 崩溃与基础事件上报

---

## 4. 账号与登录

### 4.1 策略（策划已定）

1. **游客**：首次打开生成 `guestId`（设备侧稳定 ID），可完整游玩，**支持云存档**。
2. **绑定**：上架后提供 **Sign in with Apple**、**Google**，用于找回/合并进度。
3. **不做邮箱**。

### 4.2 合并规则

- 游客绑定第三方：若第三方账号无进度 → 挂载当前游客进度。  
- 若第三方已有进度 → 比较 `updatedAt`（或 `clearedLevel`），**保留较新/较高者**，另一份写入 `progress_archive` 备查；客户端弹一次确认（文案由市场/策划补）。  
- Apple：若提供其他第三方登录，上架必须提供 Sign in with Apple。

### 4.3 主要接口

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/v1/auth/guest` | body: `{ deviceId }` → `{ userId, accessToken, refreshToken }` |
| POST | `/v1/auth/apple` | Apple identityToken → 登录或绑定 |
| POST | `/v1/auth/google` | Google idToken → 登录或绑定 |
| POST | `/v1/auth/refresh` | refreshToken → 新 access |
| POST | `/v1/auth/bind` | 已登录游客绑定 Apple/Google（可与上合并） |

---

## 5. 进度（云存档）

### 5.1 存档内容（建议字段）

```json
{
  "clearedLevel": 12,
  "inventory": { "move": 3, "undo": 1, "shuffle": 0 },
  "removeAds": false,
  "settings": { "muted": false },
  "updatedAt": "2026-10-09T05:00:00.000Z",
  "clientVersion": "1.0.0"
}
```

### 5.2 同步策略

- 过关、消耗道具、购买成功后：防抖上传（如 1s）+ 关键时补传。  
- 冲突：服务端 `updatedAt` 新者胜；客户端覆盖本地并可选 toast。  
- 首版**不**做逐操作反作弊；明显异常订单/刷道具靠服务端发货与限流。

### 5.3 接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/v1/progress` | 拉取当前用户进度 |
| PUT | `/v1/progress` | 全量或合并上传；带 `updatedAt` / `If-Match` |

---

## 6. 内购（IAP）

### 6.1 商品表（策划已定）

| SKU（建议 ID） | 价格 | 内容 |
|----------------|------|------|
| `pack_small` | $2.99 | 移出×5、撤回×5、洗牌×5 |
| `pack_medium` | $4.99 | 各×12 |
| `pack_large` | $9.99 | 各×30 |
| `remove_ads` | $2.99 | 终身去广告（本版可先不上广告，SKU 预留） |

- 不卖通关、不卖战力。  
- 不开体力、不开订阅。  
- App Store / Play Console **双端 SKU 对齐**（可映射表）。

### 6.2 发货流程（必须服务端验单）

1. 客户端向商店发起购买。  
2. 客户端拿到收据 / purchaseToken，调用 `POST /v1/iap/verify`。  
3. 服务端向 Apple / Google 验单（防重放：`transactionId` / `orderId` 唯一）。  
4. 验单成功 → 写入订单表 → 发放道具或 `removeAds=true` → 返回最新 `inventory` / 权益。  
5. 客户端以服务端结果为准刷新 UI。

### 6.3 接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/v1/iap/products` | 可与 config 合并；或商店价以商店 SDK 为准、权益以服务端为准 |
| POST | `/v1/iap/verify` | `{ platform, productId, receipt \| purchaseToken, ... }` |
| GET | `/v1/iap/orders` | 可选：用户订单历史（客服用） |

---

## 7. 远程配置

启动（及进前台）拉取，失败用内置默认。

### 7.1 下发内容（策划已定）

```json
{
  "version": 3,
  "levelsEnabled": true,
  "levelFlags": { "1": true, "50": true },
  "products": [
    { "id": "pack_small", "move": 5, "undo": 5, "shuffle": 5, "priceUsd": 2.99 },
    { "id": "pack_medium", "move": 12, "undo": 12, "shuffle": 12, "priceUsd": 4.99 },
    { "id": "pack_large", "move": 30, "undo": 30, "shuffle": 30, "priceUsd": 9.99 },
    { "id": "remove_ads", "lifetime": true, "priceUsd": 2.99 }
  ],
  "toolUnitHint": { "move": 1, "undo": 1, "shuffle": 1 },
  "adsEnabled": false
}
```

### 7.2 接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/v1/config` | query: `appVersion`, `platform`；可 CDN 缓存短 TTL |

---

## 8. 后台数据模型（简表）

- `users`：id, guest_device_id, apple_sub, google_sub, created_at  
- `progress`：user_id PK, payload JSONB, updated_at  
- `orders`：id, user_id, platform, product_id, store_txn_id UNIQUE, status, raw JSONB, created_at  
- `config`：key, value JSONB, updated_at（或单行当前配置 + version）  
- `progress_archive`：合并冲突备份（可选）

---

## 9. 安全与合规

- 全站 HTTPS；密钥仅存服务端环境变量。  
- IAP **禁止**仅信客户端回调。  
- 接口鉴权：Bearer JWT；游客与正式用户同一 user 体系。  
- 限流：auth / verify 按 IP + user。  
- 隐私政策、用户协议 URL（上架必备）；美国休闲受众，文案英文。  
- 数据最小化：进度与订单，不做无关采集。  
- 儿童：若标 4+，遵守双方商店儿童/隐私要求（市场确认年龄分级）。

---

## 10. 上架清单

**账号与工程**

- [ ] Apple Developer Program  
- [ ] Google Play Console  
- [ ] App ID / Bundle ID、包名确定并全局唯一  
- [ ] 证书、描述文件、Play App Signing  

**商店物料**

- [ ] 1024 图标、截图、简介、关键词（@市场）  
- [ ] 隐私政策 / 条款线上地址  
- [ ] 年龄分级、内容分级问卷  

**IAP**

- [ ] 四档商品双端创建，沙盒账号测通验单  
- [ ] `remove_ads` 预留，与 `adsEnabled` 配置联动  

**审核辅助**

- [ ] 演示账号（若强制登录页；游客流可说明）  
- [ ] 审核备注：玩法说明、IAP 说明  

---

## 11. 仓库与工程拆分（建议）

```
farm-match-app/          # Flutter（新建仓库或 monorepo packages/app）
farm-match-api/          # NestJS API
farm-match/              # 现有 HTML 演示（Pages），保持独立
docs/TECH_ARCHITECTURE.md
```

CI：Flutter → TestFlight / Play Internal；API → 容器部署。

---

## 12. 里程碑（建议）

| 阶段 | 交付 |
|------|------|
| M1 | API：guest auth + progress + config 骨架；Flutter 空壳登录与存档 |
| M2 | 玩法迁移可玩；本地道具 |
| M3 | IAP 验单打通（沙盒）；商店页 |
| M4 | Apple/Google 绑定；合并进度 |
| M5 | 商店物料 + 内测 → 提审 |

---

## 13. 已定与待定

**已定**

- 共用一套后台；Flutter 一套出双端（默认锁定；若改 RN，接口契约不变）  
- 游客云存档；上架加 Apple / Google；无邮箱  
- 商品：小/中/大道具包 + 去广告；无订阅/体力/卖通关  
- 配置：关卡开关、商品表、道具单价提示、是否开广告  

**实现时可再确认（不阻塞本文）**

- 云厂商最终选型（GCP vs AWS）  
- 后台语言若不用 Nest，保持本文 REST 契约即可  
- 进度冲突时 UI 文案（策划/市场）  
- HTML 演示与 Flutter 是否长期双轨  

---

## 14. 修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| v1.0 | 2026-10-09 | 首版：架构 + 接口草案 + 策划内购/登录/配置锁定 |
