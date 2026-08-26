# EVE SSO 与身份授权能力边界研究

## 1. 研究范围与来源约束

本文回答单一主军团私有 LP 转账信息管理平台的身份与授权能力边界。结论只依据以下第一方资料：

- [EVE Developers：EVE SSO](https://developers.eveonline.com/docs/services/sso/)
- [EVE Developers：ESI](https://developers.eveonline.com/docs/services/esi/)
- [EVE Developers：API Explorer](https://developers.eveonline.com/api-explorer)
- [官方 ESI OpenAPI 文档](https://esi.evetech.net/meta/openapi.json)
- [EVE SSO OAuth Authorization Server Metadata](https://login.eveonline.com/.well-known/oauth-authorization-server)
- [EVE SSO JWKS](https://login.eveonline.com/oauth/jwks)
- [RFC 6749：OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749)
- [RFC 7636：PKCE](https://www.rfc-editor.org/rfc/rfc7636)
- [RFC 7009：Token Revocation](https://www.rfc-editor.org/rfc/rfc7009)
- [RFC 7519：JWT](https://www.rfc-editor.org/rfc/rfc7519)
- [RFC 8725：JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725)

所有 ESI endpoint、scope 和响应字段应以部署时获取的官方 OpenAPI 文档为最终机器可读依据；本文按官方 OpenAPI `2025-01-01` compatibility date 的路径名称与扩展字段复核，不把第三方 SDK 的命名当作依据。部署时仍须按应用实际采用的 compatibility date 重新核对。

## 2. 结论摘要

1. **SSO 能证明当前浏览器操作者成功授权了一个具体 EVE 角色**，稳定主键是 JWT `sub` 中的角色身份（形如 `CHARACTER:EVE:<character_id>`）。它不能直接证明现实世界中的“真人”，也没有 API 枚举同一真人跨账号控制的全部角色。[EVE SSO](https://developers.eveonline.com/docs/services/sso/)；[RFC 7519 §4.1.2](https://www.rfc-editor.org/rfc/rfc7519#section-4.1.2)
2. **角色当前军团是公开游戏事实**：`GET /characters/{character_id}/` 返回 `corporation_id`，无需 ESI scope。平台可将其与配置的主军团 ID 比较，但应定期重查，不能把首次登录结果永久缓存。[官方 ESI OpenAPI](https://esi.evetech.net/meta/openapi.json)
3. **角色自己的当前军团角色与头衔可分别查询**：`GET /characters/{character_id}/roles/` 使用 `esi-characters.read_corporation_roles.v1`；`GET /characters/{character_id}/titles/` 使用 `esi-characters.read_titles.v1`。前者返回 EVE 军团角色集合，后者返回军团授予该角色的头衔；二者都不等于平台中的财务、执行者或审计业务角色。[官方 ESI OpenAPI](https://esi.evetech.net/meta/openapi.json)
4. **军团全体成员与角色是更高权限的军团级读取**：`GET /corporations/{corporation_id}/members/` 和 `GET /corporations/{corporation_id}/roles/` 使用 `esi-corporations.read_corporation_membership.v1`；除 scope 外，授权角色必须属于目标军团，且官方 OpenAPI 明确要求 `Director`。scope 只是用户同意，不会绕过游戏内角色校验。[官方 ESI OpenAPI](https://esi.evetech.net/meta/openapi.json)
5. **无法可靠证明“一个真人控制某成员结算公司”这一完整业务命题**。可验证的最强直接事实通常是：已授权角色当前是该公司的 `ceo_id`，或该角色在该公司具有 `Director` 等军团角色。前者证明“这个已授权角色是当前 CEO”，后者证明“这个角色具有相应游戏权限”；两者都不能证明现实真人身份、跨账号同一人、唯一控制或最终受益所有权。
6. **成员、财务、执行者、审计都是平台业务角色**。EVE 事实只能作为授予、保持或复核这些角色的前置条件/证据；最终平台权限必须由平台策略或管理员授予并审计，不能把同名或近似 EVE 军团角色直接映射成平台权限。
7. 对单一主军团私有平台，最小授权基线是：**登录只做 SSO；用公开角色信息检查主军团归属；只有确实需要读取某角色军团角色或头衔时，才分别增量请求 `esi-characters.read_corporation_roles.v1` 或 `esi-characters.read_titles.v1`**。不要默认请求军团全体成员/角色 scope，更不要为登录捆绑钱包等无关 scope。

## 3. 能验证什么，不能验证什么

### 3.1 登录角色所有权

Authorization Code 流程完成后，平台取得由 EVE SSO 签发、面向本应用的 token。成功验证 JWT 后，可把 `sub` 对应的 `character_id` 视为“本次授权所代表的角色”。这证明的是**对该角色完成 EVE SSO 授权的能力**，不是身份证意义上的真人身份。[EVE SSO](https://developers.eveonline.com/docs/services/sso/)；[RFC 6749 §4.1](https://www.rfc-editor.org/rfc/rfc6749#section-4.1)

JWT 中的 `owner`/角色所有者哈希可用于识别角色所有者变化（例如角色转移后重新核验绑定）。它不应被扩大解释成真人 ID，也不能可靠合并一个真人持有的多个 EVE 账号。平台应保存角色 ID，并在采用该 claim 时保存最后验证的 owner 值；owner 变化应触发解绑、冻结或人工复核，而不是静默继承历史权限。[EVE SSO](https://developers.eveonline.com/docs/services/sso/)

因此：

- 一个角色必须通过自己的 SSO 授权才能建立“平台身份—EVE 角色”绑定；
- 多角色归集为一个“成员”是平台关系，不能由 ESI 自动完备推导；
- 跨账号角色属于同一真人，只能依赖分别登录后的平台绑定流程及必要的管理员复核；
- 角色转让、离开军团或 token 被撤销后，历史绑定不能继续被视为当前事实。

### 3.2 角色当前所属军团

`GET /characters/{character_id}/` 是公开角色信息 endpoint，响应包含当前 `corporation_id`，不需要 ESI scope。平台可在登录、敏感操作和周期复核时读取它，并与主军团 ID 比较。[官方 ESI OpenAPI](https://esi.evetech.net/meta/openapi.json)

可选的公开批量接口 `POST /characters/affiliation/` 可返回多个角色当前的 `corporation_id`、`alliance_id`、`faction_id`，也不替代对每个绑定角色的 SSO 所有权证明。[官方 ESI OpenAPI](https://esi.evetech.net/meta/openapi.json)

边界：

- `corporation_id == 主军团 ID` 只证明查询时该角色是主军团成员；
- 它不证明该角色过去一直在主军团，也不证明该角色背后的真人仍满足平台制度；
- 角色的历史军团记录可由 `GET /characters/{character_id}/corporationhistory/` 查看，但历史不能替代当前归属检查；
- ESI 缓存和军团变更传播存在时间窗口，敏感授权应采用短缓存并允许拒绝/复核，而不是把 ESI 响应当作零延迟强一致信号。

### 3.3 军团职位/角色

以下能力属于不同授权层级，不能混用：

| 目的 | Endpoint | Scope | 结果与边界 |
|---|---|---|---|
| 查询已授权角色自己的军团角色 | `GET /characters/{character_id}/roles/` | `esi-characters.read_corporation_roles.v1` | 路径角色必须是 token 自身；返回其 `roles`、按地点角色和可授予角色等集合，OpenAPI 无额外军团角色要求。|
| 查询已授权角色自己的军团头衔 | `GET /characters/{character_id}/titles/` | `esi-characters.read_titles.v1` | 路径角色必须是 token 自身；返回其获授头衔，OpenAPI 无额外军团角色要求。头衔名称不能直接授予平台权限。|
| 查询军团成员列表 | `GET /corporations/{corporation_id}/members/` | `esi-corporations.read_corporation_membership.v1` | 返回成员角色 ID 列表；token 角色必须属于目标军团，且 OpenAPI 要求 `Director`，不应作为普通成员登录必需项。|
| 查询军团全体成员的角色 | `GET /corporations/{corporation_id}/roles/` | `esi-corporations.read_corporation_membership.v1` | 返回成员角色分配；token 角色必须属于目标军团，且 OpenAPI 要求 `Director`。|
| 查询军团定义的头衔及其角色组合 | `GET /corporations/{corporation_id}/titles/` | `esi-corporations.read_titles.v1` | 返回军团头衔定义；授权 token 的角色必须属于目标军团，且 ESI OpenAPI 明确要求 `Director`。|
| 查询军团公开资料 | `GET /corporations/{corporation_id}/` | 无 | 包含 `ceo_id`、`creator_id`、`member_count` 等公开字段，可核验当前 CEO。|

表中 endpoint、scope 与 schema 均来自[官方 ESI OpenAPI](https://esi.evetech.net/meta/openapi.json)，也可在[官方 API Explorer](https://developers.eveonline.com/api-explorer)按路径查阅。

重要区分：

- OAuth scope 表示角色对应用读取一类数据的同意；
- EVE 军团角色（如 `Director`、`Personnel_Manager`、`Accountant` 等）是游戏内权限事实；
- 平台业务角色是本系统授权。三者不等价。

应用收到 `403` 时不得把它解释为“角色不在军团”这一单一结论；也可能是 scope、授权角色、军团身份或所需游戏内职权不满足。应按 ESI 错误语义记录为无法验证，并要求重新授权或人工复核。

## 4. 能否验证成员对“成员结算公司”的控制

### 4.1 可直接验证的证据

对申报的成员结算公司，可组合以下事实：

1. 申报一个 `corporation_id`；
2. 用 `GET /corporations/{corporation_id}/` 读取公开 `ceo_id`；
3. 要求该 `ceo_id` 对应角色完成本平台 SSO；
4. 验证 JWT 所代表角色 ID 与 `ceo_id` 一致；
5. 用 `GET /characters/{character_id}/` 再确认其当前 `corporation_id`；
6. 保存验证时间和证据快照，并周期复核 CEO/军团归属。

这能支持精确表述：**“完成本次 SSO 的角色在查询时是该 EVE 公司的当前 CEO。”** 公开公司资料 endpoint 与字段见[官方 ESI OpenAPI](https://esi.evetech.net/meta/openapi.json)。

如果制度接受 Director 作为操作控制证据，可让该公司中的候选角色单独授权 `esi-characters.read_corporation_roles.v1`，通过 `GET /characters/{character_id}/roles/` 核验其 `Director` 等角色。精确表述只能是：**“完成本次 SSO 的角色当前拥有该 EVE 军团角色。”**

### 4.2 无法由 EVE 事实可靠证明的部分

上述证据不能单独证明：

- 主军团角色与结算公司 CEO/Director 角色背后是同一现实真人；
- 该真人是公司唯一控制人或最终受益人；
- 其他 Director、股东或角色无法共同/替代控制公司；
- 该公司只服务这名成员，或未来不会转让；
- 一个普通公司成员对公司具有实际控制；
- 平台定义的“唯一有效成员结算公司”关系天然存在于 EVE。

因此，“成员—唯一有效成员结算公司”的绑定是**平台管理事实**。EVE SSO/ESI 提供可复核证据，但绑定的建立、变更、失效日期、冲突处理和例外批准必须由平台流程记录。最稳妥的证据是 CEO 角色亲自 SSO；接受 Director、其他职位或人工材料属于后续业务决策，本文不替代该决策。

## 5. 平台业务角色能力矩阵

| 平台业务角色 | EVE 可验证的事实 | 不能由 EVE 自动推出的权限 | 建议授权责任 |
|---|---|---|---|
| 成员 | 某个已 SSO 角色的 ID；该角色当前 `corporation_id` 是否为主军团；可逐个证明成员控制了若干登录角色 | 多个角色是否属于同一真人；该真人是否有资格查看某一平台成员账户；成员结算公司唯一绑定 | 平台根据“至少一个已验证主军团角色”等政策授予；管理员处理多角色归集和例外 |
| 财务 | 经额外 scope 可见 `Accountant`、`Junior_Accountant`、`Director` 等 EVE 军团角色事实 | 审批 LP 权益、改余额、导入期初数、查看平台全局财务；EVE 同名角色不等于平台财务职责 | 只能由平台管理员/既定平台治理流程授予；EVE 角色可作前置条件或复核信号 |
| 执行者 | 可核验其是否为主军团成员及某些游戏内军团角色 | 谁被授权在客户端人工执行 LP 转账、登记执行结果、处理失败；ESI 没有代表平台授予该职责的事实 | 平台管理员授予并保留授予/撤销审计记录 |
| 审计 | 可核验角色身份、军团归属及可能存在的 EVE 军团角色 | 平台全局只读、导出、审计日志访问；即使 EVE 出现近似 `Auditor` 角色，也不等同平台审计权限 | 平台管理员授予；保持只读并与财务/执行写权限分离 |

结论不是“完全忽略 EVE 角色”，而是：**EVE 事实用于认证和资格判断，平台策略用于授权**。即使采用自动规则，也应把规则版本、证据时间、授予结果写入平台审计日志，而不是把每次请求直接等同于 ESI 当前返回值。

## 6. Authorization Code、PKCE 与 token 生命周期

### 6.1 授权流程

EVE SSO 的官方 metadata 公布授权、token、撤销和 JWKS 地址，应从[官方 OAuth Authorization Server Metadata](https://login.eveonline.com/.well-known/oauth-authorization-server)读取，而不是依赖第三方常量。当前常用地址包括：

- Authorization endpoint：`https://login.eveonline.com/v2/oauth/authorize`
- Token endpoint：`https://login.eveonline.com/v2/oauth/token`
- Revocation endpoint：`https://login.eveonline.com/v2/oauth/revoke`
- JWKS：`https://login.eveonline.com/oauth/jwks`

Web 平台应使用 Authorization Code；客户端在发起授权前生成不可预测的 `state`，回调时精确匹配，以防登录 CSRF/授权响应混淆。[RFC 6749 §10.12](https://www.rfc-editor.org/rfc/rfc6749#section-10.12)

同时使用 PKCE：每次授权生成高熵 `code_verifier`，发送其 `S256` `code_challenge`，换 token 时提交原 verifier。不得使用固定 verifier，也不得退回 `plain`。[RFC 7636 §4](https://www.rfc-editor.org/rfc/rfc7636#section-4)

`redirect_uri` 必须与应用注册和授权请求精确一致；authorization code 只能使用一次、短时有效，并由后端换取 token。机密客户端的 client secret 只能保存在后端，不能进入浏览器、日志或仓库。[RFC 6749 §4.1.3](https://www.rfc-editor.org/rfc/rfc6749#section-4.1.3)

### 6.2 access token

EVE SSO access token 是短寿命 JWT；具体有效期以 token 的 `exp` 与 token 响应 `expires_in` 为准，不在业务代码中假设永久或写死为授权期限。[EVE SSO](https://developers.eveonline.com/docs/services/sso/)；[RFC 7519 §4.1.4](https://www.rfc-editor.org/rfc/rfc7519#section-4.1.4)

每次接受 token 时至少验证：

- 签名，且公钥来自官方 JWKS；
- `kid` 对应当前 JWKS key，允许正常 key rotation；
- 预期算法白名单，不信任 token 自报任意算法；
- `iss` 与 metadata 中 issuer 精确匹配；
- `aud` 同时包含本应用的 `client_id` 与 EVE 资源受众 `EVE Online`，并强制 `azp == 本应用 client_id`；不要沿用只验证 `aud="EVE Online"` 的旧示例；
- `exp` 尚未到期，必要时仅允许很小的时钟偏差；
- `sub` 格式和角色 ID；
- 实际授予的 `scp` 不超出本次操作需要，并以 token 内实际 scope 为准。

依据：[EVE Developers：EVE SSO](https://developers.eveonline.com/docs/services/sso/)、[官方 metadata](https://login.eveonline.com/.well-known/oauth-authorization-server)、[官方 JWKS](https://login.eveonline.com/oauth/jwks)、[RFC 7519](https://www.rfc-editor.org/rfc/rfc7519)、[RFC 8725 §3](https://www.rfc-editor.org/rfc/rfc8725#section-3)。

不要用“能解析 JWT”代替验证，也不要把角色名当主键。日志不得记录完整 access token、authorization code、client secret 或 refresh token。

### 6.3 refresh token、过期与撤销

refresh token 是长期敏感凭据，只能发送到 token endpoint；应加密存储、限制读取主体，并避免在只需一次登录且不需后台 ESI 的场景持有它。刷新后必须以新 access token 的实际 `sub`、`scp`、`exp` 重新验证，不因数据库中曾经授予过 scope 就假设 token 仍有该 scope。[EVE SSO](https://developers.eveonline.com/docs/services/sso/)；[RFC 6749 §6](https://www.rfc-editor.org/rfc/rfc6749#section-6)

应用必须把以下情况当作正常失效路径：

- access token 到达 `exp`；
- refresh token 被用户或应用撤销；
- EVE SSO 拒绝刷新；
- 用户重新授权后 scope 缩减；
- 角色所有者发生变化；
- 角色离开主军团或失去游戏内角色。

本地“退出登录”至少终止平台 session；若产品语义是“解除 EVE 授权/解绑”，还应调用 metadata 公布的 revocation endpoint，并删除本地 token。撤销按[RFC 7009](https://www.rfc-editor.org/rfc/rfc7009)处理；调用成功也不能继续保留本地可用副本。平台角色撤销与 OAuth token 撤销是两个动作：前者撤销本系统权限，后者撤销 EVE API 授权，必要时两者都做。

不应声称 refresh token 永不过期。即使没有固定短 `exp`，它仍可能被撤销或失效；系统必须能回到重新授权流程。

## 7. 最小权限建议（不选择最终业务方案）

### 基线层：所有登录成员

1. 使用 EVE SSO Authorization Code + PKCE 完成角色登录；
2. 登录请求不捆绑钱包、军团成员列表或军团角色 scope；
3. 以已验证 JWT 的角色 ID 调用公开 `GET /characters/{character_id}/`；
4. 检查 `corporation_id == 主军团 ID`，建立短期验证结果；
5. 由平台 session 承载已登录状态，不把 ESI access token 本身当长期浏览器 session；
6. 每个附加角色分别 SSO，再由平台完成多角色归集。

这一层足以证明“登录角色”和“该角色当前是主军团成员”，不要求军团级高权限 token。

### 按需层：需要核验候选管理者的 EVE 职位

仅对需要以游戏内角色作为资格证据的角色增量请求：

- `esi-characters.read_corporation_roles.v1`
- 调用 `GET /characters/{character_id}/roles/`

只有后续规则明确使用军团自定义头衔时，才另行请求 `esi-characters.read_titles.v1` 并调用 `GET /characters/{character_id}/titles/`。不要仅为了普通成员登录请求这两个 scope。平台财务、执行者、审计权限仍由管理员授予；角色或头衔变化可触发复核或暂停，但是否自动撤权属于后续业务规则。

### 结算公司控制证据层

优先考虑无需额外 scope 的 CEO 证据：结算公司 `GET /corporations/{corporation_id}/` 的 `ceo_id` 对应角色亲自 SSO。若业务后来接受 Director 证据，再对该角色请求 `esi-characters.read_corporation_roles.v1`。无论哪种，成员与结算公司的唯一绑定都应有管理员确认、有效期和变更历史。

### 可选军团运维层

只有产品明确需要后台同步主军团完整成员/角色时，才给一名专门的主军团服务角色请求：

- `esi-corporations.read_corporation_membership.v1`
- `GET /corporations/{corporation_id}/members/`
- `GET /corporations/{corporation_id}/roles/`

该 token 应与普通登录 token 分开保管、单独审计、可独立撤销，并接受 ESI 对授权角色当前军团职权的额外校验。对于“单一主军团、管理员可管理平台角色”的初始版本，这一层不是登录基线。

### 明确不请求

在本票据范围内，没有依据为登录或业务角色授权请求钱包、资产、合同、邮件、位置、技能、市场或其他无关 scope。平台不直接执行 LP 转账，因此也不应以未来可能需要为由扩大授权。最小权限原则见 OAuth 对 scope 的定义：[RFC 6749 §3.3](https://www.rfc-editor.org/rfc/rfc6749#section-3.3)。

## 8. 建议保留的审计证据

为使 EVE 事实与平台授权可追溯，建议每次授予或复核保存：

- 平台成员 ID、EVE `character_id`、验证后的 JWT `sub`；
- owner 值（若采用）及变化检测结果；
- 查询时的 `corporation_id`、目标主军团 ID；
- 结算公司 `corporation_id`、公开 `ceo_id`、证明角色 ID；
- 经授权读取到的相关 EVE 军团角色子集；
- 实际 token scope（不保存完整 token 到审计日志）；
- ESI 查询时间、缓存/响应标识及验证结果；
- 平台业务角色的授予人、授予理由、依据规则版本、有效期与撤销时间。

这些记录用于说明“当时依据什么第一方事实作出平台决定”，而不是保证该事实永远不变。

## 9. 留给后续规格的决策点

本研究不选择最终方案，后续仍需明确：

1. 成员资格是“任一绑定角色当前在主军团”还是指定主角色必须在主军团；
2. 成员离团后的 session、申请和历史数据如何处理；
3. 结算公司只接受 CEO 证据，还是也接受 Director/人工复核；
4. CEO/Director 变化后的宽限期、冻结和重新绑定流程；
5. 平台财务、执行者、审计角色由谁授予，是否要求某个 EVE 角色作为前置条件；
6. 是否值得持有军团级 `esi-corporations.read_corporation_membership.v1` token 做后台同步；
7. 身份复核频率、ESI 不可用时的 fail-closed 策略与人工应急流程。

在这些决策完成前，安全默认值应是：公开信息能满足时不用 scope；单角色 scope 能满足时不用军团级 scope；EVE 事实只作资格证据，平台高权限始终显式授予并可审计。
