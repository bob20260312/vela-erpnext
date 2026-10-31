# ERPNext–Vela SSO Adapter 设计

## 意图

为 ERPNext 提供可由 Vela 自动部署和管理的 SSO 集成能力。组织只需在 Vela 启用 ERPNext 集成并完成一次组织级配置；获准成员从 Vela 入口打开 ERPNext 时，通过 OIDC 登录并自动建立或复用 ERPNext 用户。账号身份以 Vela 的稳定 `(issuer, sub)` 为准，权限跟随当前授权，撤销后 ERPNext 在约定时限内拒绝访问。

本设计以 Vela 仓库的 `docs/superpowers/specs/2026-10-30-erpnext-vela-sso-integration-requirements.md` 为需求契约，并以 `docs/superpowers/plans/2026-10-30-erpnext-vela-sso-adapter.md` 为联合实现目标。需求所指的 `integrations/erpnext/app/vela_erpnext` companion app 已存在于 Vela 仓库；代码交付目标是该 app 及其部署/验收链路，不是把 Vela 专用逻辑并入上游 ERPNext 核心。本设计文档暂存于当前可写的 ERPNext 工作区，便于当前任务评审。

## 目标与非目标

### 目标

- 提供标准 OIDC Authorization Code + PKCE S256 登录；验证 state、nonce、签名、issuer、audience、时效、算法和 key id。
- 由可信部署控制面幂等下发组织级配置和 secret；终端用户无需输入个人配置，secret 不进入普通配置读回、日志或导出。
- 使用 `(issuer, sub)` 唯一定位受 Vela 管理的 ERPNext 用户；首次获准登录时 JIT 创建最小权限用户。
- 对同邮箱异 subject、无组织成员授权、停用成员和无效 token fail closed；不静默关联或继承已有账户权限。
- 将 Vela 授权同步到受管理的 ERPNext 角色/业务范围，并为用户停用、成员移除和角色撤销提供可重复执行的撤销机制。
- 暴露版本化 capability、幂等配置和健康检查契约；Vela 无法确认兼容能力时不得将集成标为可用。
- 以真实浏览器、真实 Vela 会话和实现了该契约的 ERPNext 版本完成联合验收。

### 非目标

- 替换 Frappe 原生用户、角色、权限引擎或为所有本地用户启用 Vela 登录。
- 按 email 自动合并账号，或把本地 ERPNext 高权限角色映射成 Vela 默认角色。
- 将本地 OIDC simulator、mock ERPNext 或单元测试结果标记为跨系统验收通过。
- 要求逐用户预建 ERPNext 账号或手工绑定 subject。

## 现状和边界

- Vela 集成 app 已实现 OIDC Authorization Code + PKCE、state/nonce、ID Token 验签、固定 subject 查找、登录回调、Vela entitlement 获取、角色/范围同步，以及 `before_request` 对受管会话逐请求的授权核对。当前仍依赖 `site_config` 中的 `vela_oidc` / `vela_permission_sync`，并要求管理员逐人绑定 subject；没有 JIT、Vela capability/configure/health 契约或主动撤销接收端。
- 当前 `sso.py` 只按已绑定 `(issuer, sub)` 查找现存用户；`bind_subject` 仅允许管理员绑定现存 System User。subject 字段自身为 unique，但需求主键是 `(issuer, sub)`，需调整为复合唯一映射。
- 当前 `permission_sync.py` 已有角色/范围映射配置、最小权限检查、禁用用户时清除 session，并在每个受管请求 fail closed；但它以 Vela 权限为 subject 用户的全部授权真相，会替换该用户的角色和 User Permissions，并删除直接 DocShare。需求要求仅管理明确标记的 Vela-owned 授权，因此需重构投影策略。
- 当前工作区声明依赖 Frappe `>=16.50.0,<17.0.0`，但 Frappe framework 的认证实现不在此仓库内。实施者必须在 Vela 集成实际使用的 Bench/Frappe 版本验证 hook、secret store 和 session 生命周期，不可只凭 ERPNext 仓库推断。
- ERPNext 的业务用户/角色继续由 Frappe 管理。集成需区分 Vela-managed 身份/授权与本地管理项，避免同步时删除后者。
- Vela 负责稳定用户 ID、组织授权、角色 allowlist、OIDC client 生命周期和部署密钥通道；ERPNext 负责验证身份、账号映射/JIT、权限投影、会话和能力报告。两端不得互相宣称对方尚未实现的能力。

### 设计评审发现

- Vela 集成当前锁定 ERPNext `v16.52.1`（`integrations/erpnext/release.env`）；运行时 Frappe 精确版本需从目标 Bench capability 中报告并绑定验收记录。
- 当前 `custom_vela_oidc_subject` 是单字段 unique，不能正确表达 `(issuer, sub)`；设计改为 `Vela SSO Identity` 映射与复合唯一约束，并迁移现存绑定。
- 当前权限同步会整体替换 User roles / User Permissions 并删除所有 DocShare；这与“保留明确标记的本地授权”冲突。设计改为逐授权来源标记，只删除 Vela-owned 明细，本地授权保留。
- 当前 OIDC 与 entitlement client secrets 从 `site_config` 读取；设计改为两个用途分开的加密 Password 字段，并由受信任部署 API 幂等写入。
- 当前已有逐受管请求的在线 entitlement 核验；v1 将其作为撤销路径，不引入未经需求证明的事件队列。提议的撤销边界是状态对查询 API 可见后、下一受保护请求进入业务逻辑前完成核验。空闲 session 的物理清理时限及 API 可见性保证仍需双方确认。
- 当前 Vela entitlement 请求以 `provider` + OIDC `subject` 查找并在 repeatable-read 快照中返回 Vela `user_id`、`lifecycle_version`、`scope_revision`、active、roles、scopes。由于角色集合无独立 revision，ERPNext 对同一身份的“获取 snapshot + 应用权限”使用分布式互斥锁，避免并发旧响应覆盖新响应；锁失败或超时 fail closed。provider/subject 到稳定 Vela user 的绑定由 Vela 身份服务维护，不由 ERPNext 管理员逐人绑定。

## 用户流程

1. Vela 管理员启用 ERPNext 集成。一次组织级安装为部署控制面建立专用 `Vela SSO Configurator` Frappe service User；它只有 SSO configure API 权限，API key/secret 进入 Vela secret manager。控制面发现 ERPNext 的 SSO capability 和合同版本；若能力缺失或主版本不兼容，保持“未就绪”，不创建可用入口。
2. 控制面为该 ERPNext 站点创建或轮换 OIDC client，并用上述 service User 通过 TLS 配置 issuer、client ID、精确 callback、Vela entitlement client、角色/范围版本和 secret。重复配置返回相同有效配置版本，不在读回中返回 secret。
3. 获准成员从 Vela 中打开 ERPNext。ERPNext 发起 OIDC Authorization Code + PKCE；使用短时、一次性并绑定浏览器会话的 state/nonce。
4. 回调完成 token 和 claim 校验后，以 `(issuer, sub)` 查询本地映射，并用现有 `provider` + `subject` entitlement contract 确认 Vela 用户 ID、目标组织成员资格、账号状态、角色和业务范围。授权服务不可用或授权资料不完整时拒绝登录。
5. 若 subject 尚无映射且 Vela 明确授权，ERPNext 创建受管 System User，`username` 使用 `vela_` + `(issuer, sub)` SHA-256 的小写 Base32 前 32 字符；email/name 仅作为资料。若相同 email 已属于不同 identity，生成审计冲突并拒绝；不尝试静默关联或自动送审。
6. 按双方版本化 allowlist 同步受管角色/范围；首次登录只获得 Frappe 自动角色和 Vela 明确授予的业务角色。绝不从 email 或原有角色推导权限；Administrator/System Manager 等特权角色永不接受此 v1 SSO 映射。
7. 创建 ERPNext session 并仅跳转到校验过的站内相对 path；locale 只映射到 ERPNext 支持的语言。
8. Vela 停用用户、移除组织成员或撤销角色后，entitlement 查询返回最新状态；ERPNext 在下一受保护请求前幂等更新映射、角色/范围，或拒绝并清除 session。恢复访问需比已记录停用状态更高的 `lifecycle_version`，不能因 JIT 自动启用旧账户。

## 架构与组件

ERPNext 侧继续通过现有可安装/升级的 `integrations/erpnext/app/vela_erpnext` Frappe app 扩展，避免把 Vela 特定流程散落到 ERPNext 业务模块中。Frappe 自带用户、角色和 session 机制作为执行底座；集成 app 提供协议适配、映射和生命周期逻辑。实现和发布应在 Vela 仓库配套的 app、容器/站点初始化及验收脚本中完成。

| 组件 | 职责 |
|---|---|
| Vela SSO Settings / secret binding | 从当前 `site_config` 人工配置迁移到站点级机器配置；保存版本、issuer/client/callback、Vela API base/provider/org、两类 client ID 和启用状态；OIDC RP secret 与 Vela entitlement client secret 分开存为 Frappe Password 字段，部署控制面只通过受限的 Frappe API 用户写入，脱敏读回 |
| OIDC relying-party adapter | discovery/JWKS、PKCE、state/nonce、签名与 claims 校验、回调和站内回跳校验 |
| Vela identity/authz client | 通过服务端 client credentials 查询组织授权；验证响应来源、授权版本、租户和时效 |
| Identity mapping | 新建 `Vela SSO Identity` 映射，持久化唯一 `(issuer, sub)` → ERPNext User，并保存 Vela provider/user ID、组织、active 状态、`lifecycle_version` 和 `scope_revision`；不以 email 为键。迁移现存 User 自定义字段绑定，保留旧字段供兼容期回滚 |
| Provisioning and role projector | 冲突检查、JIT、最小权限、现有角色 allowlist/范围映射；User Role / User Permission 以 `custom_vela_managed` 标记区分来源，只撤销 Vela 管理行，保留本地角色、User Permissions 和 DocShare |
| Session/revocation manager | 保留当前 `before_request` 每次在线核对；每次操作前同步角色/范围，inactive 时禁用用户并清除 session。提供幂等状态 reconcile；若要求早于下一请求的撤销 SLA，再增加 Vela 主动推送 |
| Admin/control-plane API | 受部署控制面认证的 configure、health、capabilities 和撤销入口；幂等、限流、审计且禁止 secret 回读 |
| Audit/observability | 记录脱敏 subject、ERPNext user、组织、授予/撤销角色、配置版本、时间和结果；禁止记录 code/token/secret/完整敏感 claims |

Frappe hook 选择须以目标 Frappe 版本验证。官方 hook 文档将 `auth_hooks` 描述为请求认证 hook，将 `on_session_creation` 描述为 session 创建生命周期 hook；前者本身不等于对每个已有 cookie session 做在线授权检查。因此实现需分别覆盖浏览器 OIDC 登录、session 建立、受管 session 的后续验证和撤销，不能把 login hook 当作撤销机制。

## ERPNext ↔ Vela 契约

合同版本为 `vela_sso_contract_version`，采用语义版本。Vela capability 响应至少包含：合同版本、ERPNext/Frappe build、支持的 OIDC/PKCE/自动配置/JIT/权限同步/撤销能力、健康状态和当前配置摘要。Vela 要求的任一能力缺失、健康不通过或不兼容主版本时，连接不可就绪。

ERPNext 侧使用以下 Frappe method API 路径；Vela adapter 按 capability contract 调用：

- `GET /api/method/vela_erpnext.sso.capabilities`：公开非敏感 build/能力/版本信息。
- `POST /api/method/vela_erpnext.sso.configure`：仅供部署控制面调用，幂等配置 issuer、client ID、secret、callback、Vela API client、角色/范围 allowlist 版本；通过专用 ERPNext `Vela SSO Configurator` service User 的 Frappe API key/secret 授权，限定只能调用此配置接口；返回配置版本和 issuer/client ID 摘要，不返回 secret。该 API credential 只保存在 Vela secret manager。
- `GET /api/method/vela_erpnext.sso.health`：返回配置存在性、issuer、合同版本、依赖可用性和最近授权核对状态；不得泄露 secret 或 token。
- 现有授权状态查询：`POST /api/v1/internal/erpnext/entitlements`，由 `/api/v1/oauth2/token` 的 `erpnext:sync` client-credentials token 保护；请求含 `provider` 与 OIDC `subject`，repeatable-read 响应已含 `organization_id`、Vela `user_id`、`active`、`lifecycle_version`、`scope_revision`、`roles`、`scopes`。租户成员身份由查询限定到 client token 的 organization，未绑定、失活或不属于该组织的 subject 返回 `active=false`。此接口的响应语义供 JIT 复用，不新增逐用户 ERPNext 绑定 API。
- 撤销/状态核对：首版沿用逐受保护请求读取 entitlement 并幂等应用状态，确保撤销状态生效后下一次受保护请求先于业务操作更新角色或拒绝访问；若 SLA 要求早于下一请求失效，再实现签名事件接收、租户/重放校验、重试队列和主动 session 清除。

所有用户字段和权限声明都按合同版本解析。`iss` 必须精确匹配 discovery issuer，`sub` 必须稳定且不可变；多 audience 时校验 `azp`。Entitlement 请求用配置中的 `provider` + `sub` 查询，响应必须含与当前 site 绑定的 `organization_id`、稳定 Vela `user_id`、`active`、`lifecycle_version`、`scope_revision`、roles 和 scopes。`active` 必须为 true 才能进入；HTTPS、TLS 验证、受信任 discovery/JWKS 来源、算法 allowlist 和精确 callback 白名单为硬约束。

## 身份、权限和会话规则

- 新建 `Vela SSO Identity` Frappe DocType，字段为 `issuer`、`subject`、`provider`、`vela_user_id`、`organization_id`、`user`、`active`、`lifecycle_version`、`scope_revision`、`managed_by_vela` 和 `last_synced_at`；`(issuer, subject)` 以数据库复合唯一约束保证唯一，不同 issuer 的相同 sub 是不同身份。
- JIT `username` 是 `vela_` + `base64.b32encode(sha256((issuer + "\\0" + sub).encode()).digest()).decode().rstrip("=").lower()[:32]`；Frappe `User.name`/email 不作为身份键。若 Frappe 目标版本要求 email，缺省使用该 username 加站点受控 `.invalid` 域名；已被其他 User 占用的真实 email 触发冲突拒绝。新建时不发欢迎邮件、不设可用本地密码。
- 只有有效的 Vela 组织成员授权才能创建或启用受管 ERPNext System User。首次授权仅包含 Frappe 创建 System User 所需的自动角色，不隐式添加业务角色；业务角色只从现有 `DEFAULT_ROLE_MAP` 映射且必须由 Vela 明确授予。若没有至少一个有效业务角色或必需 scope，拒绝登录。禁用用户只有在 `active=true` 且 `lifecycle_version` 高于已记录停用状态时才可恢复。
- 同邮箱异 subject 必须拒绝静默关联；对浏览器仅返回通用拒绝页面，内部安全审计记录冲突；v1 不提供管理员审核 UI。
- 同步只能增加/移除带 `custom_vela_managed=1` 的 Vela-owned `Has Role` 和 `User Permission` 行，不改动未标记的本地角色、User Permissions 或 DocShare。角色被两边同时授予时本地授权独立保留；审计结果要区分其来源。Vela 的空/缺失授权响应不能解释成“保留旧 Vela 高权限”。
- ERPNext 最终有效权限由 Vela-owned grants 与明确的本地 grants 合并形成。Vela 停用用户会禁用整个 ERPNext User；单独撤销 Vela 角色/范围只移除 Vela-owned grant，本地独立授权仍有效。运维和审计必须展示此来源差异，避免把本地授权误报为 Vela 权限同步成功。
- 账号状态按 `lifecycle_version`、范围按 `scope_revision` 拒绝旧快照覆盖新状态。由于响应本身是 Vela 数据库 repeatable-read 快照，ERPNext 还需对每个身份串行化“取 snapshot + reconcile”；同一身份并发同步不能让先前请求的旧 roles 覆盖后续请求的更新。重复请求幂等。
- **本设计的撤销 SLA 建议**：Vela entitlement 状态一旦对查询 API 可见，ERPNext 在下一次受保护请求执行业务逻辑前完成在线核验；inactive 时拒绝并清除该用户所有 session，角色/范围变更先完成同步再继续请求。无授权缓存；Vela 不可用 fail closed。空闲浏览器的 Redis session 可在下次请求时清除，本首版不承诺用户无请求期间的物理 session 清理时限。若业务要求数值型 wall-clock SLA 或无请求也立即清除 session，需加 Vela 推送撤销和可靠投递。
- Vela 身份/授权服务不可用时，对 Vela-managed 登录及受管 session 的权限敏感操作 fail closed。不得以陈旧授权无限期放行。
- 回跳拒绝绝对 URL、协议相对 URL、反斜杠和控制字符；仅接受站内相对 path。locale 映射到已启用语言，未知 locale 使用站点默认语言。

## 安全和运维

- OIDC callback 的 state 一次性消费并绑定 browser session；nonce 与 ID Token 精确比对；验证签名算法、kid、iss、aud、azp（多 audience 时）、exp、iat 和必需 claim。
- client secret 存在 Frappe secret store 或部署 secret manager，仅以引用穿过控制面；不进入普通配置导出、日志、进程参数、API 响应或镜像层。轮换允许受控重叠期并能撤回旧 secret。
- configure 仅接受受信任控制面身份，要求 service User 的 `custom_vela_config_organization_id` 与请求 `organization_id` 相等；配置绑定组织后拒绝无显式重绑定的跨组织覆盖。浏览器登录回调与管理 API 分离。
- `Vela SSO Configurator` Frappe service User 仅能调用 configure，且其组织绑定字段由部署安装器设置、API 不可修改；API key/secret 只保存在 Vela secret manager。配置 API 不在 request/error logs 中记录 body；OIDC RP secret 与 entitlement service secret 分别保存在 Frappe Password 字段。
- Secret 轮换由 configure 接收新 secret 和 `secret_overlap_seconds`（`0..86400`）；服务端将旧 secret 移入 previous Password 字段并设定过期时刻。token/exchange 请求优先新 secret，仅在有效重叠期内对认证失败回退旧 secret。
- 错误响应统一且不透露用户是否存在、token 内容或内部配置；TLS 与 endpoint 域校验失败即拒绝。
- 安装/升级、重复 configure、授权版本乱序和 Vela outage 可恢复且幂等；health 能区分未配置、不兼容、授权依赖不可用和健康。
- 运维指标至少覆盖配置版本、capability 健康、最近授权核对时间和 entitlement 失败类型；日志仅保留脱敏 subject 标识与审计所需字段。

## 验收边界

### ERPNext 组件可验证范围

- OIDC 正向/负向协议校验、state 重放、nonce 错误、错误 issuer/audience/签名/过期 token 拒绝。
- 身份唯一映射、不同 issuer 同 sub、同邮箱异 subject 冲突、JIT 最小权限、角色增加/移除、用户停用和恢复授权。
- 配置幂等与租户隔离、secret redaction/轮换、健康/能力版本门控、回跳校验、旧授权版本拒绝和撤销后的受保护请求拦截。
- 在目标 Frappe/ERPNext Bench 上验证真实 login/session hooks 与 session revoke 行为。

### 联合验收才可声称端到端通过

使用版本固定的 ERPNext/Frappe build、真实 Vela 组织会话和真实浏览器，覆盖从 Vela 入口到 ERPNext 用户/权限/目标页面，再到组织成员撤销及既有 session 失效。记录两端版本、合同版本、脱敏 subject、ERPNext user、角色、回跳目标、撤销时刻和实测传播时间。mock IdP、fixture、组件测试或隔离站点单独成功均不能升级为端到端通过。

## 未决契约，实施前须由双方确认

1. Vela 授权服务 owner 是否承诺 entitlement 查询在成员/角色更新提交后读到新状态；这是“下一受保护请求前拒绝/收敛”的前提。
2. 是否接受上述“下一次受保护请求”作为首版既有 session 撤销 SLA，还是要求数值型 wall-clock SLA（后者需 Vela 主动推送与可靠重试）。
3. 是否接受当前 `DEFAULT_ROLE_MAP` 和 `ROLE_REQUIRED_SCOPES` 作为合同 v1 allowlist；首次 JIT 仅加 Frappe 自动角色，业务角色必须有 Vela 明确 grant，且禁止所有高权限角色。
4. 同邮箱异 subject 首版一律拒绝并记录安全审计；管理员审核 UI 延后，不通过邮箱自动合并。
5. 配置 API 使用专用 Frappe `Vela SSO Configurator` service User/API key；需在目标 Frappe build 验证 User permission 限定、Password 字段 secret 保存、OAuth callback 和 session invalidation API。
6. Vela adapter owner 冻结 capability/configure/health 请求响应 schema；首发合同为 `1.0.0`，app 随 Vela ERPNext companion image 一并升级。

以上未决项中，首版建议采取 fail-closed、冲突拒绝、无业务角色默认授予、逐请求在线授权核验和“状态对查询可见后下一受保护请求前收敛”的撤销 SLA。该 SLA 及 Vela endpoint 的可见性保证需在联合实现计划冻结前确认；更严格 SLA 需增补推送撤销。

## 本设计的实现判断

- 在现有 Frappe companion app `integrations/erpnext/app/vela_erpnext` 上迭代，而不是将登录逻辑耦合进 ERPNext 会计/业务模块。
- 保留现有 OIDC 严格校验和 entitlement API；改造重点是可信组织级配置、JIT 映射、复合 subject 键、授权版本同步、保留本地授权边界与在线撤销收敛，不重写 Vela IdP。
- 以当前 Vela entitlement API、client-credentials 基础设施、`DEFAULT_ROLE_MAP` 和 `ROLE_REQUIRED_SCOPES` 作为合同 v1 起点；角色为空或未映射时拒绝 JIT 登录。
- 为 OIDC RP 与 Vela entitlement client secrets 分别配置 Frappe 加密 Password 字段；部署控制面用独立、最小权限的 Frappe API service User 调用幂等配置接口，并将 API key/secret 存入 Vela secret manager。
- 将邮箱冲突默认设为拒绝并留审计；不在首版创建审核 UI。将 Vela Role/User Permission 来源标记在明细行上，仅清理由 Vela 创建的授权；本地 DocShare 保留为独立授权。
- capability/configure/health 使用明确版本化契约；路径和字段在 adapter 与 ERPNext app 双方落地时共享 schema fixture，能力不完整时 UI 阻断。
- 用真实目标 Bench 验证 Frappe 生命周期接口；不假设仅增加一个 hook 即能满足现存 session 撤销 SLA。

## 参考

- Vela 需求契约：`vela/docs/superpowers/specs/2026-10-30-erpnext-vela-sso-integration-requirements.md`
- Vela 适配计划：`vela/docs/superpowers/plans/2026-10-30-erpnext-vela-sso-adapter.md`
- [Frappe hooks：认证与 session 生命周期](https://github.com/frappe/frappe_docs/blob/master/frappe_docs/www/docs/user/en/python-api/hooks.md)
- [Frappe auth 实现](https://github.com/frappe/frappe/blob/develop/frappe/auth.py)
