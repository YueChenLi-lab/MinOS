# 2026-06-23 下一次评测路线

## 当前可信基线

| 证据 | 结论 |
|---|---|
| 最新可见官方结果 | 2026-06-22 18:46:51，`Accepted / 484.1693353980349`；basic=204、BusyBox=98、Lua=18、libcbench=57.16933539803484、libctest=107、lmbench=0 |
| 最新高分结果 | 2026-06-21 13:15:41，`Accepted / 484.26735406790885`；iozone-lite 撤回后已恢复 |
| 通过基线得分构成 | RISC-V basic `204`、BusyBox `98`、Lua `18`、libcbench 约 `57`、musl static libctest `107` |
| `c941356` 结果 | 24-command lmbench 队列在线上耗时约 16 分 41 秒，串口出现大量 25s timeout，lmbench 仍为 0 |
| 当前修复方向 | 撤回长 lmbench 队列，使用短轮次 lmbench 保持诊断；主攻 dynamic libctest 小批量真实 case |
| 当前已知边界 | LoongArch 占位 ELF；iozone、lmbench、ltp、网络/性能测试仍未稳定得分 |

这轮目标不是扩大测试面，而是在 484 基线附近寻找低风险新增分。`c941356` 已证明
长 lmbench 队列消耗评测时间但不进分；本轮先恢复短命令队列，把新增风险集中在
独立 `libctest-musl-dynamic` 组。dynamic 组必须能独立跳过、独立超时，不能影响
static libctest 107 分。

## 本轮提交门禁

1. 强制离线 `make all`，vendor checksum 保持 `53/0`。
2. 隐藏文件过滤后的干净导出仍能恢复 Cargo 配置并构建。
3. `kernel-rv` 为 RISC-V executable ELF，入口 `0x80200000`。
4. submit 默认构建 feature 仍为 `submit tmpfs`，不带默认 `stack_trace`。
5. 现有 basic、BusyBox、Lua、libcbench、musl static libctest staging 顺序不回退。
6. lmbench 队列只保留 9 条短轮次命令，使用 `-W 1 -N 10` 与 10s timeout。
7. dynamic libctest 使用独立 `libctest-musl-dynamic` marker，只跑 8 个低风险 case。
8. dynamic libctest 只暂存 `entry-dynamic.exe` 和 PT_INTERP 需要的最小 musl runtime。
9. dynamic staging 超过 4MiB、缺 runtime 或缺 PT_INTERP 时跳过该组并继续后续组。
10. Git 状态不包含本地镜像、日志、验证夹具或构建产物。

## 下一次官方评测验收

- Compile 阶段通过，不再出现 vendor、Cargo registry 或隐藏文件过滤导致的错误。
- RISC-V basic 保持 `204/204`。
- RISC-V BusyBox 保持 `98/98`。
- RISC-V Lua 保持 `18/18`。
- libcbench 维持约 `56-58` 分区间。
- `libctest-musl` static 保持 `107/107`。
- `libctest-musl-dynamic` 至少出现真实 START 与 case START；如果仍为 0，日志必须能区分缺 PT_INTERP、缺 runtime、`execve` errno 或 timeout。
- lmbench 若仍为 0，可以接受；本轮不允许因 lmbench timeout 把总运行时间明显拉长。
- RISC-V 输出中没有 `Panicked`，最终输出 `!TEST FINISH!` 并主动关机。
- 若总分低于 480，优先撤回 dynamic libctest probe，回到 484 基线。

## 7 天提分顺序

1. 第 1 天：提交本轮 dynamic libctest 小批量探针，确认 484 基线不回退，并读取官方串口。
2. 第 2 天：若 dynamic 有 case 进入 `main`，按官方通过情况把 8 case 扩到 16-24 case；若全是 loader/runtime 失败，先修动态解释器路径和 musl 搜索路径。
3. 第 3 天：若 dynamic 已有新增分，再只扩 dynamic libctest；暂不碰 iozone、ltp、网络。
4. 第 4 天：若 dynamic 无法进分，回到 lmbench，优先修已经能输出的 `lat_syscall`、`lat_select`、`lat_sig` parser/分组问题，不恢复 24-command 队列。
5. 第 5 天：只在有明确串口证据时补 `pipe/fork/exec/wait` 热路径，否则不扩大 lmbench。
6. 第 6 天：保留最后一次高分提交，做隐藏文件过滤、离线构建、无盘启动和官方参数启动回归。
7. 第 7 天：只做止血和文档，不再引入 iozone、ltp、网络、LoongArch 或大规模重构。

## 本轮暂缓

- 不新增 iozone、iperf、netperf 或 ltp 正式执行组。
- 不处理网络、多核和 LoongArch。
- 不恢复 24-command lmbench 队列。
- 不修改 static libctest allowlist、timeout 或 `C` 队列协议，除非官方回归。
