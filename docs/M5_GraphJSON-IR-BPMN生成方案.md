# M5 Graph-JSON IR 与 BPMN 生成方案

本文是一份**设计方案**（非代码精读），把 Smart-Workflow / Byteval 的编辑态思想，与"用节点-边表达的 graph-JSON 作为 IR 来生成 BPMN"的研究设想融合成一套可落地的方法论。覆盖：

- graph-JSON IR 的结构设计（图原生，忠于 BPMN 图论本质）
- NL → 澄清 → 局部图（function call）→ 组合 → validate → BPMN 的六阶段 pipeline
- SESE / RPST 作为"block 生成"与"graph 表示"之间的桥
- 两层 validate（片段级 + 组合级）检查项清单
- graph-JSON → BPMN 2.0（+DI）的 render 映射规则
- soundness 的形式化定义与 sound-by-construction 定理

先给结论：

```text
用 graph-JSON 做图原生 IR（忠实 BPMN），但把"模型生成"约束在嵌套的 SESE 片段上：
每次 function call 只吐一个结构完备的局部图。于是——
  澄清阶段先钉死网关语义 → 片段按结构化方式组合 → 结构化即 sound-by-construction，
  局部 validate 通过 ⟹ 全局 soundness 基本免费，只有 rigid（非结构化）片段才需重型验证。
最后 render 成 BPMN 2.0（+DI）。编辑仍走 function call，建模为保不变式的图变换。
```

一句话把 block/graph 之争收束：

```text
表示层用 graph（忠实 BPMN 的回边、共享汇聚、事件），
分析/生成层用 block（SESE/RPST 的模块性）。
两者不再二选一 —— 这是本方案的承重墙。
```

---

## 1. 融合命题与设计原则

### 1.1 核心张力：block vs graph

LLM 生成结构化流程模型时，业界主流是"块状/树形 JSON"（易生成、易校验，但难表达回边与共享汇聚）；忠于 BPMN 图论本质的是"图结构 JSON"（表达力强，但生成不稳定、验证难）。本方案的裁决：

```text
representation（表示） := graph-JSON      —— 图原生，不退化成树
analysis（分析/生成）  := SESE / RPST     —— 把图看成"嵌套块"

一句话：图是唯一真相，块是分析视角。
两个视角通过 RPST（Refined Process Structure Tree）无损互转。
```

### 1.2 五条设计原则（继承 Byteval + BPMN 适配）

```text
P1 图原生 IR       扁平 nodes[]/flows[]，关系与结构分离，容忍半成品   [继承 Byteval IR]
P2 语义左移        网关语义在"生成前"由澄清钉死，生成期只消费不猜测    [新增，soundness shift-left]
P3 结构化优先      function call 粒度 = SESE 片段；rigid 仅兜底         [新增，sound-by-construction]
P4 机器算候选      scope 用连通性∩类型算合法集，语义决策交给 Agent      [继承 Byteval scope]
P5 视图分离        layout 剥离为平行表 → 渲染成 BPMN-DI                 [继承 Byteval 改④ / 对齐 BPMN DI]
```

其中 P2、P3 是本方案相对 Byteval 的**增量贡献**，P1、P4、P5 是**直接继承**。

---

## 2. Byteval 思想 × 研究设想：逐条映射

| 研究设想（五步） | Byteval 可借鉴思想 | 融合后的升级 |
|---|---|---|
| ① 先需求澄清，控制网关无歧义 | ASCII Brief：**用户确认拓扑，Agent 管技术绑定**；**只在多候选/语义不确定时才问**（interaction-guide） | Brief 升级为**网关消歧协议**，产出结构化 `Intent Spec`，把 XOR/OR/AND、default、join 类型、循环边界在建图前钉死 |
| ② function call + 生成局部图 | `plan-apply` 批量子图；`add-node/add-edge/bind` 增量原语；`scope = 连通性∩类型` 把猜变选 | function-call **粒度上抬到 BPMN 模式片段（SESE）**，不是裸节点；`scope` 复用为数据引用候选收敛 |
| ③ 伴随 validate，局部+全局无问题 | `validate` 吃 IR、**容忍半成品**、一次收集全部、error/warn 两级、0-error 才准 upload；`deps = edges∪bindings` | **两层 validate**：片段 soundness + 组合 soundness；**补 Byteval 缺的反向可达（proper completion）** |
| ④ 转成 BPMN | IR→DSL `render` 六步；**layout 剥离为平行表**；`type::uuid` ID 生成 | `render`：graph-JSON → BPMN 2.0 XML；`layout` → **BPMN-DI**；readable-id → BPMN element id |
| ⑤ 编辑也走 function call | `remove-node` **级联清理**悬空边/绑定；`bind` **同端口覆盖幂等**；session **读写分离**（改内存、Save 才落盘） | 编辑 = **图变换规则**（前置/后置条件），保不变式 + **增量校验**（只重验受影响片段 + 缝合处） |

---

## 3. graph-JSON IR 结构设计

### 3.1 顶层结构

继承 Byteval IR 的四条编辑友好特征——**扁平 / 关系与结构分离 / 字段最小 / 容忍半成品**——但对 BPMN 做一处关键分叉：**控制流与数据流分开存**。

```jsonc
{
  "meta": {
    "name": "贷款审批",
    "notation": "bpmn",
    "version": "0.1",
    "source": ""                       // clone 来源（复用优先）
  },
  "nodes": [
    { "id": "start_0", "type": "event",   "subtype": "start" },
    { "id": "task_1",  "type": "task",    "subtype": "service", "name": "信用评估" },
    { "id": "gw_1",    "type": "gateway", "subtype": "exclusive", "default": "flow_3" },
    { "id": "gw_1j",   "type": "gateway", "subtype": "exclusive" },
    { "id": "end_ok",  "type": "event",   "subtype": "end" }
  ],
  "flows": [                            // 控制流 = BPMN sequenceFlow（网关只作用于它）
    { "id": "flow_1", "source": "start_0", "target": "task_1" },
    { "id": "flow_2", "source": "gw_1", "target": "task_2", "condition": "amount > 1e6" },
    { "id": "flow_3", "source": "gw_1", "target": "task_3" }   // default 分支
  ],
  "data":      [ { "id": "do_1", "type": "dataObject", "name": "Application", "dataType": "object" } ],
  "dataAssoc": [ { "id": "da_1", "source": "task_1", "target": "do_1", "direction": "out" } ],
  "layout":    { "task_1": { "x": 300, "y": 0 } },            // 视图分离 → BPMN-DI
  "fragments": [                                              // RPST 骨架（分析视角，见 §5）
    { "id": "f_xor1", "kind": "bond", "entry": "gw_1", "exit": "gw_1j",
      "gateway": "exclusive", "children": ["flow_2", "flow_3"] }
  ]
}
```

### 3.2 节点类型体系（对齐 BPMN 元素）

```text
type      subtype 取值                              BPMN 元素
--------  -------------------------------------     ----------------------
event     start / end / intermediate /              startEvent / endEvent /
          message / timer / error / signal          intermediateCatch/Throw
task      user / service / script / manual /         userTask / serviceTask /
          send / receive / businessRule / subProcess  scriptTask / ...
gateway   exclusive / parallel / inclusive /         exclusiveGateway (XOR) /
          eventBased / complex                       parallelGateway (AND) / inclusiveGateway (OR)
```

字段最小原则：节点只带 `id/type/subtype/name`，其余（表单、脚本、边界事件）延迟到 render 或按需 `params` 挂载——对齐 Byteval "application 只需 id+kind+app_id，schema 由 app-schema 回填"。

### 3.3 关键分叉：控制流 ⊥ 数据流（表示分开，分析合并）

这是相对 Byteval 执行引擎的**语义修正**。

```text
Byteval（执行引擎）：deps = edges ∪ ref-bindings   —— 合并
  理由：运行时"引用了谁就必须等谁先跑完"，数据引用天然是控制依赖。

BPMN（建模记法）  ：sequenceFlow ⊥ dataAssociation  —— 分开
  理由：BPMN 网关的分支/汇聚语义只作用于控制流；
        数据关联（dataInputAssociation）不参与 token 流动，不能影响网关判定。

融合裁决：
  表示层  flows 与 dataAssoc 分开存 → 忠于 BPMN 语义
  分析层  需要"上游可见性"时，临时算  vis = 传递闭包(flows ∪ dataAssoc)
          —— 这正是 scope 候选计算所需（§6.3），也复用 Byteval 的 ∪ 思想
```

一句话：**Byteval 把两者永久合并是执行态的正确；BPMN 需要分表存、按需合并，是建模态的正确。**

---

## 4. 六阶段生成 Pipeline

```text
NL 需求
  │
  ① 澄清循环 Clarification   ── 消解网关歧义 ──►  Intent Spec（网关语义已定的结构化意图）
  │        [借：ASCII Brief + 只在歧义时问]
  ▼
  ② 骨架规划 Skeleton        ── 出 SESE 片段树（RPST 骨架），每片段一个占位
  │        [借：复用优先 clone-ref]
  ▼
  ③ 片段生成 Fragment (FC)   ── 模型逐片段调 emit_xor/emit_parallel/... 吐局部 graph-JSON
  │        [借：plan-apply 批量子图 + scope∩type]     │
  │                                                    ├─► 局部 validate（片段 soundness）★
  ▼                                                    ▼   不过 → 退回该 fc 重生成
  ④ 组合缝合 Composition     ── 片段按骨架缝成全局 graph-JSON
  │        [借：validate 容忍半成品 / 两级 / 一次收集]  ├─► 全局 validate（组合+可达+数据流）★
  ▼                                                    ▼   不过 → 定位到片段/缝合处
  ⑤ 渲染 Render              ── graph-JSON → BPMN 2.0 XML（+ layout→BPMN-DI）
  │        [借：IR→DSL render 六步，layout 分离]        ├─► BPMN 层校验（schema + 语义）
  ▼
  ⑥ 编辑迭代 Edit (FC)       ── 每次编辑=图变换规则，保不变式 → 增量重验（片段+缝合处）
           [借：remove 级联清理 + bind 幂等覆盖 + 读写分离]
```

两条贯穿线：

```text
1. graph-JSON 是唯一编辑对象；BPMN 只在 ⑤ 生成，从不被增量编辑（编辑回到 graph-JSON）。
   —— 完全对应 Byteval "改 IR，render 出 DSL；DSL 不被增量改"。
2. validate 无处不在但分两层：③ 局部即时、④ 全局缝合、⑥ 增量。早失败早退回。
```

---

## 5. SESE 与 RPST：连接 block 与 graph 的桥（核心）

这是整个方案的承重墙，也是解决"图难生成/难验证"的关键。

### 5.1 SESE 片段定义

```text
定义 5.1（SESE 片段）
  给定工作流图 G=(N, F)（N 节点，F 控制流边）。
  片段 F ⊆ F 是一个 SESE（Single-Entry-Single-Exit）片段，当且仅当：
    ∃ 唯一入边 e_in 和唯一出边 e_out，使得
    F 中每条边都在从 e_in 到 e_out 的某条路径上，
    且 F 与图其余部分仅通过 e_in、e_out 相连。
```

直觉：SESE 片段是"一个能被单入单出封装的子流程"，可以整体当作一个抽象节点替换掉。

### 5.2 RPST 与片段分类

```text
定义 5.2（RPST，Refined Process Structure Tree）
  把工作流图唯一地、O(n) 线性时间地分解为嵌套 SESE 片段的树。
  每个片段属于以下四类之一：

  trivial   单条边（原子）
  polygon   顺序：子片段串行首尾相连（A → B → C）
  bond      网关配对：一个 split 网关 + 一个 join 网关，中间多条并列分支
            —— 这是 XOR / AND / OR 分支的结构化形态
  rigid     不可约：无法进一步分解为 polygon/bond 的复杂拓扑
            —— 非结构化（如跨分支同步、任意跳转、共享汇聚）
```

为什么它是桥：

```text
graph 表示 ：graph-JSON 忠实保留图结构（回边、共享汇聚、事件），不退化成树。
block 分析 ：RPST 把这张图看成"嵌套块"，于是同时获得三种能力——
             生成：每次 fc 只填一个 SESE 占位（块状、LLM 友好 → 命中"块状易生成"）
             验证：soundness 沿 RPST 自底向上组合（见 §5.3）
             复用：片段 = 可 clone 的模式（借 Byteval clone-ref）
```

### 5.3 Soundness 与 sound-by-construction 定理

```text
定义 5.3（Workflow-net Soundness，van der Aalst）
  一个工作流网 WF-net 是 sound 的，当且仅当同时满足：
    (S1) option to complete   ：从初始标识可达的任意标识，都能继续到达终止标识
    (S2) proper completion     ：到达终止标识时，网中不残留其他 token
    (S3) no dead transitions   ：每个变迁（活动）在某条执行路径上可被触发

映射到 BPMN：
    S1 ≈ 无死锁（每个 token 都能推进）
    S2 ≈ 每个活动都能反向到达 end event（★ Byteval 当前缺这条，见 §7）
    S3 ≈ 无不可达活动（Byteval 已实现正向可达）
```

```text
定理 5.4（sound-by-construction，结构化工作流）
  若工作流图 G 是"结构化"的——即 RPST 中不含 rigid 片段，
  且每个 bond 片段的 split 与 join 同类型配对（XOR↔XOR、AND↔AND），
  则在自由选择网（free-choice net）语义下，G 的 soundness 由构造保证，
  无需展开状态空间验证。

  证明思路（归纳）：
    - trivial / polygon 片段平凡 sound；
    - 配对的 bond 片段 sound（split 产生的 token 恰被同类型 join 消费）；
    - sound 片段的嵌套组合仍 sound（SESE 边界保证 token 不泄漏）。
```

**这条定理是本方案最省力的地方**：

```text
                  ┌────────────────────────────────────────────────┐
推论 5.5：验证负担坍缩 → 只有 rigid（非结构化）片段才需重型 soundness 检查。 │
                  └────────────────────────────────────────────────┘

策略：
  - 引导模型只用结构化片段（emit_xor/emit_parallel 自带配对 join）→ 绝大多数图 sound-by-construction；
  - 仅当必须非结构化（跨分支同步、事件驱动路由）才落 rigid，并对该 rigid 片段单独跑
    soundness（有界状态空间探索 / woflan-LoLA 式检查）。
```

---

## 6. 逐阶段深入

### 6.1 阶段①：澄清 → 网关消歧 → Intent Spec

Byteval 的原则是**别过度打扰**（interaction-guide：只在多候选/语义不确定时才问）。你要的"网关无歧义"正好给这个原则一个**精确触发条件**——只对下列五类结构歧义发问，其余不问：

```text
Q1 分支互斥性  多条件可同真时，是 XOR（择一，需优先级）/ OR（择多）/ 还是本就 AND（恒并行）?
Q2 默认路径    XOR/OR 若无条件命中，走哪? —— 缺 default ⟹ 潜在死锁（违反 S1）
Q3 汇聚语义    并行分支要同步（AND-join）还是简单合并（XOR-join）? —— split/join 错配是经典 bug
Q4 循环边界    回边起止、退出条件、while/until、最大迭代
Q5 隐式并行    "做 A、B、C" 是顺序还是并行?
```

产物是焊进 `Intent Spec` 的**网关语义表**：

```jsonc
{
  "gateways": [
    { "id": "gw_1", "semantics": "exclusive",
      "branches": [
        { "cond": "amount > 1e6", "goto": "人工复核" },
        { "cond": "else", "goto": "自动放款", "default": true }
      ],
      "join": { "type": "exclusive", "at": "gw_1j" }    // split/join 类型显式配对
    }
  ]
}
```

作用：**让 §6.3 的片段发射器"消费确定语义"，而不是让模型去猜网关类型**——这就是 P2 语义左移。职责边界同 Byteval：**用户确认业务拓扑（ASCII 图），Agent 处理技术绑定（不展示网关配对/DI 细节）**。

### 6.2 阶段②：骨架规划（RPST 占位树）

先出结构，再填内容——对齐 Byteval "复用优先，clone-ref"：

```text
1. 先搜可复用的完整/子流程（若有 → clone 成 graph-JSON 片段）
2. 无可复用 → 按 Intent Spec 生成 RPST 占位树：
     polygon[ start, «task占位», bond_xor(gw_1..gw_1j)[ «分支A», «分支B» ], end ]
3. 每个 «占位» 记一个待生成片段，交给阶段③ 逐个 emit
```

### 6.3 阶段③：function-call 片段生成（两层 API）

借 Byteval 两种建图模式（增量 `add-*` / 批量 `plan-apply`），但**上抬到 BPMN 模式粒度**，分两层：

```text
Tier-1 片段发射器（结构完备 by construction）—— 主力，对应"生成局部图"
  emit_sequence(entry, steps[], exit)
  emit_exclusive(split_props, branches[{cond, body}], default, join)    // 自带配对 XOR-join
  emit_parallel(branches[{body}], join)                                  // 自带 AND-join
  emit_inclusive(...) / emit_loop(body, exit_cond, kind) / emit_event_based(...)
     每个 emit_* 只吐一个 SESE 片段 → 立即跑局部 validate → 通过才纳入组合

Tier-2 图原语（细粒度，供编辑/修复）—— 对应 Byteval add-node/add-edge/bind
  add_node / add_flow / set_condition / set_default / bind_data /
  remove_node / remove_flow / replace_gateway_type
  scope(node, port)         // 借 Byteval：连通性∩类型 → 数据引用候选，把猜变选
  validate(scope=frag|all)  // 借 Byteval：两层
```

`scope` 的复用（数据绑定收敛）：某 task 的输入要绑哪个上游 dataObject / 活动输出时，用 `vis = 传递闭包(flows ∪ dataAssoc) ∩ 类型` 算合法候选，模型只从候选里选语义。算法与 Byteval `scopeCandidates`（连通性上游 ∩ `type_match`）同构——**机器算合法集，Agent 做语义决策**（P4）。

Function call 的 JSON schema 示例（emit_exclusive）：

```jsonc
{
  "name": "emit_exclusive",
  "description": "发射一个结构化 XOR 分支片段（自带配对的 split/join，保证 SESE）",
  "parameters": {
    "type": "object",
    "properties": {
      "entry":    { "type": "string", "description": "片段入口节点 id" },
      "branches": { "type": "array", "items": {
        "type": "object",
        "properties": {
          "cond": { "type": "string", "description": "conditionExpression；default 分支填 'else'" },
          "body": { "type": "array", "description": "该分支内的活动序列（graph-JSON 节点）" }
        }, "required": ["body"] } },
      "default_branch": { "type": "integer", "description": "default 分支下标（无命中时走）" }
    },
    "required": ["entry", "branches"]
  }
}
```

### 6.4 阶段③+④：两层 validate 检查项清单

继承 Byteval `validator.Validate` 的**性格**——吃 IR、容忍半成品、一次收集全部、error/warn 两级——但**分两层**，并补 BPMN 语义：

```text
【局部层】片段级（每次 emit_* 后即时跑，不过则退回重生成）
  frag_is_sese          error    片段恰好单入单出
  gateway_balanced      error    bond 内 split/join 同类型配对
  xor_or_default        error    XOR/OR split 有 default 或条件可证穷尽（防死锁）
  cond_expr_present     error    每条 XOR/OR 出边有 conditionExpression（default 除外）
  data_ref_in_scope     error    片段内数据引用可在 scope（vis 闭包）内解析
  parallel_no_cond      warning  AND split 出边不应带条件（BPMN 语义：并行无条件）

【全局层】组合级（缝合后跑，对齐 soundness S1-S3）
  start_unique          error    恰好 1 个 start event          [借 Byteval start_count]
  end_exists            error    至少 1 个 end event            [借 Byteval end_missing]
  forward_reachable     warning  从 start 正向可达每个节点      [借 Byteval，已实现]
  proper_completion     error    每节点能反向到达某 end event   [★ 补 Byteval 缺口，= S2]
  no_dead_activity      warning  无死活动                       [= S3]
  global_gateway_balance error   经 RPST 校验结构化；rigid 片段单独跑 soundness  [= S1]
  data_flow_sound       warning  无 missing-data（读了没人产）/ redundant-data（产了没人用）
  type_compatible       error    数据关联类型兼容               [借 Byteval type_mismatch]
  cross_boundary_ref    error    跨 pool/lane/subProcess 引用合规 [借 cross_scope_ref_forbidden]
  no_cycle_unless_loop  error    非 loop 片段内无回边           [借 Byteval cycle 检测]
```

两点专门强调：

```text
1. 补齐 Byteval 已知缺口：其 checkReachability 只做正向 BFS（bfs(startID, next)），
   缺"反向到达 End"。在 BPMN 里反向可达 = proper completion（soundness S2 必要条件），
   本方案必须补一个从 end event 反向的对称 BFS。

2. 增量校验：因 fc 增量，校验也增量——只重验被动片段 + 它与邻居的缝合边，而非全图重算。
   对齐 Byteval 每命令 Save 的读写分离。
```

### 6.5 阶段⑤：render（graph-JSON → BPMN 2.0）

对标 Byteval `Renderer.Render`（IR→DSL 六步），映射到 BPMN：

```text
1. ID 生成    readable(task_1) → BPMN id（Activity_x）；维护 idMap        [借 render 步骤1]
2. 节点展开   node.type/subtype → task / gateway / event 元素
3. 边转换     flow{source,target,condition} → <sequenceFlow> + <conditionExpression>
4. 数据关联   data/dataAssoc → <dataObject> + <dataInput/OutputAssociation>
5. 网关配对   Intent Spec 的 join 语义 → 生成配对 join 元素 + gatewayDirection
6. 布局分离   layout → <bpmndi:BPMNDiagram>（DI）；纯 API 路径可不填 → 自动布局   [借改④]
```

BPMN-DI 分离的官方依据：**BPMN 的 Diagram Interchange（DI）与语义模型本就分离**——正好对齐 Byteval 把 position 从 Node 剥离成平行 `layout` 表（改④）。你的 graph-JSON 天然对齐这条标准。

render 片段示例：

```xml
<exclusiveGateway id="Gateway_1" default="Flow_else" gatewayDirection="Diverging"/>
<sequenceFlow id="Flow_2" sourceRef="Gateway_1" targetRef="Task_Review">
  <conditionExpression xsi:type="tFormalExpression">amount &gt; 1000000</conditionExpression>
</sequenceFlow>
<sequenceFlow id="Flow_else" sourceRef="Gateway_1" targetRef="Task_AutoPay"/>
```

### 6.6 阶段⑥：function-call 编辑（图变换 + 不变式）

编辑复用同一套 fc，但每个操作建模为**带前置/后置条件的图变换规则**，借 Byteval 两个语义：

```text
借 bind 同端口覆盖幂等：set_condition / bind_data 对同目标覆盖（不重复追加）
借 remove-node 级联清理：删活动 → 自动清悬空 flow / dataAssoc

新增 BPMN 不变式保持（否则报错，而非静默破坏）：
  删网关一个分支 → 要么自动 re-balance（补 default / 退化为顺序），要么报 gateway_unbalanced
  替换网关类型   → 检查 split/join 仍配对
  删活动落在 bond 内 → 维持 SESE，否则触发片段修复
每次编辑后：增量重验（受影响片段 + 缝合处），不全图重算。
```

本质是**图重写 / 图文法**：编辑规则集若能证明"保 soundness"，则整个编辑会话不变式恒真——落在形式化验证的领地。

---

## 7. 相对 Byteval 的三个明确增量

```text
增量①（语义左移）：Byteval 在建好后 validate 网关；本方案在生成前澄清钉死网关语义，
                    让模型消费而非猜测 → 从源头减少非法图。

增量②（sound-by-construction）：Byteval 逐条静态检查；本方案用 SESE/RPST 让结构化图
                    的 soundness 由构造保证，验证负担坍缩到 rigid 片段。

增量③（补反向可达）：Byteval 的 checkReachability 只有正向 BFS（memory 已记为已知缺口）；
                    本方案补 proper_completion（从 end 反向 BFS），对齐 soundness S2。
```

反过来，三个**直接继承**：graph-JSON 的扁平/关系分离/容忍半成品（IR 哲学）、scope 的连通性∩类型（把猜变选）、layout 平行分离（→ BPMN-DI）。

---

## 8. 贯穿微例子：贷款审批

```text
NL："金额超100万走人工复核，否则自动放款"

① 澄清：Q1 互斥? → 是（XOR）；Q2 默认? → 自动放款；Q3 汇聚? → 两分支都到"放款完成"（XOR-join）
   Intent Spec: gw_1 = exclusive, default = 自动放款, join = exclusive@gw_1j

② 骨架：polygon[ start, 信用评估, bond_xor(gw_1 .. gw_1j), end ]

③ fc：emit_sequence(start → 信用评估 → gw_1)
      emit_exclusive(gw_1, [{amount>1e6 → 人工复核}, {else → 自动放款, default}], join=gw_1j)
      → 局部 validate：frag_is_sese ✓  gateway_balanced ✓  xor_or_default ✓

④ 缝合全局 → validate：proper_completion ✓（两分支都反向可达 end）  no_dead_activity ✓

⑤ render → <exclusiveGateway id="Gateway_1" default="Flow_else"> +
           <conditionExpression>amount &gt; 1000000</conditionExpression>

⑥ 编辑："再加金额>500万走董事会" → replace 单条件 bond 为 emit_exclusive 三分支
        → 增量重验 bond_xor 片段即可，全局不动。
```

---

## 9. 关键取舍（答辩/写作用）

| 张力 | 选择 | 理由 |
|---|---|---|
| block vs graph | **graph 表示 + block（SESE/RPST）分析** | 忠实 BPMN 图论 + 获得块状的生成/验证模块性 |
| 控制流 / 数据流 | **表示分开，分析合并** | 忠于 BPMN 语义（网关只看控制流）+ 复用 Byteval `∪` 做 scope |
| 网关语义 | **澄清期锁定，生成期消费** | soundness 左移；模型不猜网关，只填已定语义的片段 |
| 生成粒度 | **SESE 片段优先，rigid 兜底** | 结构化 ⟹ sound-by-construction，验证负担坍缩 |
| 何时问用户 | **仅五类结构歧义** | 借 Byteval "别过度打扰"，给精确触发条件 |
| 验证时机 | **局部即时 + 全局缝合 + 增量** | 借 Byteval 容忍半成品 + 早失败早退回 |
| 编辑对象 | **只编辑 graph-JSON，BPMN 只读产物** | 对齐 Byteval "改 IR，不改 DSL" |

---

## 10. 落地优先级与风险

```text
P0 骨架：graph-JSON schema + RPST 分解 + 局部/全局 validate（先把 soundness 闭环打通）
P1 生成：Tier-1 片段发射器（emit_*）+ 澄清网关消歧协议
P2 渲染：graph-JSON → BPMN 2.0（+DI）；BPMN schema / 语义校验
P3 编辑：Tier-2 图变换 + 增量校验；scope 数据绑定收敛
```

风险与缓解：

```text
- rigid 片段 soundness 验证是唯一重型环节（状态空间可能爆炸）
    → 用有界探索 / 自由选择网化简 / 尽量引导结构化避免 rigid
- 澄清过度会打断体验
    → 严守"仅五类结构歧义"触发条件（P2 精确触发）
- OR-join（inclusive）语义在 BPMN 里 non-local、易错
    → MVP 先禁 OR-join，只 XOR/AND（结构化），后续再补
- 数据流 soundness（missing/redundant data）常被忽略
    → 纳入全局 validate，别只查控制流
- LLM 生成 rigid 片段时结构不稳定
    → function call schema 强约束；生成后立即局部 validate，不过即退回
```

---

## 11. 一句话总结

```text
本方案用 graph-JSON 做图原生 IR（忠实 BPMN），把 Byteval 的"改 IR / render 出产物、scope 把猜
变选、validate 容忍半成品、layout 分离"直接继承过来；再叠加三个增量——网关语义在澄清期左移锁定、
用 SESE/RPST 让结构化图 sound-by-construction、补上 Byteval 缺的反向可达（proper completion）。
生成走 function call 逐片段吐局部 SESE 图并即时局部 validate，缝合后跑全局 soundness，最后
render 成 BPMN 2.0（layout→DI）；编辑同样走 function call，建模为保不变式的图变换 + 增量重验。
```
