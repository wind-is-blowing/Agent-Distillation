# 03 Harness 契约与搜索空间

## 3.1 设计目的

自进化系统必须先定义“什么可以进化”。如果候选可以同时修改策略、权限、预算和评分，任何分数提升都无法解释。Harness 契约用于固定边界、缩小搜索空间并保证新旧版本可比较。

## 3.2 标准接口

建议把可演化 harness 暴露为一个纯配置加少量策略函数的包：

```python
class Harness(Protocol):
    manifest: HarnessManifest

    def bootstrap(self, task: TaskSpec, env: EnvView) -> list[Message]: ...
    def render_observation(self, event: ToolResult, state: AgentState) -> Message: ...
    def select_context(self, history: list[Event], budget: ContextBudget) -> list[Message]: ...
    def next_action_policy(self, state: AgentState) -> ActionDirective: ...
    def completion_checklist(self, state: AgentState) -> list[Check]: ...
    def summarize_for_resume(self, history: list[Event]) -> ResumeState: ...
```

Harness 只返回策略结果，不直接创建容器、修改预算计数或写入评分数据库。工具执行和权限检查由 runtime 完成。

## 3.3 Manifest

每个候选必须携带机器可校验的 `harness.yaml`：

```yaml
schema_version: 1
name: context-tail-preservation
entrypoint: harness.impl:CandidateHarness
parent_version: H_003
proposal_id: prop_01J...
allowed_capabilities:
  - context.compose
  - observation.render
limits:
  max_bootstrap_bytes: 32768
  max_observation_bytes: 30720
  max_resume_state_bytes: 16384
compatibility:
  runtime_api: ">=1.0,<2.0"
```

候选不能在 manifest 中申请超过实验配置的权限或预算。控制器使用“实验上限与候选声明的交集”作为有效能力。

## 3.4 可演化模块

建议按风险逐步开放。

| 阶段 | 模块 | 可搜索内容 | 主要风险 |
|---|---|---|---|
| MVP | `system.md` | 角色、完成标准、工具使用提醒 | 提示过长、任务特化 |
| MVP | `bootstrap.py` | 初始仓库和环境信息选择 | 时间开销、敏感信息泄漏 |
| MVP | `observation.py` | 长输出截断、错误突出、原始定位 | 删除关键上下文 |
| V1 | `context.py` | 历史选择、压缩、恢复摘要 | 遗忘关键决策 |
| V1 | `completion.py` | 完成前检查和重复验证抑制 | 死循环、过早结束 |
| V2 | `planning.py` | 恢复策略、失败后的策略切换 | 过度约束模型 |
| V2 | `memory.py` | 任务内记忆内容和读取时机 | 污染、成本增长 |

MVP 每个候选只改一个主要模块；跨模块重构作为显式的独立实验。

## 3.5 受保护模块

下列路径由代码所有权和运行时挂载共同保护：

```text
control/
evaluation/
benchmarks/
security/
schemas/
runtime/budget.py
runtime/tool_executor.py
runtime/event_writer.py
tests/hidden/
```

保护不能只依赖提示词。候选 worktree 使用路径白名单生成补丁；运行容器只读挂载受保护代码；提交前再次比较 Git diff。

## 3.6 变更预算

为提高归因质量，每个候选设置变更预算：

- 最大修改文件数；
- 最大净新增行数；
- 最大系统提示长度；
- 最大新增依赖数，MVP 建议为 0；
- 最大 bootstrap 时间和输出字节；
- 禁止任务 ID、答案片段和 benchmark 专用字符串。

变更预算不是越小越好。大范围重构可以通过专门实验申请更大预算，但必须独立评测，不能与多个局部修复混在同一候选中。

## 3.7 候选生成协议

改造学生的输出包括：

```json
{
  "base_version": "H_003",
  "proposal_id": "prop_01J...",
  "changed_files": ["harness/observation.py"],
  "claimed_effect": "在固定字节预算中同时保留工具输出头尾",
  "known_risks": ["中部日志不可见"],
  "self_checks": [
    {"command": "pytest tests/contract/test_observation.py", "exit_code": 0}
  ],
  "candidate_commit": "<full git sha>"
}
```

控制器重新计算 `changed_files`、diff 统计和 commit SHA，不信任候选自报结果。

## 3.8 契约测试

任何候选进入任务评测前必须通过：

1. 包可导入，entrypoint 可实例化。
2. 所有返回对象通过 schema 校验。
3. 同一固定输入下不产生越权副作用。
4. 消息与工具调用配对有效。
5. 截断与压缩严格服从字节/token 上限。
6. bootstrap 超时或失败时能退化为原流程。
7. completion 状态机不会在常见路径中死循环。
8. 受保护路径、依赖和配置没有变化。

这些测试证明候选可运行，并不证明它能提高任务能力；能力由第 08 章的 benchmark 评测裁决。

## 3.9 初始基线

初始 `H_000` 应保持简单和可解释：

- 一个稳定系统提示；
- 原生工具调用；
- 确定性的头尾截断；
- 基本完成清单；
- 不跨任务共享记忆；
- 记录完整原始输出与实际展示内容。

基线越复杂，教师越难从轨迹中归因。已有成熟 Coding Agent 可以作为基线，但要先把其固定控制逻辑与可演化策略拆开。
