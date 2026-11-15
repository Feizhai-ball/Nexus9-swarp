# Nexus 9 (flounder) zRAM 1 GiB / LZ4 / 2-streams 修复与验证报告

> 状态：设备已正式刷写并完成压力验证  
> 设备：HTC Nexus 9（产品代号 `flounder`）  
> 系统：Android 7.1.2 / CM14.1 系列 userdebug  
> 内核：Linux 3.10.103-g5cef0dc1890-dirty  
> 报告整理日期：2026-11-15

## 1. 结论摘要

本次修改解决了启动阶段 fstab 覆盖 init.rc 中 zRAM 压缩流设置的问题。最终通过在活动配置文件 `fstab.flounder` 的 swap flags 中显式加入 `zramstreams=2`，使 fs_mgr 在初始化 zRAM 时设置双压缩流。

最终镜像已通过 fastboot 正式刷写到 boot 分区，并在设备启动后验证：

- zRAM 容量：1,073,741,824 bytes（1 GiB）
- 压缩算法：LZ4（sysfs 显示 `lzo [lz4]`，方括号内为当前选中项）
- `max_comp_streams`：2
- Swap：`/dev/block/zram0` 已启用
- 压力测试：最高分配并触碰 768 MiB 匿名内存，观察到实际 swap-out 和 swap-in；测试后系统内存恢复。

**最终结论：1 GiB LZ4 zRAM、双压缩流配置已在该 Nexus 9 上正式生效，并通过实际换页压力测试。**

## 2. 设备与原始镜像信息

设备识别：
- fastboot product：`flounder`
- Android：7.1.2
- 内核：`3.10.103-g5cef0dc1890-dirty`
- 内核构建信息：`razorloves@razorloves-VBox`
- 构建时间：`Mon Apr 11 22:05:41 CDT 2022`
- RAM：`MemTotal 1879792 kB`（约 1.79 GiB）

原始 boot 镜像备份：
- 文件：`LNX_original_pull.img`
- 大小：33,554,432 bytes（32 MiB）
- SHA-256：`9c5df7f7113bf49902564bc79eeffb51dbe681df3bf1602e9541c0bf05b2dd18`

原始镜像的 fstab swap 配置为：

```text
/dev/block/zram0 none swap defaults zramsize=533413200,notrim
```

原始容量约 508.7 MiB，且未显式配置 zRAM streams。

## 3. 根因分析

原始 `/init.flounder.rc` 中有：

```rc
on init
    write /sys/block/zram0/comp_algorithm lz4
    write /sys/block/zram0/max_comp_streams 2

on post-fs
    swapon_all /fstab.flounder
```

看起来 init 已经将压缩流数设置为 2，但 CM14.1 对应的 fs_mgr 在解析 fstab 时：
1. zRAM streams 默认值为 1；
2. 只有 fstab 明确提供 `zramstreams=` 才会覆盖默认值；
3. `swapon_all` 初始化 zRAM 时重新写入 `/sys/block/zram0/max_comp_streams`；
4. 因此原有 init.rc 写入的 2 会被 fs_mgr 的默认值 1 覆盖。

所以问题不在 GA10x/压缩算法驱动，而是 fstab 与 fs_mgr 的初始化顺序和默认值覆盖。

## 4. 最终修复内容

只修改 ramdisk 中活动配置文件 `fstab.flounder` 的 zRAM swap flags。

最终配置：

```text
/dev/block/zram0 none swap defaults zramsize=1073741824,zramstreams=2,notrim
```

改动：
- zRAM 容量由原来的 533413200 bytes 改为 1073741824 bytes；
- 增加 `zramstreams=2`；
- 保留 `notrim`；
- 压缩算法仍由原有 init 配置设置为 LZ4；
- 未修改内核二进制；
- 未修改附加 DTB 数据；
- 只修改活动的 `fstab.flounder`，未修改 `fstab.flounder64`。

### 4.1 最终候选镜像校验

- 文件名：`LNX_zram_1024MiB_zramstreams2_candidate.img`
- 大小：33,554,432 bytes（32 MiB）
- SHA-256：

```text
b0d4887df788e87a0c14f3bddf7cd064701ac26a35a99082850710f89b05d1bc
```

镜像静态审计确认：
- boot image 总大小保持 32 MiB；
- kernel 内容与原始镜像完全一致；
- ramdisk 中仅活动 `fstab.flounder` 被修改；
- 原始镜像附加数据保持不变；
- 附加数据中的 4 个 FDT 均保留；
- CPIO、gzip ramdisk 和 boot header 均通过重建后检查；
- boot image ID 按 AOSP legacy SHA-1 规则重新计算。

原始 kernel SHA-256：
`52409f2b6dede549866522f5f06b774a56bc69c7055576209eb410fd9a813147`

原始 ramdisk 页对齐后附加数据 SHA-256：
`8642ba4267ed4554c438220ceec9eb35e1d77c4f1c809c73254409459f09f5d8`

## 5. 刷写与启动验证

先使用：

```bash
fastboot boot LNX_zram_1024MiB_zramstreams2_candidate.img
```

进行临时启动验证。临时启动后通过 root shell 检查：
- `comp_algorithm`：`lzo [lz4]`
- `max_comp_streams`：`2`
- `disksize`：`1073741824`
- `initstate`：`1`
- `/proc/swaps`：zRAM swap 已启用，容量约 1 GiB。

之后重启回原系统，再正式执行：

```bash
fastboot flash boot LNX_zram_1024MiB_zramstreams2_candidate.img
fastboot reboot
```

正式刷写后重新检查，仍得到：
- 当前压缩算法 LZ4；
- streams = 2；
- disksize = 1073741824；
- `/proc/swaps` 中 `/dev/block/zram0` 已启用。

设备 boot 分区大小为 0x2000000（32 MiB），因此使用 `fastboot flash boot`，不是刷写不存在的 `LNX` 分区。

## 6. 压力测试过程

测试程序通过 Ubuntu NDK 交叉编译为 ARM64 Android 可执行文件，在设备上以匿名 mmap 分配内存，并逐字写入伪随机数据，避免只写零页造成不真实的高压缩结果。

测试步骤：
- 第一轮：最高 256 MiB，按 32 MiB 递增；
- 第二轮：最高 384 MiB，按 32 MiB 递增；
- 最终验证轮：最高 768 MiB，按 32 MiB 递增；
- 最终轮每阶段等待 15 秒，达到上限后保持 30 秒，然后释放内存；
- 768 MiB 版本设置 MemFree 低于 80 MiB 时停止继续分配；
- 测试过程中通过 `/proc/meminfo`、`/proc/vmstat`、`/sys/block/zram0/mm_stat` 和 `/proc/swaps` 监控。

### 6.1 早期测试结果

256 MiB 与 384 MiB 测试均正常分配、触碰并释放内存，但没有触发 swap：
- `pswpout = 0`
- `pswpin = 0`
- Swap Used = 0
- zRAM mm_stat 保持基线。

这说明单纯分配 256/384 MiB 不足以在当时的系统回收状态下触发匿名页换出，并不表示 zRAM 故障。

### 6.2 768 MiB 压力测试期间

监控快照：

```text
MemFree:          127724 kB
Cached:           243108 kB
SwapCached:            64 kB
SwapTotal:       1048572 kB
SwapFree:        1039804 kB

pswpin 16
pswpout 2192

mm_stat:
8617984 1684263 2101248 0 2101248 73 0

/proc/swaps:
 /dev/block/zram0 partition 1048572 8768 -1
```

关键解释：
- `pswpout=2192`：发生了实际页面换出；
- `pswpin=16`：发生了实际页面换入；
- Swap Used = 8768 KiB：zRAM swap 已实际承载交换页；
- `mm_stat` 的原始数据约 8.22 MiB，压缩数据约 1.61 MiB，zRAM 实际内存占用约 2.00 MiB；
- `same_pages=73`。

按 `orig_data_size / compr_data_size` 计算，该快照压缩比例约 5.12:1。此比例仅代表该测试快照的数据样本，不应视为所有应用负载下的固定压缩率。

### 6.3 释放后的结果

测试程序释放内存后：

```text
MemFree:          959776 kB
Cached:           252432 kB
SwapCached:            64 kB
SwapFree:        1039788 kB

pswpin 16
pswpout 2196

mm_stat:
8634368 1689071 2105344 0 2105344 73 0

/proc/swaps:
 /dev/block/zram0 partition 1048572 8784 -1
```

结果：
- MemFree 恢复至约 937 MiB；
- 设备未出现持续内存压力；
- Swap Used 保留约 8.58 MiB；
- `pswpout` 在释放阶段结束时增加到 2196；
- zRAM 统计仍保留少量交换槽数据，这是正常现象，释放测试进程并不保证所有 swap slot 立即归零。

测试期间没有观察到 OOM、设备重启或系统崩溃。测试结束后设备内存恢复正常。

## 7. 内核内存管理补充检查

检查结果：
- `swappiness=60`
- `min_free_kbytes=5286`
- `allocstall=0`
- 设备存在传统 Android Low Memory Killer 参数目录；
- 未挂载 memory cgroup；
- `/proc/modules` 不存在，但 zRAM sysfs 设备存在且工作。zRAM 可能编译进内核，不能以 `/proc/modules` 缺失判断模块故障。
- 后台 kswapd 曾扫描和回收 DMA32 区域页面；早期压力测试没有触发 swap-out。
- `compact_success=0` 与 zRAM 是否工作无直接关系，内存整理和 swap 换页是不同机制。

## 8. 重要文件与校验值

| 文件 | 用途 | SHA-256 / 状态 |
|---|---|---|
| `LNX_original_pull.img` | 原始 boot 镜像备份 | `9c5df7f7113bf49902564bc79eeffb51dbe681df3bf1602e9541c0bf05b2dd18` |
| `LNX_zram_1024MiB_zramstreams2_candidate.img` | 最终有效镜像 | `b0d4887df788e87a0c14f3bddf7cd064701ac26a35a99082850710f89b05d1bc` |
| `zram_stress.c` | 初始 256/384 MiB 测试源码 | 本地生成 |
| `zram_stress_v2.c` | 带换页统计的测试源码 | 本地生成，最终 768 MiB 版本按本报告参数调整 |

仓库中已有的 `LNX_zram_1024MiB_candidate.img` 是此前上传的旧候选文件。它的文件名和内容校验值不能证明其包含本报告所述的 `zramstreams=2` 修复，因此**不要把它当作本报告的最终有效镜像刷写**。

最终有效镜像在本次报告整理时尚未以二进制文件形式上传到仓库。应将本地且 SHA-256 与本报告完全一致的 `LNX_zram_1024MiB_zramstreams2_candidate.img` 上传到仓库根目录后，再把上面的镜像链接更新为最终文件链接。若文件哈希不一致，不要刷写。

## 9. 恢复与安全说明

- 保留原始镜像 `LNX_original_pull.img` 作为回退备份。
- 只有在确认文件 SHA-256 与本报告一致后，才使用最终镜像。
- 本次正式刷写目标是 boot 分区。
- 不需要修改 kernel、DTB 或 SELinux 策略。
- 不建议继续做 1 GiB 级别压力测试；768 MiB 已经验证实际换页且释放后系统恢复，继续增加压力收益有限。
- 设备电量应充足，刷写期间保持 USB 连接，避免在低电量状态下操作。

## 10. 最终结论

本次修复采用最小化改动：只调整活动 fstab 的 zRAM 容量和 streams 参数，保留原内核及 DTB。最终镜像已正式刷写并验证。压力测试观察到真实的 swap-in、swap-out、zRAM 压缩数据及测试后的内存恢复。

**最终配置：1 GiB zRAM + LZ4 + 2 compression streams。**

**最终有效镜像 SHA-256：**
`b0d4887df788e87a0c14f3bddf7cd064701ac26a35a99082850710f89b05d1bc`
