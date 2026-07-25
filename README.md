# cache-keepalive

Claude Code 与 Codex 的提示缓存自动保温方法。它解决的是：长会话暂时没人输入时，
怎样用有上限、可验真的短请求刷新“正在使用的同一个会话前缀”，避免回来后冷重建。

> 保温请求仍是一次真实模型调用，会消耗套餐额度或 API 费用。不同产品链路的 TTL
> 不能互相外推；先做本机冷/热对照和链式实测，再冻结节拍。

## 当前建议

| 环境 | 建议 |
|---|---|
| Claude Code 订阅主会话 | 50 分钟，默认最多 2 拍；正式启用前验证 2×50 分钟连续命中 |
| Claude API / 子代理等 5 分钟环境 | 270 秒起步，逐拍验证 usage |
| OpenAI API | 优先使用官方 `prompt_cache_options.ttl` |
| ChatGPT 订阅下 Codex CLI | 不套用 API TTL，按同 thread 本机实测决定 |

作者环境在 2026-07-25 的 Codex 受控结果：两个独立约 45k input 链在冷重建后
+6、+8、+10 分钟均命中，15 和 25 分钟未命中会话长前缀，因此选择 450 秒、
最多 8 拍。样本量是每档 2，不代表未来版本或其他账户的固定 TTL。
同版本同 thread 的 60 秒对照还显示 `low→medium` 会使 44,800 cached 降为 0，
恢复 `low` 后重新命中；自动 resume 因此必须继承原 turn 的 model+effort。
一次性 `codex exec` worker 不应保温；实现应依据 rollout 的 `source=exec`
（或 `originator=codex_exec`）排除，并允许自动化用 `CODEX_KEEPALIVE_SKIP=1` 显式跳过。
作者实现先以 `codex-keepalive-ctl mode verify --interval 450 --cap 8` 取证，至少
一拍真实命中后才切生产；CLI 版本变化还要复查 resume 后原 thread 的 session source
仍是 `cli`。

完整的实验判据、自动状态机、安全边界与验收清单见 [SKILL.md](SKILL.md)。

## 设计要点

- 程序在真实回合结束后默认登记；模型不负责决定是否启动。
- 保温轮只允许返回 `KEEPALIVE_CONTINUE` 或 `KEEPALIVE_STOP`，任何异常停链。
- 合成轮在 hook 层拒绝工具，不能只靠提示词约束。
- 用户真实输入永远优先；缓存优化不得阻断正常工作。
- 设静默期、thread 生命周期、每日请求和全局并发多层上限。
- 版本变化、休眠过期、限流、配额或 cache-read 不足时自动熔断。
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
- [agent-orchestration](https://github.com/ruodou233/agent-orchestration)：长任务/过夜流程编排，Agent 自主跑、自主省 token，不用你盯
- [cross-review](https://github.com/ruodou233/cross-review)：跨模型双审，让 AI 自己把活干完整，不用你擦屁股
- [upgrade-audit](https://github.com/ruodou233/upgrade-audit)：AI 每天自主升级知识体系，教一遍就会，不用反复纠正

完整目录见 [GitHub 主页](https://github.com/ruodou233)。
