# DeepSeek Harness 二次开发、FDE 与创业设计手册

本目录面向要把 DeepSeek Harness 用作产品底座的二次开发工程师、FDE（Forward Deployed Engineer，前线部署工程师）和早期创业团队。它解释如何把已有的插件化 agent runtime 组合成可交付的业务系统，而不是重复生成的 API 目录或逐个包的 README。

本文档集只描述当前仓库中已经存在的运行时机制；关于客户系统、数据源和领域流程的内容均以“建议”明确标注，不把建议写成产品已具备的能力。

## 阅读路径

| 读者 | 建议顺序 | 可获得的结果 |
| --- | --- | --- |
| 架构师或技术负责人 | [系统架构](architecture.md) → [设计原则](design-principles.md) → [生产参考架构](production-reference-architecture.md) | 能确定运行时边界、部署形态和长期演进规则。 |
| 二次开发工程师 | [系统架构](architecture.md) → [二次开发](extension-development.md) → [设计原则](design-principles.md) | 能把需求放到正确的插件、事件、工具或持久化扩展点。 |
| FDE | [FDE 交付方法](fde-delivery.md) → [生产参考架构](production-reference-architecture.md) → [二次开发](extension-development.md) | 能把客户目标拆成可验收的试点、集成和运营闭环。 |
| 创始人、产品负责人 | [创业与产品化策略](venture-strategy.md) → [FDE 交付方法](fde-delivery.md) → [生产参考架构](production-reference-architecture.md) | 能把定制交付沉淀为可配置产品，而非不断复制项目。 |

## 本目录的边界

DeepSeek Harness 的权威运行时说明仍在[架构文档](../architecture.zh.md)、[能力 seam 图](../capability-seams.zh.md)和[子系统参考](../subsystems/README.zh.md)中。这里的目标是把这些基础能力组织成“做什么、在哪里扩展、怎样交付、如何运营”的决策材料。

不要把本目录当作接口定义。工具参数与结果以生成的[工具目录](../tool-catalog.zh.md)为准；Cordis 配置字段以[配置目录](../config-catalog.zh.md)为准；某个包的配置、限制和模型可见效果以该包 README 为准。

## 一句总览

DeepSeek Harness 是一个由 Cordis 插件组成的 agent harness：Bundle 和 Profile 决定每次部署装入什么，服务与事件提供协作接口，Session 事件日志保留模型可见历史，Agent loop 将输入、模型、工具和后续步骤组织为可观察轮次。

二次开发的核心不是修改 loop，而是把业务能力实现为可组合插件：用 Service Definition 声明跨实现接口，用 Service Provider 接入数据或执行系统，用 Consumer 把受控能力呈现给模型、UI 或外部协议。

## 实施中的优先级

先选择一个高频、边界清楚、可人工复核的工作流，再接入最少的只读数据和受控动作。把结果、审批、审计、成本和失败处理设计为第一期功能。只有当试点能稳定复用时，才把客户专属规则上移为配置、Preset、Skill 或独立 Provider。
