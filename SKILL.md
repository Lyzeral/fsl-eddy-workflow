---
name: fsl-eddy-workflow
description: DWI 弥散成像预处理第二步「FSL eddy 涡流与头动校正」的完整工作流与 QC 规范。用于：准备/执行 eddy 校正（生成 acqparams/index、提取 b0、BET 掩码、运行 eddy）、批量处理多例 DWI、解读头动 QC 指标（movement_rms/parameters/outlier_report）、判断掩码是否可修补、排查 FSL 安装与 WSL 运行问题、整理 QC 汇报。触发场景：用户提到 eddy、涡流校正、头动校正、DWI 预处理、BET 掩码、acqparams、bvec/bval、FSL 安装问题、QC 汇报。
---

# FSL eddy 涡流与头动校正工作流

## 背景（数据与环境）

- 项目口径：14 被试 × 2 时间点 = 28 例 DWI（sub-MCT16/17/20/21/22/24/25/28/29/30/32/33/34/35，ses-baseline/endpoint）
- 采集参数（28 例实测）：Philips Achieva 3T，128×128×75×33，2mm 各向同性，33 卷 = 1 个 b0（位于第 33 卷，0 起索引 32）+ 32 个 b=1000 方向；TR≈9.97s、TE 91.5ms、SENSE 2、DwiSE 2D SE-EPI
- 相位编码轴 j，json 无极性（Philips 通病）→ acqp 用 `0 1 0 0.049`；无反相 PE 场图（FE_EPI 是静息 BOLD）→ **不做 topup**
- adc（扫描仪重建图）单独导出、**不参与 eddy**
- 环境：WSL2 Ubuntu 24.04.5 + FSL 6.0.7.23（`/home/song/fsl`，eddy 实际用 `eddy_cpu`）

## 方法（单例处理步骤）

进入输出目录后依次执行：

1. 生成 acqparams.txt（每例 1 行，PE 方向 + 总读出时间）：
   `echo "0 1 0 0.049" > acqparams.txt`
2. 生成 index.txt（nvols 个 1，本项目 33 个）：`seq 33 | awk '{printf "1 "} END{print ""}' > index.txt`
3. 提取 b0（4D 提 t 索引 32）：`fslroi dwi.nii.gz b0.nii.gz 0 -1 0 -1 0 -1 32 1`
   - 多个 b0 时：逐个提取后 `fslmerge -t` + `fslmaths -Tmean` 平均
4. BET 脑提取与掩码：`bet b0.nii.gz b0_brain.nii.gz -f 0.2 -m -R`
5. 运行 eddy：
   `eddy_cpu --imain=dwi.nii.gz --mask=b0_brain_mask.nii.gz --acqp=acqparams.txt --index=index.txt --bvecs=dwi.bvec --bvals=dwi.bval --out=<sub>_<ses>_dwi_eddy --nthr=16 --repol --cnr_maps --residuals --data_is_shelled`
   - 二进制名自动识别顺序：eddy_cpu → eddy_openmp → eddy（`pick_eddy`）
   - 不装 GPU 库时用 `eddy_cpu`（CPU 版足够本项目）
6. 批量处理：`bash scripts/batch_eddy.sh [sub] [ses]`
   - 幂等：已有 `<out>.nii.gz` 自动跳过；`FORCE=1` 强制重跑；`RUN_QUAD=1` 追加 eddy_quad
   - 退出码：0=全成功/跳过；1=环境错误；2=存在失败例
   - 日志：`data/processed/dwi/logs/eddy_*.log`

## 掩码检查原则（影像科主任意见，重要）

- b0 掩码**只检查、不手工修补**：b0 是低分辨率 EPI 图，本身易几何变形，手工描边修正的是"看起来对、实际坐标错"的边界
- 精细修补掩码是 **T1 结构像**的流程（高分辨率边界清晰才可分割精修），不属于 b0/eddy 环节
- 检查发现掩码不合格（漏脑/明显偏出）→ **调 BET 阈值 -f 重跑**（如 0.2 → 0.3），而不是手动改掩码
- eddy 只需"粗略完整包住脑"的掩码，BET 粗掩码即可，无需解剖级精度

## 结果（产出与 QC）

### 产物（输出到 data/processed/dwi/<sub>/<ses>/）

| 文件 | 用途 |
| --- | --- |
| `<out>.nii.gz` | **校正后 DWI（核心产物）** |
| `<out>.eddy_rotated_bvecs` | **校正后梯度方向——后续 DTI 建模必须用它，禁用原始 bvec！** |
| `<out>.eddy_parameters` | 逐卷 12 参数（平移/旋转/缩放/剪切） |
| `<out>.eddy_movement_rms` / `.eddy_restricted_movement_rms` | 头动 RMS（绝对/相对），33 行 2 列 |
| `<out>.eddy_outlier_report` | 异常切片报告（--repol 修复记录） |
| `<out>.eddy_cnr_maps.nii.gz` / `.eddy_residuals.nii.gz` | CNR 信噪比图 / 残差图 |

### QC 判定阈值（与 qc_eddy_matlab.m 一致）

| 指标 | 警告阈值 |
| --- | --- |
| 绝对头动 RMS 均值 | > 1.5 mm |
| 相对头动 RMS 均值 | > 1.0 mm |
| 相对头动 RMS 最大 | > 3.0 mm |
| 平均旋转角 | > 1.0° |

实测基准（sub-MCT16 baseline）：0.264 / 0.045 / 0.340 mm / 0.001° → 全部通过。异常切片由 `--repol` 自动修复（`eddy_outlier_report` 5 条 + `eddy_outlier_free_data.nii.gz`），**无需人工处理**。

### QC 归档

- 每例 QC 汇报 → `docs/04_eddy_qc_reports/<sub>_<ses>_QC.md`（总览表 + 逐卷明细 + 异常切片 + 产物清单 + 结论），索引登记在 `README.md`
- 逐卷明细 CSV → `data/processed/dwi/qc/`（用 `scripts/gen_eddy_qc_csv.py <sub> <ses>` 生成）

## 检查（阶段门与陷阱）

### 运行前/后必查

- 数量闭环：28 例全覆盖核对（跳过 = 已有结果，失败要有名单）
- 输出完整性：`<out>.nii.gz` 存在、`eddy_rotated_bvecs` 存在
- 头动指标是否超阈；`eddy_outlier_report` 数量合理（过多提示数据质量问题）

### 已知陷阱（勿重踩）

1. **WSL 后台任务会随最后一个终端退出被清理**（发行版停止，nohup 也没用）→ 长任务用 Windows 侧保活：`Start-Process wsl.exe -ArgumentList '...' -WindowStyle Hidden`（隐藏会话持有发行版直到任务结束）
2. **电脑睡眠会暂停 WSL** → 长任务期间电源设为"从不睡眠"；中断后重跑脚本（幂等）即续
3. FSL 安装失败根因（fslinstaller "2/4 Installing conda" tar 报错）= 精简镜像**缺 bzip2** → `apt-get install -y bzip2` 后重跑；磁盘空间不足不是常见原因（先 `df -h` 排除）
4. 国内网络：GitHub 不可达时用清华镜像 `ubuntu-releases/24.04/ubuntu-24.04.5-wsl-amd64.wsl` + `wsl --import`；FSL 安装器主体在 conda-forge，慢是正常的
5. fslinstaller 用 `-c none` 跳过 CUDA（省 1GB+，CPU 版足够）；`-o` 覆盖、`-r` 不注册
6. PowerShell → wsl 参数转义易碎（反斜杠/引号被吞）→ 复杂 bash 写成 `.sh`（LF）放 `D:\tools\wsl\` 再执行
7. eddy 输出名：`<sub>_<ses>_dwi_eddy`（**含 `_dwi_`**），脚本/CSV 引用时勿漏

### 参考文件

- 完整教程：`docs/03_涡流与头动校正_FSL_eddy教程.md`
- 批量脚本：`scripts/batch_eddy.sh`（参数集中区：PE_VECTOR / TOTAL_READOUT / BET_FRAC / NTHREADS / FORCE / RUN_QUAD）
- MATLAB QC：`scripts/qc_eddy_matlab.m`；CSV 工具：`scripts/gen_eddy_qc_csv.py`
