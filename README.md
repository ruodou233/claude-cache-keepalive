# cache-keepalive

> 跨 Agent / 跨厂商的缓存保温方法

长会话聊到一半，去吃顿饭、开个会，回来又要花一笔 token 重读上下文。这个 skill 帮你判断缓存该不该保温、隔多久续一次、最多续几次：先用真实会话测命中、算收益，值得做再让程序自动续温。作者环境里，Claude 侧收益显著为正，Codex 侧算下来反而亏，最后关掉了——这两种结果都很有用。

## 什么时候用得上

- **长会话中途离开**：“我经常聊到一半去忙别的，帮我看看这段空档值得保温吗？”结合上下文长度、离开多久和真实消耗来算。
- **给常用 Agent 配自动续温**：“这台电脑上的 Claude Code 怎么配比较合适？”先测冷、热对照，再确定间隔和拍数，不用自己定时发一句“在吗”。
- **查缓存为什么没命中**：“刚才还命中，换了模型或思考档位就没了，查一下。”沿着同一会话的 usage 找变化。
- **核算已有方案**：“保温已经跑了一周，算算究竟省了还是多花了。”把续温本身的调用也算进去，再决定继续、调节拍还是关闭。

## 先判断你该不该保温

**不要直接抄下面的节拍。**按顺序问自己：

1. 你的运行形态有没有可归因的合成请求入口？没有就到此为止。
2. 默认缓存寿命是不是已经覆盖了你的静默窗口？覆盖了就不需要保温。
3. 你能不能读到 usage 验证命中？读不到就别上自动链——无法验证命中就无法熔断，
   而每一拍都是真实计费请求。
4. 算下来划不划算？usage 只能证明**命中**，不能证明**省钱**。

触发点也不是越晚越好，它该贴着**命中率开始衰减的那一点**，而不是贴着 TTL 边界。

各家缓存寿命差异极大：从数分钟到数小时甚至数天，部分链路（如 DeepSeek 磁盘缓存）
按闲置状态自动清理、没有固定 TTL。**同一套方法在不同链路上可能给出相反结论**——
作者环境按真实记录测算后，Claude 侧收益显著为正，Codex 侧却是净负而整体关闭。

## 当前建议

| 环境 | 建议 |
|---|---|
| Claude Code 订阅主会话 | 55 分钟，默认最多 2 拍；正式启用前验证 2 次连续命中 |
| Claude API / 子代理等 5 分钟环境 | 270 秒起步，逐拍验证 usage |
| OpenAI API | 优先使用官方 `prompt_cache_options.ttl` |
| ChatGPT 订阅下 Codex CLI | 不套用 API TTL；作者环境按真实记录测算后收益为负，已关闭 |

作者环境在 2026-07-25 的 Codex 受控结果：两个独立约 45k input 链在冷重建后
+6、+8、+10 分钟均命中，15 和 25 分钟未命中会话长前缀，因此选择 450 秒、
最多 8 拍。样本量是每档 2，不代表未来版本或其他账户的固定 TTL。
同版本同 thread 的 60 秒对照还显示 `low→medium` 会使 44,800 cached 降为 0，
恢复 `low` 后重新命中；自动 resume 因此必须继承原 turn 的 model+effort。
一次性 `codex exec` worker 不应保温；实现应依据 rollout 的 `source=exec`
（或 `originator=codex_exec`）排除，并允许自动化用 `CODEX_KEEPALIVE_SKIP=1` 显式跳过。
作者实现先以 `codex-keepalive-ctl mode verify --interval 450 --cap 8` 取证，至少
一拍真实命中后才切生产；若真实 resume 把原 thread 改写成 worker 身份则停用。

完整的实验判据、自动状态机、安全边界与验收清单见 [SKILL.md](SKILL.md)。

## 设计要点

- 程序在真实回合结束后默认登记；模型不负责决定是否启动。
- 保温轮只允许返回 `KEEPALIVE_CONTINUE` 或 `KEEPALIVE_STOP`，任何异常停链。
- 合成轮在 hook 层拒绝工具，不能只靠提示词约束。
- 用户真实输入永远优先；缓存优化不得阻断正常工作。
- 设静默期、thread 生命周期、每日请求和全局并发多层上限。
- 休眠过期、限流、配额或 cache-read 不足时自动熔断。
- 最近一轮 input 少于 20k 时跳过，避免短会话无收益调用。

每一组合成提示和回复都会永久进入会话历史；当前 CLI 没有通用、安全的删除办法。
固定短文本、自我限定措辞和生命周期上限只能控制污染。

## 安装 skill

Claude Code：

```bash
git clone https://github.com/ruodou233/claude-cache-keepalive.git \
  ~/.claude/skills/cache-keepalive
```

Codex：

```bash
git clone https://github.com/ruodou233/claude-cache-keepalive.git \
  ~/.agents/skills/cache-keepalive
```

这个公开仓提供设计与验收规范，不假定你的 CLI 版本、账户链路或本机 hook 配置。
让 Agent 读取 `SKILL.md` 后先盘点环境、做受控实测，再生成适配本机的安装方案。

## 官方资料

- [Claude Code prompt caching](https://code.claude.com/docs/en/prompt-caching)
- [OpenAI prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching)

## Changelog

| 时间 | 变更 |
|---|---|
| 2026-07-25 | 改为 Claude Code + Codex 双侧自动保温；加入链式实测、程序调度、工具拒绝与多层上限 |
| 2026-07 | 首次开源发布 |

## 反馈

欢迎在 [GitHub](https://github.com/ruodou233/claude-cache-keepalive) 提 issue 或 PR，
尤其欢迎附 CLI 版本、账户链路、冷/热对照和链式 usage 的可复现实测。

## 相关 Skill 推荐

<!-- 本表由维护脚本生成，勿手工编辑 -->
- [agent-orchestration](https://github.com/ruodou233/agent-orchestration)：复杂任务跑到半夜，你不可能一直盯着。让 Agent 分工跑长任务和批量工作，你只管第二天早上收结果。
- [cross-review](https://github.com/ruodou233/cross-review)：AI 的活总差一点，总要你擦屁股，总打丑补丁？让另一家 AI 挑刺复查，自己把活干完整，不用你一直兜底。
- [upgrade-audit](https://github.com/ruodou233/upgrade-audit)：把你教过 AI 的东西留下来：沉淀偏好、复盘踩坑、更新 skill 和工作流程

完整目录见 [GitHub 主页](https://github.com/ruodou233)。
