---
name: codex
description: 派发子任务给 codex CLI:tmux 窗格跑可视 TUI(agent-team 布局),notify 完成唤醒主会话,不在 tmux 自动降级 exec 黑盒。任何 codex 派发、resume 追问、并行批发,动手前先加载本 skill 拿调用细则;用户 /codex <任务> = 把该任务包成简报交 codex 执行。
argument-hint: <要交给 codex 的任务描述>
---

# codex 派发细则

脚本:本 skill 目录内 `scripts/codex-tmux`,随目录分发、零安装。**下文所有 `codex-tmux` 均指 `bash <本 skill 目录绝对路径>/scripts/codex-tmux`**(加载本 skill 时目录已知,调用时自行展开为绝对路径;使用者若已将其 symlink 进 PATH,直呼亦可,非必需)。

依赖:bash≥4.4、tmux≥3.2、codex CLI、GNU coreutils;jq/node/python3 任一用于提取终报(全缺时终报只留原始 JSON 指针,流程不受影响)。

## 标准调用(新任务)

简报落盘后,Bash `run_in_background`(退出自动唤醒主会话,不阻塞):

```
codex-tmux -t <任务名> -o <终报路径> -C <工作目录> --brief <简报文件> -- -c 'model_reasoning_effort="max"'
```

- 路径全绝对;不传 `-m`。bypass 沙箱、工作目录预信任(不弹信任提示)、(降级时)`--skip-git-repo-check` 均已内置,不重复传。
- 效果:tmux 分窗格跑真 codex TUI(镜像 agent-team 布局与同款比例:首个右切 70%——cx 侧占大头,后续取 cx 区中位窗格奇偶交替分割,>2 窗格时 main-vertical、主窗格收 30%;每窗格独立边框色,标题钉死 `cx:<任务名>`、完成变 ✅,codex 改不掉),进度可视、可直接在窗格里插手对话;首个 turn 完成走 codex 官方 notify 接口写 `-o` 终报(尾部附 session-id 与续聊命令)并退出唤醒,窗格保留可续聊。
- 不在 tmux 时自动降级为 `codex exec` 黑盒等价形态(`-o` 语义不变)。

## effort 纪律

- **每次调用(含 resume)必须显式传 `-c 'model_reasoning_effort="..."'`**,不依赖 config 默认——config 声称值与实际可长期背离(2026-07-12 实证事故:config 实为 medium 而文档称 max,历史任务全低档跑)。
- 默认 `"max"`;**简单机械任务(模板化改写、批量重命名、格式整理、照清单执行)显式降 `"medium"`/`"high"`**——不是思考越多越好,高 effort 更慢;判断类(设计、审查、debug、性能、并发)一律 max。
- 产出存疑时核实实际档位:`grep -m1 '"reasoning_effort"' ~/.codex/sessions/<日期>/rollout-*<session-id>*.jsonl`。模型走 `~/.codex/config.toml` 默认(`gpt-5.6-sol`)。

## 沙箱

- 本机默认 bypass(脚本内置 `--dangerously-bypass-approvals-and-sandbox`):本容器 bwrap 起不来,`-s workspace-write` 会让 codex 空手拒工(实证);环境本身已隔离。
- 他人环境不需要 YOLO 时:`CODEX_TMUX_BYPASS=0` 改走 codex 默认审批/沙箱——窗格本来就是交互的,审批弹窗直接在窗格里人工处理。

## 简报与并行

- 简报自包含:合同/判据/上下文文件/产物路径全进 prompt,产物落盘 + 终报。
- **分层终报契约(每份简报照抄进交付要求)**:终报双文件——`<名>.report.md` 摘要 **≤50 行**(必含:判决/交付物清单/各门禁一行退出码/**任何军规触碰、偏离、未达标、自作判断**——偏离只许写在摘要正文,不许用"见附录"替代)+ `<名>.evidence.md` 附录(全部原始证据:门禁日志、grep 输出、数字详表)。唤醒后**只读摘要**,附录按需 grep/定点 Read,永不全文读;lastmsg 转储仅报告缺失时兜底。
- codex 执行命令慢:拆独立自包含子任务,一条消息批发 N 个后台任务,先完者先处理;批内不写同一文件、不抢同一 GPU/核区,真串行才链式。每任务独立 `-t`/`-o`;窗格没空间时自动转独立 window(`cx:<任务名>`)。

## resume(小追问;大转向重开)

```
codex-tmux --resume <session-id|last> -t <任务名> -o <新终报> --brief <追问文件> -- -c 'model_reasoning_effort="max"'
```

- **不传 `-C`** 即沿用原会话 cwd:脚本会从 rollout 反查会话最后记录的 cwd 并显式传给 codex(不能真靠"继承"——codex 拿进程当前目录与会话记录比对,不一致会弹交互式目录选择器,无人值守窗格永久卡住,2026-07-18 实证;反查失败会打警告并退回旧行为,此时用 `tmux send-keys` 选 "Use session directory" 解卡)。session-id 直接抄上一份终报尾部 `[codex-tmux] session-id` 行(兜底:按 workdir grep `~/.codex/sessions/<日期>/rollout-*.jsonl`,文件名尾段 36 位即 id)。
- `-c` effort 不随 session 持久,每次要传。

## 窗格管理(镜像 agent-team 登记/清理逻辑)

- `codex-tmux list`:列出登记窗格——pane_id、任务名、状态(运行中/完成/死亡)、位置、终报路径;每次先按现存 pane 对账清死条目。
- `codex-tmux kill <任务名|%pane_id|done|all>`:定向/批量关闭并注销(`done`=只关已完成,`all`=全关;重名时会列出候选并要求改用 `%pane_id`)。用户说"关掉某个/所有 codex 窗格"时用它,不要手拼 `tmux kill-pane`。
- 登记簿:`${XDG_CACHE_HOME:-~/.cache}/codex-tmux/registry.tsv`(pane_id/任务名/终报/工作目录/启动时间/边框色)。

## 失败样态与排障

- 窗格死亡未完成 → exit 2:现场保留在死窗格 + `<终报>.pane.log`;notify 原始 JSON 在 `<终报>.notify.json`。
- `--timeout <秒>` 到点 → exit 124:codex 本体继续跑,稍后自查 `<终报>.done` / `.notify.json`。
- 辅件全在终报同目录:`.brief` `.launch.sh` `.notify.sh` `.notify.json` `.done` `.pane.log` `.stamp`。

## 旋钮

- 环境变量:`CODEX_TMUX_MODE=pane|window|exec`、`CODEX_TMUX_LAYOUT=main-vertical|none`、`CODEX_TMUX_PANE_WIDTH`(首切宽,默认 `70%`)、`CODEX_TMUX_MAIN_WIDTH`(主窗格宽,默认 `30%`)、`CODEX_TMUX_BYPASS=1|0`(默认 1)、`CODEX_HOME`(session 反查,默认 `~/.codex`)。
- 单次:`-w`=独立 window;`--close`=完成即关窗格;`--timeout <秒>`。

## 用户 /codex <任务> 直呼时

把 <任务> 扩写为自包含简报(判据、上下文、产物路径齐全)落盘,按标准调用派发;机械任务降档 effort。派发后照常推进自己的主线工作,唤醒后读终报向用户汇报。
