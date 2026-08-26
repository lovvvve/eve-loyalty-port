# 确认 EVE SSO 与身份授权能力边界

Type: research
Status: resolved
Parent: [规划 LP 转账信息管理平台](../map.md)
Blocked by:

## Question

基于 EVE Online 官方 SSO、OAuth 与 ESI 文档，确认平台可如何验证登录角色的所有权、角色当前所属军团及必要的军团职位或权限；能否验证成员结算公司的控制关系；token、scope、撤销、刷新和最小权限有哪些约束？形成身份能力矩阵，并指出成员、财务、执行者和审计角色中哪些信息能由游戏事实验证，哪些必须由平台管理员授予。

研究结论还应给出适合单一主军团私有平台的最小授权建议，但不替后续业务决策选择最终登录方案。

## Answer

完整研究见：[EVE SSO 与身份授权能力边界研究](../research/eve-sso-authorization-capabilities.md)。

精炼结论：经正确验证的 EVE SSO JWT 只能证明当前操作者已授权某个具体角色，不能证明现实真人身份或自动归集其全部角色；角色当前 `corporation_id` 与公司当前 `ceo_id` 可由公开 ESI 查询。单角色军团角色与头衔分别需要 `esi-characters.read_corporation_roles.v1`、`esi-characters.read_titles.v1`，军团全员/角色/头衔读取则属于权限更高的军团级能力，不应成为普通登录基线。要求结算公司 CEO 角色亲自 SSO，可证明“该已授权角色当前是公司 CEO”，但仍不能证明同一真人、唯一控制或最终受益所有权，因此成员—唯一有效成员结算公司的绑定必须作为平台管理事实保留确认与变更审计。

成员的当前主军团归属可由 EVE 事实验证；财务、执行者、审计以及多角色归集等平台权限只能由平台管理员或既定治理规则授予，EVE 军团角色/头衔最多作为资格证据。单一主军团私有平台的最小授权基线是 Authorization Code + PKCE、登录不附带无关 ESI scope、用公开角色信息复核主军团归属，并仅在后续规则确有需要时增量请求单角色 scope；不在本票据中替后续规格选择最终登录或授权业务方案。
