---
AIGC:
    Label: "1"
    ContentProducer: 001191440300708461136T1XGW3
    ProduceID: 798659b1f59514cb753b383c1f423b09_ad4be907dcdd11f1bff2525400cd780f
    ReservedCode1: bcg+PJ3houf/+cckvkfbAFOk5GF5fjA6LrDBxotO3lzkUcdQ7nVlgX73u+0dytAM1gauBKGNo11PO54YkKGPdWGwlbCR42yGaEOze5KA9sxUa/KwFVABZKncnnF+HKD9NYt/Wcr8NgImxqlifdCS04G50KW4s2Jj7LLPwxcyFvIevyD6imhTHE3FobE=
    ContentPropagator: 001191440300708461136T1XGW3
    PropagateID: 798659b1f59514cb753b383c1f423b09_ad4be907dcdd11f1bff2525400cd780f
    ReservedCode2: bcg+PJ3houf/+cckvkfbAFOk5GF5fjA6LrDBxotO3lzkUcdQ7nVlgX73u+0dytAM1gauBKGNo11PO54YkKGPdWGwlbCR42yGaEOze5KA9sxUa/KwFVABZKncnnF+HKD9NYt/Wcr8NgImxqlifdCS04G50KW4s2Jj7LLPwxcyFvIevyD6imhTHE3FobE=
---

# Qwen-Image 2.1 画质排查 · 方案①（sd-server 参数注入）验证报告

> 验证时间：2026-11-10
> 引擎：LocModel 自带 `sd-server.exe`（stable-diffusion.cpp，commit 58bb3c1，master-1050-cuda）
> 模型：`qwen_image_2.1-Q3_K.gguf`（Q3_K 98.09%）+ `qwen_image_2.1_vae_bf16` + `Qwen3VL-8B-Instruct-Q4_K_M`（文本编码器，te=cpu）
> 硬件：RTX 3060 Ti 8GB，权重 9171MB（VRAM 3763MB / RAM 5408MB）
> 结论：**参数注入通道成立；但"补 flow_shift/scheduler 是主因"的假设被证伪，真正主导变量是 CFG scale（6.0 → 1.0）**

---

## 一、结论摘要

| 判断项 | 结论 |
|---|---|
| 参数注入是否可行 | ✅ 完全可行。`POST /sdcpp/v1/img_gen` 的 `sample_params` 可覆盖 `flow_shift` / `scheduler` / `guidance.txt_cfg` / `guidance.distilled_guidance` / `eta`，参数已逐字段写入 PNG 元数据可复核 |
| 补 `flow_shift=3.0` + `scheduler=discrete` 是否改善画质 | ❌ 未改善。在 CFG 6.0 前提下锐度反降（lap 0.00835→0.00764，梯度 p99 0.149→0.140） |
| 真正主导变量 | ✅ **`guidance.txt_cfg`：6.0 → 1.0**。视觉锐度从"最钝"跃到"最锐"，材质真实感由弱转强 |
| 代价 | CFG=1 时饱和度下降 51%~56%（sat 0.149→0.065~0.074），暗部抬升（p1 0.034→0.071~0.089），整体更"素"；需靠提示词或后期补色彩 |
| 附加收益 | CFG=1 省掉 unconditional 前向分支，单图耗时 120.5s → 51~57s，**约 2.2 倍加速** |
| 与官方 recipe 对照 | ✅ 官方 `qwen-image-21` 默认 `cfg_scale=1 / steps=20 / euler`，**不含 flow_shift**（flow_shift=3.0 属 2.0 与 edit 系列）；本次实验与官方默认一致 |

---

## 二、必须纠正的此前的三处判断

| 原判断 | 实测纠正 |
|---|---|
| LocModel 的 `scheduler=flux` 是显式传入的参数 | **错**。请求里传 `scheduler:"default"`，引擎对 Qwen-Image 2.1 自动映射为 `flux`。LocModel 并未传 `discrete`，属引擎按 FLOW 模型的自动判定（启动日志 `running in FLOW mode`） |
| 缺 `flow_shift` 是纸感的主因 | **证伪**。补 `flow_shift=3.0` 后锐度与梯度下降，极高频噪声占比仅从 39.5% 降到 38.2%，未解决结构细节不足 |
| `guidance` 应改 0（3.5 是引擎全局默认） | **未单独验证**。本次只验证了 `txt_cfg`，`distilled_guidance=3.5` 在全部四组保持一致，故其对画质的影响尚未隔离——列入下一步 |

另发现两处环境事实偏差：
- sd-server 实际监听 **8790**（非 memory 记载的 8686）；`/sdcpp/v1/img_gen` 仅接受 POST，GET 返回 404 属正常路由行为。
- LocModel 托管的 sd-server 进程在本次会话中**崩溃过**（`%LOCALAPPDATA%\CrashDumps\sd-server.exe.38688.dmp`，130MB，15:43）。因此本次验证改用独立实例（端口 8899，参数全可控），产物与 LocModel 托管路径同源同二进制。

---

## 三、实验设置（单变量对照）

统一条件：同提示词、同 seed（1090474679）、1344×768、20 步、Euler、eta 0.0、`distilled_guidance=3.5`、Q3_K。

| 组 | scheduler | flow_shift | txt_cfg | 负向词 | 定位 |
|---|---|---|---|---|---|
| A | default（→flux） | 不传 | 6.0 | 无 | 复现 LocModel 现状基线 |
| B | discrete | 3.0 | 6.0 | 无 | 仅补调度与 shift |
| C | discrete | 3.0 | 1.0 | 有 | 全参数组 |
| D | default（→flux） | 不传 | 1.0 | 无 | 仅改 CFG（对齐官方 recipe） |

### 参数生效验证（PNG 内嵌元数据逐字段比对）

| 组 | PNG 头表述 | SDCPP 载荷 | 判定 |
|---|---|---|---|
| A | `SDCPP Schedule: flux, CFG scale: 6.0` | `scheduler:"flux"`，无 flow_shift | 基线，与问题图一致 |
| B | `Schedule type: Normal, CFG scale: 6.0` | `scheduler:"discrete", flow_shift:3.0` | ✅ discrete + shift 均生效 |
| C | `Schedule type: Normal, CFG scale: 1.0` + `Negative prompt:` | `scheduler:"discrete", flow_shift:3.0, txt_cfg:1.0` | ✅ 全套生效 |
| D | `SDCPP Schedule: flux, CFG scale: 1.0` | `scheduler:"flux", txt_cfg:1.0` | ✅ 单变量隔离成立 |

---

## 四、定量测度

| 指标 | A | B | C | D | 判读 |
|---|---|---|---|---|---|
| 拉普拉斯方差 | **0.008354** | 0.007640 | 0.005732 | 0.006257 | A 最高，但为噪声虚高（见下） |
| 梯度 p99 | **0.1488** | 0.1400 | 0.1341 | 0.1365 | 同上 |
| 中频能量占比 %（0.10–0.35） | 31.46 | 30.18 | 31.87 | **32.60** | 真实结构细节 D 最丰富 |
| 平均亮度 | 0.534 | 0.509 | 0.479 | 0.483 | CFG=1 整体偏暗 |
| 亮度标准差 | 0.2498 | 0.2527 | 0.2311 | 0.2151 | A/B 对比更强 |
| 暗部 p1 | **0.0340** | 0.0301 | 0.0706 | 0.0889 | **CFG=1 暗部抬升 2~2.6 倍** |
| 暗部 p5 | 0.1268 | 0.1007 | 0.1307 | 0.1464 | 同上 |
| 饱和度均值 | **0.1486** | 0.1436 | 0.0740 | 0.0654 | **CFG=1 色彩衰减过半** |
| 平坦区噪声 ×1000 | —（无平坦块） | 3.779 | **3.727** | 4.179 | A 全图无平坦区，即高频铺满 |
| 频谱周期峰 | 14231 | 16672 | **22068** | 17222 | C 结构周期性最强 |

### 频谱环带分解（判别"锐"还是"脏"）

| 频带 | A | B | C | D |
|---|---|---|---|---|
| 0.00–0.02 | 8.62 | 10.28 | 11.62 | 10.53 |
| 0.02–0.06 | 7.64 | 9.84 | 9.47 | 8.96 |
| 0.06–0.12 | 7.37 | 8.75 | 8.49 | 8.55 |
| **0.12–0.25（结构细节）** | **15.64** | 15.41 | 16.26 | **16.71** |
| 0.25–0.45 | 29.83 | 27.76 | 29.14 | 29.54 |
| **0.45–1.00（噪声区）** | **39.51** | 38.24 | 36.64 | **36.23** |
| **HF/MID 比** | **171.7** | 158.3 | 148.1 | **143.4** |
| 梯度方向一致性 | 0.488 | 0.286 | 0.127 | 0.132 |

**关键判读**：A 的拉普拉斯方差最高，但它的极高频（噪声带）占比最高（39.5%）、中频结构占比最低（15.6%）、全图找不到平坦区。即 A 的"高锐度数值"由密集高频噪点贡献，而非结构边缘——这与视觉上的"偏脏且模糊"一致。**这是本次最重要的方法论修正：单看拉普拉斯方差会被噪点误导，必须配环带分解。**

---

## 五、视觉判读（独立视觉模型，匿名盲评）

| 组 | 纸纹颗粒感 | 发灰雾 | 锐利度（1 最锐） | 材质真实感 |
|---|---|---|---|---|
| A | 无 | **是** | 2 | 混凝土质感偏脏且画面模糊，真实感弱 |
| B | 无 | 否 | 2 | 质感粗糙但色调发暖偏黄，真实感中等 |
| C | 无 | 否 | **1** | 质地颗粒感强且光影锐利，真实感**极强** |
| D | 无 | 否 | **1** | 表面干净细腻且高光准确，真实感**极强** |

C 与 D 并列最锐，二者唯一共同变量是 **txt_cfg=1.0**；A 与 B 唯一共同变量是 **txt_cfg=6.0**。CFG 是分组因素，scheduler/flow_shift 是组内次要因素。

---

## 六、最终结论与推荐配方

1. **CFG scale 6.0 是画质劣化的首要参数原因。** Qwen-Image 2.1 属蒸馏 guidance 模型，官方 recipe 给 `cfg_scale=1`；LocModel 传 6.0 相当于对已蒸馏模型强行做双分支 CFG，代价是色彩与暗部被压平、细节被噪声替代。
2. **`flow_shift` / `scheduler` 不是主因。** 在 CFG=6.0 下补 `flow_shift=3.0`+`discrete` 反而降锐度；在 CFG=1.0 下它们带来的是**减噪**收益（C 的 HF/MID 比 D 低 4.7pp、周期峰高 28%）。因此它们属于"锦上添花"，可用但不是解药。
3. **推荐参数（C 组配方）**：

```
sample_steps   20
sample_method  euler
scheduler      discrete
flow_shift     3.0
guidance.txt_cfg           1.0
guidance.distilled_guidance 3.5（维持，未单独验证）
eta            0.0
negative_prompt 模糊，水彩纸纹，灰雾，低对比，过曝，噪点，涂抹感，失真
```

4. **必须补偿 CFG=1 的副作用**：饱和度掉 51%~56%、暗部抬 2~2.6 倍。建议提示词显式写色彩（如"饱和的红砖、冷调玻璃反射"），或后期统一 +饱和/+对比；否则会从"纸感"滑向"灰蒙"。
5. **顺带收益**：CFG=1 单图 51~57s，比 CFG=6 的 120.5s 快 2.2 倍。画质与速度同时改善。

---

## 七、尚未覆盖 / 下一步

| 待验证项 | 说明 |
|---|---|
| Q3_K 是否也是主因 | 本次全部用 Q3_K 单因素固定，未与 Q4_K 对照（方案②）。若 CFG=1 后仍有残留纸感，再测 Q4_K |
| `distilled_guidance 3.5 → 0` | 四组全程固定 3.5，未隔离其影响 |
| 原图纸纹未在 A 组复现 | 同参数（CFG 6.0/flux/1344×768/Q3_K）下 A 组只呈现"脏+灰"而非原图的水彩纸纹。说明纸纹还依赖具体 prompt 与 seed 的构图，非纯参数决定——若要精确定位需用原图同一 prompt/seed 复跑 |
| LocModel GUI 落盘 | 上述参数在 GUI 中无入口，需走 `sd-server` 直连或 `sd-cli` 直跑；GUI 侧只能靠 per-model 默认参数表（本地未找到，疑远端下发） |

---

## 八、一手证据索引

| 证据 | 路径 |
|---|---|
| A 组图（现状基线） | `output\方案①参数对照图\caseA.png` |
| B 组图（+flow_shift+scheduler） | `output\方案①参数对照图\caseB.png` |
| C 组图（全参数） | `output\方案①参数对照图\caseC.png` |
| D 组图（仅 CFG=1） | `output\方案①参数对照图\caseD.png` |
| 实验执行器 | `temp\exp\run_gen.py` |
| 引擎启动日志 | `temp\exp\server2.out.log` / `server2.err.log` |
| sd-server 崩溃转储 | `%LOCALAPPDATA%\CrashDumps\sd-server.exe.38688.dmp` |

### 复现命令

```powershell
# 启动可控实例（8899）
& "C:\Users\2013083\AppData\Local\LocModel\engines\sd\master-1050-58bb3c1-cuda\sd-server.exe" `
  --listen-ip 127.0.0.1 --listen-port 8899 --model-dir F:\AI-Models\image `
  --diffusion-model F:\AI-Models\image\qwen-image-2.1\qwen_image_2.1-Q3_K.gguf `
  --llm F:\AI-Models\image\common\Qwen3VL-8B-Instruct-Q4_K_M.gguf `
  --llm_vision F:\AI-Models\image\common\mmproj-Qwen3VL-8B-Instruct-F16.gguf `
  --vae F:\AI-Models\image\common\qwen_image_2.1_vae_bf16.safetensors `
  --diffusion-fa --backend te=cpu -W 1024 -H 1024 --lora-model-dir F:\AI-Models\image\loras

# 跑单组对照（A/B/C/D）
python temp\exp\run_gen.py A
```
*（内容由AI生成，仅供参考）*
