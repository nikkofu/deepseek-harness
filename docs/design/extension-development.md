# 二次开发：从需求到可运行插件

本页是二次开发的工作流。具体包骨架、工具定义和 LLM 适配器示例分别以[添加包](../cookbook/adding-a-package.zh.md)、[添加工具](../cookbook/adding-a-tool.zh.md)和[添加 LLM 适配器](../cookbook/adding-an-llm-adapter.zh.md)为准。

## 需求分流表

| 需求 | 主要扩展点 | 需要一起设计的内容 |
| --- | --- | --- |
| 增加只读或写入业务能力 | 业务 Service Definition + Provider + 工具 Consumer | schema、授权、错误、幂等、呈现、审计。 |
| 增加领域上下文 | 系统提示词 section、Skill 或受控注入 | 来源、更新时机、字节/token 上限、是否进入 Session 日志。 |
| 在调用前做策略 | `tools/pre-execute` 或 `ctx.tools.guard()` | 允许/拒绝/询问决策、最终执行点的防绕过检查。 |
| 对调用套超时、重试、指标 | `tools/execute` | cancellation、重试条件、异常规范化、资源释放。 |
| 观察最终结果 | `tools/result` | 不修改权威结果，只提交审计、遥测或投影。 |
| 新增持久对话状态 | `SessionEventMap` 与投影 | 版本、重放、模型可见性、历史恢复。 |
| 新增跨会话业务数据 | storage domain 或外部服务 | schema、并发、事务边界、租户隔离、保留期。 |
| 接入人类审批或提问 | approval / user questions / command Consumer | 等待语义、取消、前端或协议适配器。 |
| 接入外部协议或客户端 | ACP、SDK、Web Host 或协议桥 | 认证、连接取消、会话 owner、事件流和完全停稳。 |

若没有一项现有扩展点能表达需求，应先证明这是运行时主干缺口，而不是业务代码放错层。改动 Agent loop 会影响所有能力，必须同步更新[架构文档](../architecture.zh.md)。

## 推荐的包结构

当业务能力需要独立演进时，按职责拆为三个或四个包：

```text
packages/<group>/business-service/       Service Definition、类型、事件
packages/<group>/business-provider-x/    一个外部系统实现
packages/<group>/tool-business/          模型 Consumer 与呈现
packages/<group>/ui-business/            可选的 Web/协议 Consumer
```

Service Definition 依赖通用类型与核心服务，不依赖 Provider。Consumer 依赖 Definition，不依赖某个 Provider。Bundle 负责把某个 Provider 与 Consumer 共同装入部署。这样客户切换数据源、沙箱或模型时，无需复制工具逻辑。

## 实施顺序

1. 写清业务动作及其授权边界：谁可以读写什么、输入如何验证、何时需要审批、如何回退。
2. 为 Service Definition 定义面向消费者的请求、结果和稳定错误码。跨包不使用裸 `string` 表示不透明 id。
3. 用一个真实 Provider 实现调用、认证、限流、取消、超时和上游错误转换。凭据使用 `ctx.credentials` 的引用，不把密钥写入 config 或工具参数。
4. 实现 Consumer：工具需定义 JSON schema、模型可见说明、呈现类型和执行策略；UI 或协议 Consumer 从 Session 事件和业务视图渲染。
5. 确定持久化：模型可见事实扩展 Session 事件；领域记录进入 storage domain 或权威系统；不要让缓存成为事实来源。
6. 写实际组合：在最小可运行的 `cordis.yml` 或 Bundle patch 中装配 Service、Provider、Consumer、权限和持久化。
7. 增加验证：包测试覆盖业务规则，真实 Loader 组合测试覆盖装载与卸载，关键模型/用户可见行为通过可重放快照覆盖。

## 工具设计检查表

工具是模型的产品 API，而不是内部函数的转发器。一个工具至少需要回答以下问题。

| 问题 | 设计要求 |
| --- | --- |
| 模型能否正确选择它？ | 名称、描述和 schema 使用业务语言，避免暴露数据库表或内部 HTTP 路径。 |
| 调用是否最小化风险？ | 查询与写入分开；写入声明目标、影响范围、预览或审批要求。 |
| 是否可重复？ | 写动作携带幂等语义或在 Provider 中明确拒绝重复。 |
| 输出是否足够且受控？ | 返回任务所需字段，限制字节、条数和敏感字段；大结果使用已有 spill/摘要策略。 |
| 人能否理解执行过程？ | 预先确定 `generic`、`terminal` 或 `diff` 呈现意图与 locations。 |
| 是否可以绕过策略？ | 授权在最终 Provider/执行路径再次检查，而不只依赖 tool schema 或 prompt。 |

## 配置、凭据与客户隔离

Profile 和 Bundle patch 用于部署级配置；Agent Preset 用于单会话的能力集合；用户设置可覆盖允许变化的选择；凭据引用在每次请求时解析。客户专属的 endpoint、默认模型、最大并发、允许的空间和 Provider 路由应是验证过的配置字段。

租户 id、用户身份和权限集合必须来自已认证入口并显式传入业务服务。不要让模型传入任意租户、任意数据库连接或不受限制的文件路径。外部 Provider 的查询也应在服务端附加租户与范围条件，而不是相信模型生成的筛选条件。

## 调试与验证路径

先用 `dsh --profile <profile> --dump-config` 检查最终插件树和条目覆盖。然后在真实组合中验证：服务是否能装入、工具是否出现在预期 Agent scope、审批拒绝是否阻止了最终 Provider 调用、会话恢复是否能重建模型历史、dispose 是否移除了注册和后台资源。

测试不要只 mock 业务函数。至少保留一个通过 Loader 和运行时入口装配的测试，验证客户配置、凭据缺失、权限拒绝、上游失败和资源释放。对模型可见文案、工具 schema 或 transcript 的改动，增加或更新可重放快照。
