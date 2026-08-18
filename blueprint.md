# GoGoal 技能开发蓝图

> 文档类型：产品与技术开发蓝图  
> 当前状态：核心方案已收敛，Markdown 文档规范与看板页面设计待专项讨论  
> 最后整理日期：2026-08-14  
> 项目名称：GoGoal  
> Skill、Plugin、CLI、仓库及项目数据目录标识：`gogoal`

本文档整理 GoGoal 多轮方案讨论中已经确认的产品定位、工作流、数据格式、CLI、日志、看板技术架构、Git 约束、Skill 结构和开发要求，作为后续实现与验收的指导蓝图。

当前仅有以下两个部分尚未定版：

1. 目标 Markdown 与 AI 任务 Markdown 的具体章节、内容要求、更新时机和模板。
2. 看板页面的信息架构、视觉布局、组件、交互和最终展示设计。

在上述专项讨论完成前，实现不得擅自固化对应模板或页面设计。其他章节均按当前已收敛方案执行；后续如有变更，应同步修订本蓝图。

## 目录

1. [产品定义](#1-产品定义)
2. [命名与标识](#2-命名与标识)
3. [总体架构](#3-总体架构)
4. [领域对象与核心原则](#4-领域对象与核心原则)
5. [状态模型](#5-状态模型)
6. [目标与任务工作流](#6-目标与任务工作流)
7. [项目数据目录](#7-项目数据目录)
8. [配置文件标准](#8-配置文件标准)
9. [目标数据标准](#9-目标数据标准)
10. [任务数据标准](#10-任务数据标准)
11. [管理日志标准](#11-管理日志标准)
12. [Markdown 文档边界](#12-markdown-文档边界)
13. [CLI 设计](#13-cli-设计)
14. [一致性、并发与异常处理](#14-一致性并发与异常处理)
15. [Git 工作流](#15-git-工作流)
16. [看板技术架构](#16-看板技术架构)
17. [Skill 与开源仓库结构](#17-skill-与开源仓库结构)
18. [测试与质量保障](#18-测试与质量保障)
19. [开发阶段规划](#19-开发阶段规划)
20. [待专项决策事项](#20-待专项决策事项)

## 1. 产品定义

### 1.1 产品定位

GoGoal 是一个可安装的 AI 目标与任务管理 Skill。用户在项目中启用后，可以通过自然语言与 AI 共同完成目标分析、方案讨论、任务拆分、启动授权、自主实施、状态维护、验证、目标验收和归档，并通过只读可视化看板了解项目中全部目标与任务的整体情况。

GoGoal 不是独立于项目之外的通用项目管理平台。目标、任务、说明文档和管理日志均保存在项目仓库内，与实现代码一起接受 Git 版本管理。

### 1.2 要解决的问题

- 将已经验证有效的目标授权、AI 自主实施和目标级验收流程沉淀为可复用 Skill。
- 避免 AI 仅凭会话上下文管理长目标，降低上下文压缩、中断或切换会话导致的信息丢失。
- 通过确定性 CLI 维护编号、状态、时间、归档位置和日志，减少 AI 手工编辑结构化数据的错误。
- 通过 Markdown 保留适合人和 AI 阅读的目标分析、任务计划、实施记录和交付说明。
- 通过看板集中展示目标、AI 任务、用户任务、阻塞信息、归档数据和管理时间线。
- 通过 Git 保留文件变更、实现提交与管理动作的完整审计历史。

### 1.3 产品边界

GoGoal 第一阶段不包含：

- 中央数据库、网络数据库或云端多租户服务。
- 独立账号、组织、成员和权限系统。
- 从看板直接修改目标或任务。
- 跨设备无人值守调度器。
- 自动扩大 AI 权限或绕过宿主环境审批。
- 旧版 Markdown 表格格式的自动导入器。
- 强制性的验证附件或 `evidence/` 目录。
- 结构化验收标准及其自动覆盖率计算。

### 1.4 启用条件

只有用户明确要求将事项作为目标或任务管理，或明确要求使用 GoGoal 时，Skill 才建立正式记录。普通咨询、解释、检查或临时操作默认不进入 GoGoal。

项目尚未初始化时，Skill 只有在用户明确要求初始化或登记目标时才创建 `gogoal/` 数据目录，不得在任意项目中自动写入管理文件。

## 2. 命名与标识

所有公开和内部标识统一使用 `gogoal`，不再增加 `ops` 后缀：

| 用途 | 名称 |
| --- | --- |
| 产品展示名称 | `GoGoal` |
| GitHub 仓库 | `gogoal` |
| Plugin ID | `gogoal` |
| Skill 名称 | `gogoal` |
| CLI 命令 | `gogoal` |
| 项目数据目录 | `gogoal/` |
| Skill 目录 | `skills/gogoal/` |

目标和任务详情文件使用仅包含编号的稳定文件名：

```text
gogoal/targets/1.md
gogoal/tasks/1.md
```

编号永久冻结，标题允许修改。标题修改不引起文档重命名，因此文档路径和外部链接保持稳定。

## 3. 总体架构

### 3.1 组件

```text
用户自然语言
    ↓
GoGoal Skill
    ├── 判断是否进入目标管理
    ├── 分析目标和拆分任务
    ├── 控制授权、执行与验收流程
    ├── 编写目标和 AI 任务 Markdown
    └── 调用 GoGoal CLI
             ↓
        GoGoal CLI
        ├── 查询和修改四个聚合 JSON
        ├── 维护 log.json
        ├── 校验状态与文件一致性
        ├── 提供紧凑上下文
        └── 提供只读看板服务
             ↓
        项目内 gogoal/ 数据
             ↓
        Git 审计与只读看板
```

### 3.2 组件职责

| 组件 | 核心职责 |
| --- | --- |
| `SKILL.md` | 定义触发条件、核心操作顺序、必须遵守的边界和引用资料路由。 |
| Skill `references/` | 保存工作流、数据格式、CLI 和 Git 规则等详细规范。 |
| Skill `scripts/` | 实现确定性的 CLI、校验器、日志维护和看板服务。 |
| Skill `assets/dashboard/` | 保存固定的看板前端资源，不保存具体项目数据。 |
| 四个聚合 JSON | 保存目标与任务的当前或归档状态，是结构化状态事实源。 |
| 目标和任务 Markdown | 保存详细分析、计划、实施、验证和交付材料。 |
| `log.json` | 保存 CLI 自动追加的业务管理时间线。 |
| Git | 保存实际文件差异、实现提交、管理提交和历史恢复依据。 |
| 看板 | 只读展示 JSON、日志、Markdown 和可选 Git 信息。 |

### 3.3 数据权威关系

| 信息 | 权威来源 |
| --- | --- |
| 当前编号、标题、描述、状态和时间 | 四个聚合 JSON |
| 目标与任务之间的基础关联 | 四个聚合 JSON |
| 当前阻塞摘要 | 四个聚合 JSON |
| 用户任务类型和结果 | 四个聚合 JSON |
| 目标和 AI 任务的详细内容 | 对应 Markdown |
| 管理动作时间线 | `log.json` |
| 文件变更前后差异和提交归属 | Git |

AI 不直接读取或修改四个聚合 JSON 和 `log.json`。AI 通过 CLI 获取所需清单、上下文和状态，并通过 CLI 完成结构化修改，以减少上下文占用和格式错误。

看板通过 CLI 提供的只读 HTTP 服务读取结构化数据和 Markdown。Git 和看板不得绕过 CLI 修改结构化状态。

## 4. 领域对象与核心原则

### 4.1 领域对象

GoGoal 包含三类业务对象：

1. **目标**：描述最终要实现的结果、方案、任务规划和交付情况，是用户授权启动、提交验收、验收完成和指定归档的单位。
2. **AI 任务**：由 AI 在目标边界内负责实施并完成必要验证的执行单元。
3. **用户任务**：目标执行过程中必须由用户处理的事项。

每个新任务必须关联一个目标。目标、AI 任务和用户任务分别独立编号，编号从 `1` 开始并永久不复用。

### 4.2 用户任务类型

| 类型 | JSON 值 | 含义 |
| --- | --- | --- |
| 外部依赖 | `dependency` | 用户需要提供账号、权限、环境、服务器、域名、资料或第三方配置等外部依赖。 |
| 用户决定 | `decision` | 执行过程中出现矛盾、歧义、业务判断或方案选择，需要用户确认。 |
| 其他事项 | `other` | 无法合理归入前两类，但确实必须由用户处理的事项。 |

`other` 仅作为兜底类型使用，其任务描述必须清楚说明用户需要完成的具体事项。

### 4.3 核心原则

1. **显式纳管**：只有用户明确要求时才登记目标或任务。
2. **目标授权**：用户明确批准启动目标前，AI 不得实施目标计划内任务。
3. **边界内自主执行**：目标启动后，AI 可以在目标边界内新增、拆分、排序、启动、阻塞、恢复、完成和取消 AI 任务，无需逐任务请求批准。
4. **用户依赖显式化**：外部依赖、矛盾、问题和必要决定必须登记为用户任务，不得伪装成 AI 已完成事项。
5. **目标级验收**：AI 只能把目标提交为待验收，目标完成必须由用户明确验收通过。
6. **终态不可重开**：已完成或已取消的目标和任务不得重新打开；后续处理通过新增任务或目标完成。
7. **验收修改新增任务**：目标待验收后收到范围内修改意见时，应新增任务，不得改写已结束任务的历史事实。
8. **范围不可暗中扩大**：修改目标结果、范围、交付物或验收要求时，必须回到目标讨论或建立新目标。
9. **活动与归档唯一位置**：同一对象只能存在于活动 JSON 或归档 JSON 中的一处。
10. **结构化数据确定性维护**：所有 JSON 状态变更必须由 CLI 完成。
11. **文档与状态一致**：结构化状态、Markdown、实现结果和 Git 记录必须相互一致。
12. **变更隔离**：管理或任务提交不得混入用户原有改动或其他任务改动。
13. **高风险单独授权**：目标启动不替代生产操作、破坏性操作、外部发布、资金支出、敏感信息使用等行为所需的单独授权。
14. **默认中文**：目标、任务、评审记录、代码注释和提交说明默认使用中文。
15. **敏感信息禁止入库**：JSON、Markdown、日志和 Git 提交不得保存密码、密钥、令牌、个人敏感信息或生产隐私数据。

## 5. 状态模型

内部状态全部使用单个英文单词；CLI 和看板根据 `locale` 映射为中文显示。

### 5.1 目标状态

| JSON 值 | 中文 | 含义 |
| --- | --- | --- |
| `pending` | 未启动 | 目标已经登记，正在分析、讨论或等待用户批准。 |
| `active` | 进行中 | 用户已经启动目标，AI 正在拆分或实施任务。 |
| `blocked` | 阻塞中 | 整个目标不存在可继续执行的路径。 |
| `review` | 待验收 | 目标范围内必要任务和 AI 可执行验证已经完成，等待用户验收。 |
| `completed` | 已完成 | 用户已经明确验收通过。 |
| `cancelled` | 已取消 | 用户已经明确取消目标。 |

允许的主要迁移：

```text
pending → active → review → completed
    │         ↕       │
    │       blocked   └→ active（验收修改）
    └────────────────────→ cancelled
active / blocked / review → cancelled
```

精确规则：

- `pending → active`：用户明确启动目标。
- `active → blocked`：整个目标的所有可执行路径均被阻断。
- `blocked → active`：目标重新具备可执行路径。
- `active → review`：必要任务和验证全部完成。
- `review → active`：用户提出目标范围内的验收修改。
- `review → completed`：用户明确验收通过。
- 非终态目标可以根据用户明确要求进入 `cancelled`。
- `completed` 和 `cancelled` 不得迁移到其他状态。

### 5.2 AI 任务状态

| JSON 值 | 中文 | 含义 |
| --- | --- | --- |
| `pending` | 未启动 | AI 任务已经登记但尚未实施。 |
| `active` | 进行中 | AI 正在实施或验证任务。 |
| `blocked` | 阻塞中 | 任务暂时无法继续，但未来可能恢复。 |
| `completed` | 已完成 | 任务实施和 AI 可执行验证已经完成。 |
| `cancelled` | 已取消 | 任务不再需要或无法在目标范围内成立。 |

允许的主要迁移：

```text
pending → active → completed
             ↕
           blocked
pending / active / blocked → cancelled
```

`completed` 和 `cancelled` 为终态，不得重新打开。

### 5.3 用户任务状态

| JSON 值 | 中文 | 含义 |
| --- | --- | --- |
| `pending` | 未完成 | 用户尚未提供依赖、决定或其他必要结果。 |
| `completed` | 已完成 | 用户已经提供结果并由 AI 如实记录。 |
| `cancelled` | 已取消 | 用户任务不再需要。 |

用户任务不细分未启动、进行中和阻塞中。用户任务阻断整个目标时，通过目标的 `blocked` 状态表达。

### 5.4 归档

归档不是业务状态：

- 只有 `completed` 或 `cancelled` 对象可以归档。
- 归档后保留原终态。
- 归档通过从活动 JSON 迁移到归档 JSON 表达。
- 详情 Markdown 永久保留原路径，不移动、不删除。

## 6. 目标与任务工作流

### 6.1 项目初始化

1. 用户明确要求初始化 GoGoal 或登记第一个目标。
2. Skill 检查项目根目录是否已有 `gogoal/config.json`。
3. 未初始化时调用 `gogoal init` 创建配置、四个空状态 JSON、空日志和 Markdown 目录。
4. 执行 `gogoal validate` 确认初始化结果。
5. 初始化本身不自动创建目标。

### 6.2 目标登记与讨论

1. 用户明确要求将事项作为目标管理。
2. AI 识别目标标题、基础描述和已知上下文。
3. AI 通过 CLI 查询活动目标，检查是否可能重复。
4. 确认创建后调用 `gogoal goal create`，以 `pending` 状态分配目标编号。
5. CLI 返回稳定文档路径 `targets/<id>.md`。
6. AI 创建并编写目标 Markdown。
7. AI 与用户继续讨论目标结果、范围、方案、任务规划和验收要求。
8. 讨论期间可使用 `gogoal goal update` 更新标题或基础描述。
9. 每次完整管理动作结束前执行 `gogoal validate`。

### 6.3 目标启动

只有用户明确批准目标计划后才能启动：

1. AI 确认影响实施边界的待定项已经解决。
2. AI 按最终确定的 Markdown 规范记录用户批准信息。
3. 调用 `gogoal goal start <id>` 将目标从 `pending` 改为 `active`。
4. 根据任务规划登记初始 AI 任务和用户任务。
5. AI 为每个 AI 任务创建稳定路径的任务 Markdown；用户任务不创建详情文档。
6. 校验并创建相应管理提交。
7. 进入目标范围内的自主执行循环。

目标启动不扩大宿主环境权限，也不授权高风险外部操作。

### 6.4 AI 自主执行循环

```text
通过 goal context 获取目标上下文
→ 获取活动任务清单
→ 判断依赖和可执行路径
→ 选择一个 AI 任务
→ 完善任务计划
→ 将任务启动为 active
→ 实施并验证
→ 完成、阻塞或取消任务
→ 校验并提交对应变更
→ 继续选择下一个任务
→ 必要任务全部结束后提交目标待验收
```

默认同一时间只实施一个 AI 任务。任务阻塞后，可以切换到其他不受影响的任务；只有整个目标没有可执行路径时，目标才进入 `blocked`。

### 6.5 任务新增与调整

- 任务只能从已登记目标的方案和实施发现中产生。
- AI 可以在已启动目标范围内新增、拆分或取消 AI 任务。
- 新增任务必须说明具体结果、边界和关联目标。
- 需要用户提供依赖、解决矛盾或做出决定时，建立用户任务。
- 改变目标结果、范围、交付物或验收要求时，不能通过新增任务暗中扩展目标。

### 6.6 阻塞与恢复

AI 任务阻塞时：

1. 确认阻塞原因和解除条件。
2. 调用 `gogoal task block` 保存状态和结构化阻塞摘要。
3. 按最终 Markdown 规范补充详细记录。
4. 如果阻塞需要用户处理，新增或更新用户任务。
5. 继续执行其他不受影响的任务。

整个目标阻塞时：

1. 确认不存在其他可执行路径。
2. 调用 `gogoal goal block` 保存状态和结构化阻塞摘要。
3. 明确向用户说明所需条件。

条件满足后调用对应的 `resume` 命令。恢复操作清空当前结构化 `blocker`，历史阻塞事件继续保留在 `log.json` 和 Git 中。

### 6.7 用户任务完成与取消

- 用户提供依赖或决定后，AI 如实记录结果并调用 `gogoal task complete --type user --result ...`。
- AI 能验证用户交付时应执行必要验证；无法验证时不得声称已经验证。
- 用户明确取消或目标调整使任务不再需要时，调用用户任务取消命令，并在 `result` 中保存取消原因。

### 6.8 提交目标待验收

目标进入 `review` 前必须满足：

- 所有必要 AI 任务均为 `completed`，不再需要的任务为 `cancelled`。
- 所有必要用户任务均已完成，取消的用户任务不会破坏目标结果。
- 相关规范、项目文档和功能实现已经同步。
- AI 可以执行的验证全部完成并通过。
- 无法由 AI 执行的检查已经明确列为用户验收事项。
- 目标和任务 Markdown 已经按最终规范更新。
- JSON、Markdown、实现和 Git 状态一致。

满足条件后调用 `gogoal goal submit <id>`，将目标从 `active` 改为 `review`。CLI 校验关联任务状态，但不解析 Markdown 中的验收内容。

### 6.9 验收修改

用户对 `review` 目标提出修改时：

1. 判断反馈是否仍属于原目标范围。
2. 超出范围时建立新目标，不扩展当前目标。
3. 范围内反馈通过新增 AI 任务或用户任务处理。
4. 已完成或已取消任务保持终态，不重开、不改写历史事实。
5. 调用 `gogoal goal revise <id>` 将目标恢复为 `active`。
6. AI 自主完成新增任务并重新提交待验收。

### 6.10 目标完成

只有用户明确验收通过后：

1. AI 按最终 Markdown 规范记录验收结论。
2. 确认交付和任务记录完整。
3. 调用 `gogoal goal complete <id>`。
4. CLI 将状态从 `review` 改为 `completed` 并填写 `endedAt`。
5. 校验并创建目标完成提交。

AI 的自行验证不能替代用户验收。

### 6.11 目标和任务取消

- 目标只能根据用户明确要求取消。
- 取消目标时，CLI 同时取消全部仍处于非终态的关联任务。
- 已存在的项目改动不得自动删除、回滚或擅自提交。
- AI 任务只有在目标范围调整、任务被替代、已经不再需要或无法在目标内成立时才能自主取消。
- 取消原因通过命令参数进入日志；是否以及如何进入 Markdown，留待 Markdown 规范定版。

### 6.12 归档

- 用户指定归档已完成或已取消任务时，CLI 将其从 `task.json` 迁移到 `task-archive.json`。
- 用户指定归档已完成或已取消目标时，CLI 一并归档仍在活动 JSON 中的关联终态任务，然后迁移目标。
- 归档时填写 `archivedAt` 并追加日志。
- 详情 Markdown 保持原路径。
- 归档对象不得在活动和归档 JSON 中重复存在。

## 7. 项目数据目录

### 7.1 目录结构

```text
项目根目录/
└── gogoal/
    ├── config.json
    ├── target.json
    ├── target-archive.json
    ├── task.json
    ├── task-archive.json
    ├── log.json
    ├── targets/
    │   ├── 1.md
    │   └── 2.md
    └── tasks/
        ├── 1.md
        └── 2.md
```

### 7.2 文件职责

| 文件 | 作用 |
| --- | --- |
| `config.json` | 保存项目级 GoGoal 配置和数据格式版本。 |
| `target.json` | 保存所有未归档目标的基础信息和当前状态。 |
| `target-archive.json` | 保存已归档目标。 |
| `task.json` | 保存所有未归档 AI 任务和用户任务。 |
| `task-archive.json` | 保存已归档 AI 任务和用户任务。 |
| `log.json` | 保存 CLI 自动追加的管理动作日志。 |
| `targets/<id>.md` | 保存目标详细内容。 |
| `tasks/<id>.md` | 保存 AI 任务详细内容。 |

用户任务不创建独立 Markdown。

### 7.3 基础格式规则

- JSON 文件使用 UTF-8 编码和稳定格式化风格。
- 尚未发生或不适用的 JSON 值使用真正的 `null`，不得写为字符串 `"—"`。
- CLI 和看板在面向用户展示时可将 `null` 映射为 `—`。
- 时间统一使用 `config.json` 中的时区，第一版格式为 `YYYY-MM-DD HH:mm`。
- `recordedAt` 在首次登记时生成，之后保持不变。
- `endedAt` 只在进入 `completed` 或 `cancelled` 时生成。
- `archivedAt` 只在执行归档时生成。
- 目标、AI 任务、用户任务和日志分别独立编号，均按历史最大编号加一。
- 编号扫描必须覆盖活动和归档 JSON；日志编号扫描 `log.json`。

## 8. 配置文件标准

### 8.1 配置清单

配置文件固定为 `gogoal/config.json`：

| 配置 | 默认值 | 作用 |
| --- | --- | --- |
| `format` | `1` | 数据格式版本；CLI 据此判断当前数据是否兼容。 |
| `project` | 当前项目目录名 | 看板显示的项目名称。 |
| `locale` | `"zh-CN"` | CLI 和看板显示语言。 |
| `timezone` | `"Asia/Shanghai"` | 目标、任务和日志使用的时区。 |
| `git.enabled` | `true` | 是否启用 Git 状态检查、提交建议和补充审计信息。 |
| `git.branchPrefix` | `"goaltask/"` | 需要任务隔离分支时使用的默认分支前缀。 |
| `dashboard.host` | `"127.0.0.1"` | 本地只读看板监听地址，默认仅本机可访问。 |
| `dashboard.port` | `4173` | 本地看板监听端口。 |
| `dashboard.refreshSeconds` | `60` | 页面重新获取最新数据的间隔秒数。 |
| `dashboard.autoOpen` | `false` | 启动服务后是否尝试自动打开浏览器。 |
| `dashboard.gitActivity` | `true` | 是否在看板中补充展示匹配管理提交格式的 Git 信息。 |

`git.branchPrefix` 配置为 `goaltask` 时，CLI 应自动规范化为 `goaltask/`。该配置只用于确实需要独立分支的任务，不代表每个任务都强制创建分支。

### 8.2 配置示例

```json
{
  "format": 1,
  "project": "gogoal",
  "locale": "zh-CN",
  "timezone": "Asia/Shanghai",
  "git": {
    "enabled": true,
    "branchPrefix": "goaltask/"
  },
  "dashboard": {
    "host": "127.0.0.1",
    "port": 4173,
    "refreshSeconds": 5,
    "autoOpen": false,
    "gitActivity": true
  }
}
```

### 8.3 `format` 的作用

`format` 只在配置文件保存一次，不在其他 JSON 重复保存。新版 CLI 遇到不兼容的数据格式时必须停止修改并提示升级或迁移，不得猜测字段含义后继续写入。

第一版不提供数据迁移器，但内部设计应为后续增加迁移命令保留空间。

## 9. 目标数据标准

### 9.1 `target.json`

根结构：

```json
{
  "targets": []
}
```

目标字段：

| 字段 | 类型 | 必填 | 作用 |
| --- | --- | --- | --- |
| `id` | 整数 | 是 | 目标独立编号，创建后永久不变。 |
| `title` | 字符串 | 是 | 目标标题，允许通过 CLI 更新。 |
| `description` | 字符串 | 是 | 目标基础描述，用于查询和看板摘要。 |
| `status` | 字符串 | 是 | 当前目标状态。 |
| `document` | 字符串 | 是 | 相对于 `gogoal/` 的目标 Markdown 路径。 |
| `recordedAt` | 字符串 | 是 | 首次登记时间。 |
| `endedAt` | 字符串或 `null` | 是 | 完成或取消时间。 |
| `blocker` | 对象或 `null` | 是 | 整个目标当前阻塞摘要。 |
| `blocker.reason` | 字符串 | 阻塞时 | 整个目标无法推进的原因。 |
| `blocker.condition` | 字符串 | 阻塞时 | 解除目标阻塞所需条件。 |

示例：

```json
{
  "targets": [
    {
      "id": 1,
      "title": "建立任务管理看板",
      "description": "提供目标与任务的全局可视化能力",
      "status": "active",
      "document": "targets/1.md",
      "recordedAt": "2026-08-14 10:00",
      "endedAt": null,
      "blocker": null
    }
  ]
}
```

### 9.2 `target-archive.json`

根结构：

```json
{
  "targets": []
}
```

归档目标沿用活动目标全部字段，并增加：

| 字段 | 类型 | 必填 | 作用 |
| --- | --- | --- | --- |
| `archivedAt` | 字符串 | 是 | 用户指定归档目标的实际时间。 |

归档目标只允许 `completed` 或 `cancelled`，必须同时具有 `endedAt` 和 `archivedAt`。

## 10. 任务数据标准

### 10.1 `task.json`

根结构：

```json
{
  "aiTasks": [],
  "userTasks": []
}
```

### 10.2 AI 任务字段

| 字段 | 类型 | 必填 | 作用 |
| --- | --- | --- | --- |
| `id` | 整数 | 是 | AI 任务独立编号，创建后永久不变。 |
| `title` | 字符串 | 是 | AI 任务标题，允许通过 CLI 更新。 |
| `description` | 字符串 | 是 | AI 任务要实现的具体结果和边界。 |
| `status` | 字符串 | 是 | AI 任务当前状态。 |
| `goalId` | 整数 | 是 | 关联目标编号。 |
| `document` | 字符串 | 是 | 相对于 `gogoal/` 的 AI 任务 Markdown 路径。 |
| `recordedAt` | 字符串 | 是 | AI 任务首次登记时间。 |
| `endedAt` | 字符串或 `null` | 是 | AI 任务完成或取消时间。 |
| `blocker` | 对象或 `null` | 是 | AI 任务当前阻塞摘要。 |
| `blocker.reason` | 字符串 | 阻塞时 | 任务无法推进的原因。 |
| `blocker.condition` | 字符串 | 阻塞时 | 解除任务阻塞所需条件。 |

AI 任务示例：

```json
{
  "id": 1,
  "title": "实现数据校验器",
  "description": "校验目标、任务、状态和文档路径的一致性",
  "status": "active",
  "goalId": 1,
  "document": "tasks/1.md",
  "recordedAt": "2026-08-14 10:30",
  "endedAt": null,
  "blocker": null
}
```

### 10.3 用户任务字段

| 字段 | 类型 | 必填 | 作用 |
| --- | --- | --- | --- |
| `id` | 整数 | 是 | 用户任务独立编号，创建后永久不变。 |
| `title` | 字符串 | 是 | 用户任务标题，允许通过 CLI 更新。 |
| `description` | 字符串 | 是 | 用户需要提供、选择、确认或处理的内容。 |
| `kind` | 字符串 | 是 | `dependency`、`decision` 或 `other`。 |
| `status` | 字符串 | 是 | 用户任务当前状态。 |
| `result` | 字符串或 `null` | 是 | 完成时保存用户交付或决定；取消时保存取消原因。 |
| `goalId` | 整数 | 是 | 关联目标编号。 |
| `recordedAt` | 字符串 | 是 | 用户任务首次登记时间。 |
| `endedAt` | 字符串或 `null` | 是 | 用户任务完成或取消时间。 |

用户任务不建立 `document`，也不使用 `blocker`。

用户任务示例：

```json
{
  "id": 1,
  "title": "确认看板发布范围",
  "description": "确认看板仅在本地使用还是公开部署到 GitHub Pages",
  "kind": "decision",
  "status": "pending",
  "result": null,
  "goalId": 1,
  "recordedAt": "2026-08-14 10:35",
  "endedAt": null
}
```

### 10.4 `task-archive.json`

根结构：

```json
{
  "aiTasks": [],
  "userTasks": []
}
```

归档 AI 任务和用户任务分别沿用对应活动字段，并增加：

| 字段 | 类型 | 必填 | 作用 |
| --- | --- | --- | --- |
| `archivedAt` | 字符串 | 是 | 任务归档的实际时间。 |

归档任务只允许 `completed` 或 `cancelled`，必须同时具有 `endedAt` 和 `archivedAt`。

## 11. 管理日志标准

### 11.1 定位

`gogoal/log.json` 是 CLI 自动维护的追加式业务活动日志，用于审计目标与任务的管理动作，并为看板提供时间线。

职责区分：

| `log.json` | Git |
| --- | --- |
| 记录业务管理动作 | 记录实际文件差异 |
| 直接提供目标任务时间线 | 提供提交归属和历史恢复 |
| 未启用 Git 时仍然可用 | 依赖 Git 仓库 |
| 保存状态前后值和动作说明 | 保存字段和 Markdown 的完整改动 |

四个状态 JSON 是当前状态事实源；`log.json` 是管理历史；Git 是文件变更和恢复依据。

### 11.2 根结构

```json
{
  "logs": []
}
```

### 11.3 日志字段

| 字段 | 类型 | 必填 | 作用 |
| --- | --- | --- | --- |
| `id` | 整数 | 是 | 日志编号，永久不复用。 |
| `time` | 字符串 | 是 | 管理动作发生时间。 |
| `entity` | 字符串 | 是 | 对象类型：`goal`、`ai` 或 `user`。 |
| `entityId` | 整数 | 是 | 对应目标或任务编号。 |
| `goalId` | 整数 | 是 | 所属目标编号；目标日志与 `entityId` 相同。 |
| `title` | 字符串 | 是 | 动作完成后的对象标题快照。 |
| `action` | 字符串 | 是 | 与 CLI 管理子命令一致的动作。 |
| `statusFrom` | 字符串或 `null` | 是 | 动作前状态；创建时为 `null`。 |
| `statusTo` | 字符串或 `null` | 是 | 动作后状态；不改变状态时与前值相同。 |
| `note` | 字符串或 `null` | 是 | 阻塞、恢复、取消、更新等动作的简短说明。 |

日志不保存动作来源，不包含 `actor`；标题更新不保存 `titleFrom` 或 `titleTo`。旧标题和完整字段差异由 Git 提供。

### 11.4 动作枚举

目标动作：

```text
create
update
start
block
resume
submit
revise
complete
cancel
archive
```

AI 任务动作：

```text
create
update
start
block
resume
complete
cancel
archive
```

用户任务动作：

```text
create
update
complete
cancel
archive
```

标题或描述修改统一记录为 `update`，不使用 `rename`。一次 `update` 命令同时修改多个字段时只追加一条日志。

### 11.5 日志示例

```json
{
  "logs": [
    {
      "id": 1,
      "time": "2026-08-14 10:00",
      "entity": "goal",
      "entityId": 1,
      "goalId": 1,
      "title": "建立任务管理看板",
      "action": "create",
      "statusFrom": null,
      "statusTo": "pending",
      "note": null
    },
    {
      "id": 2,
      "time": "2026-08-14 10:15",
      "entity": "goal",
      "entityId": 1,
      "goalId": 1,
      "title": "建立 GoGoal 看板",
      "action": "update",
      "statusFrom": "pending",
      "statusTo": "pending",
      "note": "更新目标标题"
    }
  ]
}
```

### 11.6 维护规则

- 只有 CLI 可以追加日志。
- 不提供新增、修改或删除日志的公开写命令。
- 每个成功的单对象业务修改命令为受影响对象追加一条日志。
- 目标取消或归档引起关联任务级联变化时，每个受影响任务分别追加一条日志，最后再追加目标日志；因此一次 CLI 命令可以产生多条日志。
- 配置读写、查询、校验和看板读取不写业务日志。
- 日志按实际发生顺序追加，不重新排序。
- 状态文件修改和日志追加必须在同一文件锁保护下完成。
- CLI 只有在状态数据和日志均成功写入后才报告业务动作成功。
- `note` 不得包含密码、密钥、令牌或敏感数据。
- 第一版不进行日志裁剪或归档。
- 日志与当前状态不一致时，`validate` 必须报告问题，不得根据日志擅自覆盖当前状态。

## 12. Markdown 文档边界

### 12.1 已经确定的规则

- 每个目标保留一个 `gogoal/targets/<id>.md`。
- 每个 AI 任务保留一个 `gogoal/tasks/<id>.md`。
- 用户任务不建立详情 Markdown。
- Markdown 由 AI 根据用户目标、项目事实和实施结果编写与修改。
- CLI 不编写、改写或解析 Markdown 的业务正文。
- CLI 只校验文档路径存在、位于 `gogoal/` 内，并可校验一级标题与 JSON 标题的一致性。
- JSON 保存基础状态，Markdown 不作为编号和状态的事实源。
- 修改标题时，CLI 更新 JSON，AI 更新 Markdown 一级标题，文件路径不变。
- 看板按需读取并渲染 Markdown。
- Markdown 不得保存敏感信息。

### 12.2 尚未确定的内容

以下内容必须在后续专项讨论中定版：

- 目标 Markdown 的固定章节及各章节内容要求。
- AI 任务 Markdown 的固定章节及各章节内容要求。
- 目标计划批准、验收修改、验收通过、取消和归档意见如何记录。
- AI 任务阻塞、恢复、取消、验证和交付如何记录。
- Markdown 在各 CLI 动作前后应何时更新。
- Markdown 标题格式和内部链接规则。
- 看板如何拆分、摘要和展示 Markdown 各章节。

在专项规范完成前，CLI 不得通过解析 Markdown 内容决定是否允许状态迁移。

## 13. CLI 设计

### 13.1 总体原则

- 文档中的逻辑命令统一写为 `gogoal`。
- Skill 可以通过 `python3 <skill目录>/scripts/gogoal.py` 执行，无需用户额外全局安装 CLI。
- AI 不直接读取或写入结构化 JSON，只通过 CLI 查询和修改。
- 查询命令默认输出紧凑文本，减少 AI 上下文占用。
- 查询命令支持 `--json` 时输出稳定 JSON，供看板或其他程序使用。
- 修改命令负责状态校验、编号分配、时间生成、日志追加和一致性检查。
- `config set` 只修改项目配置，不属于目标或任务业务动作，因此不追加 `log.json`。
- CLI 不接受也不记录动作来源，不提供 `--actor` 参数。
- CLI 不编写 Markdown 正文，不默认推送、创建 PR 或执行外部发布。

### 13.2 初始化与配置命令

| 命令 | 作用与使用方式 |
| --- | --- |
| `gogoal init` | 在当前项目创建 `gogoal/`、配置、四个空状态 JSON、空日志和详情目录。 |
| `gogoal init --project "项目名称"` | 初始化并指定看板项目名称。 |
| `gogoal config list` | 列出全部有效配置和值。 |
| `gogoal config get dashboard.port` | 使用点路径查询单个配置。 |
| `gogoal config set dashboard.port 4180` | 修改单个配置；CLI 校验类型和允许范围。 |

### 13.3 全局查询与校验命令

| 命令 | 作用与使用方式 |
| --- | --- |
| `gogoal summary` | 返回未归档目标、AI 任务、用户任务及各状态数量。 |
| `gogoal summary --archive` | 摘要同时包含归档数据。 |
| `gogoal validate` | 校验配置、JSON、编号、状态、关联、时间、归档位置、日志和 Markdown 路径。 |
| `gogoal validate --strict` | 将警告也视为失败，适用于提交或 CI 前。 |

### 13.4 目标查询命令

| 命令 | 作用与使用方式 |
| --- | --- |
| `gogoal goal list` | 列出全部未归档目标。 |
| `gogoal goal list --status active` | 只列出指定状态的目标。 |
| `gogoal goal list --archive` | 只列出归档目标。 |
| `gogoal goal list --all` | 同时列出活动和归档目标。 |
| `gogoal goal show 1` | 返回目标 1 的完整基础信息和文档路径。 |
| `gogoal goal context 1` | 返回目标 1 及其关联 AI 任务、用户任务的紧凑上下文。 |

### 13.5 目标修改命令

| 命令 | 作用与使用方式 |
| --- | --- |
| `gogoal goal create --title "标题" --description "描述"` | 分配目标编号，以 `pending` 状态登记，并返回 `targets/<id>.md`。 |
| `gogoal goal update 1 --title "新标题"` | 修改目标标题并追加 `update` 日志；文档路径不变。 |
| `gogoal goal update 1 --description "新描述"` | 修改目标基础描述并追加 `update` 日志。 |
| `gogoal goal start 1` | 将目标从 `pending` 改为 `active`。 |
| `gogoal goal block 1 --reason "原因" --condition "解除条件"` | 将整个目标改为 `blocked` 并保存阻塞摘要。 |
| `gogoal goal resume 1` | 将目标从 `blocked` 恢复为 `active` 并清空当前阻塞摘要。 |
| `gogoal goal submit 1` | 检查关联任务后将目标从 `active` 改为 `review`。 |
| `gogoal goal revise 1 --note "验收修改摘要"` | 将目标从 `review` 恢复为 `active`。 |
| `gogoal goal complete 1` | 将用户已验收目标从 `review` 改为 `completed` 并填写结束时间。 |
| `gogoal goal cancel 1 --reason "取消原因"` | 取消目标并取消其非终态关联任务。 |
| `gogoal goal archive 1` | 归档终态目标及其仍未归档的关联终态任务。 |

目标标题和描述可以在一次命令中同时更新：

```bash
gogoal goal update 1 \
  --title "新的目标标题" \
  --description "新的目标描述"
```

该命令只追加一条 `update` 日志。

### 13.6 任务查询命令

AI 任务和用户任务分别独立编号，因此操作单个任务时必须传递 `--type ai` 或 `--type user`。

| 命令 | 作用与使用方式 |
| --- | --- |
| `gogoal task list` | 列出全部未归档 AI 任务和用户任务。 |
| `gogoal task list --goal 1` | 只列出目标 1 的关联任务。 |
| `gogoal task list --type ai` | 只列出 AI 任务。 |
| `gogoal task list --type user` | 只列出用户任务。 |
| `gogoal task list --status blocked` | 只列出指定状态任务。 |
| `gogoal task list --archive` | 只列出归档任务。 |
| `gogoal task show 2 --type ai` | 返回 AI 任务 2 的基础信息和文档路径。 |
| `gogoal task show 2 --type user` | 返回用户任务 2 的基础信息和结果。 |

### 13.7 任务修改命令

| 命令 | 作用与使用方式 |
| --- | --- |
| `gogoal task create --type ai --goal 1 --title "标题" --description "描述"` | 为目标 1 登记 AI 任务并返回 `tasks/<id>.md`。 |
| `gogoal task create --type user --kind dependency --goal 1 --title "标题" --description "描述"` | 登记外部依赖用户任务。 |
| `gogoal task create --type user --kind decision --goal 1 --title "标题" --description "描述"` | 登记用户决定任务。 |
| `gogoal task create --type user --kind other --goal 1 --title "标题" --description "描述"` | 登记其他必须由用户处理的事项。 |
| `gogoal task update 2 --type ai --title "新标题"` | 修改 AI 任务标题并追加 `update` 日志。 |
| `gogoal task update 2 --type user --description "新描述"` | 修改用户任务描述并追加 `update` 日志。 |
| `gogoal task start 2 --type ai` | 将 AI 任务从 `pending` 改为 `active`。 |
| `gogoal task block 2 --type ai --reason "原因" --condition "解除条件"` | 将 AI 任务改为 `blocked` 并保存阻塞摘要。 |
| `gogoal task resume 2 --type ai` | 将 AI 任务从 `blocked` 恢复为 `active`。 |
| `gogoal task complete 2 --type ai` | 将实施和验证完成的 AI 任务改为 `completed`。 |
| `gogoal task complete 2 --type user --result "用户结果"` | 保存用户交付或决定，并完成用户任务。 |
| `gogoal task cancel 2 --type ai --reason "取消原因"` | 取消 AI 任务。 |
| `gogoal task cancel 2 --type user --result "取消原因"` | 取消用户任务并保存原因。 |
| `gogoal task archive 2 --type ai` | 归档终态 AI 任务。 |
| `gogoal task archive 2 --type user` | 归档终态用户任务。 |

### 13.8 日志查询命令

| 命令 | 作用与使用方式 |
| --- | --- |
| `gogoal log list` | 返回最近 20 条管理日志。 |
| `gogoal log list --limit 50` | 指定返回日志数量。 |
| `gogoal log list --goal 1` | 返回目标 1 及全部关联任务日志。 |
| `gogoal log list --entity goal --id 1` | 只查看目标 1 的日志。 |
| `gogoal log list --entity ai --id 2` | 只查看 AI 任务 2 的日志。 |
| `gogoal log list --action block` | 查看全部阻塞动作。 |
| `gogoal log show 15` | 查看日志 15 的完整信息。 |
| `gogoal log list --json` | 使用稳定 JSON 格式输出日志。 |

不提供 `log add`、`log update` 或 `log delete`。

### 13.9 看板命令

| 命令 | 作用与使用方式 |
| --- | --- |
| `gogoal dashboard serve` | 启动本地只读看板服务。 |
| `gogoal dashboard serve --port 4180` | 临时覆盖监听端口，不修改配置。 |
| `gogoal dashboard serve --open` | 启动服务后尝试打开浏览器。 |
| `gogoal dashboard export --output dist/dashboard` | 导出可部署到 GitHub Pages 等环境的静态站点。 |

本地使用只需要 `dashboard serve`，不需要 `dashboard export`。

### 13.10 CLI 不负责的事项

CLI 不负责：

- 根据自然语言自行决定目标、任务和业务方案。
- 编写或改写 Markdown 正文。
- 判断用户是否已经真实批准目标或验收目标。
- 判断实现和验证是否真实完成。
- 默认创建 Git 提交、推送、Pull Request 或发布外部站点。
- 绕过宿主的沙箱、审批和安全限制。

这些事项由 Skill、AI、用户和宿主环境共同控制。

## 14. 一致性、并发与异常处理

### 14.1 修改命令统一顺序

```text
定位 gogoal/config.json
→ 获取管理文件锁
→ 在锁内重新读取配置、四个状态 JSON 和 log.json
→ 校验源对象、状态和关联关系
→ 计算新状态、时间和日志
→ 将受影响 JSON 写入同目录临时文件
→ 校验临时文件
→ 原子替换单个正式文件
→ 对可处理的替换失败执行回滚
→ 执行写后一致性检查
→ 释放文件锁
→ 返回修改文件、动作结果和建议提交消息
```

单个文件通过临时文件和原子替换保证不会出现半写文件。跨状态文件和日志属于同一受锁保护的逻辑管理动作；对正常异常执行回滚，进程被强制终止等极端情况由下一次 `validate` 检出，不宣称具有数据库级跨文件事务。

### 14.2 第一版必须实现的保障

- 状态迁移校验。
- JSON 结构与字段类型校验。
- 单文件原子写入。
- 本地管理文件锁。
- 获取锁后重新读取全部相关数据。
- 编号覆盖活动与归档数据扫描。
- 目标与任务关联校验。
- 活动和归档单一位置校验。
- 终态、结束时间和归档时间组合校验。
- 阻塞状态、原因和解除条件组合校验。
- Markdown 路径存在性和目录边界校验。
- 标题一致性校验。
- 日志编号、动作、对象和状态前后值校验。
- 写后校验和失败时不报告成功。
- 安全错误信息，不输出敏感配置或文件内容。

### 14.3 第一版不需要的机制

- SQLite 或其他数据库事务。
- 每条记录独立版本号。
- 文件校验和。
- 自动备份目录。
- 分布式锁。
- 网络数据库。

### 14.4 并发约束

- 多个 AI 会话不得绕过 CLI 并发修改 `gogoal/`。
- 本机 CLI 并发写入由文件锁串行化。
- 多工作树可以分别实施项目任务，但不得同时修改同一套 GoGoal 管理数据。
- Git 合并冲突涉及目标、任务或日志业务含义时必须暂停，不得擅自覆盖一方。

### 14.5 异常处理

- JSON 无法解析时停止所有修改，只允许诊断和人工修复。
- 数据格式版本不兼容时停止修改。
- Markdown 暂时缺失时，创建类管理动作不得向用户宣称整体登记完成；AI 创建文档并通过校验后才完成本次操作。
- JSON、Markdown、实现或 Git 不一致时，先恢复一致性，再继续状态流转。
- 写入、验证、提交或合入失败时，不得宣称对应动作已经成功。
- 已完成任务后发现问题时新增任务，不改写原完成事实。

## 15. Git 工作流

### 15.1 Git 的职责

- 保存 JSON、日志和 Markdown 的完整差异。
- 保存 AI 任务实现及验证记录对应的提交。
- 支持审计旧标题、旧描述和详细内容变化。
- 支持在管理数据损坏时恢复历史版本。

`log.json` 不替代 Git，Git 也不替代业务日志。

### 15.2 分支与工作树

- `git.branchPrefix` 默认使用 `goaltask/`。
- 示例任务分支：`goaltask/goal-1-task-2-dashboard`。
- 独立任务分支只在用户改动、其他任务改动或并行实施需要隔离时创建。
- 当前工作树干净且无需并行时，可以按项目既有工作流在当前分支实施。
- 基线分支和目标分支以仓库实际规则为准，不在 GoGoal 中硬编码为固定分支。
- 工作树和分支只有在交付安全合入且无未提交改动后才能清理。

### 15.3 提交消息

默认沿用中文管理提交格式：

```text
目标-操作-编号-标题
AI任务-操作-编号-标题
用户任务-操作-编号-标题
```

目标操作：

```text
登记、更新、启动、阻塞、恢复、待验收、完成、取消、归档
```

AI 任务操作：

```text
登记、更新、启动、阻塞、恢复、完成、取消、归档
```

用户任务操作：

```text
登记、更新、完成、取消、归档
```

### 15.4 提交边界

- 每个管理提交只包含对应目标或任务范围内的文件。
- 暂存前检查工作树和暂存区差异。
- 必须精确指定文件暂存，不使用会混入无关改动的宽泛暂存方式。
- JSON 状态变化、对应日志和不可分割的 Markdown 更新应位于同一管理提交。
- AI 任务完成提交可以同时包含任务实现、验证记录和管理数据。
- 无法安全隔离同一文件中的其他改动时必须暂停并说明。
- 本地提交不代表获得推送、创建 PR、修改远程历史或发布的授权。

## 16. 看板技术架构

### 16.1 已确定的产品边界

- 第一版看板只读。
- 用户继续通过对话操作目标和任务。
- 本地使用动态只读服务，不将数据嵌入并重复生成 HTML。
- 页面定时获取最新数据并局部更新，不需要整页刷新。
- 发布静态站点时才执行导出。
- 页面具体布局、组件和交互留待专项设计。

### 16.2 本地运行

```bash
gogoal dashboard serve
```

默认访问地址：

```text
http://127.0.0.1:4173
```

运行关系：

```text
固定看板前端
→ 每 refreshSeconds 秒请求本地只读 API
→ 服务读取四个状态 JSON、log.json 和所需 Markdown
→ 页面更新目标、任务、日志和文档内容
```

本地服务不是必须长期驻留的后台进程。用户需要查看看板时启动，关闭后不影响目标任务数据。

### 16.3 初始只读 API 边界

具体 URL 可在实现时根据前端框架调整，但第一版至少需要提供等价能力：

| 能力 | 作用 |
| --- | --- |
| 获取全局摘要 | 返回目标和任务状态统计、项目配置及校验摘要。 |
| 获取目标列表 | 返回活动或归档目标。 |
| 获取目标详情 | 返回目标基础数据、关联任务和目标 Markdown。 |
| 获取任务列表 | 返回活动或归档 AI 任务和用户任务。 |
| 获取 AI 任务详情 | 返回任务基础数据和任务 Markdown。 |
| 获取日志 | 支持按目标、对象、动作和数量筛选时间线。 |
| 获取可选 Git 信息 | 返回当前分支、工作树摘要和匹配的管理提交。 |

不提供任何 `POST`、`PUT`、`PATCH` 或 `DELETE` 写接口。

### 16.4 服务安全边界

- 默认只监听 `127.0.0.1`。
- 只读取当前项目 `gogoal/` 白名单文件和必要的只读 Git 信息。
- 拒绝路径穿越和任意文件读取。
- Markdown 渲染必须执行 HTML 安全过滤。
- API 不返回敏感环境变量或 Git 凭据。
- 监听非本机地址应要求用户明确配置和确认。

### 16.5 静态导出

```bash
gogoal dashboard export --output dist/dashboard
```

建议输出结构：

```text
dist/dashboard/
├── index.html
├── assets/
├── data/
│   ├── target.json
│   ├── target-archive.json
│   ├── task.json
│   ├── task-archive.json
│   └── log.json
└── documents/
    ├── targets/
    └── tasks/
```

静态页面通过相对 URL 读取导出数据。源项目数据变化后需要重新导出和部署。公开发布前必须提醒用户检查数据和 Markdown 是否包含不应公开的信息。

### 16.6 页面设计待决

页面的导航、总览、目标看板、任务看板、用户任务、归档、时间线、健康检查、搜索筛选和详情展示方式将在后续专项设计中确定。本蓝图只确定数据来源、只读边界和运行方式。

## 17. Skill 与开源仓库结构

### 17.1 仓库结构

```text
gogoal/
├── .codex-plugin/
│   └── plugin.json
├── .agents/
│   └── plugins/
│       └── marketplace.json
├── skills/
│   └── gogoal/
│       ├── SKILL.md
│       ├── agents/
│       │   └── openai.yaml
│       ├── references/
│       │   ├── workflow.md
│       │   ├── data-format.md
│       │   ├── cli-reference.md
│       │   └── git-workflow.md
│       ├── scripts/
│       │   ├── gogoal.py
│       │   └── gogoal/
│       │       ├── config.py
│       │       ├── storage.py
│       │       ├── validation.py
│       │       ├── transitions.py
│       │       ├── commands.py
│       │       ├── logging.py
│       │       └── dashboard.py
│       └── assets/
│           └── dashboard/
│               ├── index.html
│               ├── app.js
│               └── style.css
├── tests/
├── examples/
│   └── demo-project/
├── .github/
│   └── workflows/
│       └── ci.yml
├── README.md
├── LICENSE
└── .gitignore
```

### 17.2 Skill 内部职责

| 文件或目录 | 作用 |
| --- | --- |
| `SKILL.md` | 保存精简的触发条件、核心工作流、边界和资源路由。 |
| `agents/openai.yaml` | 保存界面展示名称、简短说明和默认提示。 |
| `references/workflow.md` | 保存目标任务状态机、授权、执行、验收和归档规则。 |
| `references/data-format.md` | 保存配置、四个状态 JSON 和日志的完整标准。 |
| `references/cli-reference.md` | 保存 CLI 命令、参数、返回值和使用示例。 |
| `references/git-workflow.md` | 保存分支、工作树、提交和变更隔离要求。 |
| `scripts/gogoal.py` | CLI 唯一入口。 |
| `scripts/gogoal/` | 保存 CLI 内部实现模块。 |
| `assets/dashboard/` | 保存固定的只读看板前端资源。 |

当前仓库中的 `work-management-spec.md` 只是创建 GoGoal 的外部实践参考，不原样复制进 Skill。应提取其中已经验证有效的经验，融合为新的 `SKILL.md` 和 `references/` 内容。

### 17.3 Skill 外部文件职责

| 文件或目录 | 作用 |
| --- | --- |
| `.codex-plugin/plugin.json` | 定义 Plugin 标识、版本、说明和包含的 Skill。 |
| `.agents/plugins/marketplace.json` | 允许 GitHub 仓库作为可添加的 Plugin Marketplace 源。 |
| `tests/` | 测试存储、状态迁移、日志、校验、CLI 和看板服务。 |
| `examples/demo-project/` | 提供可直接体验的示例目标任务数据。 |
| `.github/workflows/ci.yml` | 在提交和 PR 中执行 Skill 校验和自动测试。 |
| `README.md` | 面向用户说明功能、安装、使用、演示和安全边界。 |
| `LICENSE` | 明确开源使用、修改和分发许可。 |
| `.gitignore` | 排除缓存、测试产物和临时文件。 |

第一版不需要额外的 `packages/`、`schemas/` 或大量 `docs/` 目录。数据规则由 CLI 校验和 `references/data-format.md` 共同定义。

### 17.4 渐进披露

- Skill 元数据只负责准确触发。
- `SKILL.md` 保持精简，只包含每次使用都必须知道的规则。
- 工作流、数据、CLI 和 Git 详细规则拆入一级 `references/`。
- CLI 脚本可直接执行，不要求 AI 每次读取实现代码。
- 看板模板作为 `assets/` 使用，不进入 AI 上下文。

### 17.5 分发方式

- GitHub 仓库保存完整开源项目。
- Plugin 使用 `.codex-plugin/plugin.json` 包装 `skills/gogoal/`。
- Marketplace 文件支持用户通过 GitHub 仓库添加安装源。
- 发布使用语义化版本。
- 数据格式版本 `format` 与 Plugin 版本分别管理。
- 对外发布前提供安装示例、演示项目、看板截图和安全说明。

## 18. 测试与质量保障

### 18.1 单元测试

至少覆盖：

- 配置默认值、读取、更新和非法值。
- 目标、AI 任务、用户任务和日志编号分配。
- 所有合法状态迁移。
- 所有非法状态迁移。
- 标题和描述更新。
- 阻塞、恢复及 `blocker` 校验。
- 用户任务三种 `kind`。
- 结束时间和归档时间生成。
- 活动与归档迁移。
- 目标归档时关联任务归档。
- 目标取消时非终态任务取消。
- 日志动作、状态前后值和标题快照。
- 文件锁和并发写入。
- 单文件原子替换和失败回滚。
- 文档路径边界和标题一致性。

### 18.2 CLI 集成测试

至少覆盖以下完整路径：

```text
init
→ goal create
→ goal update
→ goal start
→ ai/user task create
→ task start/block/resume/complete
→ goal submit/revise/submit/complete
→ task/goal archive
→ validate --strict
```

还需覆盖目标取消、任务取消、JSON 损坏、日志不一致、重复编号、缺失文档和格式版本不兼容。

### 18.3 看板测试

- 本地服务只读路由。
- JSON、日志和 Markdown 读取。
- 定时刷新。
- 归档数据读取。
- Markdown 安全过滤。
- 路径穿越拦截。
- 监听地址和端口覆盖。
- 静态导出完整性。
- 无 Git 仓库或关闭 Git 时正常降级。

### 18.4 Skill 验证

- 使用 Skill 标准校验器检查 `SKILL.md` 和 `agents/openai.yaml`。
- 使用真实示例场景前向测试目标登记、启动、阻塞、验收修改和归档。
- 测试时向新 AI 会话只提供 Skill 和原始用户请求，不泄露预期答案。
- 验证 AI 是否始终通过 CLI 管理 JSON、是否遵守启动授权和目标级验收。

## 19. 开发阶段规划

### 阶段一：数据内核与 CLI

- 建立标准 Skill 和 Plugin 骨架。
- 实现 `config.json` 和四个状态 JSON。
- 实现状态机、编号、查询、修改和归档。
- 实现 `log.json` 自动维护。
- 实现文件锁、原子写入和 `validate`。
- 完成数据与 CLI 自动测试。

### 阶段二：Markdown 规范专项设计

- 确定目标 Markdown 模板。
- 确定 AI 任务 Markdown 模板。
- 确定评审、阻塞、取消、验证和交付记录规则。
- 确定各 CLI 动作与 Markdown 更新顺序。
- 将最终规则写入 Skill references 和模板资源。

### 阶段三：看板页面专项设计与实现

- 确定信息架构、页面层级和组件。
- 确定目标、任务、用户任务、归档和时间线展示。
- 实现只读 HTTP 服务和前端。
- 实现定时刷新和静态导出。
- 完成视觉和安全验证。

### 阶段四：Skill 联调与发布

- 编写精简 `SKILL.md` 和界面元数据。
- 通过真实目标场景前向测试。
- 完成示例项目、README、CI 和开源许可证。
- 打包 Plugin 和 Marketplace 条目。
- 创建版本发布并验证安装体验。

## 20. 待专项决策事项

### 20.1 目标和任务 Markdown

需要另行确定：

- 目标文档章节与字段。
- AI 任务文档章节与字段。
- 目标计划批准和验收记录方式。
- 任务计划、实施、验证和交付记录方式。
- 阻塞、恢复和取消的详细记录方式。
- Markdown 更新与 CLI 状态修改的精确先后顺序。
- 文档模板、链接和渲染约束。

### 20.2 看板页面设计

需要另行确定：

- 页面导航和整体信息架构。
- 总览指标和数据卡片。
- 目标和任务看板布局。
- 用户任务突出方式。
- 目标和任务详情布局。
- 归档、日志、健康检查和搜索筛选页面。
- 状态颜色、空状态、错误状态和响应式布局。
- Markdown 章节在页面中的摘要和展开方式。

除上述两部分外，本文档其余要求已经作为 GoGoal 第一版开发基线收敛。
