# LangGraph Loop Architect

一个面向通用业务场景的 Codex Skill，用于在以下架构之间做有依据的选择，并输出可执行的技术方案：

- 单 Agent 全量工具暴露；
- 渐进式领域披露；
- 确定性快路径与 Supervisor Loop 组合模式；
- 大规模动态工具检索；
- 固定状态机与有限 Agent 节点；
- 开放式 ReAct / Code Agent Loop。

Skill 会根据工具规模、领域边界、依赖不确定性、延迟、成本、权限和副作用风险，设计 Supervisor/Worker 职责、LangGraph 状态流转、JSON 契约、动态依赖、能力边界、降级/回滚/补偿、人工介入、测试和上线步骤。

ASR、TTS、翻译、视觉、图像生成和 3D 等均为场景条件能力。没有相应业务输入或输出时，Skill 会将其标记为 `not_applicable`，不会强制引入。

## 安装

通过 Codex 的 `$skill-installer` 安装本仓库，或者将仓库克隆到：

```text
~/.codex/skills/langgraph-loop-architect
```

重新进入一个 Codex 会话后即可自动发现，也可以显式调用：

```text
$langgraph-loop-architect
```

## 使用示例

```text
使用 $langgraph-loop-architect 分析这个售后业务场景。
当前有订单、物流、退款、会员和语言五个领域共 48 个 API，
部分退款操作有副作用，要求给出工具暴露策略、Supervisor Loop、
JSON 状态契约、动态依赖处理、性能权衡和测试方案。
```

## 目录

```text
SKILL.md
agents/openai.yaml
references/
  architecture-selection.md
  capability-boundaries.md
  supervisor-loop.md
  json-contracts.md
  resilience-human-control.md
  evaluation-playbook.md
  delivery-template.md
  worked-examples.md
```

详细使用规则和引用路由见 [`SKILL.md`](SKILL.md)。
