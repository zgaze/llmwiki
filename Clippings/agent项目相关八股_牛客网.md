---
title: "agent项目相关八股_牛客网"
source: "https://www.nowcoder.com/discuss/880875757633888256"
author:
published:
created: 2026-05-07
description: "1. 怎么把 RAG 接入到你的 Agent 执行链路里的？是在调用工具之前先检索，还是在生成结果之后再做校验？我们的 RAG 是通过 Spring AI 的 Advisor 接入到 ChatClient 的：模型调用前在 before() 做向量检索，把命中文档拼成上下文注入 prompt；模型生_牛客网_牛客在手,offer不愁"
tags:
  - "clippings"
---
[![头像](https://static.nowcoder.com/head/header0006.png?x-oss-process=image%2Fresize%2Cw_72%2Ch_72%2Cm_mfit)](https://www.nowcoder.com/users/337341244)

[尽管沉淀](https://www.nowcoder.com/users/337341244)

05-04 18:07 已编辑 门头沟学院 Java 发布于广东

## 1\. 怎么把 RAG 接入到你的 Agent 执行链路里的？是在调用工具之前先检索，还是在生成结果之后再做校验？

我们的 RAG 是通过 Spring AI 的 Advisor 接入到 ChatClient 的：模型调用前在 before() 做向量检索，把命中文档拼成上下文注入 prompt；模型生成后在 after() 把命中文档回写到响应 metadata 便于追踪。当前是“先检索再生成”的 pre-RAG，并没有单独做生成后的事实校验，后续可以增加 verifier 节点来做 post-check。

## 2\. MCP 是完全自研的吗？和市面上成熟的 Agent 框架相比，你的方案在扩展性上有什么优势？

## 结论

- **MCP 不是完全自研协议** ：我用的是 **MCP 标准协议** ，在 Java 侧主要做的是 **工程化落地与平台化接入** ——把 MCP Server（SSE/StdIO 两种传输）动态装配进 Spring 容器，并注入到 `ChatClient` 的 tool-calling 链路里。
- **自研的部分** 在于：围绕 MCP 做了\*\*配置化编排、动态装配、可观测执行链路（SSE 轨迹）\*\*这一整套“企业集成层”，而不是重新造一个协议。

## 1) 你的 MCP 到底自研了什么、复用了什么？

### 复用（标准/成熟组件）

- **协议层** ：遵循 MCP 的工具描述与调用模型（Tool schema、call/response）。
- **传输层** ：支持 **SSE** （远程工具服务）和 **StdIO** （本地工具进程）。
- **调用链** ：通过 Spring AI 的 tool-calling 支持，把 MCP client 聚合成 tool callback provider（你代码里是 `SyncMcpToolCallbackProvider` 一类的注入方式）。

### 自研（平台落地能力）

- **动态装配** ：从 DB/配置加载 MCP 工具配置，在运行时创建 MCP client，并注册成 Spring Bean（按工具 id 生成 beanName）。
- **按 Agent/Client 绑定工具集** ：不同 Client 可组合不同 MCP 工具集合，避免“全局一锅端”。
- **装配到执行引擎** ：最终构建 `ChatClient` 时将工具回调注入，使 AutoAgent/FlowAgent 的不同 step 可以复用同一套工具能力。
- **可观测** ：工具调用结果可以写入执行轨迹，并通过 SSE 推给前端（便于排障与解释）。

## 2) 和成熟 Agent 框架相比，你的扩展性优势是什么？

我会从三点讲“扩展性”，每点都能落到工程事实：

### 优势 A：在 Spring 体系下的“配置化扩展 + 动态装配”

- 成熟框架（LangChain/LangGraph）在 Python 生态扩展快，但在 Java 企业环境里经常要重新做：
- 我这套做法把工具、模型、RAG、Prompt 都当成 **配置驱动的模块** ，通过规则树按依赖顺序装配成 Bean。 **扩展一个新 MCP Server** 更多是“加配置 + 注册”，不需要侵入执行链路代码。

### 优势 B：工具接入与 Agent 执行解耦（横切能力复用）

- RAG 用 Advisor 这种横切组件接入；MCP 工具也以“可插拔能力”注入 `ChatClient` 。
- 结果是：同一个工具集可以被多个 Agent/多个 Step 复用；新增工具不会导致执行链路膨胀。

### 优势 C：工程化能力更贴近生产（可观测/可控/可治理）

成熟框架很多 demo 很快，但生产要补的能力很多。我的扩展点围绕“生产化”：

- SSE 全链路事件输出（分析/执行/总结）可观测
- 规则树装配顺序保证一致性
- 支持按 Client 绑定工具白名单（易做权限治理）
- 易插入限流、重试、超时、审计等治理层

MCP 协议本身不是我自研的，我遵循 MCP 标准，在 Java 侧主要做工程化落地：支持 SSE/StdIO 两种传输，把 MCP 客户端按配置动态创建并注册到 Spring 容器，最终注入到 ChatClient 的 tool-calling 链路里。相比市面成熟 Agent 框架，我的优势不在于算法，而在于 Java/Spring 生态下的扩展性：工具、模型、RAG、Prompt 都配置化装配，新增工具低侵入；同时具备更强的生产化集成空间——权限白名单、审计、限流和 SSE 可观测链路都能统一治理。

[#发面经攒人品#](https://www.nowcoder.com/creation/subject/c121d108a0d94e04b8821f14c3acadd5)

评论 5 20 真题解析

浏览 444

大家都在搜：agent项目

一键发评

先检索再生成

mark配置化扩展

SSE可观测强

快捷表情

畅所欲言吧～

图片

话题

[程序员花海](https://www.nowcoder.com/users/664521299)

05-05 19:04

[复旦大学 Java](https://www.nowcoder.com/users/664521299)

![](https://uploadfiles.nowcoder.com/files/20240514/510894044_1715657211013/tiezhi.png)

[我在大厂做 AI Agent 真实日常：和自学版完全两回事!](https://www.nowcoder.com/discuss/881252877627383808?sourceSSR=post)[大家好，我是@程序员花海。最近后台好多同学问我，跟着网上教程把 Agent Demo 跑通了，RAG、工具调用、多轮对话都能实现，为什么一投简历没回音，面试稍微深挖一点就接不住？今天不聊通用学习路线，也不堆... 查看更多](https://www.nowcoder.com/discuss/881252877627383808?sourceSSR=post)[聊聊我眼中的AI](https://www.nowcoder.com/creation/subject/f8880e4ec9b74009bdf83deeaf85e0b3?entranceType_var=%E5%86%85%E5%AE%B9%E6%9D%A1%E7%9B%AE)

[上岸绝不摆烂](https://www.nowcoder.com/users/808766168)

05-04 08:15

[门头沟学院 Java](https://www.nowcoder.com/users/808766168)

[宇树科技实习AI agent开发一面分享](https://www.nowcoder.com/feed/main/detail/641237a9125a4e488e7731cb300e8d2d?sourceSSR=post)[发一下问题给大家参考，攒攒人品！  
1.请介绍一个你参与的、与AIAgent相关的项目，并说明你的角色和贡献。... 查看更多](https://www.nowcoder.com/feed/main/detail/641237a9125a4e488e7731cb300e8d2d?sourceSSR=post)查看10道真题和解析

[杰尼龟要实习](https://www.nowcoder.com/users/216541357)

05-02 13:53

已编辑

[吉首大学 后端工程师](https://www.nowcoder.com/users/216541357)

[27届双非Agent开发春招实习选择](https://www.nowcoder.com/discuss/879349992144568320?sourceSSR=post)

投票[本人27届双非，去年开始从Java开发转Agent开发，目前有一段三个月金蝶的Agent开发实习经历，4月份离职，准备春招。原本目标是找一份有名气的中厂Agent开发实习，但是官网上投的所有中大厂基本上都是简历挂了... 查看更多](https://www.nowcoder.com/discuss/879349992144568320?sourceSSR=post)去向（单选）

45人参与投票

[双非应该如何逆袭？](https://www.nowcoder.com/creation/subject/b9a4a1c6569f4a4cbcf04909adc0c06f?entranceType_var=%E5%86%85%E5%AE%B9%E6%9D%A1%E7%9B%AE)

04-17 17:35

[门头沟学院 算法工程师](https://www.nowcoder.com/users/103181999)

[阿里淘天 日常实习agent开发一面分享](https://www.nowcoder.com/feed/main/detail/aec43d1ad5064a4892e80be8ba7e277a?sourceSSR=post)[发一下问题给大家参考，攒攒人品！  
1.介绍RAG技术... 查看更多](https://www.nowcoder.com/feed/main/detail/aec43d1ad5064a4892e80be8ba7e277a?sourceSSR=post)查看14道真题和解析

[Hcoco](https://www.nowcoder.com/users/984902720)

05-05 16:10

[华为\_系统工程师](https://www.nowcoder.com/users/984902720)

![](https://uploadfiles.nowcoder.com/files/20240514/510894044_1715657211013/tiezhi.png)

[【面试真题】美团Agent 方向面经整理（思路引导 + 推荐回答）](https://www.nowcoder.com/discuss/881209147377664000?sourceSSR=post)[Agent / LLM 方向面经整理（思路引导 + 推荐回答） 每章开头有一小段本章思路引导（这类题整体上在考什么、怎么组织话）。每道题下先有一行思路（答题时先想什么），再是推荐回答（可参考的表述骨架）。请把里面... 查看更多](https://www.nowcoder.com/discuss/881209147377664000?sourceSSR=post)评论

5

20

真题解析