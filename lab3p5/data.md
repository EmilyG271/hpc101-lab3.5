## 环境信息

- 日期：2026-09-04
- DevPod：arm64-910B
- 镜像：hpc101-lab35:latest
- CANN 版本：8.5.0
- 硬件：Ascend 910B4
- 代码 commit：a51517f

## 实验目的

1. 使用 Ascend C 实现 `fused_add_rmsnorm`，完成融合残差加法与 RMSNorm：
   `residual_out = x + residual`，`y = residual_out / sqrt(mean(residual_out^2) + eps) * weight`。
2. 通过全部公开正确性 case，并保持非对齐 H、B=1、大 B、大 H 路径的正确性。
3. 在固定评分 shape `[256, 1024]`、FP16、`eps=1e-6` 下优化 `Task Duration(us)`。
4. 使用 `msprof op` 完成性能分析，并保存 profiling 结果。

## 评分依据

| 项目 | 配置 |
| --- | --- |
| Shape | `[256, 1024]` |
| 数据类型 | FP16 |
| eps | `1e-6` |
| 指标 | msprof op 的 `Task Duration(us)` |
| 预热 | 本地脚本 `checker/profile.sh` 使用 20 次 |

| Task Duration | 得分 |
| ---: | ---: |
| 15.0 us | 0 分 |
| 4.5 us | 100 分 |
| 3.5 us 及以下 | 120 分 |

评分区间使用对数插值：

```text
4.5 < t <= 15.0: score = 100 * ln(15.0 / t) / ln(15.0 / 4.5)
3.5 < t <= 4.5: score = 100 + 20 * ln(4.5 / t) / ln(4.5 / 3.5)
t <= 3.5:       score = 120
```

## 正确性结果

测试命令：

```bash
hpc submit -p lab3p5 bash checker/run.sh
```

作业 ID：`246286`

| Case | Shape | y | residual_out | 结果 |
| ---: | --- | --- | --- | --- |
| 0 | `32 x 4096` | PASS | PASS | PASS |
| 1 | `256 x 1024` | PASS | PASS | PASS |
| 2 | `1 x 4096` | PASS | PASS | PASS |
| 3 | `1997 x 3037` | PASS | PASS | PASS |
| 4 | `2048 x 4096` | PASS | PASS | PASS |

结论：5/5 个公开 case 全部通过。

## 性能结果

测试命令：

```bash
hpc submit -p lab3p5 bash checker/profile.sh
```

作业 ID：`246312`

| 项目 | 数值 |
| --- | ---: |
| Task Duration | `6.720000 us` |
| 对应得分 | `66.69` 分 |
| Block Dim | `40` |
| 当前频率 / 额定频率 | `1650 MHz / 1650 MHz` |
| Kernel | `FusedAddRmsNorm_f5a8bf5afa6875e75a8ea27e3b64457a_0` |

本地 profiling 数据保存在 `lab3p5/prof_out_final/`，远端原始目录为：

```text
/home/h3250102096/src/lab3p5/prof_out/OPPROF_20260904144320_JTFEOXTIVJOZFCXN
```

### 关键 msprof 指标

以下区间取自有效计算 block 的 `PipeUtilization.csv` / `MemoryUB.csv` / `Memory.csv`：

| 指标 | 数值 |
| --- | ---: |
| 有效 AIV 时间 | `4.28 - 5.81 us` |
| Vector 时间 | `0.98 - 1.41 us` |
| Vector 利用率 | `20.7% - 33.1%` |
| Scalar 时间 | `1.42 - 3.41 us` |
| MTE2 时间 | `1.43 - 2.85 us` |
| MTE3 时间 | `0.32 - 0.41 us` |
| UB 读带宽峰值 | `66.40 GB/s` per AIV |
| UB 写带宽峰值 | `51.08 GB/s` per AIV |
| GM -> UB 带宽峰值 | `6.71 GB/s` per AIV |
| UB -> GM 带宽峰值 | `6.24 GB/s` per AIV |
| L2 总命中率均值 | `11.45%` |

性能结论：最终版本的主要瓶颈仍集中在 Scalar 控制开销和 MTE2 搬运，Vector 利用率约三分之一以内，MTE3 写回时间很小。继续压到 4.5 us 需要显著减少逐行标量调度或进一步重构输入搬运与规约路径。

## 优化历程

| 版本 | 主要修改 | Task Duration | 相对 baseline 加速比 | 正确性 |
| --- | --- | ---: | ---: | --- |
| baseline | 原始逐行实现，每行独立加载、规约和写回 | `15.141 us` | `1.00x` | 全部通过 |
| v1 | 连续行打包，减少每行搬运和启动开销 | `6.920 us` | `2.19x` | 全部通过 |
| v2 | 多行压缩归约，规约结果保留在 Vector pipe | `6.420514 us` | `2.36x` | 全部通过 |
| v3 | 8 行 tile、`RepeatReduceSum`、`Brcb` 广播、单缓冲 | `6.320000 us` | `2.40x` | 草案在 `2048x4096` 失败 |
| v4 | 修复 `H>2040` repeat stride 溢出，大 H 回退单行；host 按 B 上限裁剪空核 | `6.720000 us` | `2.25x` | 全部通过 |

补充 A/B 实验：

| 实验 | 修改 | Task Duration | 结论 |
| --- | --- | ---: | --- |
| 32 AIV | host 只启动 32 个 block，每 block 8 行，强制走 `Brcb` 路径 | `6.920554 us` | Vector 时间上升到约 `1.81 us`，不采用 |
| 40 AIV | 每核约 7 行，走压缩归约和逐行归一化路径 | `6.320000 - 6.720000 us` | 最终采用，板间波动约 6% |

`v4` 的最终实测耗时高于历史最好值 `6.320000 us`，但两者是同一核心算法，差异主要来自板端负载和 profile 采样波动。`v4` 额外修复了 `H=4096` 时 repeat stride 超出 `uint8_t` 导致 `2048x4096` 错误的问题，因此作为最终提交版本。

## 测试结论

1. 最终 Ascend C 实现全部 5 个公开正确性 case 通过，`y` 与 `residual_out` 均正确。
2. 最终 profile 的 `Task Duration` 为 `6.720000 us`，按给定评分公式估算为 `66.69` 分。
3. 历史最好实测为 `6.320000 us`，估算约 `71.78` 分，但该草案存在 `2048x4096` 正确性问题；最终提交以正确且可复现的 `v4` 为准。
4. 相比 15.0 us baseline，最终版本仍有约 `2.25x` 加速。
5. 尚未达到 4.5 us 的 100 分目标；后续优化应优先减少 Scalar issue 与控制开销，并重新组织 MTE2 输入流水。

## 2026-09-05 optimization_plan_v2 执行记录

### A0 基线复现

| 项目 | 结果 |
| --- | ---: |
| 作业 ID | `250491` |
| Task Duration | `6.940000 us` |
| 结论 | 与 v4 的 `6.720000 us` 同量级，可作为本轮对照 |

### A1/A2 短行组补齐与批量广播

**修改内容：**

- 在 `ProcessAlignedRowGroups()` 中将真实的 7 行扩展为片上 8 行，以启用 `Brcb` 批量广播。
- GM 读写和算术长度仍使用真实行数，只影响片上规约与广播路径。
- 对 dummy lane 使用 `SetValue` 并补充 `S_V` 同步，修复 32B 对齐问题。

**验证结果：**

| 项目 | 结果 |
| --- | ---: |
| 正确性作业 | `250566` |
| 公开 case | 5/5 PASS |
| 关键边界 | `2048x4096`、`1997x3037` 均通过 |
| 性能作业 | `250584`、`250611` |
| Task Duration | `6.980558 us` / `6.940555 us` |
| 中位数 | `6.960557 us` |
| 估算得分 | `63.77` |

**关键 msprof 指标：**

| 指标 | 数值 |
| --- | ---: |
| AIV 时间 | `2.507 - 6.112 us` |
| Vector 时间 | `0.017 - 1.705 us` |
| Scalar 时间 | `1.447 - 3.852 us` |
| MTE2 时间 | `1.032 - 2.521 us` |
| MTE3 时间 | `0.001 - 0.421 us` |
| GM -> UB 峰值带宽 | `6.123 GB/s` |
| UB -> GM 峰值带宽 | `5.691 GB/s` |
| UB 读峰值带宽 | `72.814 GB/s` |
| UB 写峰值带宽 | `53.506 GB/s` |

**结论：** A1/A2 虽然保持 5/5 正确性，但 Task Duration 与 A0 基线基本持平，且高于 v4 最终版本，未带来可采纳的性能收益。因此该实验不进入最终提交版本，代码回退到 v4；后续优化应转向 MTE2 输入流水与更安全的 row-tile 选择器。
