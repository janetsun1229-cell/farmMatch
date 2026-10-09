# 农场配对（Farm Match）游戏设计 / 功能规格说明书

| 字段 | 内容 |
|------|------|
| 文档版本 | v1.0 |
| 产品中文名 | 农场配对 |
| 产品英文名 | Farm Match |
| 受众 | 产品负责人 birdy、客户端 / 服务端工程师 |
| 范围 | 核心玩法、关卡发牌、道具、界面流程、动效音效、本地进度与 IAP、远程配置建议 |
| 语言约定 | 本文档正文为中文；**玩家可见 UI 一律英文**，不出现中文文案 |

---

## 1. 概述

### 1.1 产品定位

Farm Match 是一款面向美国休闲市场的 **三消牌堆（Triple Match Tile）** 游戏。视觉为卡通 3D、明亮饱和色；操作以单击为主，面向中老年友好（按钮大、文案短、无复杂操作）。

- 商店叙事（Store Story）基调：集市结束，先收拾门廊，再收拾谷仓。
- **商店首行文案不得出现 match / 消除** 等字样；叙事以「清理空间」为主。
- 玩家 UI 语言：**仅英文**。文档中的中文仅供内部沟通。

### 1.2 一句话玩法

牌桌上多层偏移堆叠与少量全遮挡牌堆；玩家只能点击**完全未被遮挡的顶牌**，牌飞入底部 7 格托盘；托盘中出现三张相同图案立即消除并左压；托盘满且无法三消则失败。

### 1.3 技术落地说明（产品约束）

- 当前 HTML 原型：道具次数局内固定，**无广告补道具**。
- Flutter 正式产品：默认游客本机玩；第 5 / 8 关引导绑 X 或 Facebook；登录后云存档 + 好友每日互赠道具；腾讯云国际做验单、配置、账号与云档。

---

## 2. 术语

| 术语 | 英文 / 代码建议 | 定义 |
|------|-----------------|------|
| 牌 / 卡牌 | Card / Tile | 一张可点击的图案牌，固定尺寸 |
| 图案类型 | ItemType | 15 种物品之一；每关每种出现 6 张（可被 3 整除） |
| 偏移堆叠 | Offset Stack | 上层只遮挡下层部分图标；任意遮挡即不可点 + 灰纱 |
| 全遮挡牌堆 | Full-Occlusion Pile | 同向微移 1/10 牌宽，侧边露层，图案全藏，仅顶牌可点 |
| 托盘 | Tray | 底部 7 槽，左→右入槽；三消后左压 |
| 暂存区 | Hold | Move 道具将托盘最左 3 张移至托盘上方，最多 6 张 |
| 遮挡深度 | Cover Depth | 覆盖当前牌的层数，决定灰纱不透明度 |
| 灰纱 / 遮罩 | Veil | 被遮挡牌上的灰色半透明层 |
| 发牌种子 | Level Seed | 以关卡号为种子；重开不变 |
| 关卡带 | Difficulty Band | L1–5 / L6–10 / … 等难度分段 |
| 道具 | Power / Booster | Move / Undo / Shuffle |
| 清除徽章 | Cleared Badge | 通关 L50 后主页标题下展示 |

---

## 3. 模块

### 3.1 核心消除

#### 3.1.1 点击规则

1. 仅当牌为**完全未被任何上层覆盖的顶牌**时可点击。
2. 偏移堆叠中：只要存在任意像素级覆盖（逻辑上「上层压住下层」），下层 **不可点击**，并显示灰纱。
3. 全遮挡牌堆：仅最顶一张可点击；下层图案完全隐藏。
4. 飞行动画或三消动画期间：**输入锁定**，忽略点击。

#### 3.1.2 入槽与三消

1. 合法点击后，牌沿弧线飞入托盘**当前最右空槽**（从左到右填充）。
2. 入槽完成后，立刻检测托盘中是否存在 **3 张相同 ItemType**：
   - 有：三张聚拢 → 弹出消除 → 托盘剩余牌**向左压实**（compact left）。
   - 无：保持现状。
3. 同一入槽可能只触发一组三消；若消除后托盘内又形成新三消（当前规则下一般不会连锁，因一次只入一张），仍按「入槽后扫描」处理，保证实现一致。
4. **清除托盘牌不会改变牌桌剩余牌的位置或尺寸**（牌桌布局静态）。

#### 3.1.3 失败条件

- 托盘 7 槽全部占用，且入槽后**无法形成三消** → 失败。
- Move 将牌移出托盘可腾出空位，不视为失败判定的例外；失败仅在「托盘满且无三消」时触发。

#### 3.1.4 过关条件

- 牌桌上所有牌与托盘内所有牌均已消除（牌桌空 + 托盘空）→ 过关。

#### 3.1.5 牌尺寸与布局边界

| 规则 | 值 |
|------|-----|
| 牌尺寸 | 固定为 **L1 原始牌宽/高的 85%**，全关卡终身不变 |
| 列宽上限 | 最多 **5–6 列**，不得超过屏幕宽度 |
| 纵向 | 若牌桌高度超出可视区，**仅允许纵向滚动** |
| 移除后布局 | 不重排、不缩放剩余牌 |

---

### 3.2 牌面与堆叠

#### 3.2.1 物品列表（15 种，无鸡蛋）

苹果替代鸡蛋。每关每种类型固定 **6 份**。

| 序号 | 英文 UI / 资源名建议 | 中文内部名 | 关卡引入顺序（Add Order） |
|------|----------------------|------------|---------------------------|
| 1 | scissors | 剪刀 | 1 |
| 2 | bucket | 水桶 | 2 |
| 3 | brush | 刷子 | 3 |
| 4 | carrot | 胡萝卜 | 4 |
| 5 | gloves | 手套 | 5 |
| 6 | corn | 玉米 | 6 |
| 7 | wool | 羊毛 | 7 |
| 8 | milk（milk bottle） | 奶瓶 | 8 |
| 9 | hay | 草堆 | 9 |
| 10 | pitchfork | 叉子 | 10 |
| 11 | pumpkin | 南瓜 | 11 |
| 12 | apple | 苹果 | 12 |
| 13 | mower | 割草机 | 13 |
| 14 | boots | 靴子 | 14 |
| 15 | watering can | 水壶 | 15 |

**美术约束（锁定）：**

- 道具图为**透明底 PNG**，禁止白底矩形。
- 奶瓶、羊毛等本体为白色的物品：**保留白色本体**，不得为「去白底」而抠掉本体。

#### 3.2.2 偏移堆叠（Offset Stacks）

- 上层仅覆盖下层**部分图标**。
- 任意覆盖 ⇒ 下层不可点 + 灰纱。
- 灰纱按**被盖层数（depth）**：

| 被盖层数 | 灰纱不透明度 |
|----------|--------------|
| 1 | 25% |
| 2 | 35% |
| 3 | 45% |
| 4+ | 55% |

- 上层离开后，下层阴影/灰纱立即清除（可点则无纱）。
- **禁止微旋转**：所有牌旋转角恒为 **0°**。

#### 3.2.3 全遮挡牌堆（Full-Occlusion Piles）

- 同向平移叠放，偏移量 = **牌宽的 1/10**；侧边露出层数轮廓。
- 非顶牌：**物品图案完全隐藏**，仅顶牌可点。
- 放置位置：相对偏移堆簇，**随机在上方或下方**（禁止左/右并排策略作为主规则）。
- 可与偏移簇**贴边相触**，但**不得与偏移牌重叠**，牌堆之间也**不得互相重叠**。
- 历史：侧置时约 1/5 间隙；现行：**不要盖住**偏移牌。
- 牌堆内部仅 1/10 偏移，**无旋转**。
- 每个牌堆内图标种类：**≥ 3 种不同 ItemType**。

#### 3.2.4 L1 布局特例

- L1：3 层偏移堆，牌数分布 **12 / 8 / 4**（合计 24）。
- L1–L2：**无全遮挡牌堆**。

---

### 3.3 关卡与发牌

#### 3.3.1 总关卡

- 共 **50** 关。
- 免费开放 **L1–L20**；L21–35、L36–50 由关卡包解锁（见商店）。

#### 3.3.2 难度带与规模

| 关卡带 | 图案种类数 | 总牌数 | 全遮挡牌堆 | 单堆厚度（张） | 备注 |
|--------|------------|--------|------------|----------------|------|
| L1–5 | 4 | 24 | L1–2：0；L3–5：2 | L3–5：4 | L1 三层 12/8/4；L1–2 无全堆；L3 起 2 堆×4 |
| L6–10 | 6 | 36 | 2 | 4 | |
| L11–15 | 8 | 48 | 3 | 5 | |
| L16–25 | 10 | 60 | 3 | 5 | |
| L26–40 | 12 | 72 | 4 | 6 | |
| L41–50 | 15 | 90 | 5 | 7 | |

说明（锁定口径）：

- L1–2：**无全遮挡牌堆**。
- 自 L6 起按上表「牌堆数 × 厚度」生成全遮挡牌堆。
- 偏移层数 / 覆盖比例随带升高：早期约 **1/4** 牌被较深覆盖，后期约 **1/2**。
- 类型按 **Add Order** 依次解锁引入；某带「N 种」= 取引入顺序前 N 种。
- **每种类型在一关内恒为 6 张**（总牌数 = 种类数 × 6，可被 3 整除）。

#### 3.3.3 位置抖动（自 L3）

- L1–L2：相对整齐，无独立位置抖动（或抖动≈0）。
- 自 L3 起：每张牌独立位置抖动，按带递增；**种子 = 关卡号**，重开布局不变。
- **邻牌间隙 ≤ 牌宽的 1/8**。
- **无旋转**。

| 关卡带 | maxX（相对牌宽） | maxY（相对牌高） |
|--------|------------------|------------------|
| L3–10 | ~1/4 | ~1/5 |
| L11–25 | ~1/3 | ~1/4 |
| L26–50 | ~2/5 | ~1/3 |

#### 3.3.4 图案洗牌约束（Face Shuffle）

- 洗牌种子 = 关卡号。
- 避免同图标大块聚集（no same-icon clumps）。
- **L1–L2**：开局保证存在**一组已完全暴露的三连**（便于教学）。
- **自 L3**：开局**不得**出现三张相同且全部暴露。
- **自 L31**：顶层相邻牌不得同类型（top-adjacent not same type）。

#### 3.3.5 发牌流程（逻辑顺序）

1. 按关卡带确定种类数 N、总牌数、全堆数与厚度。
2. 取 Add Order 前 N 种，各复制 6 张 → 牌池。
3. 以关卡号种子洗牌，应用聚集/暴露/邻顶约束。
4. 生成偏移堆布局（L1 用 12/8/4）；应用抖动（L3+）。
5. 生成全遮挡牌堆（L6+），随机置于偏移簇上或下，碰撞检测：不重叠。
6. 计算每张牌的 Cover Depth → 灰纱与可点性。

---

### 3.4 道具

#### 3.4.1 通用

- 三道具按钮**始终可见**。
- 未解锁：显示锁定，文案 **Lv.5 / Lv.8 / Lv.12**（对应 Move / Undo / Shuffle）。
- **解锁当关**：赠送该道具 **3 次使用** + **一次手指引导**。
- 解锁之后：每关开局该道具次数重置为 **3**（当前 HTML 原型口径；正式版次数可与本地库存叠加，见进度模块）。
- **当前 HTML 原型：无广告补充道具次数。**

#### 3.4.2 Move（L5 解锁）

- 将托盘**最左侧连续最多 3 张**牌移到托盘上方的 **Hold** 区。
- Hold 上限：**6** 张。
- 点击 Hold 中的牌：若托盘有空位，则该牌回到托盘（入最右空槽，并走入槽后三消检测）。
- 若托盘空位不足或 Hold 已满导致无法执行，则点击无效（可轻提示，无强制弹窗）。

#### 3.4.3 Undo（L8 解锁）

- 撤销：**上一张进入托盘且尚未被三消清除、也未被 Move 移出**的牌，退回其**原始牌桌格子**。
- 不可撤销已消除的三消；不可把已在 Hold 的牌用 Undo 直接回桌（Hold 回托盘用点击 Hold）。

#### 3.4.4 Shuffle（L12 解锁）

- 仅打乱**仍留在牌桌上的牌**的图案（faces），位置/堆叠几何不变。
- 不打乱托盘与 Hold 中的牌。

#### 3.4.5 软辅助（Soft Help）

1. 托盘仅剩 **1** 个空槽时：托盘条变为**橙色**警示。
2. 若检测到牌桌**暂时无解**（无任何可点且可形成进展的步骤）且托盘占用 **≥ 5**：对仍可用的 **Move / Shuffle** 做**一次**闪烁提示（本局每种条件触发一次即可，避免刷屏）。

---

### 3.5 界面流程

#### 3.5.1 启动与主页

- 启动 → **仅进入 Home**，**无关卡地图**。
- Home 要素：
  - 夸张 3D 标题 **Farm Match**
  - 左侧剪纸风农夫：白边描边 + 轻微摇摆
  - 方形主按钮：显示**当前关卡号**
  - 静音开关（Mute toggle）
  - 通关 L50 后：标题下方显示 **Cleared** 徽章

#### 3.5.2 进入关卡

- 点击主按钮 → 进入当前关卡。
- **L1 首次进入**：对一张可点的空闲牌播放**一次**手指引导（finger hint）。

#### 3.5.3 过关

- 播放彩带（confetti）约 **1.5s** + 欢呼音效（cheer SFX）。
- **无英文过关文案**（No English win copy）。
- 自动返回 Home；主按钮变为**下一关**编号。
- **L50**：彩带 → Home；按钮保持 **50**（可重玩）；显示 Cleared。

#### 3.5.4 失败

1. 遗憾音效（regret SFX）
2. 全屏灰色遮罩
3. 遮罩上方对话框文案：**Try this level again?**
4. 按钮：**Try again** / **Home**
5. 关闭任一路径均移除遮罩 + 对话框

#### 3.5.5 局内导航

- **Retry**：重开本关（同种子布局；道具次数按产品规则重置/扣减，见进度）。
- **Home**：回主页，**不记失败**（无 fail）。

#### 3.5.6 输入锁定

- 飞牌、三消聚拢/弹出动画期间锁定输入。

---

### 3.6 音效动效（Juice，美术锁定）

| 效果 | 规格 |
|------|------|
| 飞入托盘 | 弧线飞行 **0.22s** |
| 入槽落地 | 弹性缩放峰值 **1.08**（bounce） |
| 三消 | 先聚拢再弹出，缩放约 **1.4**；**6–8** 个火花，持续 **0.35s** |
| 过关彩带 | 更宽、高饱和、带摇摆（wider saturated sway） |
| 解锁动画 | **已移除**（随关卡地图一并删除） |
| 过关音效 | cheer SFX |
| 失败音效 | regret SFX |

---

### 3.7 本地进度与商店（含关卡包门槛）

#### 3.7.1 身份、绑定与存档

- **默认免登录**：本机随机身份 + 可改昵称；进度先只写本机。
- **绑定时机（锁定）**
  - **第 5 关过关后**：轻提示绑定 **X 或 Facebook**，理由是云存档；**可跳过**，不挡继续玩。
  - **第 8 关（撤回解锁）后**：若仍未绑定，再推绑定；并引导**邀请好友**开启每日互赠道具。文案侧重「送好友道具 / 存进度」，不要硬写「请登录」。
  - **不做**固定第 10 关强制绑定。
- **登录用户（已绑定）**：关卡进度、道具次数、已购权益（关卡包 / 去广告）**云同步到后台**；换机同账号可拉回；卸载后可找回。
- **游客**：仍可只存本机；卸载后进度不找回（与旧规则一致）。
- 后端：在验单 + 远程配置之外，增加 **账号绑定、云存档、好友与互赠** API。

#### 3.7.1b 好友互赠道具

- 需已绑定账号并互为好友（邀请链接 / X / Facebook 关系导入，实现择一，产品要求：能邀请到好友）。
- 每人每天可向好友**赠送同一种道具**（Move / Undo / Shuffle **三选一**，当日固定）最多 **3 次**（按送出计数）。
- 接收：不强制与送出同上限；同一好友同一天最多收 **1 次**，防刷。
- 赠送不扣自己库存以外的「赠送额度」以外资源；具体是「从每日免费赠送池发送」还是「从自己库存扣 1」：默认 **每日免费赠送池**（不扣库存），避免伤付费。
- 失败或道具为 0 时可展示「向好友要一次」，跳转邀请 / 好友列表。

#### 3.7.1c 双向邀请奖励

- **触发**：已绑定用户分享邀请链接 / 码；被邀请人**新安装**并**通关第 1 关**后，双方各得奖励。
- **奖励（锁定）**：双方各得 **Move +2、Undo +2、Shuffle +2**（一次成功邀请结算一次）。
- **归因**：被邀请人首次打开带邀请参数；通关 L1 后写入「已归因」；同一账号 / 同一设备只归因一次。
- **邀请人上限**：每日成功邀请结算最多 **5** 次；终身成功邀请奖励次数上限 **50**（防刷）。
- **被邀请人**：必须是新用户（本机无旧进度，或账号创建 < 24h 且未过 L1）；老号不可冒领。
- **未登录邀请人**：L8 引导绑定后再发有效邀请链接；未绑定分享只给「装好游戏」链，不发双向奖励。
- **文案方向（英文 UI）**：`Invite friends — you both get free boosts`；结算 Toast：`You and {name} got boosts!`

#### 3.7.1d 深链落地（Deep Link）

- **目标**：点分享 / 推送 / 广告链进 App，**不得落空白首页**，要直达意图页。
- **链路类型**
  | 类型 | 示例参数 | 落地 |
  |------|----------|------|
  | 邀请 | `invite?code=` | 首页叠「接受邀请」或自动归因 + 轻提示去打第 1 关 |
  | 赠礼 | `gift?from=&type=` | 打开礼物收件 / Toast 领取，再回首页 |
  | 挑战 | `challenge?level=` | 若关卡已解锁则进该关；未解锁则提示并停在首页方钮 |
  | 商店 | `shop?sku=` | 打开商店对应商品（二期可做） |
- **技术**：iOS Universal Links + Android App Links；自定义 scheme `farmmatch://` 作兜底；未安装走商店 + **延迟深链**（Firebase Dynamic Links / 等价方案，实现选型由研发定，产品要求：装完首次打开仍能带上邀请码）。
- **冷启动顺序**：解析深链 → 游客/登录态就绪 → 再导航；解析失败才进普通首页。
- **网页版**：本版**不做**深链与邀请结算（仅 Flutter `farmMatch-app`）。

#### 3.7.1e 软引导评星

- **触发（满足其一，且未在冷却中）**
  1. **连续过关 ≥ 3**（同次会话或累计连胜，失败打断连胜）；或
  2. 首次通关 **第 10 关**；或
  3. 首次通关 **第 20 关**。
- **形态**：系统原生 In-App Review（iOS StoreKit / Google Play In-App Review）；**不**自做跳商店的强弹窗（除非系统 API 不可用时降级为一次可跳过的「Enjoying Farm Match?」→ Rate / Later）。
- **规则**：可跳过；**失败弹框后不评星**；同一自然日最多展示 1 次；展示过后冷却 **90 天**；用户点过 Rate 后本安装不再主动弹。
- **不与**绑定弹窗、邀请弹窗同一帧叠出：优先绑定 / 邀请，评星延后到下次过关满足条件时。

#### 3.7.2 免费与关卡包门槛

| 区间 | 获取方式 |
|------|----------|
| L1–20 | 免费 |
| L21–35 | IAP **Barn Bundle** $1.99 |
| L36–50 | IAP **Harvest Bundle** $2.99 |

- 未购买时：主按钮若进度停在门槛前，点击锁关应引导购买对应 Bundle（或显示锁定态）；不得通过「单关通关售卖」绕过。
- **不卖单关通关**；**无体力**；**无订阅**。

#### 3.7.3 IAP 商品

| 商品 | 价格 | 内容 |
|------|------|------|
| Prop Pack Small | $2.99 | Move / Undo / Shuffle 各 **5** |
| Prop Pack Mid | $4.99 | 各 **12** |
| Prop Pack Large | $9.99 | 各 **30** |
| Remove Ads | $2.99 | 终身去广告 |
| Barn Bundle | $1.99 | 解锁关卡 **21–35** |
| Harvest Bundle | $2.99 | 解锁关卡 **36–50** |

#### 3.7.4 Restore Purchases

- 仅恢复：**Remove Ads** + **关卡包（Barn / Harvest）**。
- **不恢复**：关卡进度、道具库存、昵称等本地状态。

#### 3.7.5 道具库存与「每关 3 次」（正式产品建议口径）

- HTML 原型：解锁后每关 3 次，无广告补。
- Flutter：本地库存；进入关卡时若库存 > 0，可按「本关可用次数」封顶显示为 3（或直接消耗库存，由实现选定一种并在远程配置开关）。**不得**与本文锁定的解锁关卡冲突。

---

### 3.8 远程配置字段建议

建议下发 JSON（示例字段名，供工程师映射 Param ID）：

```json
{
  "free_level_cap": 20,
  "tray_slots": 7,
  "hold_max": 6,
  "card_scale_vs_l1": 0.85,
  "powers": {
    "move_unlock_level": 5,
    "undo_unlock_level": 8,
    "shuffle_unlock_level": 12,
    "unlock_grant_uses": 3,
    "per_level_uses": 3
  },
  "timing": {
    "fly_seconds": 0.22,
    "tray_bounce_scale": 1.08,
    "match_pop_scale": 1.4,
    "spark_seconds": 0.35,
    "spark_count_min": 6,
    "spark_count_max": 8,
    "confetti_seconds": 1.5
  },
  "veil_opacity_by_depth": { "1": 0.25, "2": 0.35, "3": 0.45, "4_plus": 0.55 },
  "pile_internal_shift_w": 0.1,
  "neighbor_gap_max_w": 0.125,
  "jitter_bands": [
    { "from": 3, "to": 10, "max_x_w": 0.25, "max_y_h": 0.2 },
    { "from": 11, "to": 25, "max_x_w": 0.333, "max_y_h": 0.25 },
    { "from": 26, "to": 50, "max_x_w": 0.4, "max_y_h": 0.333 }
  ],
  "difficulty_bands": [
    { "from": 1, "to": 2, "types": 4, "cards": 24, "piles": 0, "pile_size": 0 },
    { "from": 3, "to": 5, "types": 4, "cards": 24, "piles": 2, "pile_size": 4 },
    { "from": 6, "to": 10, "types": 6, "cards": 36, "piles": 2, "pile_size": 4 },
    { "from": 11, "to": 15, "types": 8, "cards": 48, "piles": 3, "pile_size": 5 },
    { "from": 16, "to": 25, "types": 10, "cards": 60, "piles": 3, "pile_size": 5 },
    { "from": 26, "to": 40, "types": 12, "cards": 72, "piles": 4, "pile_size": 6 },
    { "from": 41, "to": 50, "types": 15, "cards": 90, "piles": 5, "pile_size": 7 }
  ],
  "soft_help": {
    "tray_warn_slots_left": 1,
    "dead_board_tray_min": 5
  },
  "iap": {
    "prop_small": { "price_usd": 2.99, "each": 5 },
    "prop_mid": { "price_usd": 4.99, "each": 12 },
    "prop_large": { "price_usd": 9.99, "each": 30 },
    "remove_ads": { "price_usd": 2.99 },
    "barn_bundle": { "price_usd": 1.99, "levels": [21, 35] },
    "harvest_bundle": { "price_usd": 2.99, "levels": [36, 50] }
  }
}
```

L1–2 无全堆、L1 的 12/8/4 层分布建议作为关卡脚本常量或 `level_overrides`，不强制全部进 remote。

---

## 4. 详细状态机

### 4.1 总览状态

```
AppLaunch → Home
Home → Playing（点主按钮且关卡已解锁）
Playing → Home（过关 / 点 Home / 失败选 Home）
Playing → FailDialog（托盘满无三消）
FailDialog → Playing（Try again）
FailDialog → Home（Home）
```

### 4.2 开局（Enter Level）

1. `InputLock = true`（可选极短，防连点）。
2. 校验关卡解锁（免费 cap / Bundle）。
3. `seed = levelId`，生成牌桌（偏移 + 全堆 + 抖动 + 洗牌约束）。
4. 初始化托盘空、Hold 空；道具次数按规则装载。
5. 计算可点集合；播放入场（若有）。
6. 若 `levelId == 1` 且本地 `l1_finger_hint_shown == false`：播放手指引导一次并标记已示。
7. 若本局刚解锁某道具：发放 3 次 + 手指引导一次。
8. `InputLock = false`；状态 = `Playing`。

### 4.3 点牌（Tap Card）

```
前提: Playing && !InputLock && card.isFullyUncovered
→ InputLock = true
→ 牌从牌桌移除（布局占位不重排）
→ 飞入托盘最右空槽（0.22s 弧线）
→ 落地 bounce 1.08
→ 扫描三消（见 4.4）
→ 若托盘满且无三消 → 转失败（4.5）
→ 若牌桌+托盘+Hold 皆空 → 转过关（4.6）
→ 更新灰纱/可点性
→ Soft Help 检测
→ InputLock = false
```

非法点击（被盖）：忽略。

### 4.4 三消（Match Clear）

```
入槽完成或 Hold 回托盘后：
→ 查找托盘内任意 ItemType 计数 ≥ 3
→ 取该类型 3 张：聚拢 → pop~1.4 + 火花 6–8 / 0.35s
→ 从托盘移除这 3 张
→ 托盘左压
→（实现可选：再次扫描，防极端连锁）
```

### 4.5 失败（Fail）

```
条件: tray.occupied == 7 && 入槽后无三消发生
→ 播放 regret SFX
→ 全屏灰纱 + 对话框 "Try this level again?"
→ Try again: 关对话框与纱 → 同关重新 Enter Level
→ Home: 关对话框与纱 → Home（不记失败统计也可，产品不要求 fail 标记）
```

### 4.6 过关（Win）

```
条件: board empty && tray empty && hold empty
→ confetti ~1.5s + cheer SFX
→ 无 win 英文文案
→ 若 level < 50: currentLevel = level + 1
→ 若 level == 50: currentLevel = 50; clearedBadge = true
→ 回 Home
```

### 4.7 道具子状态

| 道具 | 成功条件 | 效果 |
|------|----------|------|
| Move | 解锁且次数>0 且托盘≥1 且 Hold 空间足够容纳将移动的 1–3 张 | 最左最多 3 张入 Hold；次数−1 |
| Undo | 解锁且次数>0 且存在可撤销的「上张入槽未消除未移出」记录 | 牌回原桌位；次数−1 |
| Shuffle | 解锁且次数>0 且牌桌仍有牌 | 仅打乱桌面 faces；次数−1 |

动画中禁止使用道具（InputLock）。

---

## 5. 数值表

> 每行含 **Param ID**，供工程映射配置 / 远程字段。

### 5.1 难度带：种类与牌数

| Param ID | 关卡 | 种类数 | 总牌数 | 每类型份数 |
|----------|------|--------|--------|------------|
| DIFF_TYPES_L1_5 | L1–5 | 4 | 24 | 6 |
| DIFF_TYPES_L6_10 | L6–10 | 6 | 36 | 6 |
| DIFF_TYPES_L11_15 | L11–15 | 8 | 48 | 6 |
| DIFF_TYPES_L16_25 | L16–25 | 10 | 60 | 6 |
| DIFF_TYPES_L26_40 | L26–40 | 12 | 72 | 6 |
| DIFF_TYPES_L41_50 | L41–50 | 15 | 90 | 6 |
| DIFF_COPIES_PER_TYPE | 全局 | — | — | 6 |
| DIFF_TOTAL_LEVELS | 全局 | — | 50 关 | — |

### 5.2 全遮挡牌堆数量与厚度

| Param ID | 关卡 | 牌堆数 | 单堆张数 | 备注 |
|----------|------|--------|----------|------|
| PILE_COUNT_L1_2 | L1–2 | 0 | 0 | 禁止全堆 |
| PILE_COUNT_L3_5 | L3–5 | 2 | 4 | 与 L6–10 同档厚度 |
| PILE_COUNT_L6_10 | L6–10 | 2 | 4 | |
| PILE_COUNT_L11_15 | L11–15 | 3 | 5 | |
| PILE_COUNT_L16_25 | L16–25 | 3 | 5 | |
| PILE_COUNT_L26_40 | L26–40 | 4 | 6 | |
| PILE_COUNT_L41_50 | L41–50 | 5 | 7 | |
| PILE_MIN_DISTINCT_TYPES | 全局 | — | — | 每堆 ≥3 种不同图标 |
| PILE_INTERNAL_SHIFT | 全局 | — | 0.1 × 牌宽 | 同向微移 |
| PILE_PLACE_AXIS | 全局 | — | 上或下 | 相对偏移簇，禁左右主策略 |
| PILE_NO_OVERLAP | 全局 | — | true | 不与偏移牌/他堆重叠 |

### 5.3 L1 偏移层分布

| Param ID | 值 |
|----------|-----|
| L1_OFFSET_LAYER_COUNTS | 12 / 8 / 4 |
| L1_OFFSET_LAYER_COUNT | 3 |

### 5.4 灰纱不透明度

| Param ID | 被盖层数 | 不透明度 |
|----------|----------|----------|
| VEIL_OPACITY_DEPTH_1 | 1 | 25% |
| VEIL_OPACITY_DEPTH_2 | 2 | 35% |
| VEIL_OPACITY_DEPTH_3 | 3 | 45% |
| VEIL_OPACITY_DEPTH_4_PLUS | 4+ | 55% |

### 5.5 位置抖动 maxX / maxY

| Param ID | 关卡 | maxX | maxY |
|----------|------|------|------|
| JITTER_L1_2 | L1–2 | 0 | 0（整齐） |
| JITTER_L3_10 | L3–10 | ~1/4 牌宽 | ~1/5 牌高 |
| JITTER_L11_25 | L11–25 | ~1/3 牌宽 | ~1/4 牌高 |
| JITTER_L26_50 | L26–50 | ~2/5 牌宽 | ~1/3 牌高 |
| JITTER_NEIGHBOR_GAP_MAX | 全局 | ≤1/8 牌宽 | — |
| JITTER_SEED | 全局 | levelId | 重开不变 |
| ROTATION_DEG | 全局 | 0 | 禁止旋转 |

### 5.6 覆盖比例指引（偏移堆）

| Param ID | 阶段 | 约略深覆盖比例 |
|----------|------|----------------|
| COVER_RATIO_EARLY | 早期带 | ~1/4 |
| COVER_RATIO_LATE | 后期带 | ~1/2 |

### 5.7 托盘与 Hold

| Param ID | 值 |
|----------|-----|
| TRAY_SLOTS | 7 |
| HOLD_MAX | 6 |
| MOVE_TAKE_COUNT | 最多 3（托盘最左） |
| CARD_SCALE_VS_L1 | 0.85 |
| BOARD_MAX_COLUMNS | 5–6 |
| BOARD_SCROLL | 仅纵向 |

### 5.8 道具解锁与次数

| Param ID | 道具 | 解锁关 | 解锁赠送 | 之后每关 |
|----------|------|--------|----------|----------|
| POWER_MOVE_UNLOCK | Move | 5 | 3 + 手指引导 | 3 |
| POWER_UNDO_UNLOCK | Undo | 8 | 3 + 手指引导 | 3 |
| POWER_SHUFFLE_UNLOCK | Shuffle | 12 | 3 + 手指引导 | 3 |
| POWER_UNLOCK_GRANT_USES | — | — | 3 | — |
| POWER_PER_LEVEL_USES | — | — | — | 3 |
| POWER_LOCKED_LABEL_MOVE | UI | Lv.5 | — | — |
| POWER_LOCKED_LABEL_UNDO | UI | Lv.8 | — | — |
| POWER_LOCKED_LABEL_SHUFFLE | UI | Lv.12 | — | — |
| POWER_ADS_REFILL_HTML | HTML 原型 | false | 无广告补次数 | — |

### 5.9 软辅助

| Param ID | 值 |
|----------|-----|
| SOFT_TRAY_WARN_SLOTS_LEFT | 1（托盘条橙色） |
| SOFT_DEAD_BOARD_TRAY_MIN | 5 |
| SOFT_FLASH_POWERS | Move / Shuffle 闪一次 |

### 5.10 发牌约束开关

| Param ID | 关卡条件 | 规则 |
|----------|----------|------|
| DEAL_EXPOSED_TRIPLE_L1_2 | L1–2 | 开局需一组全暴露三连 |
| DEAL_NO_EXPOSED_TRIPLE_FROM_L3 | ≥L3 | 开局禁止三张相同全暴露 |
| DEAL_TOP_ADJ_DIFF_FROM_L31 | ≥L31 | 顶层相邻不同型 |
| DEAL_NO_ICON_CLUMPS | 全局 | 避免同图标团块 |
| DEAL_FACE_SEED | 全局 | levelId |

### 5.11 动效时间与尺度

| Param ID | 值 |
|----------|-----|
| ANIM_FLY_SECONDS | 0.22 |
| ANIM_TRAY_BOUNCE_SCALE | 1.08 |
| ANIM_MATCH_POP_SCALE | ~1.4 |
| ANIM_SPARK_SECONDS | 0.35 |
| ANIM_SPARK_COUNT_MIN | 6 |
| ANIM_SPARK_COUNT_MAX | 8 |
| ANIM_CONFETTI_SECONDS | ~1.5 |
| ANIM_UNLOCK_WITH_MAP | 已移除 |

### 5.12 IAP 与关卡门槛

| Param ID | 商品 / 规则 | 值 |
|----------|-------------|-----|
| IAP_PROP_SMALL_PRICE | Prop Pack Small | $2.99 |
| IAP_PROP_SMALL_EACH | 各道具数量 | 5 |
| IAP_PROP_MID_PRICE | Prop Pack Mid | $4.99 |
| IAP_PROP_MID_EACH | 各道具数量 | 12 |
| IAP_PROP_LARGE_PRICE | Prop Pack Large | $9.99 |
| IAP_PROP_LARGE_EACH | 各道具数量 | 30 |
| IAP_REMOVE_ADS_PRICE | Remove Ads | $2.99 lifetime |
| IAP_BARN_BUNDLE_PRICE | Barn Bundle | $1.99 |
| IAP_BARN_BUNDLE_RANGE | 解锁关卡 | 21–35 |
| IAP_HARVEST_BUNDLE_PRICE | Harvest Bundle | $2.99 |
| IAP_HARVEST_BUNDLE_RANGE | 解锁关卡 | 36–50 |
| IAP_FREE_LEVEL_CAP | 免费最高关 | 20 |
| IAP_NO_STAMINA | — | true |
| IAP_NO_SUBSCRIPTION | — | true |
| IAP_NO_SINGLE_LEVEL_CLEAR_SELL | — | true |
| IAP_RESTORE_SCOPE | Restore | 仅去广告 + 关卡包 |

### 5.13 物品引入顺序（Add Order）

| Param ID | Order | Item |
|----------|-------|------|
| ITEM_ORDER_01 | 1 | scissors |
| ITEM_ORDER_02 | 2 | bucket |
| ITEM_ORDER_03 | 3 | brush |
| ITEM_ORDER_04 | 4 | carrot |
| ITEM_ORDER_05 | 5 | gloves |
| ITEM_ORDER_06 | 6 | corn |
| ITEM_ORDER_07 | 7 | wool |
| ITEM_ORDER_08 | 8 | milk |
| ITEM_ORDER_09 | 9 | hay |
| ITEM_ORDER_10 | 10 | pitchfork |
| ITEM_ORDER_11 | 11 | pumpkin |
| ITEM_ORDER_12 | 12 | apple |
| ITEM_ORDER_13 | 13 | mower |
| ITEM_ORDER_14 | 14 | boots |
| ITEM_ORDER_15 | 15 | watering can |

---


### 5.14 账号绑定与好友互赠

| Param ID | 值 |
|----------|-----|
| AUTH_GUEST_DEFAULT | true |
| AUTH_BIND_PROMPT_LEVEL | 5（过关后轻提示云存档，可跳过） |
| AUTH_INVITE_PROMPT_LEVEL | 8（撤回解锁后推绑定+邀请） |
| AUTH_PROVIDERS | X, Facebook |
| AUTH_CLOUD_SAVE_ON_LOGIN | true |
| SOCIAL_GIFT_DAILY_SEND_MAX | 3 |
| SOCIAL_GIFT_TYPES | move / undo / shuffle（当日三选一） |
| SOCIAL_GIFT_SAME_FRIEND_DAILY_RECV_MAX | 1 |
| SOCIAL_GIFT_FROM_FREE_POOL | true（不扣自己库存） |


### 5.15 邀请、深链与评星

| Param ID | 值 |
|----------|-----|
| INVITE_REWARD_BOTH | move+2, undo+2, shuffle+2 |
| INVITE_REQUIRE_CLEAR_LEVEL | 1 |
| INVITE_INVITER_DAILY_SUCCESS_MAX | 5 |
| INVITE_INVITER_LIFETIME_SUCCESS_MAX | 50 |
| INVITE_REQUIRE_INVITER_BOUND | true |
| INVITE_NEW_USER_MAX_ACCOUNT_AGE_H | 24 |
| DEEPLINK_SCHEME | farmmatch:// |
| DEEPLINK_TYPES | invite / gift / challenge / shop |
| DEEPLINK_DEFERRED | true（未安装保留归因） |
| RATE_PROMPT_STREAK_CLEARS | 3 |
| RATE_PROMPT_LEVELS | 10, 20（各首次） |
| RATE_PROMPT_COOLDOWN_DAYS | 90 |
| RATE_PROMPT_MAX_PER_DAY | 1 |
| RATE_PROMPT_SKIP_ON_FAIL | true |
| RATE_PROMPT_NOT_WITH_AUTH_SHEETS | true |

## 6. 调参指引

### 6.1 提高难度（更难通关）的旋钮

| 旋钮 | Param ID 示例 | 方向 | 副作用 |
|------|---------------|------|--------|
| 增加种类数 | DIFF_TYPES_* | ↑ | 托盘更易堵；可读性下降 |
| 增加全堆数/厚度 | PILE_COUNT_* | ↑ | 信息隐藏↑；前期挫败↑ |
| 提高深覆盖比例 | COVER_RATIO_* | ↑ | 可点面减少 |
| 加大抖动 | JITTER_* | ↑ | 遮挡关系更乱；可能误触感↑ |
| 收紧邻间隙 | JITTER_NEIGHBOR_GAP_MAX | ↓ | 更挤、更难辨认边缘 |
| 推迟道具解锁 | POWER_*_UNLOCK | ↑ | 新手更难 |
| 减少每关道具次数 | POWER_PER_LEVEL_USES | ↓ | 容错↓ |
| 自 L31 邻顶异色等约束 | DEAL_TOP_ADJ_* | 更早启用 | 开局更「闷」 |

### 6.2 提高可读性 / 友好度的旋钮

| 旋钮 | 方向 | 说明 |
|------|------|------|
| 灰纱对比 | VEIL_OPACITY_* 略↑ | 被盖更明显（过浓会脏屏） |
| 抖动 | JITTER_* ↓ | 堆叠更整齐，中老年更友好 |
| L1–2 暴露三连 | DEAL_EXPOSED_TRIPLE_L1_2 | 保持开启，降低首障 |
| 牌缩放 | CARD_SCALE_VS_L1 | 略↑更清晰，但列数可能不够 |
| Soft Help | SOFT_* | 保持：1 槽橙条 + 死局闪道具 |
| 列数上限 | BOARD_MAX_COLUMNS ≤6 | 避免超屏缩放 |

### 6.3 调参安全边界（建议）

- 总牌数必须 = 种类数 × 6，且可被 3 整除（已满足）。
- 全堆不得与偏移牌重叠；调大抖动后必须重跑碰撞。
- 托盘槽位不建议 <7 或 >7（本产品锁定 7）。
- 旋转角锁定 0，禁止用旋转「增加难度」。
- 商店首行文案禁止 match/消除用词。

---

## 7. 版本记录

| 版本 | 日期 | 说明 |
|------|------|------|
| v1.0 | 2026-10-09 | 首版完整功能规格：核心三消、双堆叠、50 关难度带、道具与软辅助、Home 流程、Juice、本地进度与 IAP、远程配置建议与 Param ID 数值表 |
| v1.1 | 2026-10-09 | 默认免登录；L5 过关提示绑 X/Facebook 云存档（可跳过）；L8 再推绑定+邀请好友；登录后云同步；好友每日互赠同种道具最多送出 3 次 |
| v1.2 | 2026-10-09 | 双向邀请奖励（双方道具）；深链落地 invite/gift/challenge；软引导 In-App Review；实现范围仅 farmMatch-app，网页版不做 |

---

**文档结束。** 实现时若与本锁定事实冲突，以本文件为准；新增系统须经产品负责人 birdy 书面确认后再修订版本号。
