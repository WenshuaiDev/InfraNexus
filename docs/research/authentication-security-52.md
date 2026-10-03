# 密码认证与浏览器会话的安全约束

核对日期：2026-10-03。研究票：[核实密码认证与浏览器会话的安全约束](https://github.com/WenshuaiDev/InfraNexus/issues/52)。承接：[SaaS 登录认证：决策地图](https://github.com/WenshuaiDev/InfraNexus/issues/50)。

这是支撑后续讨论的事实矩阵，不是实施规范。NIST 的 SHALL 仅在采用其符合性目标时构成相应要求；OWASP 是安全工程建议；浏览器和数据库行为是实现必须面对的机制约束。本项目未选择认证保障等级，不因本研究新增生产合规、MFA、账号恢复或权限体系。

当前已确认：管理员账号只是 CLI 创建的第一个账号；本期所有已登录账号都能创建普通账号，不校验管理员身份。修改密码、查看和撤销会话针对本人。以下未决问题不能改变这一范围。

## 事实矩阵

| 主题 | 一手来源事实及适用条件 | 对本期的影响与尚未决定事项 |
| --- | --- | --- |
| 密码长度与字符 | NIST SP 800-63B-4 §3.1.1.2：单因素密码至少 15 字符；建议允许至少 64 字符上限；不附加强制字符组合，不周期强制改密；允许 Unicode 时按码点计数并建议 NFC。建立或修改密码时检查完整弱密码值，不能截断验证。详见 [NIST](https://pages.nist.gov/800-63-4/sp800-63b.html)。 | 这是采用该标准时的要求/建议区别，不自动冻结本项目长度、Unicode、空白处理、弱密码字典或升级策略。创建账号、CLI 初始化与改密应采用一致规则。 |
| 失败尝试与交互 | NIST 要求账号维度限制失败尝试；§3.2.2 的 100 次是标准上限且关联禁用/重新绑定，不是默认建议值。标准要求支持密码管理器和自动填充，并建议允许粘贴。[NIST](https://pages.nist.gov/800-63-4/sp800-63b.html) | 本期不含恢复路径，不能机械引入永久锁定；限流维度、窗口、阈值、冷却和依赖故障时的行为待定。 |
| 密码存储成本 | OWASP 建议使用加盐的专用慢哈希，首选 Argon2id；其一个最低配置是内存 19 MiB、迭代 2、并行度 1，亦列出不同 CPU/内存配比。单纯 SHA-256 不适于存储密码。成本需结合实际服务器负载评估。[OWASP Password Storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) | 推荐将 Argon2id 作为候选并在本地 Docker 测量并发耗时/内存；算法、参数、哈希格式及后续升级机制仍待决定。此配置不等于 NIST/FIPS 认证结论。 |
| 登录错误、改密与传输 | OWASP 建议无账号、错误凭据等使用通用失败反馈，连同响应时间/状态码避免枚举差异；修改密码要求有效会话和原密码；登录及认证后页面建议全程 TLS。[OWASP Authentication](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) | 用户已确认原密码验证。建议服务故障单独反馈，不能伪装成密码错误。已登录的账号创建冲突是否明确提示用户名占用，是本期交互选择，不照搬公开注册示例。HTTP 本地闭环只能证明功能，不能证明传输机密性。 |
| Cookie 属性与本地 HTTP | HttpOnly 阻止 JavaScript 读取 Cookie，但浏览器仍会随 fetch 请求发送；SameSite=None 要求 Secure；Secure 通常要求 HTTPS，MDN 记录 localhost 例外。__Host- 前缀要求 Secure、Path=/ 且无 Domain。[MDN Set-Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie) | 不可把 localhost 特例推断为任意局域网地址/自定义开发域名均可用。待定实际开发主机、浏览器、Cookie 名称和属性；若利用 localhost 特例，须在目标浏览器实测设置及回传。HttpOnly 不替代 CSRF 防护。 |
| 本地环境隔离 | Cookie 依主机而非端口隔离，同一主机不同端口共享其 Cookie 范围。[RFC 6265 §8.5](https://www.rfc-editor.org/rfc/rfc6265#section-8.5) | 仅给开发/测试分配不同端口不足以隔离浏览器身份。可考虑独立测试浏览器上下文、独立 Cookie 名称/主机；具体方案待定。名称隔离不等于对同主机恶意服务的安全边界。 |
| CSRF | OWASP 将 SameSite 主要作为纵深措施；same-site 不等于 same-origin。可使用 CSRF token、Origin/Referer 校验、Fetch Metadata 等；Fetch Metadata 只在可信 URL 上提供，并应规定缺失头的回退或拒绝行为。[OWASP CSRF](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) | 退出、改密、撤销会话、创建账号以及登录都须纳入威胁分析；不以 GET 改状态。具体组合待定，验收要覆盖恶意来源和缺失头，不能仅验证同源成功请求。 |
| 会话固定与轮换 | OWASP 建议采用 CSPRNG 生成会话标识，只接受服务端签发的有效标识；登录时更新标识以防会话固定。轮换应让旧标识失效。[OWASP Session Management](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) | 推荐登录签发新的不可预测令牌，不沿用匿名客户端给定值。令牌格式、长度、存储形式及改密时的当前会话处置仍待定。 |
| 超时、退出和撤销 | OWASP 区分闲置超时、绝对超时，并要求服务端执行；退出须使服务端会话失效，清除 Cookie 本身不足。其时长示例依风险与用途，不是统一默认。[OWASP Session Management](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) | 待定时长、何种活动刷新闲置时间、并发数量、改密保留/撤销范围、撤销与在途操作的分界。本人会话列表中的设备/IP 信息应是提示，不能证明实际设备身份。 |
| Redis 持久化与恢复 | RDB 会丢失最近快照后的写入；AOF everysec 仍可能丢失约一秒写入，重启按已保存内容恢复。[Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/) | **推导**：丢失会话创建只会迫使重登，丢失删除/撤销却可能恢复旧会话；“可丢失缓存”不是完整安全语义。不能把开启 AOF 等同于已撤销会话绝不复活。 |
| Redis 过期机制 | Redis 过期时间持久化为绝对 Unix 时间；实例停机期间时间继续流逝，受机器时钟影响。[Redis EXPIRE](https://redis.io/docs/latest/commands/expire/) | **推导**：恢复时已到期的键不应凭空获得新寿命，但尚未到期的旧快照键仍可能复活。TTL 不能替代撤销事实。若实施闲置续期，须同时遵守固定绝对到期时间。 |
| 事务边界与创建冲突 | PostgreSQL 事务使其中的 SQL 更新原子提交；唯一约束可保障指定列/组合的唯一性。[PostgreSQL transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html)、[constraints](https://www.postgresql.org/docs/current/ddl-constraints.html) Redis MULTI/EXEC 不提供执行时错误的事务回滚。[Redis transactions](https://redis.io/docs/latest/develop/using-commands/transactions/) | **推导**：两种存储的本地事务不会自动组成一个事务。密码已提交而撤销失败、登录与改密竞争、创建响应丢失、两个 CLI 同时初始化，都需定义结果及重试规则；“先查再写”不足以承担并发正确性。 |

## 存储方案必须回答的边界

以下是研究推导与候选路线，均未被选定：

1. **以持久存储为会话/撤销的权威来源**：密码与失效事实尽量在同一事务内变化；Redis 只作可重建缓存。但缓存命中仍需遵守撤销即时性，不能从陈旧命中放行。需要决定是否直接查询权威状态以及性能取舍。
2. **以 Redis 为在线会话状态**：需要完整的丢失、恢复及重新开放访问协议。若采用恢复时整体失效，应排除旧会话快照回灌和旧进程续写；只依靠 Redis 内同样可能回滚的“版本号”不能证明失效不可逆。若选择外部持久代际或撤销记录，还需定义验证与更新顺序。
3. **失败和未知结果**：安全状态不可读取时，推荐拒绝受保护操作；密码变更或撤销未被确认时，不向用户承诺操作成功。断连可能发生在提交之后，应区别“确知失败”和“结果未知”，再决定如何查询/幂等重试。业务错误、依赖不可用、部分结果的接口和 UI 语义均留给后续票。

“不复活”的验收应指定恢复场景和信任边界：仅 Redis 崩溃/丢失/恢复、PostgreSQL 是否保持完整、是否包含旧备份人工回灌。整个权威存储恢复到旧备份也会回滚安全事实，不能在没有外部单调状态或恢复规程时宣称任意灾难恢复都保持撤销。

## 留给决策票的选择

- 账号和密码：用户名唯一性/规范化、初始密码由谁设定与如何交付、是否首次改密、CLI 并发初始化与重复调用、平台创建后的返回与重试行为。
- 密码保护：长度/字符/字典、哈希参数及资源上限、失败尝试策略及依赖故障。
- 会话行为：令牌与 Cookie、CSRF、闲置/绝对期限、并发、改密后的会话范围、撤销生效点及在途写入。
- 数据和故障：权威存储、撤销不可逆边界、Redis 重建/恢复、跨存储部分成功及服务错误反馈。
- 本地验收：指定浏览器/URL、HTTP Cookie 实测、独立浏览器身份与数据、过期边界、旧令牌重放、并发及 Redis 故障/恢复。此处只是候选证据清单，未执行测试，也不扩展为全仓验收。

本研究没有冻结上述阈值、存储实现或操作策略；这些选择须由 HITL 决策票确认。
