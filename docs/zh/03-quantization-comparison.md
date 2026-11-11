---
AIGC:
    Label: "1"
    ContentProducer: 001191440300708461136T1XGW3
    ProduceID: 798659b1f59514cb753b383c1f423b09_a8c35eaadce611f1a0ca525400de85a5
    ReservedCode1: dEZTYWwc/DT8sthSSpDjlCbuB1zw2NW86sbXhe83fUwCJMkBFXVM17sxhtli/jOqww6E+IQuLjTcN2g7KlprLjwUGCpJtxgiQoLlpkXR9PhBiO3WetOauK/Pcv66NM+6JHwZiLm4HqP7BJzl3PEFmo+gl5/Yk7St30ZSoLzbeDsPaz8hm6sTc5+6D/Y=
    ContentPropagator: 001191440300708461136T1XGW3
    PropagateID: 798659b1f59514cb753b383c1f423b09_a8c35eaadce611f1a0ca525400de85a5
    ReservedCode2: dEZTYWwc/DT8sthSSpDjlCbuB1zw2NW86sbXhe83fUwCJMkBFXVM17sxhtli/jOqww6E+IQuLjTcN2g7KlprLjwUGCpJtxgiQoLlpkXR9PhBiO3WetOauK/Pcv66NM+6JHwZiLm4HqP7BJzl3PEFmo+gl5/Yk7St30ZSoLzbeDsPaz8hm6sTc5+6D/Y=
---

# Qwen-Image 2.1（stable-diffusion.cpp / LocModel 引擎）画质调优
## 方案②：量化档次对照 + 全量经验总结 + 最佳方案

- 完成时间：2026-11-10
- 引擎：`stable-diffusion.cpp` commit `58bb3c1`（`sd-server.exe`，LocModel 内置 master-1050-cuda 版）
- 硬件：RTX 3060 Ti 8GB / Windows 10 19045
- 固定条件：同 Prompt、`seed=1090474679`、1344×768、20 步、采样器 euler、`eta=0.0`、`distilled_guidance=3.5`

---

## 一、方案② 要回答的三个问题

| 编号 | 待验证问题 | 结论 |
|---|---|---|
| Q1 | Q3_K → Q4_K → Q6_K，画质是否随量化档提升？ | **否。Q3_K 最锐，Q6_K 最干净，Q4_K 无收益** |
| Q2 | 直觉上"更高量化档 = 更好"，实测为何不成立？是否为显存不足致 VAE 卸载？ | **VAE 卸载假设被否证**（见 §四），差异来自量化本身的噪声量 |
| Q3 | `distilled_guidance` 3.5→0 是否值得单独调？ | **不可用**：触发退化路径，VAE 解码 661 s（13 倍），已放弃 |

---

## 二、量化档对照结果（cfg=1.0 配方下，仅换模型）

| 组 | 模型 | 体积 | 总参数/显存分配 | 耗时 | 拉普拉斯方差 | 梯度p99 | 中频带% | 暗部噪声 | 饱和度 |
|---|---|---|---|---|---|---|---|---|---|
| **C** | **Q3_K** | 3.05 GB | 9171 MB（VAE 在显存） | **48~51 s** | **0.006255** | **0.1906** | **1.82** | 0.02228 | 0.1491 |
| E | Q4_K | 3.91 GB | 10055 MB（VAE 落 CPU） | 66~69 s | 0.005668 | 0.1529 | 1.36 | 0.01888 | 0.1463 |
| H | Q6_K | 5.59 GB | 11771 MB（diffusion 走磁盘） | **141.5 s** | 0.006721 | 0.1722 | 1.43 | **0.01816** | 0.1456 |

> 中频带 = 结构中频能量占比（越高代表真实结构细节越多）；暗部噪声 = 暗块内标准差中位数（越低越干净）。

**读法**：Q3_K 的中频结构能量最高（1.82）、梯度 p99 最高（0.1906），但同时噪声也最高（0.02228）。Q4_K/Q6_K 噪声降 15%~18%，代价是中频结构掉 21%~25%、耗时增 44%~192%。

### 像素级差异幅度（SSIM / MAE，全图）

| 对比 | 变化量 | SSIM | MAE | 判定 |
|---|---|---|---|---|
| A vs F | 仅量化 Q3_K→Q4_K（cfg6） | 0.9289 | 17.00 | 高度相似 |
| C vs E | 仅量化 Q3_K→Q4_K（cfg1） | 0.9027 | 16.97 | 高度相似 |
| C vs H | 仅量化 Q3_K→Q6_K（cfg1） | 0.8895 | 17.68 | 可察觉 |
| **A vs C** | **仅 CFG 6.0→1.0** | **0.6496** | **40.95** | **显著不同** |
| C vs K | 同参数重跑（复现性） | **1.0000** | **0.00** | 完全一致 |

**量化差异幅度仅为 CFG 差异幅度的 41%（MAE 17 vs 41）。** 修正 CFG 的收益，比更换量化档大一个量级。

---

## 三、视觉盲评（局部 100% 裁切 2× 放大）

| 对比 | 上=Q3_K / 下=量化更高档 | 盲评结论 |
|---|---|---|
| A vs F（cfg6） | Q3_K vs Q4_K | **Q3_K 明显更清晰**，Q4_K 线条发散、暗部噪点增多 |
| C vs E（cfg1） | Q3_K vs Q4_K | **几乎无差别**，仅暗部色调微小波动 |
| C vs H（cfg1） | Q3_K vs Q6_K | **Q6_K 细节更真实、噪点更少**，差异明显 |

盲评与数值的冲突正是本案关键：**Q3_K 的"锐"里含量化噪声成分**（噪声 0.02228 为三档最高），在高倍放大的干净度维度上 Q6_K 胜出；但 Q3_K 的真实结构能量（中频 1.82）确实最高。两者都不是"错"，是**不同维度的取舍**。

---

## 四、被否证的假设与异常路径（重要避坑）

### 4.1 "更高量化档被挤出显存导致画质劣化" —— 否证

日志显示不同量化档的自动卸载策略确实不同：

```
Q3_K: diffusion=CUDA0, vae=CUDA0            VRAM 3763 MB
Q4_K: diffusion=CUDA0, vae=cpu              VRAM 4003 MB
Q6_K: diffusion=disk,  vae=cpu              VRAM 5719 MB
```

若 VAE 精度/设备不同，解码结果应不同。于是强制 Q4_K 把 VAE 留在显存重跑（L 组，`--backend te=cpu,diffusion=CUDA0,vae=CUDA0`，VRAM 4647 MB）：

| 组 | 配置 | 耗时 | MD5 |
|---|---|---|---|
| E | Q4_K，VAE 落 CPU | 69.3 s | `c8fc349910372aac700e1097fe697e2d` |
| **L** | **Q4_K，VAE 在显存** | **66.3 s** | **`c8fc349910372aac700e1097fe697e2d`** |

**字节级完全一致**。结论：VAE 跑 CPU 还是 GPU 对输出无任何影响，画质差异 100% 来自量化权重本身。无需再为"让 VAE 上显存"折腾。

### 4.2 `distilled_guidance=0` —— 退化路径，禁用

设定 `txt_cfg=1.0, distilled_guidance=0` 后：

- 4 步探针：51.3 s 完成（看似正常）
- 20 步正式：`generating ... completed, taking 50.41s`（采样正常）→ **`latent decoded, taking 661.48s`** → 总计 **726.39 s**

采样耗时正常但 VAE 解码暴涨 13 倍，属引擎在该参数组合下的退化路径。期间还出现过进度条冻结 45 s 不动的假死相。**不要碰 `distilled_guidance`，保持默认 3.5。**

### 4.3 复现性验证

同参数同 seed 重跑（C / K 两组）MD5 相同、SSIM 1.0000、MAE 0.00。参数注入通道是**完全确定性**的，可作为回归测试基线。

---

## 五、经验总结（方法论层）

### 5.1 客观锐度指标不能单用，必须"环带分解 + 盲评"双轨

拉普拉斯方差与梯度 p99 都会被量化噪声抬高。本案 A 组（cfg6）三项锐度指标一度最高，实际视觉最脏。**先看中频带能量（结构）与暗部噪声（脏）的比值，再做放大盲评**，两者一致才可下结论；不一致时以 100% 裁切放大为准，并明确记录分歧。

### 5.2 混淆变量必须逐个用"字节级/比特级"手段排除

"更高量化更糊"的第一直觉是显存不足 → VAE 卸载。本次直接用 MD5 对比 E/L 两组，一次否证，省下大量调参时间。**凡是能靠 MD5 判定的，不要靠指标推断。**

### 5.3 差异幅度要归一化再比较

SSIM/MAE 的价值不在绝对值，而在**跨变量的相对幅度**：量化 MAE≈17、CFG MAE≈41。一眼看出"该调哪个旋钮"。调优顺序应由幅度决定，而不是由可调项多少决定。

### 5.4 高成本参数的"快探针"要配"正式步数"复核

dg=0 用 4 步探针 51 s 正常通过，20 步才暴露 726 s。**探针只能证伪不能证真**：凡是探针通过的可疑参数，正式步数必须再跑一次才能采纳。

### 5.5 耗时异常看日志分段

`generate_image completed in X` 是一行总数，真正的分段证据在 `generating N latent images completed` 与 `latent decoded` 两行。本案正是靠"采样 50 s + 解码 661 s"的荒谬比例，才定位到退化点在解码而非采样。

### 5.6 参数注入通道本身就是最大收益

LocModel GUI 不透传 `txt_cfg`，导致长期以 6.0 运行（官方 recipe 为 1.0）。仅修正这一个字段，画质提升幅度就超过所有量化、调度器、shift 的组合收益，还顺带快 2.2 倍。

---

## 六、最佳方案

### 6.1 首选配置（日常出图，画质与速度平衡最优）

| 项目 | 取值 |
|---|---|
| 模型 | `qwen_image_2.1-Q3_K.gguf`（3.05 GB） |
| VAE | `qwen_image_2.1_vae_bf16.safetensors` |
| LLM / vision | `Qwen3VL-8B-Instruct-Q4_K_M.gguf` + `mmproj-Qwen3VL-8B-Instruct-F16.gguf` |
| 分辨率 | 1344×768（或 1024×1024 基准） |
| 步数 | 20 |
| 采样器 | euler |
| **cfg_scale（txt_cfg）** | **1.0** ← 核心修正 |
| scheduler | `default`（即 D 配方，贴近官方 recipe，高光最准）；需要更强边缘时用 `discrete` |
| flow_shift | D 配方不传；C 配方传 3.0（边缘更锐） |
| distill guidance | 3.5（默认，勿改） |
| eta | 0.0 |
| 负向词 | 「模糊，水彩纸纹，灰雾，低对比，过曝，噪点，涂抹感，失真」 |
| 实测 | 约 **50 s/张**（较原配置 120 s 提速 2.4 倍） |

### 6.2 质量优先（可接受约 2.8 倍耗时）

- 模型换 `qwen_image_2.1-Q6_K.gguf`（5.59 GB），配方同上 → 约 **141 s/张**
- 收益：暗部噪声再降 18%、放大盲评细节最真实；代价：diffusion 部分走磁盘 mmap，显存 5719 MB 接近 8 GB 上限
- 风险：8 GB 卡上余量极小，长批量任务建议先小批试跑

### 6.3 明确不要用的配置

| 配置 | 原因 |
|---|---|
| **cfg_scale = 6.0** | 画质最脏最糊、耗时 2.3 倍（120~163 s），是本次所有问题的首因 |
| **Q4_K** | 相对 Q3_K 无画质收益、慢 44%，处于"两头不占"的最差位置 |
| **distilled_guidance = 0** | 触发 VAE 解码退化，总耗时 726 s（13 倍） |
| 无 backup 直接改 GUI catalog | GUI 无法透传这些字段，改了也不生效 |

### 6.4 落地方式（GUI 不透传时的正确做法）

LocModel 为 Tauri 应用，内置 `C:\Users\<user>\AppData\Local\LocModel\engines\sd\<commit>-cuda\sd-server.exe`。GUI 不传的字段由引擎默认兜底（`flow_shift=auto`、`guidance 3.5`），但 `txt_cfg` 无法在 GUI 内改。两种可行途径：

1. **HTTP 注入（推荐，零风险）**：对 `POST http://127.0.0.1:<port>/sdcpp/v1/img_gen` 的 `sample_params` 逐字段覆盖，端口以 `netstat` 实测为准（本机实测 **8790**，settings.json 记载的 8686 已失效）。
2. **独立实例 + CLI**：直接以 `sd-server.exe` / `sd-cli.exe` 拉起自有实例，参数完全可控，避免污染 GUI 会话。

> 参数注入后建议校验 PNG 内嵌元数据（`parameters` 字段，需 latin1 解码再按 utf-8 还原）确认生效。

---

## 七、遗留与边界

1. 色彩副作用：`cfg=1.0` 相对 6.0 饱和度下降约 50%（0.298→0.149），暗部 p1 由 0.031 抬到 0.087（暗部被提亮）。出图后建议 +15~25 饱和度、轻微压暗部；或在提示词补色彩描述。
2. 原图中"水彩纸纹"未被复现：同参数下只呈现"脏+灰"。说明纸纹依赖具体 Prompt 与 seed 构图，非纯参数决定，需在提示词中显式描述纸张纹理并重试 seed。
3. Q8_0（7.16 GB）未测：8 GB 卡上必然重度卸载，预期远慢于 Q6_K，且按 Q4_K/Q6_K 的趋势也不会带来结构细节提升。
4. 不同 seed 下的稳定性未做批量验证（本次均为单 seed 对照）。

---

## 附：实验组编号对照

| 组 | 模型 | cfg | scheduler / flow_shift | distilled_guidance | 备注 |
|---|---|---|---|---|---|
| A | Q3_K | 6.0 | default / 未传 | 3.5 | 原 GUI 现状基线 |
| B | Q3_K | 6.0 | discrete / 3.0 | 3.5 | 锐度反降 |
| C | Q3_K | 1.0 | discrete / 3.0 | 3.5 | 最锐 |
| D | Q3_K | 1.0 | default / 未传 | 3.5 | 最锐、高光最准 |
| E | Q4_K | 1.0 | discrete / 3.0 | 3.5 | VAE 落 CPU |
| F | Q4_K | 6.0 | default / 未传 | 3.5 | 最脏最糊 |
| G/J | Q4_K/Q3_K | 1.0 | discrete / 3.0 | **0** | 解码退化 726 s，废样 |
| H | Q6_K | 1.0 | discrete / 3.0 | 3.5 | 最干净 |
| I | Q4_K | 1.0 | discrete / 3.0 | 0 | 4 步探针，伪通过 |
| K | Q3_K | 1.0 | discrete / 3.0 | 3.5 | = C，MD5 一致 |
| L | Q4_K | 1.0 | discrete / 3.0 | 3.5 | VAE 强制显存，= E，MD5 一致 |
*（内容由AI生成，仅供参考）*
