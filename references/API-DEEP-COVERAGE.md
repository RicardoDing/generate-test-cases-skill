# 接口深度覆盖补充方法

本文档定义接口测试的 5 种补充覆盖方法，在核心 5 种方法（EP/BVA/PV/ST/EG）完成后执行，用于弥补常见的用例遗漏场景。

这些方法来源于对大量过往项目用例的分析，发现以下场景是 AI 生成用例时最容易遗漏的。

---

## 方法总览

| 方法 | 标记 | 核心动作 | 触发条件 |
|------|------|---------|---------|
| **出参验证** | `RV` | 逐字段验证响应体结构完整性 | **所有接口**的正向用例 |
| **业务状态穷举** | `BS` | 对关键业务实体的每个状态枚举值生成独立用例 | 接口行为依赖实体状态 |
| **关联关系验证** | `RL` | 验证凭证/资源的归属匹配与不匹配行为 | 接口涉及凭证与资源的绑定关系 |
| **数据库字段验证** | `DB` | 验证表结构定义 + 写操作后数据库记录正确性 | 需求涉及新建表/新增字段，或接口含写操作 |
| **Redis缓存验证** | `REDIS` | 验证缓存生成/值/TTL/更新策略/删除行为 | 需求涉及缓存操作，或接口含Token生成/刷新/注销 |

---

## 1. 出参验证 (Response Validation, RV)

### 定义

对接口响应体的**每个字段**进行存在性和正确性验证。过往项目经验表明，仅验证接口调通（status_code=200）远远不够，需要逐字段确认响应结构完整。

### 触发条件

- **所有接口**的正向用例（基本流成功场景）都必须附带出参验证
- 接口文档中定义了响应结构（response body schema）

### 生成规则

#### 1.1 接口调通验证（必须，1 条）

每个接口生成 1 条「接口调通」用例，验证：
- 接口可正常调用
- 返回完整的响应体 JSON 结构
- 预期结果中列出完整的响应体示例（含字段名和示例值）

#### 1.2 响应字段逐一验证（必须，每个字段 1 条）

对响应体中的**每个字段**（包括嵌套字段）生成独立用例：

| 字段层级 | 用例标题格式 | 示例 |
|---------|-------------|------|
| 顶层字段 | `RV-出参验证: {接口名} - 响应包含{字段名}字段` | `RV-出参验证: 获取用户信息 - 响应包含data字段` |
| 嵌套字段 | `RV-出参验证: {接口名} - 响应包含{父字段}-{子字段}字段` | `RV-出参验证: 获取用户信息 - 响应包含data-userName字段` |
| 分页字段 | `RV-出参验证: {接口名} - 响应包含page-{字段}字段` | `RV-出参验证: 查询列表 - 响应包含page-total字段` |

#### 1.3 分页接口额外验证

分页接口必须额外验证以下字段的存在性：
- `total`（或 `totalCount`）
- `pageNum`（或 `page`）
- `pageSize`（或 `size`）
- `list`（或 `data`、`records`）

### 优先级

- 接口调通：**L1**
- 顶层字段验证：**L1**
- 嵌套字段验证：**L1**

### 示例

```
接口：POST /api/lapp/friend/list
响应体：
{
  "msg": "操作成功",
  "code": "200",
  "data": {
    "inviteTime": 1,
    "friendId": "xxx",
    "name": "xxx"
  },
  "page": {
    "total": 0,
    "size": 10,
    "page": 0
  }
}

生成用例：
1. RV-出参验证: 获取好友列表 - 接口调通
2. RV-出参验证: 获取好友列表 - 响应包含msg字段
3. RV-出参验证: 获取好友列表 - 响应包含code字段
4. RV-出参验证: 获取好友列表 - 响应包含data字段
5. RV-出参验证: 获取好友列表 - 响应包含data-inviteTime字段
6. RV-出参验证: 获取好友列表 - 响应包含data-friendId字段
7. RV-出参验证: 获取好友列表 - 响应包含data-name字段
8. RV-出参验证: 获取好友列表 - 响应包含page字段
9. RV-出参验证: 获取好友列表 - 响应包含page-total字段
10. RV-出参验证: 获取好友列表 - 响应包含page-size字段
11. RV-出参验证: 获取好友列表 - 响应包含page-page字段
```

### 与现有方法的关系

- RV 用例**不与 PV/EP/BVA 去重**，因为 PV 验证的是入参，RV 验证的是出参
- 如果「响应结构验证」规则（SKILL.md 中已有）已覆盖了部分字段，RV 在此基础上补全未覆盖的字段
- RV 用例归入模块分组中的「出参验证」子目录

---

## 2. 业务状态穷举 (Business State Enumeration, BS)

### 定义

当接口的行为依赖于某个业务实体的状态时，对该实体的**每个可能状态**都生成独立的测试用例。过往项目经验表明，测试人员通常只测试正常状态，遗漏异常状态下的接口行为。

### 触发条件（满足任一即触发）

- 需求或接口文档中定义了实体状态枚举（如账号状态：available/frozen/deleted）
- 接口行为因实体状态不同而产生不同结果（如冻结账号调用接口返回特定错误码）
- 需求中出现"当XX状态为YY时"、"仅在XX状态下"等条件描述

### 生成规则

#### 2.1 识别状态实体

从需求/接口文档中提取所有状态枚举：

| 常见状态实体 | 典型枚举值 |
|-------------|-----------|
| 账号/应用状态 | available、frozen、notThrough、deleted、checkPending |
| 订单状态 | created、paid、shipped、completed、cancelled、refunded |
| 审批状态 | pending、approved、rejected、withdrawn |
| 设备状态 | online、offline、disabled |
| 任务状态 | waiting、running、success、failed、timeout |

#### 2.2 每个状态生成独立用例

对每个状态枚举值，生成 1 条独立用例，验证该状态下接口的行为：

| 场景 | 用例标题格式 | 优先级 |
|------|-------------|-------|
| 正常状态（基本流） | `BS-状态验证: {接口名} - {实体}{正常状态}时{预期行为}` | P0 |
| 异常状态 | `BS-状态验证: {接口名} - {实体}{异常状态}时返回{错误码}` | P1 |

#### 2.3 状态组合验证

当接口依赖多个实体的状态时，覆盖策略：
- 所有实体都正常的组合（1 条基本流）
- 逐个将每个实体置为异常状态，其他实体保持正常（每个异常状态 1 条）
- 不需要生成"多个实体同时异常"的用例（除非需求特别说明）

### 示例

```
实体：appInfo 状态枚举 = {available, frozen, notThrough, deleted, checkPending}
接口：POST /api/lapp/trust/device/token/get

生成用例：
1. BS-状态验证: 获取托管token - appInfo状态为available时获取成功（P0）
2. BS-状态验证: 获取托管token - appInfo状态为checkPending时获取成功（P0）
3. BS-状态验证: 获取托管token - appInfo状态为frozen时返回10015（P1）
4. BS-状态验证: 获取托管token - appInfo状态为notThrough时返回10015（P1）
5. BS-状态验证: 获取托管token - appInfo状态为deleted时返回10015（P1）
```

### 与现有方法的关系

- BS 用例与 EP 等价类有交集（状态枚举本质上是等价类划分），但 BS 强调**穷举每个状态值**，而 EP 可能只取代表值
- 如果 EP 已经为某个状态枚举生成了用例，BS 不重复生成，但需检查是否每个枚举值都已覆盖
- BS 用例归入模块分组中的「功能验证」子目录

---

## 3. 关联关系验证 (Relation Validation, RL)

### 定义

验证接口中凭证（Token/Cookie）与操作资源之间的归属关系是否正确校验。过往项目经验表明，"凭证与资源不匹配"是高频遗漏场景。

### 触发条件（满足任一即触发）

- 接口同时需要凭证（accessToken）和资源标识（如 appKey、deviceSerial、userId）
- 凭证代表的身份与资源存在归属关系（如"该 appKey 属于该 accessToken 对应的用户"）
- 需求中出现"属于""归属""关联""绑定"等描述

### 生成规则

#### 3.1 匹配场景（正向）

| 场景 | 用例标题格式 | 优先级 |
|------|-------------|-------|
| 凭证与资源匹配 | `RL-关联验证: {接口名} - {凭证}与{资源}匹配时操作成功` | P0 |

#### 3.2 不匹配场景（反向，重点）

| 场景 | 用例标题格式 | 优先级 |
|------|-------------|-------|
| 凭证与资源不匹配 | `RL-关联验证: {接口名} - {凭证}与{资源}不匹配时返回{错误码}` | P1 |
| 资源不存在 | `RL-关联验证: {接口名} - {资源}不存在时返回{错误码}` | P1 |
| 资源类型错误 | `RL-关联验证: {接口名} - {资源}为非{预期类型}时返回{错误码}` | P2 |

#### 3.3 常见关联关系模式

| 关联模式 | 匹配用例 | 不匹配用例 |
|---------|---------|-----------|
| Token ↔ AppKey | Token 对应用户拥有该 AppKey | Token 对应用户不拥有该 AppKey |
| Token ↔ DeviceSerial | Token 对应用户拥有该设备 | Token 对应用户不拥有该设备 |
| 主账号 ↔ 子账号 | 主账号操作自己的子账号 | 主账号操作其他主账号的子账号 |
| 用户 ↔ 订单 | 用户操作自己的订单 | 用户操作他人的订单 |

### 示例

```
接口：POST /lapp/user/weak/limit/get
入参：accessToken（超级账号Token）、weakAppKey（弱账号AppKey）

生成用例：
1. RL-关联验证: 查询弱账号并发 - accessToken与weakAppKey匹配时查询成功（P0）
2. RL-关联验证: 查询弱账号并发 - accessToken与weakAppKey不匹配时返回10065（P1）
3. RL-关联验证: 查询弱账号并发 - weakAppKey不存在时返回30005（P1）
4. RL-关联验证: 查询弱账号并发 - weakAppKey为非弱账号appKey时返回10001（P2）
```

### 与现有方法的关系

- RL 用例与 AUTH 越权测试有交集，但侧重点不同：
  - AUTH 关注"用户A能否操作用户B的资源"（权限边界）
  - RL 关注"凭证与资源的绑定关系是否正确校验"（数据关联）
- 如果 AUTH 已覆盖了某个不匹配场景，RL 不重复生成
- RL 用例归入模块分组中的「功能验证」子目录

---

## 4. 数据库字段验证 (Database Validation, DB)

### 定义

验证数据库表结构（DDL）的正确性，以及写操作接口执行后数据库记录（DML）的变更是否符合预期。过往项目经验表明，接口返回成功不代表数据库落库正确——字段类型错误、约束缺失、记录未写入是高频遗漏场景。

### 触发条件（满足任一即触发）

- 需求涉及**新建数据库表**或**新增字段**（DDL 变更）
- 接口包含**写操作**（创建/修改/删除），需验证数据库记录变更
- 需求文档或设计文档中提到了具体的表名、字段名
- 接口涉及**跨表联动**（如创建订单同时更新库存表）

### 生成规则

DB 验证分为两个子类型：**DDL 结构验证**和 **DML 数据验证**。

#### 4.1 DDL 结构验证（表结构）

当需求涉及新建表或新增字段时，对**每个字段**生成独立的结构验证用例。

**验证维度**（逐字段检查）：

| 验证项 | 说明 | 示例预期结果 |
|--------|------|-------------|
| 字段类型 | 数据类型是否符合设计 | `验证字段类型为 varchar(64)` |
| 非空约束 | NOT NULL 是否正确设置 | `验证字段 NOT NULL` |
| 唯一约束 | UNIQUE 是否正确设置 | `验证字段具备 UNIQUE 唯一性约束` |
| 默认值 | DEFAULT 值是否正确 | `验证默认值为 CURRENT_TIMESTAMP` |
| 自增 | AUTO_INCREMENT 是否设置 | `验证具备 AUTO_INCREMENT 自增特性` |
| 索引 | 索引是否正确创建 | `验证字段上存在索引` |
| 外键 | 外键关联是否正确 | `验证外键关联到 {关联表}.{关联字段}` |
| ON UPDATE | 更新时自动行为 | `验证具备 ON UPDATE CURRENT_TIMESTAMP 特性` |

**用例标题格式**：`DB-结构验证: {库名}.{表名} - {字段名}`

**用例名称**：直接使用字段名（如 `id`、`open_code`、`create_time`）

**预期结果格式**：`验证字段类型为 {类型}，{约束1}，{约束2}`

#### 4.2 DML 数据验证（记录变更）

当接口包含写操作时，验证操作后数据库记录的变更是否正确。

**按操作类型生成**：

| 操作类型 | 验证内容 | 用例标题格式 | 优先级 |
|---------|---------|-------------|-------|
| **创建** | 新增记录存在 + 关键字段值正确 | `DB-数据验证: {接口名} - {表名}新增记录` | L1 |
| **更新** | 目标字段已更新 + 非目标字段未变 | `DB-数据验证: {接口名} - {表名}记录更新` | L1 |
| **删除** | 记录已删除（硬删）或状态已变更（软删） | `DB-数据验证: {接口名} - {表名}记录删除` | L1 |
| **跨表联动** | 多张表的记录同步变更 | `DB-数据验证: {接口名} - {表A}和{表B}联动更新` | L1 |

**预期结果要求**：
- 创建操作：`{库名}.{表名}新增1条记录` 或 `{库名}.{表名}新增1条记录，{字段名}={预期值}`
- 更新操作：`{库名}.{表名}记录{字段名}字段更新为{预期值}`
- 删除操作：`{库名}.{表名}对应记录被删除` 或 `{库名}.{表名}记录status字段更新为{删除状态值}`
- 联动操作：逐表描述变更，如 `1.{表A}新增记录 2.{表B}对应记录更新`

### 优先级

| 子类型 | 优先级 | 说明 |
|--------|-------|------|
| DDL 结构验证 - 主键/唯一约束 | **L1** | 约束缺失会导致数据异常 |
| DDL 结构验证 - 其他字段 | **L1** | 字段类型错误影响业务逻辑 |
| DML 数据验证 - 创建/更新/删除 | **L1** | 核心写操作必须验证落库 |
| DML 数据验证 - 跨表联动 | **L1** | 数据一致性关键场景 |

### 示例

#### DDL 结构验证示例

```
需求：新建 open_app_code_user 表，包含 id、open_code、open_user_id、create_time、update_time 字段

生成用例（子功能=数据库验证，测试项={库名}.{表名}）：
1. DB-结构验证: open.open_app_code_user - id
   预期：验证字段类型为 bigint unsigned，NOT NULL，具备 AUTO_INCREMENT 自增特性
2. DB-结构验证: open.open_app_code_user - open_code
   预期：验证字段类型为 varchar(64)，NOT NULL，具备 UNIQUE 唯一性约束
3. DB-结构验证: open.open_app_code_user - open_user_id
   预期：验证字段类型为 varchar(64)，允许 NULL，具备 UNIQUE 唯一性约束
4. DB-结构验证: open.open_app_code_user - create_time
   预期：验证字段类型为 datetime，NOT NULL，默认值为 CURRENT_TIMESTAMP
5. DB-结构验证: open.open_app_code_user - update_time
   预期：验证字段类型为 datetime，默认值为 CURRENT_TIMESTAMP 且具备 ON UPDATE CURRENT_TIMESTAMP 特性
```

#### DML 数据验证示例

```
接口：POST /api/lapp/trust/device/token/get（获取托管token）
写操作：成功时在 openauth.user_access_token 表新增1条记录

生成用例（子功能=数据库验证，测试项={库名}.{表名}）：
1. DB-数据验证: 获取托管token - openauth.user_access_token新增记录
   前置条件：appInfo状态=available
   步骤：1.调用POST /api/lapp/trust/device/token/get接口，生成token成功
   预期：openauth.user_access_token表新增1条记录

接口：POST /api/lapp/trust/info/change（修改托管信息）
写操作：更新 open.open_user_info 表对应记录

生成用例：
1. DB-数据验证: 修改托管信息 - open.open_user_info记录更新
   步骤：1.调用POST /api/lapp/trust/info/change接口
   预期：更新对应的开发者的appinfo信息

接口：POST /api/lapp/trust/cancel（解除设备托管）
写操作：联动更新多张表

生成用例：
1. DB-数据验证: 解除设备托管 - openauth.ram_resource托管数据表更新
   预期：openauth.ram_resource数据表更新
2. DB-数据验证: 解除设备托管 - alarm.alarm_event报警关联关系更新
   预期：alarm.alarm_event数据表更新
3. DB-数据验证: 解除设备托管 - open.cancel_trust_history记录新增
   预期：open.cancel_trust_history数据表新增记录
```

### 信息来源

DB 验证用例的信息来源（按优先级）：

1. **需求文档中的表结构设计**：DDL 语句、ER 图、字段定义表
2. **设计文档中的数据模型**：实体关系描述、字段约束说明
3. **代码中的 Entity/DAO 层**：`@Column`、`@Table` 注解、Mapper XML
4. **接口文档中的副作用说明**：如"调用成功后会在XX表新增记录"

> **注意**：严禁仅凭字段名称推断类型（如看到 create_time 就推断为 datetime）。所有 DDL 验证的预期结果必须有明确的文档或代码依据。

如果需求文档中**未提供表结构信息**，则：
- DDL 结构验证：跳过，不生成（无法凭空推断字段类型），在用例备注中标注"缺少表结构信息"
- DML 数据验证：仅生成"新增/更新/删除记录"级别的用例，不细化到字段值

### 与现有方法的关系

- DB 用例与 RV 出参验证互补：RV 验证接口返回值，DB 验证数据库落库值
- 写操作接口建议同时生成 RV + DB 用例，形成"接口返回正确 + 数据库落库正确"的双重验证
- DB 用例归入模块分组中的「数据库验证」子目录


---

## 5. Redis 缓存验证 (Redis Cache Validation, REDIS)

### 定义

验证接口操作后 Redis 缓存的生成、值、过期时间、更新策略和删除行为是否符合预期。过往项目经验表明，接口返回成功且数据库落库正确，但缓存未正确写入/更新/删除是高频遗漏场景——缓存与数据库不一致会导致用户看到脏数据。

### 触发条件（满足任一即触发）

- 需求文档或设计文档中提到了 Redis 缓存 key（如 `OPENAPI_ACCESS_TOKEN_{token}`、`hlp:doc:cen:{yyyymmdd}{docId}{userid}`）
- 接口操作会触发缓存的生成、更新或删除
- 需求中出现"缓存"、"Redis"、"cache"、"TTL"、"过期时间"等关键词
- 接口涉及 Token 生成/刷新/注销（Token 通常缓存在 Redis 中）
- 设计文档中的 Redis 设计表格列出了缓存 key 及其行为

### 生成规则

REDIS 验证按缓存 key 维度生成，每个缓存 key 最多生成 **5 个维度**的验证用例。

#### 5.1 缓存验证 5 维度

对每个需求中涉及的缓存 key，按以下 5 个维度逐一生成用例：

| 维度 | 用例名称 | 验证内容 | 验证命令 | 优先级 |
|------|---------|---------|---------|-------|
| **缓存生成** | `缓存生成` | 操作后缓存是否成功创建 | `GET {key}` 或 `EXISTS {key}` | L1 |
| **缓存值** | `缓存值` | 缓存内容是否与预期一致 | `GET {key}` 并比对值 | L1 |
| **缓存时间** | `缓存时间` | TTL 是否符合设计要求 | `TTL {key}` | L1 |
| **更新策略** | `更新策略` | 重复操作后缓存是否按预期更新或不更新 | `GET {key}` 比对前后值 | L2 |
| **缓存删除** | `缓存删除` 或 `主动删除` 或 `过期删除` | 特定操作后缓存是否被正确删除 | `EXISTS {key}` 返回 0 | L1 |

> **注意**：并非每个缓存 key 都需要生成全部 5 个维度。根据需求描述按需生成：
> - 只读缓存（查询接口触发）：通常生成 缓存生成 + 缓存值 + 缓存时间 + 更新策略
> - 写操作缓存（创建/修改接口触发）：通常生成 缓存生成 + 缓存值 + 缓存时间
> - 缓存清除场景（删除/注销接口触发）：通常生成 缓存删除
> - 如果需求明确提到了某个维度，则必须生成对应用例
>
> **更新策略维度的生成条件**（仅在以下情况生成，避免推测）：
> - 需求明确说明"重复操作不更新缓存" → 生成更新策略用例，预期为"不更新"
> - 需求明确说明"重复操作会更新缓存并重置 TTL" → 生成更新策略用例，预期为"更新缓存值，将缓存时间重置为{N}s"
> - 需求未提及更新策略 → **不生成**该维度用例

#### 5.2 用例格式规范

**子功能**：`缓存验证`

**测试项**：缓存 key 的模式（如 `OPENAPI_USER_BASICINFO{userId}`、`hlp:doc:cen:{yyyymmdd}{docId}{userid}`）

**用例标题格式**：`REDIS-缓存验证: {接口名} - {缓存key模式} - {维度}`

**用例名称**：直接使用维度名（如 `缓存生成`、`缓存值`、`缓存时间`、`更新策略`、`缓存删除`）

**步骤格式**：
- 缓存生成/值/时间：`1.调用{接口}成功 2.使用GET/TTL命令查询缓存`
- 更新策略：`1.调用{接口}成功 2.再次调用{接口} 3.使用GET命令查询缓存比对前后值`
- 缓存删除：`1.确认缓存存在 2.调用{触发删除的接口} 3.使用EXISTS命令确认缓存已删除`

**预期结果格式**：
- 缓存生成：`成功查询到缓存` 或 `{缓存key}缓存生成`
- 缓存值：具体的值描述（如 `是1`、`写入userBasicInfo的json值`、`是msgId + , + userId`）
- 缓存时间：`{N}s`（如 `3600s`、`86400s`、`2592000s`）
- 更新策略：`不更新` 或 `更新缓存值，将缓存时间重置为{N}s`
- 缓存删除：`缓存被删除` 或 `redis中{缓存key}缓存删除`

#### 5.3 缓存操作类型与用例映射

| 缓存操作类型 | 触发场景 | 必须生成的维度 | 可选维度 |
|-------------|---------|--------------|---------|
| **缓存写入**（首次生成） | 查询接口首次触发缓存 | 缓存生成、缓存值、缓存时间 | 更新策略 |
| **缓存更新**（覆盖写入） | 写操作接口更新缓存 | 缓存值、缓存时间 | 缓存生成（如果是新key） |
| **缓存删除**（主动清除） | 删除/注销/修改密码等接口 | 缓存删除 | - |
| **缓存过期**（自然失效） | TTL 到期 | 缓存时间 | 过期删除 |
| **缓存不变**（操作不影响缓存） | 某些操作不应改变缓存 | 更新策略（预期：不更新） | - |

### 优先级

| 维度 | 优先级 | 说明 |
|------|-------|------|
| 缓存生成 | **L1** | 缓存未生成会导致后续请求全部穿透到数据库 |
| 缓存值 | **L1** | 缓存值错误会导致用户看到脏数据 |
| 缓存时间 | **L1** | TTL 错误可能导致缓存永不过期或过早失效 |
| 更新策略 | **L2** | 更新策略错误影响数据一致性，但通常不会立即暴露 |
| 缓存删除 | **L1** | 缓存未删除会导致用户看到已失效的数据（如已注销的Token仍可用） |

### 示例

#### 示例 1：查询接口触发缓存生成

```
接口：POST /api/lapp/friend/list（获取好友列表）
缓存 key：OPENAPI_USER_BASICINFO{userId}
缓存行为：首次查询时生成缓存，写入 userBasicInfo 的 JSON 值，TTL=3600s，重复查询不更新

生成用例（子功能=缓存验证，测试项=OPENAPI_USER_BASICINFO{userId}）：
1. REDIS-缓存验证: 获取好友列表 - OPENAPI_USER_BASICINFO{userId} - 缓存生成
   步骤：1.3600s内调用Post_/api/lapp/friend/list接口，获取到用户信息 2.get命令查询缓存
   预期：生成缓存

2. REDIS-缓存验证: 获取好友列表 - OPENAPI_USER_BASICINFO{userId} - 缓存值
   步骤：1.3600s内调用Post_/api/lapp/friend/list接口，获取到用户信息 2.get命令查询缓存
   预期：写入userBasicInfo的json值

3. REDIS-缓存验证: 获取好友列表 - OPENAPI_USER_BASICINFO{userId} - 缓存时间
   步骤：1.3600s内调用Post_/api/lapp/friend/list接口，获取到用户信息 2.ttl命令查询缓存
   预期：3600s

4. REDIS-缓存验证: 获取好友列表 - OPENAPI_USER_BASICINFO{userId} - 更新策略
   步骤：1.3600s内调用Post_/api/lapp/friend/list接口，获取到用户信息 2.get命令查询缓存
   预期：不更新
```

#### 示例 2：写操作触发缓存更新

```
接口：POST /lapp/user/weak/limit/set（超级账号设置弱账号并发）
缓存 key：userappro#{liveType}#userId
缓存行为：设置成功后生成/更新缓存，TTL=3600s

生成用例（子功能=缓存验证，测试项=userappro#{liveType}#userId）：
1. REDIS-缓存验证: 设置弱账号并发 - userappro#{liveType}#userId - 缓存生成
   前置条件：userappro#{liveType}#userId不存在
   步骤：1.调用Post_/lapp/user/weak/limit/set接口 2.get命令查询缓存成功
   预期：缓存生成

2. REDIS-缓存验证: 设置弱账号并发 - userappro#{liveType}#userId - 缓存值
   步骤：1.调用Post_/lapp/user/weak/limit/set接口 2.get命令查询缓存成功
   预期：缓存值是该用户此次设置的值

3. REDIS-缓存验证: 设置弱账号并发 - userappro#{liveType}#userId - 缓存时间
   步骤：1.调用Post_/lapp/user/weak/limit/set接口 2.ttl命令查询缓存成功
   预期：3600s

4. REDIS-缓存验证: 设置弱账号并发 - userappro#{liveType}#userId - 缓存更新
   前置条件：userappro#{liveType}#userId存在
   步骤：1.调用Post_/lapp/user/weak/limit/set接口 2.get命令查询缓存成功
   预期：调用Post_/lapp/user/weak/limit/set接口会更新缓存值，将缓存时间重置为3600s
```

#### 示例 3：操作触发缓存删除

```
接口：POST /api/lapp/ram/account/updatePassword（修改子账户密码）
缓存 key：AS_ACCESS_TOKEN_{accountId}_{appKey}、AS_DEVICE_TRUST_TOKEN_{accountId}_{appKey}、OPENAPI_ACCESS_TOKEN_{accessToken}
缓存行为：修改密码成功后删除相关缓存

生成用例（子功能=缓存验证）：
1. REDIS-缓存验证: 修改子账户密码 - AS_ACCESS_TOKEN_{accountId}_{appKey} - 缓存删除
   前置条件：该子账号已经生成ra token，且缓存都存在
   步骤：1.调用Post_/api/lapp/ram/account/updatePassword接口
   预期：删除AS_ACCESS_TOKEN_{子账号accountId}_{appKey}缓存

2. REDIS-缓存验证: 修改子账户密码 - AS_DEVICE_TRUST_TOKEN_{accountId}_{appKey} - 缓存删除
   前置条件：该子账号已经生成ra token，且缓存都存在
   步骤：1.调用Post_/api/lapp/ram/account/updatePassword接口
   预期：删除AS_DEVICE_TRUST_TOKEN_{子账号accountId}_{appKey}缓存

3. REDIS-缓存验证: 修改子账户密码 - OPENAPI_ACCESS_TOKEN_{accessToken} - 缓存删除
   前置条件：该子账号已经生成ra token，且缓存都存在
   步骤：1.调用Post_/api/lapp/ram/account/updatePassword接口
   预期：删除OPENAPI_ACCESS_TOKEN_{accessToken}缓存
```

#### 示例 4：PV/UV 统计缓存

```
接口：POST /web/console/doc/center/doc/access/count（统计pv和uv）
缓存 key 1：hlp:doc:cen:{yyyymmdd}{docId}{userid}
缓存 key 2：hlp:doc:cen:all:{yyyymmdd}{userid}
缓存行为：userId不为空且未访问过时生成缓存，TTL=86400s

生成用例（子功能=缓存验证）：
--- 缓存 key 1: hlp:doc:cen:{yyyymmdd}{docId}{userid} ---
1. REDIS-缓存验证: 统计pv和uv - hlp:doc:cen:{yyyymmdd}{docId}{userid} - 缓存生成
   步骤：1.调用Post_/web/console/doc/center/doc/access/count接口（userId不为空）
   预期：成功查询到缓存

2. REDIS-缓存验证: 统计pv和uv - hlp:doc:cen:{yyyymmdd}{docId}{userid} - 缓存值
   步骤：1.调用Post_/web/console/doc/center/doc/access/count接口（userId不为空）
   预期：是1

3. REDIS-缓存验证: 统计pv和uv - hlp:doc:cen:{yyyymmdd}{docId}{userid} - 缓存时间
   步骤：1.调用Post_/web/console/doc/center/doc/access/count接口（userId不为空）
   预期：86400s

--- 缓存 key 2: hlp:doc:cen:all:{yyyymmdd}{userid} ---
4. REDIS-缓存验证: 统计pv和uv - hlp:doc:cen:all:{yyyymmdd}{userid} - 缓存生成
   步骤：1.调用Post_/web/console/doc/center/doc/access/count接口（userId不为空）
   预期：成功查询到缓存

5. REDIS-缓存验证: 统计pv和uv - hlp:doc:cen:all:{yyyymmdd}{userid} - 缓存值
   步骤：1.调用Post_/web/console/doc/center/doc/access/count接口（userId不为空）
   预期：是1

6. REDIS-缓存验证: 统计pv和uv - hlp:doc:cen:all:{yyyymmdd}{userid} - 缓存时间
   步骤：1.调用Post_/web/console/doc/center/doc/access/count接口（userId不为空）
   预期：86400s
```

### 信息来源

REDIS 验证用例的信息来源（按优先级）：

1. **设计文档中的 Redis 设计表格**：缓存 key 模式、值类型、TTL、更新/删除策略
2. **需求文档中的缓存描述**：如"调用成功后生成XX缓存"、"修改密码后清除Token缓存"
3. **代码中的 Redis 操作**：`redisTemplate.opsForValue().set()`、`@Cacheable` 注解等
4. **接口文档中的副作用说明**：如"调用成功后会刷新XX缓存"

如果需求文档中**未提供缓存 key 信息**，则：
- 仅根据接口类型生成粗粒度用例（如"写操作后相关缓存应更新"）
- 不凭空编造缓存 key 模式

### 与现有方法的关系

- REDIS 用例与 DB 数据库验证互补：DB 验证数据库落库，REDIS 验证缓存层
- 写操作接口建议同时生成 RV + DB + REDIS 用例，形成"接口返回正确 + 数据库落库正确 + 缓存正确"的三重验证
- REDIS 用例归入模块分组中的「缓存验证」子目录


---

## 用例去重规则

当深度覆盖方法（RV/BS/RL/DB/REDIS）与核心方法（EP/BVA/PV/ST/EG/AUTH）产生重叠时，基于**测试点**进行去重。如果已有用例验证了相同的测试点，则不重复生成。

### 测试点识别规则

| 方法 | 测试点 = | 示例 |
|------|---------|------|
| RV | {接口名} + {字段路径} | `获取用户信息` + `data-userName` |
| BS | {接口名} + {实体} + {状态值} | `获取托管token` + `appInfo` + `frozen` |
| RL | {接口名} + {凭证类型} + {资源类型} + {匹配/不匹配} | `查询弱账号并发` + `accessToken` + `weakAppKey` + `不匹配` |
| DB-DDL | {表名} + {字段名} | `open_app_code_user` + `open_code` |
| DB-DML | {表名} + {操作类型} | `user_access_token` + `新增记录` |
| REDIS | {缓存key模式} + {维度} | `OPENAPI_ACCESS_TOKEN_{token}` + `缓存生成` |

### 去重判断流程

1. 生成深度覆盖用例前，提取该用例的测试点
2. 在已生成的核心方法用例中查找是否存在相同测试点
3. 如果存在 → 跳过，不重复生成
4. 如果不存在 → 正常生成

---

## 覆盖率检查

Phase 2.8 质量预审时，额外检查以下覆盖率：

| 检查项 | 目标 | 未达标处理 |
|--------|------|-----------|
| 出参验证覆盖率 | 每个接口至少 1 条 RV 用例 | 高亮警告未覆盖接口 |
| 业务状态覆盖率 | 每个状态枚举的每个值至少 1 条 BS 用例 | 高亮警告未覆盖状态值 |
| 关联关系覆盖率 | 每对关联关系至少 1 条匹配 + 1 条不匹配用例 | 高亮警告未覆盖关联 |
| 数据库验证覆盖率 | 每个写操作接口至少 1 条 DB-DML 用例；新建表每个字段 1 条 DB-DDL 用例 | 高亮警告未覆盖的写操作接口或未验证的字段 |
| Redis缓存验证覆盖率 | 每个涉及缓存操作的接口至少 1 条 REDIS 用例；每个缓存 key 至少覆盖缓存生成+缓存值+缓存时间 3 个维度 | 高亮警告未覆盖的缓存 key 或未验证的维度 |

---

## 模块分组规则补充

RV/BS/RL 用例在模块分组中的归属：

| 方法 | 末级目录名 | 说明 |
|------|-----------|------|
| RV 出参验证 | `出参验证` | 与「入参验证」平级 |
| BS 业务状态穷举 | `功能验证` | 归入功能验证目录 |
| RL 关联关系验证 | `功能验证` | 归入功能验证目录 |
| DB 数据库字段验证 | `数据库验证` | DDL 和 DML 验证均归入此目录 |
| REDIS 缓存验证 | `缓存验证` | 缓存生成/值/时间/更新策略/删除验证均归入此目录 |

示例：
```json
{"模块": ["账号管理", "获取托管token", "出参验证"], "用例标题": "RV-出参验证: 获取托管token - 响应包含data-accessToken字段"}
{"模块": ["账号管理", "获取托管token", "功能验证"], "用例标题": "BS-状态验证: 获取托管token - appInfo状态为frozen时返回10015"}
{"模块": ["账号管理", "获取托管token", "功能验证"], "用例标题": "RL-关联验证: 获取托管token - accessToken与appKey不匹配时返回10065"}
{"模块": ["账号管理", "获取托管token", "数据库验证"], "用例标题": "DB-数据验证: 获取托管token - openauth.user_access_token新增记录"}
{"模块": ["账号管理", "获取托管token", "缓存验证"], "用例标题": "REDIS-缓存验证: 获取托管token - OPENAPI_ACCESS_TOKEN_{accessToken} - 缓存生成"}
```
