# claude-cache-keepalive local-config 示例

> 这是字段骨架示例。数值为虚构值，请用你本机实测结果替换。复制成 `local-config.md`（已在 `.gitignore` 里）再填。

## 链路

- 产品与版本：Claude Code 2.1.xxx（终端 / 桌面版 / VS Code）
- 计费方式：订阅额度内 / 订阅超额（usage credits）/ API key / 云平台 / 自建网关
- 是否经网关：是（网关是否原样转发 `anthropic-beta`：是 / 否 / 未知）
- `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS`：未设 / 已设（原因：……）
- 主模型与 effort：<模型> / <effort>

## TTL 自检

- 命令：`claude -p "hello" --output-format json`
- 默认：`ephemeral_5m_input_tokens` = 4,000，`ephemeral_1h_input_tokens` = 0（虚构示例值）
- 设 `promptCacheTtl: "1h"` 后：`ephemeral_1h_input_tokens` = 30,000（虚构示例值）
- 跨过 5 分钟后再发：读取 30,000、写入 0 → 1 小时确实生效（虚构示例值）
- 实测日期：<YYYY-MM-DD>

## 空闲压缩

- `idleCompaction`：开 / 关
- `CLAUDE_CODE_IDLE_COMPACT_MIN_TOKENS`：默认 200000 / 自定义
- 和保温的关系：保温时关掉 / 只对阈值以下的会话保温 / 不保温

## 续温公式参数

- 价格（$/百万 token）：写入 5 分钟 <x>、写入 1 小时 <y>、读取 <z>、输出 <w>
- 续聊概率（按"已空闲到这一拍"为条件，用自己的记录统计）：
  - 第 1 拍（TTL 的 90%）：0.30（虚构示例值）
  - 第 2 拍：0.20（虚构示例值）
  - 第 3 拍：0.12（虚构示例值）
- 阈值：0.05 美元（沿用 Pi 默认）
- 由此算出的保温门槛：第 1 拍前缀 ≥ 约 4 万 token（虚构示例值）

## 回放核算

- 时间范围：<YYYY-MM-DD> 到 <YYYY-MM-DD>，主会话请求数 <n>
- 不保温：<金额>；只改 TTL：<金额>；改 TTL + 保温：<金额>
- 计价口径：官方价 / 中转站实际价 / 订阅额度（无法折算时写明）

## 决定

- 选择：什么都不做 / 只改 TTL / 改 TTL + 装 <工具名> / 5 分钟档 + 装 <工具名>
- 装工具后的验收：至少一次跨过 TTL 的真实命中（日期、读取量、写入量）
- 复查时间：<YYYY-MM-DD>（版本升级、换中转站或换计费方式后重测）
