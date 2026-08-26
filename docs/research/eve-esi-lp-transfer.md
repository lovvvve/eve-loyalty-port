# EVE Online 官方 ESI 对 Loyalty Points 转账与捐赠的支持情况

> 核查日期：2026-08-26  
> 研究范围：Tranquility 当前官方 ESI OpenAPI、EVE Developers 文档、CCP 官方帮助中心及官方补丁说明。  
> 核心规范来源：[当前官方 ESI OpenAPI](https://esi.evetech.net/meta/openapi.json?compatibility_date=2026-08-18)

## 结论

当前官方 ESI **不支持任何 Loyalty Points（LP）转账、捐赠、税率设置或 LP Store 购买写操作**。

逐项结论如下：

| 场景 | EVE 客户端 | ESI 读取 | ESI 写入 |
| --- | --- | --- | --- |
| 角色个人 LP 余额 | 可查看 | 支持；`GET /characters/{character_id}/loyalty/points`，scope 为 `esi-characters.read_loyalty.v1` | 不适用 |
| 角色个人 LP → 玩家军团 | 官方明确支持，通过 Wallet 操作 | 只能读取角色余额，不能读取该次捐赠指令 | **不支持** |
| 角色个人 LP → NPC 军团 | 官方明确不允许 | 不适用 | **不支持** |
| 玩家军团持有的 LP 余额 | 客户端存在 Corporation LP 账户及消费流程 | **无接口** | 不适用 |
| 玩家军团使用 LP Store | 官方补丁证明客户端可使用 Corporation Accounts 消费 | 只能读取公开报价，不能读取军团 LP 余额 | **不支持购买** |
| 玩家军团 LP → 另一玩家军团 | 未找到官方支持依据；官方只明确了个人 LP 捐给玩家军团 | **无接口** | **不支持** |
| NPC 军团 LP Store 报价 | 可查看 | 支持；公开 `GET /loyalty/stores/{corporation_id}/offers` | **无购买接口** |

因此，如果需求是通过第三方应用自动完成“军团之间的 LP 转账／捐赠”，当前 ESI 无法实现。应用最多可以读取角色个人 LP、读取公开 LP Store 报价并生成操作建议，最终捐赠或消费仍须在 EVE 客户端内由玩家操作。

## “军团”的两种含义

LP 语境中的 `corporation` 有两种不同角色，不能混用。

### NPC loyalty-point corporation：LP 发行方与商店所属方

角色通过任务、Factional Warfare、Incursion 等活动获得由特定 NPC corporation 发放的 LP；这些 LP 在对应发行方的 LP Store 中消费。CCP 帮助中心明确说明 LP 来自 NPC corporations，并用于授予该 LP 的 corporation 的商店：

- [CCP 帮助中心：Loyalty Points](https://support.eveonline.com/hc/en-us/articles/14141831188636-Loyalty-Points)
- [CCP 帮助中心：Currencies](https://support.eveonline.com/hc/en-us/articles/14216227951388-Currencies)

ESI 个人 LP 响应中的 `corporation_id` 表示这种 **LP 发行方**。当前 OpenAPI 将响应定义为由下列字段组成的数组：

```json
{
  "corporation_id": 1000035,
  "loyalty_points": 12345
}
```

它不表示“该玩家军团拥有多少 LP”，也不是玩家军团 LP 钱包的 ID。规范见：[当前官方 ESI OpenAPI](https://esi.evetech.net/meta/openapi.json?compatibility_date=2026-08-18)。

### Player-owned corporation：捐赠接收方与 Corporation LP 持有方

官方 Version 21.05 补丁说明的 **2023-08-01.1** 条目明确写明：

- 玩家可以把 LP 捐给任意 player-owned corporation；
- LP 捐赠功能可通过 Wallet 使用；
- 玩家不能把 LP 捐给 NPC Corporations。

这描述的是：

```text
角色个人持有的、由某 NPC corporation 发行的 LP
    → player-owned corporation 持有的相同 LP 币种
```

它不是向 NPC 发行方“捐回”LP，也不是玩家军团之间的 LP 转账。来源：[Patch Notes – Version 21.05](https://www.eveonline.com/news/view/patch-notes-version-21-05)。

## 当前 ESI 的直接 LP 端点

对 [当前官方 ESI OpenAPI](https://esi.evetech.net/meta/openapi.json?compatibility_date=2026-08-18) 的 `paths`、HTTP 方法、`security` 和 OAuth scope 清单逐项核查后，Loyalty 分类只有两个操作，且均为 `GET`。

### 读取角色个人 LP

```http
GET /characters/{character_id}/loyalty/points
```

用途：返回角色为各 LP 发行 corporation 持有的 LP 数量。

OAuth：

```text
esi-characters.read_loyalty.v1
```

该路径没有 `POST`、`PUT`、`PATCH` 或 `DELETE` 方法，因而不能用于：

- 向玩家军团捐 LP；
- 向其他角色转 LP；
- 设置 LP 税；
- 代替玩家从 LP Store 消费。

官方链接：

- [当前 ESI OpenAPI](https://esi.evetech.net/meta/openapi.json?compatibility_date=2026-08-18)
- [EVE Developers API Explorer](https://developers.eveonline.com/api-explorer)

### 读取 LP Store 报价

```http
GET /loyalty/stores/{corporation_id}/offers
```

用途：读取指定 LP Store corporation 的报价。该端点是公开读取接口，OpenAPI 未声明 OAuth security requirement。

这里的 `corporation_id` 是商店所属、发行 LP 的 corporation；它不是接收个人捐赠的 player-owned corporation。该路径同样只有 `GET`，不存在接受报价或购买商品的写方法。

官方链接：

- [当前 ESI OpenAPI](https://esi.evetech.net/meta/openapi.json?compatibility_date=2026-08-18)
- [EVE Developers API Explorer](https://developers.eveonline.com/api-explorer)

## OAuth scopes 核查

当前 OpenAPI 的 OAuth scope 清单中，与 loyalty 直接相关的 scope 只有：

```text
esi-characters.read_loyalty.v1
```

不存在以下或同类 scope：

```text
write_loyalty
manage_loyalty
read_corporation_loyalty
write_corporation_loyalty
transfer_loyalty
```

OAuth scope 只能授权 OpenAPI 已公开的操作，不能用 CEO、Director、Accountant 等客户端角色权限推导出一个未公开的 ESI 军团 LP 接口。

官方来源：

- [当前 ESI OpenAPI](https://esi.evetech.net/meta/openapi.json?compatibility_date=2026-08-18)
- [EVE Developers：Single Sign-On](https://developers.eveonline.com/docs/services/sso/)
- [EVE Developers：ESI Overview](https://developers.eveonline.com/docs/services/esi/overview/)

## 玩家军团持有 LP 与客户端操作

官方补丁说明证明 player-owned corporation 可以持有和使用 LP：

- **2023-06-22.1**：LP tax 与 NPC Bounty tax 可以分别设置；
- LP Store 中存在 `Use Corporation Accounts` 选项；
- 使用 Corporation LP 购买的物品会送至 `Corporation Deliveries`；
- LP Store 可按 corporation 可负担的报价过滤。

来源：[Patch Notes – Version 21.05](https://www.eveonline.com/news/view/patch-notes-version-21-05)。

但是，当前 ESI 没有任何对应的军团 LP 路径，例如不存在：

```http
GET  /corporations/{corporation_id}/loyalty/points
POST /corporations/{corporation_id}/loyalty/transfers
POST /characters/{character_id}/loyalty/donations
```

因此不能通过 ESI：

- 读取 player-owned corporation 的各发行方 LP 余额；
- 读取 LP 税归集流水；
- 设置 LP tax；
- 使用 Corporation LP 购买 LP Store 商品；
- 把个人 LP 捐给玩家军团；
- 把一个玩家军团持有的 LP 转给另一玩家军团。

## 玩家军团之间是否能在客户端转 LP

官方补丁明确支持的是 **Personal Donations**：玩家把个人 LP 捐给任意 player-owned corporation。相同补丁中，CCP 对 corporation-to-corporation 转移明确描述的是联盟范围内的 **EverMarks** 转移，而没有把该能力扩展到 Corporation LP。

因此，依据现有官方资料，不能把“个人可以向任意玩家军团捐 LP”推导为“玩家军团可以把已持有的 LP 转给另一玩家军团”。本研究未找到官方帮助中心、补丁说明或开发者文档证明后者存在，应按 **无官方支持依据／不支持直接军团间 LP 转账** 处理。

即使未来客户端存在额外受权限限制的操作，只要当前 ESI OpenAPI 没有相应路径和 scope，第三方应用仍不能通过 ESI 执行该操作。

## 不是 LP 转账接口的间接端点

### Character/Corporation Wallet journal

ESI 提供角色及军团 Wallet journal 读取接口，例如：

```http
GET /characters/{character_id}/wallet/journal
GET /corporations/{corporation_id}/wallets/{division}/journal
```

所需 scopes 分别为：

```text
esi-wallet.read_character_wallet.v1
esi-wallet.read_corporation_wallets.v1
```

这些是 Wallet journal 只读接口，不是 LP 余额或 LP 捐赠接口。CCP 帮助中心也把 LP Store/LP balance 与普通 Wallet transactions、Corporation Wallet divisions 分开描述：[The Wallet](https://support.eveonline.com/hc/en-us/articles/14142293829916-The-Wallet)。

### Contracts、Market 与 Assets

这些接口至多用于观察 Corporation LP 消费后形成的物品、合同、市场记录或资产。它们不会转移 LP，也不能在 LP Store 接受报价。

因此，“先用 LP 买物品，再转移物品或 ISK”只是转移 LP 的经济价值，不是 LP 转账。

规范来源：[当前官方 ESI OpenAPI](https://esi.evetech.net/meta/openapi.json?compatibility_date=2026-08-18)。

### ESI UI helper

当前 OpenAPI 的 UI 写操作只有：

- 设置 autopilot waypoint：`esi-ui.write_waypoint.v1`；
- 打开 Contract、Information、Market Details 或 New Mail 窗口：`esi-ui.open_window.v1`。

没有打开 LP 捐赠对话框、提交 LP 转账、设置 LP tax 或确认 LP Store 购买的 UI 端点。`open_window` 也只负责打开有限的客户端窗口，不会代替玩家确认交易。

规范来源：[当前官方 ESI OpenAPI](https://esi.evetech.net/meta/openapi.json?compatibility_date=2026-08-18)。

## 可行替代方案

### 1. ESI 读取与规划，客户端手动捐赠

第三方应用可以：

1. 使用 `esi-characters.read_loyalty.v1` 读取角色个人 LP；
2. 使用公开 LP Store offers 接口计算报价或估值；
3. 向用户展示 LP 发行方、数量和目标 player-owned corporation；
4. 由用户在 EVE 客户端 Wallet 中完成个人 LP 捐赠。

这是最接近自动化需求、同时与官方接口能力一致的方案。

### 2. 让角色直接捐给最终目标玩家军团

如果业务流程原计划为：

```text
角色 → 中间玩家军团 → 最终玩家军团
```

可以避开没有官方依据的军团间 LP 转账，直接使用客户端已明确支持的：

```text
角色个人 LP → 最终目标 player-owned corporation
```

### 3. 消费 Corporation LP 后转移物品或 ISK

拥有适当客户端权限的人员可以在 LP Store 中选择 Corporation Accounts 消费，所得物品进入 Corporation Deliveries，之后再通过物品、合同、市场或 ISK 结算价值。

限制是：

- 转移对象是物品或 ISK，不是 LP；
- LP Store 购买没有 ESI 写接口；
- 不能把该流程描述为 ESI LP 转账。

## 最终判定

当前官方 ESI 的 LP 能力只有：

1. 经 OAuth 读取角色个人、按 NPC 发行 corporation 区分的 LP 余额；
2. 公开读取 NPC corporation 的 LP Store 报价。

当前 ESI 没有 player-owned corporation LP 余额读取接口，也没有 LP 捐赠、转账、税率设置或 LP Store 购买写接口。客户端明确支持的是 **角色个人 LP 捐给玩家军团**；没有官方资料证明 **玩家军团持有的 LP 可以直接转给另一玩家军团**。

因此，“通过官方 ESI 实现军团之间 LP 转账／捐赠”的结论为：**不支持**。
