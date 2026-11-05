# ShellAX 统一中间件连接设计

> 状态：设计稿（仅设计，不含实现）  
> 基线：上游 `daxiangme/ShellAX` `master` @ `4ed0a8d`（v1.3.4）  
> 目标 fork 归属：`King-bj`  
> 本地克隆：`C:\Users\king\source\ShellAX`  
> 许可证：AGPL-3.0-only  
> 日期：2026-11-05（America/Antigua）

---

## 0. 一句话结论

**扩展现有 `AssetKind` + 共享连接元数据，按协议族挂 Adapter，侧栏继续「一族一入口」；不要另起一套平行资产树。**  
首批：收口 Mongo → Elasticsearch → Kafka → VictoriaMetrics；后续中间件只加 Adapter + 工作区，不改壳。

---

## 1. 愿景与边界

### 1.1 愿景

把日常运维/排查用到的中间件连接，统一收进 ShellAX，成为个人/团队的 **工具连接库（Tool Connection Hub）**：

- 同一套分组、标签、生产标记、SSH 隧道、钥匙串凭据、只读策略与审计；
- 每种协议有专用工作区（查询/浏览/窥探），而不是硬塞一张通用表格；
- 未来可被 AI / MCP 以受控只读方式消费连接与查询结果。

产品叙事从「SSH 终端 + 若干侧栏」升级为：**终端为中枢，连接库为资产，各族工作区为能力面。**

### 1.2 统一连接中心 vs 继续堆独立侧栏

| 路径 | 做法 | 评价 |
|------|------|------|
| **A. 推荐：统一资产壳 + 族工作区** | 扩展 `assets.rs` 的 `AssetKind`；共享 ConnMeta；侧栏仍按族过滤；各族独立 `*_view` | 与现有 DB/Redis/Docker 一致；扩展成本可控 |
| B. 纯堆独立侧栏 | 每加一个中间件就新模块、新配置段、新密钥约定 | 短期快，长期重复造「树/隧道/凭据/权限」 |
| C. 单一「数据平面」大一统 UI | 一个侧栏列出所有连接，靠类型过滤 | 概念干净，但迁移成本高，违背当前「模块过滤 kind」习惯 |

**推荐 A。** C 可作为远期「连接总览」页（Tools 或设置旁），不是 P0。

### 1.3 明确不做（边界）

- **不做**完整 ELK / OpenSearch Dashboards 替代品（无完整可视化编排、Index Lifecycle 管理控制台）。
- **不做** Kubernetes IDE / 完整 Prometheus 运维台 / Grafana 替代。
- **不做**默认的 Kafka 生产者轰炸、批量删索引、无确认的写路径。
- **不做**与 `assets.rs` 平行的第二套「ServiceStore」；所有连接仍是 Asset。
- **不做**把 SQL 表格 UI 硬套到 ES/Kafka/指标上。

---

## 2. 现状与差距

### 2.1 已有（须复用，勿重造）

| 能力 | 位置 / 类型 | 说明 |
|------|-------------|------|
| 统一资产 | `src/assets.rs`：`AssetKind` = `Local \| Ssh \| Database \| Redis \| Docker` | 共享分组树、标签、生产标记；侧栏按 kind 过滤 |
| SQL | `src/db/`：`DbConn`、`DbKind`/`Family`、`SqlDriver` | sqlx / mssql / clickhouse / dm / 可选 duckdb |
| Redis | `src/kv/`：`RedisConn` | 缓存/KV 工作区 |
| Docker | `src/dock/`：`DockerEndpoint` | bollard |
| Mongo | `DbKind::Mongodb` 但 `Family::Other`、非 `SqlDriver`；UI 在 `src/doc/` + `doc_view.rs` | **模型与 UI 分裂** |
| 隧道 / 密钥 / 策略 | `ssh_via`、keyring（如 `db:{id}:password`）、`read_only`、TLS、production → `policy` / `audit` | 所有新连接必须对齐 |
| 侧栏模块 | Host/Terminal、SFTP、Database、Redis、Docker、Homebrew(mac)、Ports、Tools | 一族一入口已是产品习惯 |
| AI / MCP | `ai_ctx`、`db_mcp/` 等 | DB MCP 侧重 MySQL/达梦；中间件 MCP 需新边界 |

### 2.2 缺口

| 目标 | 现状 | 差距 |
|------|------|------|
| Elasticsearch / OpenSearch | 无 `AssetKind`、无 Adapter、无工作区 | 全缺 |
| VictoriaMetrics | 无 | 全缺（可复用「查询编辑器 + 结果」壳，但协议不同） |
| Kafka | 无 | 全缺；需消息窥探护栏 |
| MongoDB | 半有：DbKind 挂名 + doc 模块 | 未进统一注册表语义；侧栏归属模糊 |
| 扩展更多中间件 | 无 Registry / Adapter 约定 | 每加一个易复制粘贴 |

### 2.3 Mongo 分裂问题（P0 必须收口）

- 配置/类型上像「Database」的一种 `DbKind`；
- 运行时不是 `SqlDriver`；
- UI 走 `doc/` 独立文档工作区。

**收口原则**：Mongo 归入 **Document 族**（推荐新增 `AssetKind::Document`，或明确 `Database` 仅表示 Relational/SQL，Document 单独 kind）。doc 模块保留为 Document 工作区实现，连接记录进入统一 Asset + Registry，而不是继续「DbKind 挂羊头」。

---

## 3. 推荐架构

### 3.1 明确推荐（主方案）

```
┌─────────────────────────────────────────────────────────┐
│  Sidebar（一族一入口，filter AssetKind）                 │
│  Host | SFTP | Database(SQL) | Document | Redis | …     │
└─────────────┬───────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────┐
│  assets.rs  Asset { id, kind, group, tags, production…} │
│  + ConnMeta { ssh_via, tls, read_only, secret_refs…}    │
└─────────────┬───────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────┐
│  Connection Registry（薄层）                             │
│  AssetId → dyn FamilyAdapter / 具体句柄                  │
│  生命周期：connect / health / disconnect / tunnel bind   │
└─────────────┬───────────────────────────────────────────┘
              │
     ┌────────┼────────┬──────────┬────────────┐
     ▼        ▼        ▼          ▼            ▼
 Relational Search  Metrics  Messaging  Document  Cache  Container
 (SqlDriver) (ES)   (VM)     (Kafka)    (Mongo)  (Redis) (Docker)
     │        │        │          │            │
     ▼        ▼        ▼          ▼            ▼
  db_view   search_   metrics_  messaging_   doc_view
            view      view      view
```

**要点：**

1. **扩展 `AssetKind`，不平行建库**  
   建议新增（命名可在实现时微调，语义固定）：
   - `Document` — MongoDB（及日后 Couch 等）
   - `Search` — Elasticsearch / OpenSearch
   - `Metrics` — VictoriaMetrics（及日后 Prometheus 兼容端点）
   - `Messaging` — Kafka（及日后 NATS/Pulsar 等）
   - 保留现有 `Local | Ssh | Database | Redis | Docker`

2. **Connection Registry**  
   - 不是第二套持久化；持久化仍是 Asset + `config.toml` 段 + keyring。  
   - Registry 负责：按 kind 分发、持有运行时连接、统一 health、统一套用 `ssh_via`。

3. **按协议族 Adapter trait（示意，非最终 API）**

   | 族 | Trait 重心 | 现有锚点 |
   |----|------------|----------|
   | Relational | 已有 `SqlDriver` | `src/db/` |
   | Document | list collections、find/aggregate、explain | `src/doc/` |
   | Search | cluster health、index tree、query DSL/_sql | 新建 `src/search/` |
   | Metrics | label browse、instant/range query、只读 rules | 新建 `src/metrics/` |
   | Messaging | brokers/topics/CG、peek（护栏）、可选受控 produce | 新建 `src/messaging/` |
   | Cache | 已有 Redis | `src/kv/` |
   | Container | 已有 Docker | `src/dock/` |

   共享超集能力（可选小 trait）：`Connect`、`Health`、`Disconnect`、`TunnelAware`、`PolicyGate`。

4. **共享 UX 壳**  
   连接树、分组、标签、生产旗标、凭据对话框、SSH 隧道、只读开关、权限引擎钩子、AI 上下文（`ai_ctx`）挂载点 —— 全部复用/抽取，不按族复制。

5. **各族专用工作区**  
   - Search：索引树 + DSL 编辑器 + 命中表/JSON  
   - Metrics：PromQL/MetricsQL + 瞬时表 + 区间折线  
   - Messaging：Topic/Partition/CG + 受限 peek 流  
   - Document：对齐现有 `doc_view`  
   - 禁止「一个 Grid 打天下」。

### 3.2 备选方案（不推荐作主路径）

| 备选 | 内容 | 为何次选 |
|------|------|----------|
| B1. 单一 `AssetKind::Service` + `protocol` 枚举 | 侧栏只留「连接」 | 颠覆现有 filter-by-kind；`main.rs`/sidebar 改动面大 |
| B2. 每族独立 Store（`EsStore`/`KafkaStore`） | 最快出 ES Demo | 与 assets 重复；隧道/权限/审计必漂移 |
| B3. 仅 WASM 扩展实现中间件 | 零改核心 | 深度 UI/隧道/keyring 集成弱；不适合当主连接库 |

**决策：主方案 A；B1 仅当用户明确要求「一个侧栏看全部」时再评估迁移。**

### 3.3 与 `assets.rs` 的约束

- 新 kind **必须**进入 `AssetKind` 与侧栏过滤，不能只活在配置里。  
- 分组树、tags、production 语义与现有一致。  
- 密钥 **禁止**进 `config.toml`；沿用 keyring 命名（建议约定见 §5）。  
- 不发明与现有 Asset 字段冲突的第二主键；连接 ID = Asset id。

---

## 4. 首批协议能力矩阵

图例：`MVP` = 首个可合并版本；`后` = 随后迭代；`—` = 明确不做或极低优先。

### 4.1 Elasticsearch / OpenSearch（`AssetKind::Search`）

| 能力 | MVP | 后 | 备注 |
|------|-----|----|------|
| 连接（URL、TLS、Basic/ApiKey、`ssh_via`） | ✓ |  | ApiKey 进 keyring |
| 集群健康（status / nodes 摘要） | ✓ |  | |
| 索引树（indices、别名展开） | ✓ |  | |
| 查询 DSL | ✓ |  | 默认只读索引偏好 |
| `_sql`（若集群支持） |  | ✓ | |
| 文档抽样（match_all / 按 id） | ✓ |  | 结果条数上限 |
| 别名管理 | 只读查看 | 写别名 | 写走 policy |
| ILM / 分片重分配 / 危险 DELETE | — | 可选强确认 | 默认隐藏 |
| OpenSearch 差异 | 基础兼容 | 特性探测 | 同一 Adapter 内分支 |

### 4.2 VictoriaMetrics（`AssetKind::Metrics`）

| 能力 | MVP | 后 | 备注 |
|------|-----|----|------|
| 连接（HTTP(S)、`ssh_via`、可选 basic） | ✓ |  | |
| MetricsQL / PromQL 即时查询 | ✓ |  | |
| 区间查询 + 折线图 | ✓ |  | 复用轻量图表，非 Grafana |
| 标签 / `__name__` 浏览 | ✓ |  | |
| 告警规则 | 只读列表 |  | 优先只读 |
| 写入 / import / delete_series | — | 强 policy | 默认关闭 |
| 多租户路径前缀 |  | ✓ | |

### 4.3 Kafka（`AssetKind::Messaging`）

| 能力 | MVP | 后 | 备注 |
|------|-----|----|------|
| 连接（bootstrap、TLS、SASL、`ssh_via`） | ✓ |  | |
| Broker / Topic / Partition 元数据 | ✓ |  | |
| 消费组列表与 lag 摘要 | ✓ |  | |
| 消息窥探（按 offset/时间、限条数、限字节） | ✓ |  | **强制护栏** |
| 受控单条 produce（非默认） |  | ✓ | 需非 production 或显式允许 |
| 批量 produce / 压测 | — | — | 产品级拒绝 |
| Schema Registry |  | ✓ | 可选二级连接 |
| 管理类（删 topic、改 ACL） | — | 强确认 | 默认不做 |

### 4.4 MongoDB（`AssetKind::Document`，收口）

| 能力 | MVP | 后 | 备注 |
|------|-----|----|------|
| 迁入 Registry + Document kind | ✓ |  | 对齐现有 `doc/` |
| 集合树 / 文档浏览 / 查询 | ✓（已有） | 增强 | |
| 与 `DbKind::Mongodb` 脱钩或标 deprecated | ✓ |  | 避免 SQL 路径误用 |
| 聚合构建器 / explain |  | ✓ | |
| 写操作 | 跟现有 policy |  | production 默认只读可配 |

---

## 5. 配置模型草案

原则：**结构可发现、密钥永不落盘明文、URI 可导入。**

### 5.1 推荐：按族分段（可读性优先）

```toml
# 示意 — 非最终字段集；实现时与 config.rs 现有风格对齐

[[document]]
id = "mongo-prod"
name = "订单 Mongo"
group = "生产/订单"
tags = ["mongo", "orders"]
production = true
read_only = true
ssh_via = "bastion-1"          # 可选，复用 SSH Asset id
uri_import = "mongodb://..."   # 导入时解析后丢弃密码到 keyring

[[search]]
id = "es-logs"
name = "日志 ES"
endpoints = ["https://es.internal:9200"]
tls_mode = "system"            # 与现有 DB TLS 枚举对齐风格
ssh_via = "bastion-1"
production = true
read_only = true
# 密码/ApiKey：keyring `search:{id}:password` / `search:{id}:api_key`

[[messaging]]
id = "kafka-core"
name = "核心 Kafka"
bootstrap = ["kafka-1:9092", "kafka-2:9092"]
security_protocol = "SASL_SSL"
sasl_mechanism = "SCRAM-SHA-256"
ssh_via = "bastion-1"
peek_max_messages = 50
peek_max_bytes = 1048576
allow_produce = false

[[metrics]]
id = "vm-main"
name = "VictoriaMetrics"
base_url = "https://vm.internal"
ssh_via = "bastion-1"
query_timeout_secs = 30
```

### 5.2 备选：统一 `[[service]]`

```toml
[[service]]
id = "es-logs"
kind = "search"   # 对应 AssetKind
# …公共字段 + 族扩展表
```

统一段利于代码生成，**TOML 手写体验较差**。推荐实现层内部用统一结构，**文件层按族分段**（或导出时按族）。

### 5.3 密钥约定（对齐现有 `db:{id}:password`）

| 族 | keyring 键示例 |
|----|----------------|
| Document | `document:{id}:password`（迁移期可读 `db:{id}:password`） |
| Search | `search:{id}:password`、`search:{id}:api_key` |
| Messaging | `messaging:{id}:sasl_password`、`messaging:{id}:ssl_key` |
| Metrics | `metrics:{id}:password`、`metrics:{id}:bearer` |

**禁止**写入 `config.toml`、日志、审计正文（与 PROJECT_BRIEF 硬约束一致）。

### 5.4 URI 导入

- `mongodb://` / `mongodb+srv://` → Document  
- `http(s)://...` + 用户声明或探测 → Search / Metrics  
- `kafka://` 或 bootstrap 列表粘贴 → Messaging  
导入向导：解析 → 创建 Asset → 密钥进 keyring → 原文密码从内存清零。

---

## 6. UI 信息架构

### 6.1 推荐：一族一入口（与现网一致）

左侧继续模块化：

| 入口 | AssetKind 过滤 | 工作区 |
|------|----------------|--------|
| 主机 / 终端 | Local, Ssh | term |
| SFTP | （经 SSH） | sftp |
| 数据库 | Database（SQL only） | db_* |
| **文档** | Document | doc_*（收口 Mongo） |
| Redis | Redis | kv_* |
| Docker | Docker | dock_* |
| **搜索** | Search | search_*（新） |
| **消息** | Messaging | messaging_*（新） |
| **指标** | Metrics | metrics_*（新） |
| Ports / Tools / Brew | 现有 | 现有 |

**迁移成本：** 中等——`sidebar` / `main` 增模块；Mongo 从 Database 心智迁到 Document 需一次引导文案与配置迁移。  
**收益：** 与现有 UX 同构；用户不会在「万能表」里迷路。

### 6.2 备选：统一数据平面 + 类型过滤

- 单一「连接库」入口 + Segmented（SQL / Doc / Search / …）  
- 适合连接数极大、跨族跳转频繁的用户  
- **成本高**（信息架构重做、快捷键、空态、onboarding）  
- 建议 **P3 之后** 以「总览页」形式试验，不替换各族入口。

### 6.3 共享壳交互清单

- 树：分组 / 搜索 / 生产标记视觉  
- 连接对话框：公共字段 + 族特定字段页签  
- 隧道状态指示（复用 ssh_via）  
- 只读徽标；写操作二次确认 + policy  
- `#` 提及 / AI 上下文：注入「当前连接元数据 + 只读查询结果摘要」

---

## 7. 权限 / 审计 / MCP

### 7.1 Policy

- 所有写路径（ES 写索引、Kafka produce、Mongo 写、VM delete_series）走现有 `policy` / `managed` 管道。  
- `production = true` 默认：写拒绝或强制确认（产品可配）。  
- `read_only = true`：Adapter 层直接禁用写 API。

### 7.2 审计

- 连接、断开、查询（可采样）、peek、写尝试 → `audit` JSONL。  
- 审计 **不含** 密钥与完整敏感报文；peek 只记 topic/partition/offset/条数。

### 7.3 未来 MCP 边界（只读优先）

| 允许 | 不允许（默认） |
|------|----------------|
| list connections（脱敏） | 读 keyring |
| health / 元数据树 | 任意写 |
| 受参数限制的查询（条数/字节/超时） | 无限制 scan / produce 轰炸 |
| 解释性摘要给 Agent | 绕过 production 标记 |

新 MCP 模块建议：`search_mcp` / `metrics_mcp` / `messaging_mcp`，模式对齐 `db_mcp/`，且默认只挂只读工具。

---

## 8. 分阶段路线图

### P0 — 骨架 + Mongo 收口

- 扩展 `AssetKind`（至少 `Document`；为 Search/Messaging/Metrics 留枚举位或同期加上）。  
- Connection Registry 薄层 + `ConnMeta` 抽取。  
- Mongo：配置迁移、`doc/` 挂 Registry、Database 侧不再假装 SQL。  
- 设计/扩展点文档落入 fork 的 `docs/`。  
- **验收：** 旧 Mongo 连接可迁移；隧道/只读/生产标记行为不回归。

### P1 — Elasticsearch / OpenSearch

- `src/search/` Adapter + 侧栏入口 + 工作区。  
- MVP 矩阵 §4.1。  
- keyring 与 `ssh_via` 打通。  
- **验收：** 经堡垒机只读查询日志索引；生产标记下无法删除索引。

### P2 — Kafka

- `src/messaging/` + peek 护栏（max messages/bytes、默认禁止 produce）。  
- Topic/CG/lag 只读视图。  
- **验收：** 生产集群可安全 peek；无默认写入口。

### P3 — VictoriaMetrics

- `src/metrics/` + Instant/Range + 标签浏览 + 规则只读。  
- 轻量折线（非 Grafana）。  
- **验收：** MetricsQL 区间图可用；写 API 默认关闭。

### P4+ — 扩展点固化

- Adapter 编写指南、配置 schema、UI 壳复用清单。  
- 候选：OpenSearch 深度、Schema Registry、Prometheus 兼容、Pulsar/NATS、ClickHouse 已在 SQL 族加强等。  
- 可选「连接总览」页（统一平面试验）。

```
P0 骨架+Mongo ──► P1 ES ──► P2 Kafka ──► P3 VM ──► 扩展文档/更多 Adapter
```

---

## 9. 风险与原则

### 9.1 体积与依赖（Rust 客户端选型原则）

1. **优先**纯 Rust、维护活跃、TLS 可插拔的库；评估 compile time 与二进制增量。  
2. **避免**为 Demo 引入巨型 C 依赖或完整 JVM 桥。  
3. Kafka：选成熟 Rust 客户端；若某功能缺口，MVP 可先元数据 + 受限 consumer，不硬上 admin 全家桶。  
4. ES：HTTP + serde JSON 往往比「官方全功能客户端」更可控。  
5. VM：HTTP + Prometheus 查询 API，依赖面最小。  
6. Feature flag：`search` / `messaging` / `metrics` 可按发布包裁剪（对齐 duckdb 可选特性思路）。

### 9.2 协议差异

- ES vs OpenSearch API 漂移 → 能力探测 + 降级。  
- Kafka 安全协议组合多 → 配置向导分步，失败给可复制诊断（无密钥）。  
- VM 单节点 vs 集群/vmselect → `base_url` 语义写清。

### 9.3 运维误操作

- Peek/查询默认限额；导出二次确认。  
- 生产标记视觉强提示。  
- 危险操作默认不进入 UI 信息架构（见 §4）。

### 9.4 工程风险

- `main.rs` / sidebar 极大：新模块用符号搜索局部接入，禁止整文件翻写。  
- AGPL-3.0：fork 与分发需合规；第三方客户端许可证记入 `THIRD_PARTY_NOTICES`.  
- 无上游写权限：实现落在 `King-bj` fork / 本地，不推 `daxiangme/ShellAX`。

---

## 10. 待用户拍板（实现前）

1. **侧栏：** 确认采用「一族一入口」新增 文档/搜索/消息/指标，还是希望更早做「统一连接总览」？  
2. **Mongo 归属：** 同意新增 `AssetKind::Document` 并从 `DbKind::Mongodb` 迁移，还是暂时保留 DbKind 挂名仅加 Registry？  
3. **Kafka 写：** 是否允许在非 production 连接上提供「受控单条 produce」，还是全版本只读 peek？

---

## 11. 参考锚点（实现时优先打开）

| 主题 | 路径 |
|------|------|
| 资产模型 | `src/assets.rs` |
| SQL | `src/db/` |
| Mongo UI | `src/doc/`、`doc_view.rs` |
| Redis | `src/kv/` |
| Docker | `src/dock/` |
| 配置 / 密钥 | `src/config.rs`、`src/secrets.rs` |
| 权限 / 审计 | `src/policy.rs`、`src/audit.rs` |
| 侧栏 / 主状态 | `src/sidebar.rs`、`src/main.rs` |
| DB MCP 模式 | `src/db_mcp/` |
| 开发速查 | `docs/PROJECT_BRIEF.md`（本机与 bot store） |

---

*本文为设计稿；实现须另开任务，并遵守 clippy 零警告与密钥硬约束。*
