# Warm-Flow 工作流引擎深度解析：从 7 张表看懂设计哲学

> **摘要**：Warm-Flow 是一款国产轻量级工作流引擎，仅用 7 张核心表就实现了完整的审批流功能。本文从数据表设计的维度出发，深入探讨设计时与运行时的边界划分、权限标识的解析机制，以及框架与业务系统解耦的架构思想。

---

## 一、为什么是 7 张表？

相比 Activiti/Flowable 的 25+ 张表，Warm-Flow 用 7 张表完成了同样完整的工作流能力。这背后遵循的是 **KISS 原则**——能用简单方案解决的问题，绝不引入不必要的复杂度。

### 1.1 7 张表全景

| # | 表名 | 一句话 | 生命周期 |
|---|------|--------|---------|
| 1 | `flow_definition` | 流程定义——流程图模板 | 设计时 |
| 2 | `flow_node` | 流程节点——图中的每个方块 | 设计时 |
| 3 | `flow_skip` | 流转条件——节点间的箭头 | 设计时 |
| 4 | `flow_instance` | 流程实例——某次具体的流程运行 | 运行时 |
| 5 | `flow_task` | 待办任务——当前轮到谁处理 | 运行时 |
| 6 | `flow_his_task` | 历史任务——已经处理完的记录 | 运行时 |
| 7 | `flow_user` | 流程用户——谁有权处理待办 | 运行时 |

### 1.2 最核心的分界线：设计时 vs 运行时

理解 Warm-Flow 架构的关键在于区分 **"定义"** 和 **"运行"**：

```
┌─────────────────────────────────┐
│          设计时（管理员画图）       │
│  flow_definition                 │
│    ├─ flow_node（方块）          │  ← 定义流程模板
│    └─ flow_skip（箭头+条件）      │
└─────────────────────────────────┘
               ↓ 发布
┌─────────────────────────────────┐
│          运行时（用户执行）         │
│  flow_instance（一次运行）        │
│    ├─ flow_task（当前待办）       │  ← 实际流转
│    ├─ flow_his_task（历史脚印）   │
│    └─ flow_user（权限分配）       │
└─────────────────────────────────┘
```

---

## 二、设计时：3 张定义表

### 2.1 定义表的内容长什么样？

以请假流程为例，管理员在设计器拖拽后，数据库写入三条记录：

**`flow_definition` 流程定义表：**

| id | flow_code | flow_name | version | is_publish |
|----|-----------|-----------|---------|------------|
| 1 | leave | 请假流程 | 1.0 | 1 |

**`flow_node` 流程节点表：**

| id | definition_id | node_code | node_name | permission_flag | node_type |
|----|--------------|-----------|-----------|----------------|-----------|
| 1 | 1 | start | 开始 | (空) | 0-开始节点 |
| 2 | 1 | apply | 填写申请 | (空) | 1-中间节点 |
| 3 | 1 | manager | 主管审批 | `role:100` | 1-中间节点 |
| 4 | 1 | hr | 人事审核 | `role:200` | 1-中间节点 |
| 5 | 1 | end | 结束 | (空) | 2-结束节点 |

**`flow_skip` 流转条件表：**

| id | definition_id | now_node_code | next_node_code | skip_condition |
|----|--------------|---------------|----------------|----------------|
| 1 | 1 | start | apply | (无条件) |
| 2 | 1 | apply | manager | (无条件) |
| 3 | 1 | manager | hr | `${days > 3}` |
| 4 | 1 | manager | end | `${days <= 3}` |

### 2.2 定义阶段的关键字段：`permission_flag`

这个字段是 Warm-Flow 设计中最值得品味的地方。它存储的是**办理人的占位标识**，支持 5 种类型：

| 类型 | 存储格式 | 例子 | 含义 |
|------|---------|------|------|
| 用户 | 直接存 ID | `"101"` | 用户 ID=101 的人处理 |
| 角色 | `role:` 前缀 | `"role:100"` | 角色 ID=100 的所有人 |
| 部门 | `dept:` 前缀 | `"dept:200"` | 部门 ID=200 的所有人 |
| 岗位 | `post:` 前缀 | `"post:50"` | 岗位 ID=50 的所有人 |
| SPEL | `$` 或 `#` 开头 | `"${managerId}"` | 从变量中动态解析 |

**多个标识用 `@@` 分隔**：`"role:100@@dept:200"`

### 2.3 发布校验：框架到底校验了什么？

```java
// FlwDefinitionServiceImpl.publish()
if (StringUtils.isBlank(flowNode.getPermissionFlag())
    && !isStartNode
    && isIntermediateNode) {
    errorMsg.add(flowNode.getNodeName());
}
```

这段代码值得认真分析——它**只校验了字符串不为空**，完全没有验证内容是否合法。这意味着：

- `"role:100"` → 通过（不空白即可）
- `"${managerId}"` → 通过
- `"随便写的字符串"` → **也通过**

**框架对 `permission_flag` 是零认知的。** 它不知道"角色"是什么，不知道"部门"对应什么，甚至不关心写的是不是合法格式。它只当这是一个字符串占位符。

为什么这样设计？因为框架不需要知道——**校验角色存不存在是业务系统的事，不在框架职责范围内**。这体现了清晰的职责边界：框架只负责"存"和"传"，业务系统负责"解释"。

---

## 三、运行时：4 张动态表

### 3.1 启动流程时发生了什么？

用户提交请假单，调用 `startCompleteTask()`，引擎的操作：

**① 创建流程实例（`flow_instance`）**

| id | definition_id | business_id | node_code | flow_status |
|----|--------------|-------------|-----------|-------------|
| 1001 | 1 | `"BIZ001"` | manager | 1-审批中 |

- `definition_id` = 1：关联到"请假流程 v1.0"
- `business_id` = `"BIZ001"`：关联到业务系统的请假单
- `node_code` = `"manager"`：当前走到"主管审批"节点

**② 创建待办任务（`flow_task`）**

| id | instance_id | node_code | node_name | flow_status |
|----|------------|-----------|-----------|-------------|
| 2001 | 1001 | manager | 主管审批 | 1-待审批 |

**③ 分配审批人（`flow_user`）**

这是最关键的环节。Warm-Flow 核心引擎在 `addTask()` 方法内部：

1. 读取 `flow_node.permission_flag`，拿到 `"role:100"`
2. 调用业务系统注入的 `PermissionHandler.convertPermissions(["role:100"])`
3. 业务系统查询数据库：角色 ID=100 对应哪些用户 → `["101", "102", "103"]`
4. 插入 `flow_user` 表

```sql
INSERT INTO flow_user (type, processed_by, associated) VALUES
(1, '101', 2001),
(1, '102', 2001),
(1, '103', 2001);
```

**`flow_user` 存的是具体用户 ID（"101"），不是角色标识（"role:100"）。**

### 3.2 flow_user：设计时 vs 运行时的"翻译层"

这是 Warm-Flow 最精妙的设计点之一。`flow_user` 充当了**设计时到运行时的翻译层**：

```
设计时（flow_node）
  permission_flag = "role:100"       ← 框架不理解的字符串
                      │
                      ▼ 运行时（PermissionHandler.convertPermissions）
  flow_user = [
    { processed_by: "101" },         ← 框架能理解的字符串
    { processed_by: "102" },
    { processed_by: "103" }
  ]
                      │
                      ▼ 查询待办时
  WHERE processed_by = '当前用户ID'  ← 纯字符串匹配，简单高效
```

**业务系统负责翻译（role:100 → [101,102,103]），框架负责匹配（"101" == "101"）。**

`flow_user` 的 `processed_by` 字段在运行时永远存储**企业系统里的真实用户 ID**。框架对它的全部操作就是字符串等值匹配——这正是它轻量的根源：不需要理解数据结构，只做最简单的字符串比对。

### 3.3 待办查询：为什么能如此简单？

```sql
-- 查询某用户的待办任务
SELECT t.id, uu.processed_by
FROM flow_task t
LEFT JOIN flow_user uu ON uu.associated = t.id
WHERE uu.processed_by = '101'          ← 当前登录用户ID
  AND uu.type IN ('1', '2', '3')       ← 审批人/转办人/委托人
```

没有复杂的角色层级解析，没有权限树的递归遍历，就是**一个 `WHERE` 条件 + 字符串等值匹配**。

这正是将"角色展开"提前到任务创建时的好处——查询时就不需要再展开，代价是每次创建任务时多做一次角色展开。对于"读多写少"的工作流场景，这是合理的取舍。

---

## 四、SPEL 表达式解析机制

### 4.1 什么是 SPEL 表达式？

SPEL（Spring Expression Language）表达式是 Warm-Flow 提供的另一种动态指定审批人的方式：

```
permission_flag = "${managerId}"
```

这里的 `managerId` 是一个**运行时的流程变量**，在调用 `startWorkflow()` 时传入：

```java
Map<String, Object> variables = new HashMap<>();
variables.put("managerId", "user_002");
variables.put("days", 5);

workflowService.startCompleteTask(dto.setVariables(variables));
```

### 4.2 解析链

解析时机在 `FlwTaskServiceImpl` 第 689 行：

```
① taskService.addTask()             → 用原始值创建 task
② ExpressionUtil.evalVariable()     → ${managerId} → "user_002"
③ fetchUsersByStorageIds()          → "user_002" → 查用户表确认存在
④ flowNode.setPermissionFlag(...)   → 写入最终值
```

### 4.3 两种动态机制的对比

| | 静态标识（`role:100`） | SPEL 表达式（`${managerId}`） |
|--|----------------------|------------------------------|
| 解析时机 | 创建 task 时 | 创建 task 后，通过 `ExpressionUtil` |
| 解析源 | 角色/部门/岗位表 | 运行时传入的 variables Map |
| 灵活性 | 需预配置角色 | 完全由业务代码决定 |
| 适用场景 | 固定角色的审批 | 动态指定审批人 |

---

## 五、分阶段数据流总览

### 5.1 定义阶段

```
管理员画图
    │
    ├─ 拖拽节点 → flow_node（node_code, node_name, permission_flag）
    ├─ 画箭头   → flow_skip（now_node_code, next_node_code, skip_condition）
    ├─ 保存模板 → flow_definition（flow_code, flow_name, version）
    │
    ▼ 点击发布
    └─ 校验：permission_flag 不能为空（仅检查空白，不验证内容）
```

### 5.2 启动阶段

```
用户提交申请
    │
    ├─ flow_instance 新增一行（关联 definition_id 和 business_id）
    │
    ├─ taskService.addTask()
    │   ├─ flow_task 新增一行
    │   └─ 读取 permission_flag
    │       ├─ "role:100" → PermissionHandler.convertPermissions → [101,102,103]
    │       │                ↓
    │       │                flow_user 插入多条（每用户一条）
    │       └─ "${managerId}" → 暂存待解析
    │
    └─ ExpressionUtil.evalVariable()
        └─ "${managerId}" → variables.get("managerId") → "user_002"
```

### 5.3 流转阶段

```
审批人处理（completeTask）
    │
    ├─ flow_task → flow_his_task（完成待办，迁移到历史）
    │
    ├─ 读取 flow_skip，判断条件
    │   ├─ days > 3  → 走人事审核分支
    │   └─ days ≤ 3  → 直接结束
    │
    ├─ 生成下一个节点的 flow_task
    │
    └─ 重复：解析 permission_flag → 插入 flow_user
```

### 5.4 结束阶段

```
到达结束节点
    │
    └─ flow_instance.flow_status = "8"（已完成）
       flow_task 全部完成
       flow_his_task 完整归档
```

---

## 六、架构设计的底层哲学

### 6.1 框架边界：只做"字符串匹配"

Warm-Flow 最核心的设计决策就是**框架不感知业务语义**。整个引擎的核心操作只有两个：

1. **存字符串**：把 `permission_flag` 从设计时原样存到 `flow_node`
2. **匹配字符串**：`flow_user.processed_by = '当前用户ID'`

至于"role:100"是什么意思、`${managerId}` 从哪取值——都是业务系统的责任。

### 6.2 解耦点：PermissionHandler 接口

```
框架核心                        业务系统
┌────────────┐               ┌─────────────────┐
│  addTask() ├─── 调用 ───→  │ convertPermissions │
│  不关心"角色"│              │  role:100 → [101,102,103]
│  只存/比字符串│              │  查数据库、查缓存    │
└────────────┘               └─────────────────┘
```

### 6.3 变化点在哪里？

| 变化 | 影响范围 |
|------|---------|
| 角色下的用户变动 | 只影响业务系统的角色表，框架无需改 |
| 新增审批人类型 | 扩展 `PermissionHandler` 即可 |
| 流程路径变化 | 设计器重画，无需改代码 |
| 审批规则变化 | 属于业务逻辑，与框架无关 |

### 6.4 轻量背后的权衡

写时展开 vs 读时展开的取舍：

```
Warm-Flow 选择：写时展开（创建 task 时把角色展开为用户 ID）
  优点：查询快（直接 WHERE 等值匹配）
  缺点：角色变化时，已创建的 task 不会自动更新

Activiti 选择：读时展开（查询时再根据角色过滤）
  优点：角色变化实时生效
  缺点：SQL 复杂，性能开销大
```

没有绝对优劣，只有场景适配。对于审批流场景，"人事变动不回溯已审批的历史"是合理假设，所以写时展开是更实用的选择。

---

## 七、写在最后

Warm-Flow 的 7 张表设计本质上是对 **"定义" 和 "运行" 两个概念在存储层面的极致分离**。3 张定义表负责"怎么走"，4 张运行时表负责"走到哪了"。

而 `permission_flag` → `flow_user` 的解析链，则展示了框架与业务系统之间的一条清晰边界线：框架做好"存字符串"和"比字符串"两件事，业务系统通过 `PermissionHandler` 接口塞入自己的语义解释。各司其职，互不入侵。

对于开发者的启示：**一个好的框架设计，不是把各种能力包揽在自己身上，而是划清楚责边界，留好扩展点，把业务交给业务系统。**

---

*本文基于 RuoYi-Plus 项目集成 Warm-Flow 的代码实现编写。Warm-Flow 开源地址：https://github.com/dromara/warm-flow*
