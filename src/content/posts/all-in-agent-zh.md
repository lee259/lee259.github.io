---
title: "All in Agent：把工作流程集成到 Agent 里"
description: "在 Agent 时代，各种操作的默认入口不再是平台 UI，而是 Agent 本身。"
date: 2026-08-09
tags: ["AI Agent", "开发者工具", "工作流"]
---

> 在 Agent 时代，各种操作的默认入口不再是平台 UI，而是 Agent 本身。

---

## 问题：Agent 在 IDE 里很强，出了 IDE 就失能

Agent 能写代码，但走不完一个完整的开发流程。

修一个 Bug 的场景：Agent 在 IDE 里定位问题、改代码，这一步很快。然后呢？手动打开 Git 托管平台创建 MR，手动打开 CI/CD 平台触发构建，手动打开管理后台检查配置，手动打开发布平台发布。Agent 在"写代码"这一步很强，出了 IDE 就失能了。

但这个问题不只是开发者的。

运营同学配置一个活动，需要在活动管理平台创建模板、在设计稿平台找素材、在发布平台配置上线、在监控平台看数据。每个平台都要打开、登录、找到对应的页面、点几个按钮。繁琐，重复，容易出错。

产品经理也一样：在需求管理平台写需求、在设计评审平台看稿子、在数据分析平台查指标、在项目管理平台跟进度。信息散落在各个平台里，每次切换都是一次上下文损失。

这不是 Agent 的问题，是系统设计的问题。这些平台在设计时只考虑了人类用户，没人想过"如果 Agent 要调用我，我该怎么暴露接口"。

![前后对比示意](/images/all-in-agent-before-after.png)

---

## "All in Agent" 的两层含义

第一，**把平台操作封装成 Agent 可调用的工具**。不是给 Agent 一个浏览器让它自己点，而是给它一个统一的接口，让它能直接操作。

第二，**把工作流程编排成 Agent 可执行的工作流**。不是让 Agent 自由发挥，而是定义好步骤和门禁，让它按流程执行。

做到这两点之后，不管什么角色，工作方式都变成：**我告诉 Agent 目标，Agent 从开始到完成，我在关键节点做判断和验收。**

---

## 把平台操作封装成工具

### CLI：最直接的 Agent 接口

把平台操作封装成工具，最直接的方式是 CLI。

CLI 有天然适合 Agent 的特性：结构化输出（JSON），Agent 可以直接解析结果；幂等操作，Agent 可以放心重试；管道组合，适合编排；无 GUI 依赖，纯文本交互；可审计，每次操作都有日志。

但很多 CLI 是给人用的——输出是彩色表格，参数有交互式提示，错误信息藏在花哨的格式里。想让 Agent 能调，两件事就够了：**默认输出 JSON，提供 dry-run 模式**。

![三层架构示意](/images/all-in-agent-three-layers.png)

### 场景一：开发者的发版流程

传统做法：打开 Git 托管平台创建 MR → 打开 CI/CD 平台触发构建 → 打开发布平台选择版本发布。

封装成 CLI 之后：

```
$ release start --branch feat/fix-bug --dry-run
→ 输出：即将创建 MR，版本号 v1.2.3，变更清单 [src/a.ts, src/b.ts]
→ 人确认：yes

$ release start --branch feat/fix-bug
→ 自动创建 MR → 等待 CI 通过 → 触发构建 → 发布
```

### 场景二：运营的活动配置

传统做法：打开运营后台创建活动 → 在设计稿平台找素材 → 在发布平台配置上线 → 在监控平台看数据。

封装成 CLI 之后：

```
$ campaign create --template spring-festival --dry-run
→ 输出：即将创建活动，关联素材 [banner.png, prize-list.json]，投放渠道 [首页弹窗、消息推送]
→ 人确认：yes

$ campaign create --template spring-festival
→ 自动创建活动 → 绑定素材 → 配置投放 → 上线

$ campaign status --id 123
→ 输出：活动状态，曝光量，点击率
```

开发者不需要离开终端，运营不需要打开五个浏览器标签页。Agent 统一执行，人在关键节点确认。

---

## 把工作流程编排成可执行的工作流

CLI 解决了"Agent 能调平台"的问题，但还不够。Agent 还得知道什么时候调什么。

定义可执行的工作流，不是让 Agent 自由发挥：

```
1. 准备阶段 → 门禁：素材齐全、文案审核通过
2. 创建活动 → 门禁：配置不重复、预算不超限
3. 投放配置 → 门禁：目标用户正确、推送时间合理
4. 上线验证 → 门禁：预览正常、数据指标正确
```

每个步骤对应一个 CLI 命令，门禁是命令的校验逻辑。卡在哪一步、为什么卡住，一目了然。

### 人机协作分工

流程编排不等于自动化。关键节点需要人判断：

| 节点 | 人负责 | Agent 负责 |
| --- | --- | --- |
| 需求理解 | 确认目标 | 检索背景、拆解任务 |
| 执行 | 确认 dry-run 结果 | 执行操作流程 |
| 验证 | 确认结果符合预期 | 检查门禁规则 |
| 复盘 | 分析根因 | 沉淀记录到知识库 |

人从执行者变成决策者，精力从"做"转向"判断"。

![工作流闭环示意](/images/all-in-agent-workflow-loop.png)

---

## 三个设计原则

### 原则一：Agent 优先，人类兼容

设计系统时先问 Agent 怎么用，再问人怎么用。CLI 默认输出 JSON，同时提供 `--pretty` 给人看。操作默认 dry-run，同时提供 `--yes` 跳过确认。错误信息既要有机器可读的 error code，也要有人类可读的描述。

### 原则二：约束进工具，不进提示词

不要在 prompt 里写"不要删除线上的活动配置"——Agent 可能忽略。在 CLI 工具里直接禁止高危操作，或者要求二次确认。让一个不存在的工具拦住它，比写十句"禁止"都管用。（[Pi 的设计哲学](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/) 的核心洞察之一——约束写进工具 API，不写进 prompt。）

### 原则三：低频能力不常驻

Agent 的上下文窗口是稀缺资源。高频能力（代码生成、活动创建）默认可用，低频能力（发版、配置修改）按需加载，用的时候再读操作说明。

---

## 什么时候不适合

All in Agent 不是万能药。有些场景不适合：

- **小任务、一次性操作** — 一个 prompt 就能搞定的事，不需要搭系统
- **平台没有 API 或 CLI** — 没有程序化接口，Agent 集成不了
- **高风险操作** — 删库、改核心配置，保持人工通道
- **团队没有"写"的习惯** — 这套东西依赖人持续维护工作流和知识库

---

## 总结

All in Agent 不是"用 Agent 写代码"，而是把各种操作的入口从平台 UI 迁移到 Agent 本身。CLI 是实现这个目标的关键桥梁——它把平台操作变成 Agent 可调用的工具，把工作流程变成 Agent 可执行的工作流。

最终状态是：**人定目标、做判断，Agent 走流程、做执行。**

---

## 参考

- [Anthropic《Building effective agents》](https://www.anthropic.com/engineering/building-effective-agents) — 关于"从一个 Agent 起步、增量加复杂度"的论述，直接支撑了"先集成、再编排"的演进路径
- [Anthropic《Effective context engineering for AI agents》](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — context rot 概念，解释了为什么不能把全部能力塞进上下文
- [Pi 作者 Mario Zechner 博客](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/) — 轻 Harness 哲学：约束进工具不进 prompt
