# 提示缓存保温｜Prompt Caching & Keepalive

缓存命中可以节省 90% token。如果你五分钟内看不完 AI 的报告，可以考虑缓存保温。

> 缓存保温值不值：先查 TTL、再算账、最后挑现成工具

**缓存保温并非一律需要。**如果通常能在缓存有效期内继续对话，或者续温的消耗高于避免冷重建的收益，就不必启用。作者环境里，Claude 侧按公式回放，账面上能省两成；Codex 侧算下来反而亏，最后关掉了。

## 三步

**① 查自己的 TTL**（time to live，缓存存活时间）

```bash
claude -p "hello" --output-format json
```

看 `usage.cache_creation`：写入记在 `ephemeral_1h_input_tokens` 就是 1 小时档，记在 `ephemeral_5m_input_tokens` 就是 5 分钟档。订阅在套餐额度内，主会话默认 1 小时；超额改用 usage credits 后自动降到 5 分钟；API key、云平台和网关默认 5 分钟。

**② 能用 1 小时就先改 TTL，不急着保温**

在 Claude Code 设置里加 `"promptCacheTtl": "1h"`（或环境变量 `CLAUDE_CODE_PROMPT_CACHE_TTL=1h`，v2.1.242+），改完重跑 ①。经 `ANTHROPIC_BASE_URL` 走网关时，网关要原样转发 `anthropic-beta` 请求头；设了 `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1` 的话，这个头里的 1 小时标记会被剥掉，一定要实测。1 小时写入是 2 倍价（5 分钟是 1.25 倍），很少空闲超过 5 分钟的人换了反而多花钱。

**③ 还要保温，挑一个现成 mod**（Claude Code v2.1.287+）

- [cache-tax](https://github.com/karanb192/cache-tax)：适合想在冷发送前被拦一下的人。`/keepwarm 90m` 开一个有边界的窗口，发消息前显示这次要重写多少钱。
- [cache-refresher](https://github.com/olddonkey/cache-refresher)：适合想先看账再决定的人。状态栏倒计时，面板并排列出过期一次和续温一次各多少钱，自动续温默认关。
- [cache-warmer](https://github.com/paulbkim-dev/claude-code-cache-warmer)：适合想装上就不管的人。直接用 Pi 的续温公式，5 分钟和 1 小时两档都支持。

三个都用 mods 的 `$.model.fork` 续温：重放最后一次请求，模型调不了工具，续温轮也不进会话历史。没有 mods 的旧版本可以看 [cachebeat](https://github.com/ARahim3/cachebeat)。

## 什么时候用得上

- **长会话中途离开**："我经常聊到一半去忙别的，帮我看看这段空档值得保温吗？"结合上下文长度、离开多久和真实消耗来算。
- **给常用 Agent 配缓存**："这台电脑上的 Claude Code 怎么配比较合适？"先查 TTL，再用公式比较改 TTL、装保温工具和什么都不做，不用自己定时发一句"在吗"。
- **查缓存为什么没命中**："刚才还命中，换了模型或思考档位就没了，查一下。"沿着同一会话的 usage 找变化。
- **核算已有方案**："保温已经跑了一周，算算究竟省了还是多花了。"把续温本身的调用也算进去，再决定继续、调节拍还是关闭。

## 先判断你该不该保温

**不要直接抄别人的节拍。**按顺序问自己：

1. 默认缓存寿命是不是已经覆盖了你的静默窗口？覆盖了就不需要保温；没覆盖，先看能不能把寿命改长。
2. 算下来划不划算？usage 只能证明**命中**，不能证明**省钱**。
3. 你的运行形态有没有可归因的合成请求入口？没有就到此为止。
4. 你能不能读到 usage 验证命中？读不到就别上自动链：无法验证命中就无法熔断，而每一拍都是真实计费请求。

第 2 问用 [Pi](https://github.com/earendil-works/pi) 的续温公式回答，取代以前的"55 分钟"经验节拍：

```text
续聊概率 × 重建代价 − 续温代价 ≥ 阈值   → 续温，否则停
```

续聊概率是"已经空闲到这一拍的前提下，下一个缓存寿命内会回来"的比例，最该换成你自己的数据。它随空闲时长下降得很快，所以公式会自己停；触发点也就自然贴着**命中率开始衰减的那一点**，而不是贴着 TTL 边界。

1 小时 TTL 还要注意空闲压缩（idle compaction）：上下文 20 万 token 以上的会话，空闲约 54 分钟、缓存过期前会被自动压缩成摘要，正好撞上"55 分钟保温"。保温就设 `"idleCompaction": false`，或只对 20 万以下的会话保温。

各家缓存寿命差异极大：从数分钟到数小时甚至数天，部分链路（如 DeepSeek 磁盘缓存）按闲置状态自动清理、没有固定 TTL。**同一套方法在不同链路上可能给出相反结论。**

## 作者环境的算例

2026-11-16 在作者本机测的，换一台机器、换一个账户都可能不同，只当算例看。

**环境**：Claude Code 2.1.325，主模型 Opus 5.5，按 API key 方式经本地网关转发到第三方中转站。桌面版启动的会话进程里自带 `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1`；终端是 2026-11-02 把命令行接到同一个网关时加的，原因是当时的上游（底层走云平台）拒收预发布 beta 头，报 400：`Unexpected value(s) … for the anthropic-beta header`。11-16 复测，把它设回 0 也能正常回复，现在的上游已经不拒收了。

**TTL 自检**：

| 条件 | `ephemeral_5m` | `ephemeral_1h` | 结论 |
|---|---|---|---|
| 默认 | 3,972 | 0 | API key 经网关，默认就是 5 分钟 |
| 加 `CLAUDE_CODE_PROMPT_CACHE_TTL=1h` | 0 | 29,036 | 1 小时写入成功 |
| 8.5 分钟后再发一次（同样设 1h） | 0 | 0 | 读取 29,036，没有重写，确实过了 5 分钟仍命中 |

所以，作者主会话近一个月 97% 的缓存写入落在 5 分钟档，原因是"没设 `promptCacheTtl`"，不是 beta 头被剥掉。本地抓包看，设了 `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1` 后 `extended-cache-ttl` 头确实不发了，但请求体里还带着 `ttl: "1h"`，这条中转链路照样认。换一条链路，结论可能相反。

**回放**（2026-10-15 到 11-15，主会话 7,136 次请求，按 Opus 5.5 官方价：基础输入 4 美元/百万 token，5 分钟写入 1.25 倍、1 小时写入 2 倍、读取 0.05 倍；只把"回来时确实冷了"的请求算作重建）：

| 方案 | 缓存写入加续温费用 | 相对现状 |
|---|---|---|
| 5 分钟，不保温（现状） | 约 1,170 美元 | — |
| 只改 1 小时 TTL | 约 1,131 美元 | −3% |
| 1 小时 TTL + 公式保温 | 约 1,034 美元 | −12% |
| 5 分钟 TTL + 公式保温 | 约 920 美元 | −21% |

本机实测的续聊概率：5 分钟档在空闲 4.5、9.5、14.5 分钟时分别是 0.28、0.21、0.14；1 小时档在 54 分钟时是 0.15，和 Pi 的 0.15 很接近。代入公式，5 分钟档第一拍要前缀 4.3 万 token 以上才值得续温，第三拍要 11 万以上；1 小时档第一拍要 5.3 万以上。

两点要说清：一是只改 1 小时 TTL 几乎打平，因为作者 90% 的空档都在 5 分钟以内，多付的写入价把省下的重建吃掉了；二是表里的美元按官方价算，中转站和订阅额度怎么折算 1 小时写入、续温读取，作者并不知道。所以作者目前没有开保温，也没有改 TTL。真要开，按公式看 5 分钟档保温账面最好；cache-warmer 的 5 分钟档用的就是这个公式（续聊概率固定 0.15），要开就先拿它按 SKILL.md 第五节实测一遍。

**一个真实的反例**：作者环境的 Codex 侧按真实记录测算（2026-08），长静默事件只占 0.24%（Claude 侧为 2.0%），前缀中位数只有 Claude 的 39%，缓存写入倍率也更低。全链重放显示回本需救回约 168 次，而覆盖窗口内可救的未命中总共只有 108 次，**即使每次都救成功仍然亏损**，因此整体关闭。

完整的检查步骤、实测对照方法和核算方法见 [SKILL.md](SKILL.md)。

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

这个仓只提供判断和验收方法，不带代码。让 Agent 读取 `SKILL.md` 后先查 TTL、按公式算账，再决定改 TTL、装工具还是什么都不做。

## 致谢与资料

- 续温决策公式来自 [Pi](https://github.com/earendil-works/pi) 的 `packages/ai/src/cache-warming.ts`（Mario Zechner），本仓只引用思路，没有复制代码。
- 推荐工具：[cache-tax](https://github.com/karanb192/cache-tax)、[cache-refresher](https://github.com/olddonkey/cache-refresher)、[cache-warmer](https://github.com/paulbkim-dev/claude-code-cache-warmer)；旧版本参考 [cachebeat](https://github.com/ARahim3/cachebeat)。
- [Claude Code prompt caching](https://code.claude.com/docs/en/prompt-caching)、[Claude Code mods API](https://code.claude.com/docs/en/plugins/mods/api)
- [OpenAI prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching)

## Changelog

| 时间 | 变更 |
|---|---|
| 2026-11-16 | 改为决策指南：先查 TTL、再按 Pi 的续温公式算账、最后挑现成 mod；删去自造的 asyncRewake 保温状态机；补上 1 小时 TTL 设置、订阅超额降档、网关 beta 头、空闲压缩；附作者环境 TTL 自检与回放 |
| 2026-07-25 | 改为 Claude Code + Codex 双侧自动保温；加入链式实测、程序调度、工具拒绝与多层上限 |
| 2026-07 | 首次开源发布 |

## 反馈

欢迎在 [GitHub](https://github.com/ruodou233/claude-cache-keepalive) 提 issue 或 PR，
尤其欢迎附 CLI 版本、账户链路、TTL 自检结果、冷/热对照和回放核算的可复现实测。

## 相关 Skill 推荐

<!-- 本表由维护脚本生成，勿手工编辑 -->
- [agent-orchestration](https://github.com/ruodou233/agent-orchestration)：复杂任务跑到半夜，你不可能一直盯着。让 Agent 分工跑长任务和批量工作，你只管第二天早上收结果。<br>Coordinate AI agents for long-running tasks, parallel work, and overnight workflows.
- [cross-review](https://github.com/ruodou233/cross-review)：AI 的活总差一点，总要你擦屁股，总打丑补丁？让另一家 AI 挑刺复查，自己把活干完整，不用你一直兜底。<br>An agent skill for independent code and design reviews across AI providers, checking correctness, complexity, and better approaches.
- [upgrade-audit](https://github.com/ruodou233/upgrade-audit)：把你教过 AI 的东西留下来：沉淀偏好、复盘踩坑、更新 skill 和工作流程<br>Review conversation history, agent memory, and skills to identify reusable lessons and propose updates to outdated instructions.

完整目录见 [GitHub 主页](https://github.com/ruodou233)。
