# 提示缓存保温｜Prompt Caching & Keepalive

**缓存命中可节省 90% 的重复输入费用。如果你五分钟内看不完 AI 的报告，可以考虑缓存保温。**

这里的 90% 指 Claude API 常见的缓存读取折扣：同样一段输入，命中缓存后只收普通输入价格的十分之一，并非 token 数量减少 90%，也不代表订阅额度按此比例节省。五分钟是常见的缓存有效期，具体取决于模型和使用方式，部分环境可保留一小时或更久。[价格与有效期说明](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)

## 为什么需要保温

你每次给 AI 发消息，它通常都要重新处理前面的对话、文档和工具结果。**缓存让它复用已经处理过的内容，减少重复输入的费用。**聊得越长，这部分开销就越值得关注。

但缓存会过期。比如 AI 写完一份长报告，你读了十分钟才追问；如果缓存已经过期，下一轮就要重新处理旧内容。

缓存保温的做法是：**在你阅读或暂时离开时，由程序向同一会话发送简短请求，命中缓存并延长它的有效期。**不用你守着计时器发“在吗”。

## 这个 skill 帮你做什么

让 Claude Code、Codex 等 Agent 读取这份 skill 后，按你的实际环境：

1. **检查缓存是否命中**，以及间隔多久后会失效。
2. **算清保温是否划算**，把保温请求本身的消耗也算进去。
3. **值得做再配置自动保温**，设置间隔和次数上限；你回来继续对话时，正常工作优先。

**保温本身也花钱，缓存命中不等于最终省钱。**如果你回复很快、会话很短，或原本的缓存就能覆盖阅读时间，通常不需要保温。作者在自己的环境里测算后，Claude 侧有收益，Codex 侧反而亏，因此关闭了 Codex 保温。这不是所有账户和版本的通用结论。

## 怎么用

在你使用的 Agent 对应目录安装：

**Claude Code：**

```bash
git clone https://github.com/ruodou233/claude-cache-keepalive.git \
  ~/.claude/skills/cache-keepalive
```

**Codex：**

```bash
git clone https://github.com/ruodou233/claude-cache-keepalive.git \
  ~/.agents/skills/cache-keepalive
```

然后对 AI 说：

> 用 cache-keepalive 检查我的会话缓存。我经常花十几分钟读报告再回复，帮我测测保温划不划算，值得的话就配置好。

**这个仓库提供的是给 Agent 使用的操作规范，安装后不会自动启动保温服务。**Agent 需要先检查你的 CLI 版本、账户和缓存表现，再生成适配本机的配置。

已有保温方案却感觉更费了，也可以直接说：“用 cache-keepalive 算算最近的保温到底省了多少。”

## 想看技术细节

[SKILL.md](SKILL.md) 包含各环境的实测方法、调度间隔、命中验证、自动停止条件与验收清单；[local-config.example.md](local-config.example.md) 提供本机配置记录示例。文中的实验参数是参考，使用前应在自己的环境验证。

保温请求会进入会话历史，因此应限制次数。遇到缓存未命中、限流或休眠过期等情况，程序应停止继续请求。

## 官方资料

- [Claude API 缓存价格与有效期](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Claude Code prompt caching](https://code.claude.com/docs/en/prompt-caching)
- [OpenAI prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching)

## 反馈

欢迎在 [GitHub](https://github.com/ruodou233/claude-cache-keepalive) 提 issue 或 PR。附上使用的 Agent、版本和实际消耗记录，会更容易判断问题。

## 相关 Skill 推荐

<!-- 本表由维护脚本生成，勿手工编辑 -->
- [agent-orchestration](https://github.com/ruodou233/agent-orchestration)：复杂任务跑到半夜，你不可能一直盯着。让 Agent 分工跑长任务和批量工作，你只管第二天早上收结果。<br>Coordinate AI agents for long-running tasks, parallel work, and overnight workflows.
- [cross-review](https://github.com/ruodou233/cross-review)：AI 的活总差一点，总要你擦屁股，总打丑补丁？让另一家 AI 挑刺复查，自己把活干完整，不用你一直兜底。<br>An agent skill for independent code and design reviews across AI providers, checking correctness, complexity, and better approaches.
- [upgrade-audit](https://github.com/ruodou233/upgrade-audit)：把你教过 AI 的东西留下来：沉淀偏好、复盘踩坑、更新 skill 和工作流程<br>Review conversation history, agent memory, and skills to identify reusable lessons and propose updates to outdated instructions.

完整目录见 [GitHub 主页](https://github.com/ruodou233)。
