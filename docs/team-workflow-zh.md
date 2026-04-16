# GSD 团队协作工作流

## 团队结构

```
Agent 仓库                              Frontend 仓库
┌────────────────────────────┐          ┌────────────────────────────┐
│  Unit 1                    │          │  Unit 1                    │
│  ├── Driver（协调/拆解）     │          │  └── 前端                   │
│  ├── 后端                   │          │                            │
│  └── 算法                   │          │  Unit 2                    │
│                            │          │  └── 前端                   │
│  Unit 2                    │          └────────────────────────────┘
│  ├── Driver（协调/拆解）     │
│  ├── 后端                   │
│  └── 算法                   │
└────────────────────────────┘
```

- 2 个仓库：Frontend 仓库、Agent 仓库
- 2 个 Unit，每个 Unit 配置 1 个前端、1 个后端、1 个算法
- 后端和算法在 Agent 仓库上迭代
- 每个 Unit 有一个 Driver，负责需求拆解和任务分配
- Driver 的工作通过线下沟通完成，不通过 GSD

## 前置条件（一次性设置）

### 开启 Team Mode

```bash
# 方式一：新项目初始化时选择
/gsd:new-project    # → 选择 "Team Mode"

# 方式二：已有项目追加开启
/gsd:settings       # → Team Mode → Yes
```

开启后自动启用：

- `unique_milestone_ids` — 防止多人 phase 文件命名冲突
- `push_branches` — 自动推送 milestone 分支到远端
- `pre_merge_check` — 合并前校验
- `.planning/.gitignore` — 共享规划文档 tracked，个人状态文件 ignored

## 工作流程

### 第一步：需求拆解（Driver，线下完成）

Driver 组织团队讨论，产出以下内容：

| 产出项   | 说明                         |
| -------- | ---------------------------- |
| 阶段划分 | 分几个阶段，每个阶段包含什么 |
| 任务分配 | 每个阶段由谁负责             |
| 依赖关系 | 哪些阶段有前后依赖           |
| 时间线   | 每个阶段的截止时间           |
| 合并顺序 | 谁先合入，谁后合入           |

Driver 通过录音/笔记 + AI 生成结构化上下文文件：

```markdown
<!-- docs/MILESTONE-CONTEXT-v1.1.md -->

# Milestone v1.1：智能推荐系统

## 目标

构建实时个性化推荐能力，支持用户画像和排序模型。

## 阶段划分

### Phase 1：用户服务 API（负责人：后端）

- 用户画像 CRUD 接口
- 行为日志采集端点
- 依赖：无，可先行开发
- 截止：4/3

### Phase 2：排序模型（负责人：算法）

- 排序模型训练 Pipeline
- 在线推理服务
- 依赖：Phase 1（需要用户画像 API Schema）
- 截止：4/5

### Phase 3：推荐看板（负责人：前端，Frontend 仓库）

- 推荐效果看板页面
- A/B 实验配置页面
- 依赖：Phase 2（需要推理接口）
- 截止：4/7

## 合并顺序

1. 后端（Phase 1）→ milestone/v1.1
2. 算法（Phase 2）→ rebase → milestone/v1.1
3. 在 milestone/v1.1 上联调
4. milestone/v1.1 → main
5. 前端（Phase 3）→ Frontend 仓库 main（Agent 仓库 API 部署后）

## 时间线

- 开发：3/27 - 4/5
- 联调：4/6 - 4/7
- 发布：4/8
```

Driver 将文件提交到仓库：

```bash
git checkout main && git pull
git checkout -b milestone/v1.1-smart-rec
cp MILESTONE-CONTEXT-v1.1.md docs/
git add docs/MILESTONE-CONTEXT-v1.1.md
git commit -m "docs: add milestone v1.1 context"
git push -u origin milestone/v1.1-smart-rec
```

### 第二步：各自开发（每个开发者独立进行）

每个开发者把分配到的阶段当作一个**独立的 GSD milestone** 来完成。

#### 后端启动 Phase 1：

```bash
# 从团队 milestone 分支切出自己的分支
git fetch origin
git checkout milestone/v1.1-smart-rec
git checkout -b milestone/v1.1-smart-rec/phase-1-user-service

# 读取上下文文件，启动 GSD
/gsd:new-milestone
# Claude 询问目标时回答：
# "请参考 docs/MILESTONE-CONTEXT-v1.1.md，我负责 Phase 1：用户服务 API"

# 正常走 GSD 流程
/gsd:discuss-phase
/gsd:plan-phase
/gsd:execute-phase
# （按需循环多个 sub-phase）
/gsd:verify-work
```

#### 算法启动 Phase 2（同时进行）：

```bash
git fetch origin
git checkout milestone/v1.1-smart-rec
git checkout -b milestone/v1.1-smart-rec/phase-2-ranking-model

/gsd:new-milestone
# "请参考 docs/MILESTONE-CONTEXT-v1.1.md，我负责 Phase 2：排序模型"

/gsd:discuss-phase
/gsd:plan-phase
/gsd:execute-phase
/gsd:verify-work
```

#### 前端启动 Phase 3（Frontend 仓库，同时进行）：

```bash
# 在 Frontend 仓库中操作
git checkout main && git pull
git checkout -b milestone/v1.1-smart-rec/phase-3-dashboard

/gsd:new-milestone
# "请参考 Agent 仓库中的 docs/MILESTONE-CONTEXT-v1.1.md，我负责 Phase 3：推荐看板"

/gsd:discuss-phase
/gsd:plan-phase
/gsd:execute-phase
```

### 第三步：按顺序合入团队分支

按照 Driver 在 MILESTONE-CONTEXT 中定义的合并顺序执行。

#### 第一个合并：后端

```bash
# 在 milestone/v1.1-smart-rec/phase-1-user-service 分支上
/gsd:merge-milestone milestone/v1.1-smart-rec
```

执行过程：

1. 拉取最新的 `milestone/v1.1-smart-rec`
2. 此时没有其他人的代码 → 干净合并
3. Squash merge 后端代码到 `milestone/v1.1-smart-rec`
4. Push

#### 第二个合并：算法

```bash
# 在 milestone/v1.1-smart-rec/phase-2-ranking-model 分支上
/gsd:merge-milestone milestone/v1.1-smart-rec
```

执行过程：

1. 拉取最新的 `milestone/v1.1-smart-rec`（已包含后端代码）
2. **Squash merge**：一次性合并算法代码，所有冲突一次性暴露
3. 如果有冲突：AI 读取双方的 `.planning/` 上下文，辅助解决
4. 全量审查冲突解决结果后提交

### 第四步：联调

所有代码合入 `milestone/v1.1-smart-rec` 后，在该分支上进行联调：

```bash
git checkout milestone/v1.1-smart-rec
# 跑测试、启动服务、验证模块间集成
npm test
npm run dev
```

发现问题时：

- 直接在 `milestone/v1.1-smart-rec` 上修复
- 或切出短期修复分支，修复后合回

### 第五步：合入 main

联调通过后，将团队分支合入 main：

```bash
git checkout milestone/v1.1-smart-rec
/gsd:merge-milestone main
```

执行过程：

1. 拉取最新的 main
2. Squash merge `milestone/v1.1-smart-rec` 到 main（一次性暴露所有冲突）
3. `.planning/` 文件冲突处理：
   - 临时文件（STATE.md、锁文件等）→ 自动解决（git checkout --ours，不删除文件）
   - 收集双方分支上下文 → 展示给用户确认
   - 共享文档（PROJECT.md、ROADMAP.md 等）→ 沿用对应 GSD 命令的更新逻辑合并
4. 代码冲突 → 基于需求上下文的 AI 合并，逐文件人工确认
5. 全量审查所有冲突解决结果
6. 影响范围评估 + 自动运行相关测试 → 测试失败则分析修复
7. 用户确认后 commit → Push

### 第六步：前后端联调 + 发布

Agent 仓库部署新 API 后，前端进行联调：

```
Frontend 仓库（前端）              Agent 仓库（main，已包含后端 + 算法）
     │                                    │
     └──── 调接口、联调 ──────────────────┘
```

```bash
# 在 Frontend 仓库中
/gsd:verify-work

# 联调通过后合入 main
/gsd:merge-milestone main
```

两个仓库的 main 都已包含本迭代全部代码，走正常发布流程。

## 跨 Unit 并行场景

两个 Unit 同时在 Agent 仓库上开发时：

```
Agent 仓库 main ──────────────────────────────────────────────────
  │                    ↑                              ↑
  ├── milestone/v1.1 (Unit 1) ───────────────────────┘│
  │   ├── phase-1-user-service（后端）                  │
  │   └── phase-2-ranking-model（算法）                 │
  │                                                    │
  └── milestone/v1.2 (Unit 2) ────────────────────────┘
      ├── phase-1-payment（后端）
      └── phase-2-fraud-detection（算法）
```

**跨 Unit 合并顺序**由两个 Unit 的 Driver 协调：

- 先完成的 Unit 先合入 main
- 后合入的 Unit：merge 时自动拿到先合入 Unit 的全部代码（冲突一次性暴露）
- 如果存在跨 Unit 代码冲突（共享配置、路由表、数据库 Schema）：两个 Unit 的相关同学坐到一起解决。AI 负责合并 `.planning/` 文件，人 + AI 一起合并代码

## 分支命名规范

```
milestone/v{版本号}-{名称}                          # 团队 milestone 分支（Driver 创建）
milestone/v{版本号}-{名称}/phase-{编号}-{描述}       # 个人开发分支
```

示例：

```
milestone/v1.1-smart-rec                             # Unit 1 团队分支
milestone/v1.1-smart-rec/phase-1-user-service        # 后端的工作分支
milestone/v1.1-smart-rec/phase-2-ranking-model       # 算法的工作分支

milestone/v1.2-payments                              # Unit 2 团队分支
milestone/v1.2-payments/phase-1-payment-api          # 后端的工作分支
milestone/v1.2-payments/phase-2-fraud-detection      # 算法的工作分支
```

## 各角色 GSD 命令速查

| 角色          | 阶段           | 操作                                                                    |
| ------------- | -------------- | ----------------------------------------------------------------------- |
| Driver        | 创建团队分支   | `git checkout -b milestone/v1.1-xxx`                                    |
| Driver        | 提交上下文文件 | `git add docs/MILESTONE-CONTEXT-v1.1.md` → `git push`                   |
| 开发者        | 启动自己的阶段 | `git checkout -b milestone/v1.1-xxx/phase-n-yyy` → `/gsd:new-milestone` |
| 开发者        | 规划           | `/gsd:discuss-phase` → `/gsd:plan-phase`                                |
| 开发者        | 实施           | `/gsd:execute-phase`（循环，直到所有 sub-phase 完成）                   |
| 开发者        | 验证           | `/gsd:verify-work`                                                      |
| 开发者        | 合入团队分支   | `/gsd:merge-milestone milestone/v1.1-xxx`                               |
| 团队          | 联调           | 在 milestone 分支上手动测试验证                                         |
| 开发者/Driver | 合入 main      | `/gsd:merge-milestone main`                                             |

## .planning/ 文件共享策略

团队模式下，`.planning/` 目录包含两类文件：**共享的项目文档**和**个人的运行时状态**。只有前者需要通过 git 在团队间同步。

### 共享范围（git tracked）

```
.planning/
  ├── config.json             # 项目配置（team mode、branching 策略等）
  ├── PROJECT.md              # 项目定义（What This Is、Core Value、Requirements、Key Decisions）
  ├── REQUIREMENTS.md         # 当前 milestone 的需求列表（REQ-IDs、categories、traceability）
  ├── ROADMAP.md              # 路线图（phase 结构、进度标记、milestone 分组）
  ├── MILESTONES.md           # 历史 milestone 记录（版本、日期、交付内容）
  ├── RETROSPECTIVE.md        # 复盘记录（跨 milestone 趋势）
  ├── phases/                 # 所有 phase 的计划、执行、验证产物
  │   ├── 01-abc123-xxx/      # 开发者 A 的 phase（unique ID 防冲突）
  │   └── 01-def456-yyy/      # 开发者 B 的 phase（不同 ID，不同目录）
  ├── milestones/             # 已完成 milestone 的归档
  ├── codebase/               # 代码库结构快照（map-codebase 产物）
  └── quick/                  # quick task 产物
```

### 不共享范围（git ignored）

```
.planning/
  ├── STATE.md                # 个人运行时状态（当前位置、最后活动、累积上下文）
  ├── STATE.md.lock / .lock   # 进程锁
  ├── auto.lock               # --auto 链式执行标记
  ├── active-workstream       # 当前工作区选择
  ├── WAITING.json            # 等待/检查点状态
  ├── metrics.json            # 个人执行速度指标
  ├── continue.md             # 会话恢复上下文
  ├── *-CONTINUE.md           # phase 级恢复上下文
  ├── research/               # 研究产物（每个开发者为自己的 scope 独立研究）
  ├── reports/                # 会话报告
  ├── forensics/              # 诊断报告
  ├── debug/                  # 调试知识库
  ├── todos/                  # 待办事项
  ├── activity/               # 会话日志
  ├── runtime/                # 子进程状态
  └── worktrees/              # git worktree 元数据
```

### 上下文完整性保障

共享的 10 个文件/文件夹覆盖了 GSD 工作流的**信息主干**：

```
PROJECT.md（项目定义）→ REQUIREMENTS.md（需求）→ ROADMAP.md（路线图）
  → phases/（计划 + 执行 + 验证）→ milestones/（历史归档）→ MILESTONES.md（记录）
```

不共享的文件属于**辅助上下文**或**运行时状态**：

| 不共享文件 | 信息是否可从共享文件恢复 | 说明 |
|-----------|------------------------|------|
| STATE.md | **大部分可恢复** | Phase 进度 → 从 ROADMAP.md 的 `[x]`/`[ ]` 推断；Quick Task 记录 → 从 `quick/` 目录推断；唯一丢失的是 `Accumulated Context`（跨 phase 积累知识），但关键决策已沉淀在 PROJECT.md Key Decisions 中 |
| research/ | **关键结论可恢复** | 技术选型和架构决策 → 已沉淀在 PROJECT.md Key Decisions 和 REQUIREMENTS.md 的 scoping 中；原始研究细节丢失但不影响合并 |
| debug/ | **不可恢复但非必需** | 调试知识库是 nice-to-have，不在信息主干上 |
| reports/ | **不可恢复但非必需** | 会话报告是参考性质，不影响工作流执行 |
| todos/ | **不可恢复但非必需** | 待办事项各自管理，不影响协作 |

**merge-milestone 的适配**：当 STATE.md 不可用时（team mode 下的常态），5b 步骤会从 ROADMAP.md 的 phase 完成标记和 phases/\*-SUMMARY.md 推断进度，不会报错或中断。

### 文档可靠性保障

多人并行修改共享文件后，文档内容的一致性和完整性由两层机制保障：

**第一层：源头隔离（防止冲突发生）**

`unique_milestone_ids` 让每个开发者的 phase 目录名天然不同：

```
开发者 A: .planning/phases/01-abc123-user-service/
开发者 B: .planning/phases/01-def456-ranking-model/
```

即使 phase 编号相同（都是 01），目录名不同 → 文件不冲突 → git merge 自动处理。

**第二层：命令逻辑合并（解决不可避免的冲突）**

对于 PROJECT.md、REQUIREMENTS.md、ROADMAP.md 这三个**必然会被多人修改**的文件，merge-milestone 沿用对应 GSD 命令的更新逻辑来合并，而非行级 diff：

| 文件 | 合并逻辑 | 为什么行级 diff 不够 |
|------|----------|---------------------|
| PROJECT.md | `complete-milestone` 的 evolution review | Validated Requirements 需要按 milestone 版本标注来源，Key Decisions 需要按时间排序合并 |
| REQUIREMENTS.md | `new-milestone` 的 define-requirements | REQ-ID 有全局唯一性约束，category 分组需要语义理解 |
| ROADMAP.md | `complete-milestone` 的 reorganize_roadmap | Phase 编号可能冲突需要重新编号，milestone grouping 需要结构化处理 |

**结论：这套共享策略在上下文完整性、文档可靠性、冲突安全性三个维度上都是充分的。**

## 合并完成后的文件结构

```
.planning/
  PROJECT.md              # 经语义合并后的项目文档（包含所有开发者的 milestone 信息）
  ROADMAP.md              # 经语义合并后的路线图（所有 phase 已标记完成）
  MILESTONES.md           # 所有 milestone 记录
  config.json
  milestones/v1.1/        # 所有 milestone 详细归档
  phases/
    01-abc123-user-service/
      01-abc123-PLAN.md       # 后端的 phase 计划（unique ID: abc123）
      01-abc123-SUMMARY.md    # 后端的 phase 总结
    01-def456-ranking-model/
      01-def456-PLAN.md       # 算法的 phase 计划（unique ID: def456）
      01-def456-SUMMARY.md    # 算法的 phase 总结

docs/
  MILESTONE-CONTEXT-v1.1.md   # Driver 的原始上下文文件（保留供参考）
```

## merge-milestone 冲突解决完整链路

团队协作中冲突不可避免：多人同时修改 `.planning/` 规划文档，或改到了同一份代码文件。`/gsd:merge-milestone` 的核心价值在于：**不是简单地做行级文本合并，而是理解双方在做什么，然后用对应领域命令的更新逻辑来智能合并。**

### 设计理念

传统 git merge 是基于文本行的 diff——它不知道 `PROJECT.md` 里的 "Validated Requirements" section 意味着什么，也不知道 `ROADMAP.md` 里的 phase 编号有全局唯一性约束。

GSD 的做法是：**每种 `.planning/` 文件都有一个 GSD 命令负责管理它**，这个命令内含了文件的结构知识和更新规则。冲突解决时，直接沿用该命令的更新逻辑。

```
PROJECT.md   ← complete-milestone 的 evolve_project_full_review 逻辑
ROADMAP.md   ← complete-milestone 的 reorganize_roadmap 逻辑
REQUIREMENTS.md ← new-milestone 的 define-requirements 逻辑
STATE.md     ← new-milestone / quick 的 state-update 逻辑
codebase/*   ← map-codebase（建议合并后重新生成）
```

### 完整流程图

```
/gsd:merge-milestone [target]
│
├─ Step 1: 检测状态 + 确定目标分支
│   ├─ 如果传入了目标分支 → 直接使用
│   └─ 如果没有传入 → 询问用户选择 main / master / 指定分支
│
├─ Step 2: 展示变更摘要
│
├─ Step 3: 确认合并方式
│   └─ Squash merge（推荐）/ Regular merge / Cancel
│
├─ Step 4: 同步 + 执行合并
│   ├─ git fetch + checkout 目标分支 + pull 最新
│   └─ git merge --squash（一次性合并，所有冲突一次性暴露）
│       ⚠ 不使用 rebase：GSD 产生大量原子 commit，rebase 会逐 commit 解冲突，轮次太多且上下文不全
│
├─ Step 5: 处理冲突（核心）
│   │
│   ├─ 5a: 自动解决临时文件冲突
│   │   └─ STATE.md, 锁文件, CONTINUE.md 等 → git checkout --ours
│   │   ⚠ 不删除任何 .planning/ 文件（用户本地可能有 STATE.md 等）
│   │
│   ├─ 5b: ★ 收集双方分支上下文 ★
│   │   ├─ 读取双方的 PROJECT.md / ROADMAP.md / REQUIREMENTS.md
│   │   ├─ STATE.md 不可用时 → 从 ROADMAP.md + SUMMARY.md 推断进度
│   │   ├─ 识别：各自的 milestone 版本、需求范围、完成进度
│   │   ├─ 判断分支关系：独立 / 同 milestone / 重叠 / 未知
│   │   ├─ 展示上下文摘要 → 用户确认
│   │   └─ 上下文不足 → 主动要求用户补充
│   │
│   ├─ 5c: ★ 命令逻辑感知的 .planning/ 合并 ★
│   │   ├─ 每个文件 → 识别对应 GSD 命令 → 沿用该命令的更新逻辑
│   │   ├─ 生成合并结果 + 决策说明
│   │   └─ 逐文件展示 → 用户确认
│   │
│   ├─ 5d: 识别剩余代码冲突
│   │
│   ├─ 5e: ★ 上下文感知的代码冲突解决 ★
│   │   ├─ 复用 5b 收集的分支上下文
│   │   ├─ 关联到具体的 requirement / plan
│   │   ├─ 分析每个冲突块：独立 / 重叠 / 矛盾
│   │   └─ 逐文件提案 → 用户确认
│   │
│   ├─ 5f: ★ 全量审查所有冲突解决结果 ★
│   │   ├─ 跨文件一致性检查（ROADMAP phase 编号 ↔ REQUIREMENTS traceability）
│   │   ├─ 代码 + 规划对齐检查
│   │   ├─ 残留冲突标记扫描
│   │   └─ import / 依赖一致性检查
│   │
│   ├─ 5g: ★ 影响范围评估 + 测试验证 ★
│   │   ├─ 分析冲突文件的依赖关系和 API 变更
│   │   ├─ 自动检测测试框架，运行相关测试
│   │   └─ 测试失败 → 分析是否由合并引起 → 修复或跳过
│   │
│   └─ 5h: 用户确认后 → commit
│
├─ Step 6: 清理（删除分支、codebase map 刷新建议）
├─ Step 7: Push
└─ Step 8: 合并摘要
```

### 阶段详解

#### 阶段一：收集双方上下文（5b）

这是整个冲突解决的基础。在解决任何冲突之前，先搞清楚**两个分支各自在干什么**。

**自动收集：**

```bash
# 从 MERGE_HEAD 读取对方分支的规划文件
git show MERGE_HEAD:.planning/PROJECT.md    # 对方的项目定义
git show MERGE_HEAD:.planning/REQUIREMENTS.md  # 对方的需求列表
git show MERGE_HEAD:.planning/ROADMAP.md    # 对方的路线图
```

**AI 从中提取：**

| 维度       | 从哪读                    | 提取什么                              |
| ---------- | ------------------------- | ------------------------------------- |
| Milestone  | PROJECT.md Current Milestone | 版本号、名称、目标                    |
| 需求范围   | REQUIREMENTS.md           | REQ-ID 列表、所属 category            |
| 完成进度   | ROADMAP.md + phase summaries | 哪些 phase 完成了、交付了什么         |
| 分支关系   | 综合以上                  | 独立 / 同 milestone / 重叠 / 未知     |

**展示给用户确认：**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD > MERGE CONTEXT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Target (milestone/v1.1-smart-rec):
  Milestone: v1.1-abc123 用户服务 API
  Scope: USER-01, USER-02, USER-03
  Progress: Phase 1 completed (2/2 plans)

Source (milestone/v1.1-smart-rec/phase-2-ranking-model):
  Milestone: v1.1-def456 排序模型
  Scope: RANK-01, RANK-02
  Progress: Phase 2 completed (3/3 plans)

Relationship: 同一 milestone 的不同 workstream（无重叠需求）

Conflicting files:
  .planning/: PROJECT.md, ROADMAP.md, REQUIREMENTS.md
  Code: src/config/models.py
```

**如果信息不足（例如某一方没有 REQUIREMENTS.md）：**

AI 不会猜测，而是直接问用户：

```
无法自动判断分支上下文：
- Source 分支缺少 REQUIREMENTS.md

请描述：
1. milestone/v1.1-smart-rec 上目前包含什么？
2. 你的分支 (phase-2-ranking-model) 负责什么？
3. 两者之间有没有重叠的功能？
```

#### 阶段二：命令逻辑感知的 .planning/ 合并（5c）

确认上下文后，对每个冲突的 `.planning/` 文件，**沿用管理该文件的 GSD 命令的更新逻辑**来合并。

##### PROJECT.md → 沿用 `complete-milestone` 的 `evolve_project_full_review`

不是简单地取"更新的版本"，而是像 complete-milestone 完成时那样做全量 review：

| Section              | 合并策略                                              |
| -------------------- | ----------------------------------------------------- |
| What This Is         | 综合两边对产品的描述，如果两边都扩展了 → 合并表述     |
| Core Value           | 如果不同 → 标记给用户决定                             |
| Validated            | 取并集，每条带 milestone 版本引用（`✓ User API — v1.1-abc123`） |
| Active               | 取并集，同 REQ-ID 不同描述 → 标记冲突                 |
| Out of Scope         | 取并集，保留两边的 reasoning                          |
| Key Decisions        | 合并两边的决策表，按时间排序，标注来源 milestone       |
| Context              | 综合两边的最新状态                                    |
| Current Milestone    | 取更晚启动的那个；如果无法判断 → 问用户               |

**展示效果：**

```
## Conflict: .planning/PROJECT.md
Resolving with: complete-milestone → evolve_project_full_review

### What each branch contributed:
- Target: shipped 用户服务 API，新增 3 个 Validated requirements
- Source: shipped 排序模型，新增 2 个 Validated requirements，新增 Key Decision "选择 ONNX Runtime"

### Merge decisions made:
- Combined Validated requirements: 5 total (3 from target + 2 from source)
- Key Decisions table: merged 4 entries from target + 2 from source, sorted by date
- Context: synthesized from both — "用户服务 API 已上线，排序模型推理服务已部署"

[Accept] / [Let me provide context] / [Skip]
```

##### ROADMAP.md → 沿用 `complete-milestone` 的 `reorganize_roadmap`

| 场景             | 处理                                                          |
| ---------------- | ------------------------------------------------------------- |
| Phase 编号冲突   | 重新编号（source 的 phase 接在 target 最后一个 phase 之后）   |
| Phase 状态标记   | 保留两边的 `[x]`/`[ ]`                                       |
| Milestone 分组   | 用 `<details>` 折叠已完成的 milestone                         |
| Progress 表格    | 合并两边的行                                                  |

##### REQUIREMENTS.md → 沿用 `new-milestone` 的 define-requirements 逻辑

| 场景                   | 处理                                          |
| ---------------------- | --------------------------------------------- |
| 不同 category 的 REQ   | 直接合并，保留各自 category 分组              |
| 同 category 不同 REQ   | 追加，保持 REQ-ID 连续编号                    |
| 同 REQ-ID 不同状态     | 取更"完成"的那个（`[x]` > `[ ]`）            |
| 同 REQ-ID 不同描述     | 标记给用户决定                                |
| Traceability table     | 合并行，同 REQ-ID 不同 phase mapping → 标记   |

#### 阶段三：上下文感知的代码冲突解决（5e）

`.planning/` 文件处理完后，处理剩余的代码文件冲突。此时 AI 已经完全理解双方分支的目标和需求。

**传统 AI 合并 vs GSD 上下文感知合并：**

```
传统：
  "这个函数有两个版本，一个加了参数 A，一个加了参数 B"
  → 机械合并，不知道为什么加

GSD：
  "Target 为了 USER-02（用户画像查询）加了 profile_id 参数"
  "Source 为了 RANK-01（实时排序）加了 model_version 参数"
  "两者独立，都需要保留"
  → 理解意图后合并，置信度 high
```

**冲突分类：**

| 关系       | 含义                       | 处理                                  |
| ---------- | -------------------------- | ------------------------------------- |
| 独立       | 两边改不同的东西           | 都保留，置信度 high                   |
| 重叠       | 同一目标，不同实现         | AI 提议最佳方案，置信度 medium        |
| 矛盾       | 互斥的设计决策             | 展示两个方案，置信度 low，必须人工决定 |

**展示效果：**

```
## Conflict: src/config/models.py

### Block 1 (line 42)

Target intent: USER-02 需要 profile 缓存配置（来自 phase-1 plan）
Source intent: RANK-01 需要 model registry 配置（来自 phase-2 plan）
Relationship: independent

Proposed resolution:
  # Profile cache config (USER-02)
  PROFILE_CACHE_TTL = 3600
  PROFILE_CACHE_SIZE = 10000

  # Model registry config (RANK-01)
  MODEL_REGISTRY_URL = "http://model-registry:8080"
  DEFAULT_MODEL_VERSION = "v1"

Confidence: high
Reason: 两个配置块完全独立，分属不同需求，合并后无副作用

[Accept all] / [Let me provide context] / [Skip]
```

### 不同场景的冲突处理对比

| 场景                           | 临时文件 | .planning/ 共享文档 | 代码文件 |
| ------------------------------ | -------- | ------------------- | -------- |
| 同 Unit 内合入团队分支         | 自动解决 | 命令逻辑合并        | 上下文 AI 合并 |
| 团队分支合入 main（无其他 Unit）| 自动解决 | 通常无冲突          | 通常无冲突 |
| 团队分支合入 main（有其他 Unit）| 自动解决 | 命令逻辑合并        | 上下文 AI 合并 |
| 两个 Unit 改了同一文件         | 自动解决 | 命令逻辑合并        | 展示两个 Unit 的需求上下文，人工决定 |

### 用户干预点

整个流程中，用户需要介入的地方：

| 步骤   | 干预内容                        | 何时触发                         |
| ------ | ------------------------------- | -------------------------------- |
| Step 1 | 选择目标分支                    | 未传入目标分支参数时触发         |
| 5b     | 确认分支上下文理解是否正确      | 每次有冲突的合并必触发           |
| 5b     | 补充缺失的上下文信息            | AI 无法从 .planning/ 推断时触发  |
| 5c     | 确认 .planning/ 文件合并结果    | 每个冲突的 .planning/ 文件       |
| 5e     | 确认代码冲突合并结果            | 每个冲突的代码文件               |
| 5f     | 审查全量冲突解决结果            | 所有冲突解决后必触发             |
| 5g     | 选择是否运行测试 + 处理测试失败 | 审查通过后触发                   |
| 5h     | 确认提交                        | 测试通过（或跳过）后必触发       |
| Step 6 | 是否刷新 codebase map           | codebase/* 有冲突时触发          |

**设计原则：AI 做重活（读上下文、理解意图、生成合并方案），人做决策（确认或纠正）。**

## 常见问题

**Q：算法需要后端的 API Schema，但后端还没开发完怎么办？**

算法可以临时把后端的分支合到自己本地进行测试：

```bash
git fetch origin
git merge origin/milestone/v1.1-smart-rec/phase-1-user-service
# 基于后端当前代码进行开发和测试
# 继续自己的开发
```

**Q：合并顺序临时需要调整怎么办？**

合并顺序只是约定，不是硬性限制。任何开发者都可以随时合并。后合的人 merge 时会自动处理冲突（所有冲突一次性暴露，AI 辅助解决）。定义顺序只是为了减少需要解冲突的人数。

**Q：两个人改了同一个文件怎么办？**

merge-milestone 的冲突解决流程会处理（详见上方「merge-milestone 冲突解决完整链路」）：

1. **收集上下文**：读取双方 `.planning/` 文件，识别各自的 milestone、需求范围和完成进度
2. **确认理解**：展示上下文摘要给用户确认，信息不足时主动要求补充
3. **`.planning/` 文件**：沿用对应 GSD 命令的更新逻辑合并（如 PROJECT.md 用 complete-milestone 的 evolution review 逻辑）
4. **代码文件**：基于已收集的需求上下文，关联到具体 requirement/plan，分析冲突块的意图关系（独立/重叠/矛盾），给出合并建议和置信度
5. **用户确认**：每个文件的合并结果都需要用户确认后才写入

**Q：一个开发者可以负责多个阶段吗？**

可以。为每个阶段创建独立分支，逐个合入：

```bash
milestone/v1.1-smart-rec/phase-1-user-service
milestone/v1.1-smart-rec/phase-4-notification-service
```

按顺序分别执行 `/gsd:merge-milestone milestone/v1.1-smart-rec`。

**Q：Driver 想查看整体进度怎么办？**

查看团队 milestone 分支的状态：

```bash
git checkout milestone/v1.1-smart-rec
git log --oneline
# 查看哪些 phase 已合入
# 查看 ROADMAP.md 中各 phase 的完成状态
```
