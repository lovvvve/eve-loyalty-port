# EVE LP 数据能力研究

## 结论摘要

截至本次核验所列的最新 ESI compatibility date `2026-08-18`，当前官方 OpenAPI **没有**玩家军团 LP 余额 endpoint，也没有军团间 LP 转账日志或执行转账 endpoint。平台因此不能通过 ESI 自动取得主军团按发行方的实际余额，也不能取得某次 LP 转账的发送方、接收方、发行方、数量和时间。

成员贡献存在一条受控的“可推导”路径：为每个 LP 发行方单独建立 `earn_loyalty_point` Corporation Project，再通过 Corporation Projects ESI 读取各 `character_id` 的累计贡献。它能证明“该项目记录了该角色累计赚取多少指定发行方 LP”，但不是 LP 税账，也没有逐笔时间、税额或军团实际入账字段；只有在项目覆盖完整、每个发行方独立建项目、税率期间受控等运营前提下，才能作为成员应税 LP 的推导输入，不能冒充游戏内税收流水。

官方明确支持的 LP 相关数据主要是：授权角色个人的当前 LP 余额、Corporation Project 的配置和累计贡献、玩家军团当前 LP 税率，以及公开 LP 商店报价。后续账本应把 ESI 推导数据、游戏内余额/转账人工凭证和平台账本分层保存，不应把任何平台生成记录伪装成 ESI 交易。

## 核验口径与来源

- 先从官方 [`/meta/compatibility-dates`](https://esi.evetech.net/meta/compatibility-dates) 取得 compatibility dates，并以核验时最新的 `2026-08-18` 请求官方 [`/meta/openapi.json`](https://esi.evetech.net/meta/openapi.json)（请求头 `X-Compatibility-Date: 2026-08-18`），逐一搜索 `loyalty`、`loyalty_points`、LP tax、Corporation Projects 及所有 corporation paths。
- CCP 已将 ESI 从 Swagger 迁移到 OpenAPI，并使用 `X-Compatibility-Date`；当前 endpoint 与字段应以官方 API Explorer/OpenAPI 为准。参见 CCP 的 [Corporation Projects early-access 说明](https://developers.eveonline.com/blog/early-access-corporation-projects)。
- 当前规范中 Loyalty tag 仍只有角色余额和 LP 商店报价两类路径；Corporation Projects 属于独立 tag。没有任何 `/corporations/{corporation_id}/loyalty/...` 路径。
- CCP 官方博客明确说明从 compatibility date `2026-07-21` 起，公开的 corporation detail 增加 LP 税率；精确的当前字段形状以 [API Explorer 的 corporation detail](https://developers.eveonline.com/api-explorer?compatibility_date=2026-08-18#/operations/GetCorporationsCorporationId) 为准，即 `tax_rates.loyalty_point`。参见 [A splash of color: Corporation Palette and a few fresh fields](https://developers.eveonline.com/blog/a-splash-of-color-corporation-palette-and-a-few-fresh-fields)。
- 官方补丁说明至少证明客户端中存在 “LP Corp Wallet” 和 “Exchange LP” 窗口；这说明游戏内可以形成人工凭证，但不等于 ESI 提供相同数据。参见 [Patch Notes - Version 23.02](https://www.eveonline.com/news/view/patch-notes-version-23-02)。

> “endpoint 不存在”是对上述最新官方 OpenAPI 的否定性核验结果，不是从旧版第三方 SDK、论坛或玩家经验推断。

## 数据能力矩阵

| 业务事实 | 官方状态 | 当前官方接口与字段 | 授权、角色、缓存、分页、历史 | 可用于什么 | 关键缺口与决策 |
|---|---|---|---|---|---|
| 玩家军团当前 LP 税率 | **官方明确支持** | [`GET /corporations/{corporation_id}`](https://developers.eveonline.com/api-explorer?compatibility_date=2026-08-18#/operations/GetCorporationsCorporationId)，`tax_rates.loyalty_point` | 公开 GET；无 OAuth scope、无军团角色；TTL 3600 秒；无分页；只返回当前值，无税率历史 | 校验主军团当前是否配置为 100% LP 税 | 不能证明某笔 LP 产生时的税率；平台必须保存带生效时间的税率快照/管理员确认 |
| 授权角色按 NPC LP 发行军团的当前个人余额 | **官方明确支持** | [`GET /characters/{character_id}/loyalty/points`](https://developers.eveonline.com/api-explorer?compatibility_date=2026-08-18#/operations/GetCharactersCharacterIdLoyaltyPoints)，每项只有 `corporation_id`、`loyalty_points` | scope `esi-characters.read_loyalty.v1`；不要求军团角色；服务端/客户端 TTL 3600 秒，支持 `ETag`、`Last-Modified`；无分页、无历史窗口 | 查看该授权角色当前个人余额；稳定快照键为 `(character_id, corporation_id, observed_at)` | 不是玩家主军团钱包；没有活动、税额、发生时间、事件 ID；余额差会被赚取、兑换、消费等共同影响，不能作为审计级税流水 |
| Corporation Project 的 `earn_loyalty_point` 配置 | **官方明确支持** | [`GET /corporations/{corporation_id}/projects/{project_id}`](https://developers.eveonline.com/api-explorer?compatibility_date=2026-08-18#/operations/GetCorporationsProjectsDetail)；`configuration.earn_loyalty_point.corporations[]` 指定 LP 发行军团；项目 `id` 为 UUID | scope `esi-corporations.read_projects.v1`；详情是 event-based cache，客户端 TTL 60 秒；无分页；项目 `last_modified` 会因贡献变化而更新 | 证明某个项目要追踪哪些 LP 发行方 | 若一个项目配置多个发行方，贡献值没有按发行方拆分；后续应强制“一发行方一项目” |
| 项目内全部成员的累计 LP 贡献 | **官方明确支持项目累计值；用于应税 LP 归因属于可推导** | [`GET /corporations/{corporation_id}/projects/{project_id}/contributors`](https://developers.eveonline.com/api-explorer?compatibility_date=2026-08-18#/operations/GetCorporationsProjectsContributors)，每项 `id`（`character_id`）、`name`、`contributed` | scope `esi-corporations.read_projects.v1`；读取全部贡献者明确要求军团角色 `Project_Manager`；event-based cache；cursor 分页，`after`/`before` 互斥，`limit` 默认 10、范围 10–100 | 在专用单发行方项目中，把累计 `contributed` 归到稳定的 `character_id`；相邻快照差可推导一段时间内新增赚取 LP | 这是项目进度，不是 LP 税入账；没有逐笔时间、税额、税后军团入账或事件 ID。项目覆盖、启停、成员状态和税率变化都可能破坏等价关系 |
| 授权角色自己的项目累计贡献 | **官方明确支持项目累计值** | [`GET /corporations/{corporation_id}/projects/{project_id}/contribution/{character_id}`](https://developers.eveonline.com/api-explorer?compatibility_date=2026-08-18#/operations/GetCorporationsProjectsContribution)，`contributed`，可有 `last_modified` | scope `esi-corporations.read_projects.v1`；规范摘要为 “your contribution”，未列军团角色；event-based cache，客户端 TTL 60 秒；无分页 | 成员自查/交叉核对其项目累计值 | 不应据此假定可读取任意角色；批量归因应使用 `contributors` 并授权 `Project_Manager` |
| 项目清单和关闭项目 | **官方明确支持，但未承诺账务历史保留期** | [`GET /corporations/{corporation_id}/projects`](https://developers.eveonline.com/api-explorer?compatibility_date=2026-08-18#/operations/GetCorporationsProjectsListing)，`state=All|Active` | scope `esi-corporations.read_projects.v1`；event-based cache；cursor 分页，`limit` 10–100；`state=All` 可请求非活跃项目 | 发现和归档项目 UUID、状态及配置 | 规范没有承诺关闭/删除项目保留多久，也没有逐笔 contribution history；平台必须自行连续快照，不能把 `state=All` 当永久历史保证 |
| 成员角色产生且实际被主军团征收的逐笔 LP/税额 | **ESI 不支持；受控条件下只能推导累计值或区间差额** | 无 LP tax journal。可组合专用项目累计贡献、平台保存的税率生效记录和成员绑定历史 | 没有可声明的税流水 scope、军团角色、缓存、分页或历史窗口 | 估计成员按发行方的新增应税 LP，作为待确认账本输入 | 必须标记 `derived`；不得声称是游戏内税入账。异常期、项目遗漏、跨发行方项目、税率变更需人工调整/确认 |
| 主军团按 LP 发行方的实际 LP 余额 | **当前 ESI 无法自动获得** | 最新 OpenAPI 中没有 corporation LP balance endpoint | 因 endpoint 不存在，scope、军团角色、缓存、分页、历史窗口均不存在 | 无 | 从客户端 LP Corp Wallet 采集带时间的余额截图/屏幕录制或人工录入；用于总量对账，不替代成员明细 |
| 军团间 LP 转账明细：发送方、接收方、发行方、数量、时间 | **当前 ESI 无法自动获得** | 最新 OpenAPI 中没有 corporation LP transfer journal；也没有 ESI LP transfer POST | 因 endpoint 不存在，scope、军团角色、缓存、分页、历史窗口均不存在 | 无 | 每次执行后提交游戏内 Exchange LP 结果/钱包变化凭证并由第二人复核；平台生成自己的 transfer UUID，状态标记为人工证据 |
| LP 商店报价 | **官方明确支持，但与本研究账务事实无关** | [`GET /loyalty/stores/{corporation_id}/offers`](https://developers.eveonline.com/api-explorer?compatibility_date=2026-08-18#/operations/GetLoyaltyStoresCorporationIdOffers)，`offer_id`、`type_id`、`quantity`、`lp_cost`、`isk_cost`、`required_items` | 公开、无 scope/角色；官方说明每日 11:05 UTC 过期；支持条件请求；无分页/历史 | 如未来需要展示兑换目录 | `offer_id`/`type_id` 不是税或转账匹配键，不能用于本平台账本关联 |

## 对成员应税 LP 的可行推导方案

### 推荐的受控方案

1. 主军团按每个 LP 发行方建立一个长期专用 `earn_loyalty_point` Corporation Project，禁止一个项目混合多个发行方。
2. 保存稳定键：`main_corporation_id + project_id(UUID) + issuer_corporation_id + character_id`。
3. 使用具有 `Project_Manager` 角色的服务角色授权 `esi-corporations.read_projects.v1`，完整遍历 contributors cursor；保存原始响应、游标、抓取时间、`ETag`/`Last-Modified`（若响应提供）及项目配置快照。
4. 每次同步以累计 `contributed` 减去上一有效快照，得到区间新增值；不要把项目 `last_modified` 当作每名成员的赚取时间。
5. 将区间新增值与平台中的角色—成员绑定历史、税率生效记录组合。只有在该区间项目持续有效、角色持续属于主军团、发行方唯一且 LP 税率已确认时，才生成 `derived` 权益输入。
6. 任何累计值回退、项目关闭/删除、角色换团、授权中断、分页未完成或税率不确定，都进入异常队列，禁止静默结算。

### 为什么仍然不是自动税账

- `contributed` 的官方语义是 Corporation Project 的贡献进度，不是玩家军团 LP 钱包 credit。
- contributors 只给累计数，没有贡献事件 ID、发生时间、税前/税后拆分或税率。
- `tax_rates.loyalty_point` 只给当前税率；ESI 不提供税率历史。
- 即使当前税率为 100%，也只能在运营约束成立时推断“项目记录的新增赚取 LP 应全部被征收”；不能从 ESI 证明主军团实际收到同样数量。

因此，建议把该数据用于**成员归因层**，把客户端主军团余额人工快照用于**总量控制层**，把每次人工转账凭证用于**出账层**，三层分别对账。

## 稳定匹配键与禁止误用

### 可用键

- EVE 实体：`character_id`、玩家主军团 `corporation_id`、LP 发行军团 `corporation_id`。
- Corporation Project：官方 UUID `project_id`。
- 项目累计归因：`(main_corporation_id, project_id, issuer_corporation_id, character_id, snapshot_at)`。
- 角色个人 LP 余额快照：`(character_id, issuer_corporation_id, snapshot_at)`。
- 当前税率快照：`(main_corporation_id, observed_at)`。

### 没有官方键的事实

税收事件和军团间 LP 转账都没有 ESI event/transaction ID。平台必须生成自己的不可变 UUID，并保存证据哈希、提交人、复核人和证据中的游戏内时间。发送方/接收方/发行方/数量/时间组成的复合值只能用于疑似重复提示，不能宣称是官方唯一键。

### 禁止误用

- 不要把 character wallet journal/transaction ID 当作 LP 税或军团 LP 转账 ID；当前 LP 规范没有建立这种关联。
- 不要把 `offer_id`、物品 `type_id`、项目 `last_modified` 或余额差额当作转账 ID。
- 不要把角色个人 LP `corporation_id` 误读为持有余额的玩家主军团；它表示角色 LP 余额所属的 NPC LP 发行军团。

## 缓存、分页和历史窗口实施注意事项

- 角色个人 LP 余额：最多按 3600 秒缓存节奏设计，使用条件请求；没有历史，平台若需要趋势必须自存快照。
- 当前军团 LP 税率：3600 秒 TTL；必须自存生效快照，官方没有历史 endpoint。
- Project detail 与个人 contribution：event-based server cache，客户端 TTL 60 秒。
- Project listing 与 contributors：event-based cache、cursor 分页；严格跟随响应 cursor，`after` 与 `before` 不可同时使用，`limit` 10–100。只有完整遍历后才能提交一轮快照。
- `state=All` 能请求所有状态的项目，但官方规范未给永久保留窗口；上线日开始持续归档，不依赖事后重建。
- 不存在的 corporation LP balance/transfer endpoints 没有任何缓存、分页或历史参数；产品规格中不得虚构。

## 人工证据与导入降级

### 主军团余额快照

建议由有游戏内权限的操作员按固定频率打开 LP Corp Wallet，逐发行方采集：

- 主军团 `corporation_id`；
- LP 发行方 `corporation_id`/名称；
- 显示余额；
- 客户端可见时间和平台采集时间；
- 原始截图或连续屏幕录制、文件 SHA-256；
- 提交人及复核人。

如客户端当时提供复制/导出能力，可接受原始导出文件，但必须保留原文件和字段映射版本；不要假定官方存在固定 CSV schema。

### LP 转账执行凭证

每笔平台申请在人工执行后至少记录：

- 平台 `transfer_id`；
- 发送方主军团 `corporation_id`；
- 接收方成员结算公司 `corporation_id`；
- LP 发行方 `corporation_id`；
- 数量；
- 游戏内显示的发生时间（若不可见则明确标记为操作员声明时间）；
- 执行前后相关余额或 Exchange LP 确认/结果画面；
- 执行人、复核人、证据哈希和上传时间。

证据状态至少区分 `manual_unverified`、`manual_verified`、`derived`，不能标记为 `esi_verified`。

### 期初与中断补录

- 上线时按发行方导入经双人确认的主军团余额和成员权益期初数。
- Project/ESI 授权中断期间，允许导入“成员—角色—发行方—数量—区间”的人工汇总，保留原始凭证和调整原因。
- 恢复后只从最后一个完整快照继续计算；不得用不完整分页或跨缺口余额差静默补账。

## 对后续账本与对账规格的直接建议

1. **平台账本是事实主线，ESI 是输入/校验来源，不是完整 LP 总账。**
2. 每条权益增加记录必须带 `source_kind`：`corp_project_derived`、`manual_import` 或 `admin_adjustment`；每条转出记录只能是人工执行并附证据。
3. 不同 LP 发行方严格分账；Corporation Project 也必须一发行方一项目。
4. 对账至少包括：项目累计贡献变化对成员权益增加、主军团人工余额快照对总账库存、人工转账凭证对已执行申请。
5. 差异不能靠覆盖余额消失，必须生成带原因、证据和审批人的调整分录。
6. 把“当前税率为 100%”“项目贡献累计值”“主军团实际余额”“转账已执行”设计成四类不同事实，避免在领域模型中合并。

## 最终判定

- **官方明确支持**：当前军团 LP 税率；授权角色个人当前 LP 余额；Corporation Project 的 LP 发行方配置、累计贡献和贡献者；公开 LP 商店报价。
- **可合理组合推导**：在“一发行方一项目”、完整快照、明确角色绑定和税率生效记录的受控条件下，按成员/角色推导某区间新增应税 LP；结论必须标记为 `derived`。
- **无法通过当前 ESI 自动获得**：主军团按 LP 发行方的实际余额；逐笔 LP 税收流水；军团间 LP 转账日志及 sender/recipient/issuer/amount/time；任何可用于这些事件的官方稳定 transaction ID。
- **必要降级**：客户端原始截图/录屏、条件性原始导出、双人复核、管理员确认和带证据哈希的人工导入。
