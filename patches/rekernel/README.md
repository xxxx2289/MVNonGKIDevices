# Re:Kernel 内核内嵌形态补丁（4.9 / 4.14 / 4.19）

来源：Sakion-Team/Re-Kernel（GPL-2.0，`Integrate/` 素材）；本地工作区快照
`localworkspace/mirrors/Re-Kernel/`（v11.6）。≤5.4 非 GKI 走内核内嵌形态；
≥5.10 上游走 ko/Magisk 模块，不在本仓库。

## 组成（4.9/、4.14/、4.19/）

三版各三件；`0002`/`0003` 的锚点按各树上下文落位，`0001` 里
`rekernel.h` 的 `jobctl_frozen()` 形态按树内是否具备 `JOBCTL_TRAP_FREEZE`
选择（4.9/4.14 带走 fallback，4.19 用上游原文）；差异见各自 README。

### 4.9/

| 补丁 | 内容 |
|---|---|
| `0001-drivers-rekernel-netlink-server.patch` | 新增 `drivers/rekernel/`（`rekernel.c`/`rekernel.h`/`Kconfig`/`Makefile`）：netlink unit 22–26 探测、user port 100 上报；上报前统一过滤（目标非冻结组、与源同 uid 丢弃） |
| `0002-binder-frozen-transaction-notify.patch` | `drivers/android/binder.c`：`binder_transaction()` 内 target_proc 建立后按 reply 分派 reply/transaction 上报，oneway 且 async 空间不足再补 overflow 上报 |
| `0003-signal-frozen-kill-notify.patch` | `kernel/signal.c`：`do_send_sig_info()` 对 SIGKILL/SIGTERM/SIGABRT/SIGQUIT 上报 |

`drivers/Kconfig` 与 `drivers/Makefile` 的 `source`/`obj` 注入由 workflow 步骤
幂等完成：这两个文件会被 root 集成步骤改写，补丁 context 会漂移。

## 与上游形态的差异

- `rekernel.h`：上游直接读 `JOBCTL_TRAP_FREEZE`；pre-freezer-v2 树（4.9/4.14）
  无此位，改为 `#ifdef` 回退，冻结判定由 `frozen_task_group()` 的
  `cgroup_freezing()` 承担（4.19 有该位，可用上游原文）。
- `binder.c`：上游另插入 `TF_UPDATE_TXN`（`0x40`）的 async 事务合并块；三棵目标树
  都未命名该位，其中 MTK 4.14 的 `0x40` 是厂商 `TF_ASYNC_BOOST`，照搬本地定义会
  撞位，故本 port 只保留上报块。判定与证据见
  `localworkspace/pipelines/rekernel/FEASIBILITY.md`。

## 接线（workflow）

当前只有 polaris（mix2s）接 `enable_rekernel`（默认 off）：步骤 apply 三件补丁 +
`drivers/Kconfig`、`drivers/Makefile` 注入 + `rekernel.config.fragment`
（`CONFIG_REKERNEL=y`、`# CONFIG_REKERNEL_NETWORK is not set`）；`merge-defconfig.sh`
断言 `CONFIG_REKERNEL` 且 `CONFIG_REKERNEL_NETWORK` 未开；AK3 显示面在
`ENABLE_REKERNEL=true` 时于特性行加入 `REKERNEL`。

辐射顺序：mix2s 运行面通过后才把同一条接线接到其余设备；在此之前，4.14/4.19 与
4.9 其余设备的三件补丁只作为可行性评估素材留在 `patches/rekernel/{4.9,4.14,4.19}/`。

## 状态

- polaris（mix2s，`lineage-22.2`）：三件补丁对 tip 洁净树真实 `git apply` 顺序通过。
  编译面实测通过（`build-polaris` `dc8e718` 起：`OK CONFIG_REKERNEL=y` /
  `OK CONFIG_REKERNEL_NETWORK is not set`、`CC drivers/rekernel/rekernel.o` →
  `LD drivers/rekernel/built-in.o`，产出 `Image.gz-dtb` 与含 `REKERNEL` 特性行的
  AK3 包）。**运行面未通过**：刷入后 freezer 相关面提示内部异常，原因待定位——需要
  dmesg / 客户端日志与 `mount | grep cgroup` 的层级信息；本 port 在 4.9 上只识别 v1
  冻结状态（该树无 `JOBCTL_TRAP_FREEZE`，判定式为 `jobctl_frozen() ||
  cgroup_freezing()`），若该机走 cgroup v2 冻结，判定不会命中。定位前不辐射其他设备。
- 4.14（RMX2117）/ 4.19（alioth）：三件补丁对 `f0c2afc4d` / `71b13e6` 洁净树真实
  `git apply` 顺序通过；评估期构建（接线已收起）里 `CONFIG_REKERNEL=y`、
  `CC drivers/rekernel/rekernel.o` → 归档、`Image` 与含 `REKERNEL` 特性行的 AK3 包
  均达成。当前未接线。
- 4.9 其余设备（beryllium `thirteen`、daisy `lineage-20`、vince `13`）：三件顺序
  `git apply --check` 通过；vince 的 binder 段为 `target_proc->tmp_ref++`（裸自增），
  0002 按该形态重锚在 `4.9/vince/`（0001/0003 取共享件）；三棵树的
  `struct binder_alloc` 均带 `buffer_size`，与 port 里 `alloc.free_async_space` /
  `alloc.buffer_size` 的引用相符。当前未接线。
- 接线步骤在 polaris 的目标树 worktree 上真实执行：`drivers/rekernel/` 四件落位、
  `drivers/Kconfig` 与 `drivers/Makefile` 各注入一行、fragment 为 `CONFIG_REKERNEL=y`
  + `# CONFIG_REKERNEL_NETWORK is not set`、binder 与 signal 上报钩子各一处。
