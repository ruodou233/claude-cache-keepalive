# claude-cache-keepalive

Claude 缓存保温：实测 TTL、按环境设计保温节拍，控制反复冷读的成本。

> **使用前必须先读**
>
> 1. **本 skill 仅适用 Claude 侧。**OpenAI 侧缓存全自动、无显式写入费，通常没有保温需求；原文结论保留在 `SKILL.md`。
> 2. **TTL 有官方口径与实测静默波动史。**本文数值截至 2026-07，随平台策略可能变化，一切以你的本机实测为准。
> 3. **保温会消耗真实额度或费用。**只在明确等待场景按上限使用，不要默认后台常开，不得用于规避限额或制造无意义请求。

## 这是什么

Claude 的提示缓存 TTL 会随使用环境变化：订阅侧主会话、超限 credits、API 直连、子代理和强制覆盖配置，都可能对应不同的缓存行为。这个 skill 帮 Agent 先做本机 TTL 实测，再按实测结果设计等待期间的 keepalive 节拍。它只适用于明确等待场景，不是后台常驻机制，也不是绕过额度限制的工具。

核心思路是：先判断环境，再经使用者同意读取 usage 做自检，然后用 `实测 TTL * 0.8` 计算节拍，并且每拍都做命中校验。命中失败就停链，避免把保温变成连续冷读。

## 核心功能

- 区分订阅侧主会话、超限 credits、API 直连、子代理和强制覆盖环境。
- 把 TTL 实测自检作为首次使用第一步，而不是套用固定数值。
- 支持两类等待场景：超长后台等待、等使用者回复的改进循环。
- 提供每拍命中校验硬规则，TTL 变化或不可判定时立即停链。
- 提供 local-config 模板，把本机 TTL、节拍和拍数上限外置。

## 安装

Claude Code：

```bash
git clone https://github.com/ruodou233/claude-cache-keepalive.git ~/.claude/skills/claude-cache-keepalive
```

Codex：

```bash
git clone https://github.com/ruodou233/claude-cache-keepalive.git ~/.agents/skills/claude-cache-keepalive
```

其他支持 `SKILL.md` 的平台：把本仓库放入其 skills 目录即可。

## 使用示例

```text
帮我判断当前 Claude 会话的缓存 TTL，要不要布 keepalive。
```

预期行为：Agent 先说明需要读取哪些 usage / transcript 数据，征得同意后做 TTL 自检，再给出是否保温和节拍建议。

```text
这个后台任务可能要 2 小时，帮我按缓存 TTL 布保温拍。
```

预期行为：Agent 判断是否属于场景 A，按本机实测 TTL 计算拍距和上限，每拍校验命中。

```text
我可能一小时后回来，你可以继续做轻量确认，但不要改核心文件。
```

预期行为：Agent 判断是否属于场景 B，只安排低风险、可丢弃的小活；没有新事实或达到上限即停。

## 首次使用：环境自适应

首次使用时，让 Agent 只读判断当前 Claude 环境类型。读取本会话 transcript / usage 数据前，Agent 必须先说明读取范围和目的，并获得你的明确同意。之后根据实测 TTL 计算节拍与拍数上限，写入：

```text
~/.config/agentops-skills/claude-cache-keepalive/local-config.md
```

无法写该路径时，可退回 skill 目录内 `local-config.md`。格式参考 `local-config.example.md`。

## Changelog

| 时间 | 变更 |
|---|---|
| 2026-07 | 首次开源发布 |

## 反馈与作者

这个 skill 我长期维护。如果你有修改方案、发现问题、或者改出了更好的版本，欢迎通过以下任一渠道找到我：

- GitHub：本仓库提 issue 或 PR
- 小红书：错误乱码
- 微信公众号：能工智人错误乱码
- B站：若逗道人

## 相关 Skill 推荐

<!-- 本表由维护脚本生成，勿手工编辑 -->
- [agent-orchestration](https://github.com/ruodou233/agent-orchestration)：长任务治理：主代理指挥、子代理干活、状态落盘、断点续跑
- [cross-review](https://github.com/ruodou233/cross-review)：跨厂商双审：让另一家公司的最强模型独立审你的方案
- [upgrade-audit](https://github.com/ruodou233/upgrade-audit)：升级审计：让 Agent 定期把对话里的知识沉淀进文档体系

完整目录见 [GitHub 主页](https://github.com/ruodou233)。
