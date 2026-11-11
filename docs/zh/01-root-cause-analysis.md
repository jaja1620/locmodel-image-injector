---
AIGC:
    Label: "1"
    ContentProducer: 001191440300708461136T1XGW3
    ProduceID: 798659b1f59514cb753b383c1f423b09_07f7c0a6dcd511f1a0ca525400de85a5
    ReservedCode1: jskqy4+9GQE9qrOsZqBhN1eiWKGI713jN4kTn6hF0rQeaAupYom0YKV750DmP+jHXEi5IHTgzE8wAZczOOuCgrSL2yDOTiADkjXCXNcstm0qEBWFEXbl/PPoNMyyrA9UouWUji2qEaTqyufTg9XoGronObSq0pSq7euE54FuYgy3pEkcN2bvv9JyRds=
    ContentPropagator: 001191440300708461136T1XGW3
    PropagateID: 798659b1f59514cb753b383c1f423b09_07f7c0a6dcd511f1a0ca525400de85a5
    ReservedCode2: jskqy4+9GQE9qrOsZqBhN1eiWKGI713jN4kTn6hF0rQeaAupYom0YKV750DmP+jHXEi5IHTgzE8wAZczOOuCgrSL2yDOTiADkjXCXNcstm0qEBWFEXbl/PPoNMyyrA9UouWUji2qEaTqyufTg9XoGronObSq0pSq7euE54FuYgy3pEkcN2bvv9JyRds=
---

# Qwen-Image 2.1 Q3_K 出图「水彩纸质感」根因分析与优化方案

> 分析对象：LocModel 3.1.0（stable-diffusion.cpp 后端）生成的两张同提示词样图
> 提示词（两者完全相同）：两个甜甜的、清纯的中国美女，穿不同的休闲装，非常自然性感，两个美女望向镜头，充满关心，充分体现女性的温柔和体贴。
> 分析日期：2026-11-10

---

## 一、结论先行

| 疑问 | 结论 |
|---|---|
| 是 LocModel 内置工作流参数有问题吗？ | **是，占主要责任**。Qwen-Image 2.1 这条路径存在两处配置缺项：`flow_shift` 未传、`scheduler` 由 2.0 的 `discrete` 改成了 `flux` |
| 是模型需要特殊提示词吗？ | **是次要原因**。不是"必须特殊咒语"，而是 Qwen-Image 2.1 基座比 Z-Image Turbo 更依赖显式的摄影/材质语汇与负向词；Z-Image Turbo 因蒸馏了审美先验，短提示也能出锐图 |
| 还有什么致命因素？ | **量化档位**。Q3_K（3.44 bpw，占权重 98.09%）远低于官方出样所用的 Q4_K_M；且本机 qwen 系 GGUF 是"均匀量化"，不像 z-image 那样给时间步/调制层保 F32 |
| 「印在水彩纸上」是模型加了滤镜吗？ | **不是**。频谱检测未发现周期性纸纹峰（图1 周期峰强度 156 < 图2 的 414）。这是"暗部被抬起 + 中高频细节塌陷 + 边缘对比变低"三者叠加造成的**视觉错觉** |

---

## 二、硬数据：两张图的客观差异

同一提示词，各取原图（1344×768）逐项测度：

| 指标 | 图1 Qwen-Image 2.1 Q3_K | 图2 Z-Image Turbo Q4_K | 差异 |
|---|---|---|---|
| 拉普拉斯方差（锐度） | 0.00072 | 0.00124 / 0.00197 | **低 42%~63%** |
| 中频能量占比（0.05–0.3 归一化频率） | 0.255% | 0.82% / 1.20% | **低 3~5 倍** |
| 梯度 p99（边缘对比强度） | 0.112 | 0.148 / 0.188 | 低 25%~40% |
| 平均亮度 | 0.718 | 0.483 / 0.604 | 偏高 0.11~0.24 |
| 暗部 1% 分位 | 0.136 | 0.047 / 0.056 | **暗部抬升 2.4~2.9 倍，画面无纯黑** |
| 饱和度和（HSV S 均值） | 0.206 | 0.269 / 0.165 | 相当 |
| 独立颜色数占比 | 9.0% | 10.3% / 10.2% | 略低（色阶偏压） |
| 频谱周期峰（纸纹检测） | 156（弱） | 414 / 261 | 图1 **不是**叠加了纸纹图案 |

**读法**：图1 的问题不是"多了一层纸"，而是"少了三层东西"——
1. 暗部被抬起（p1 从 0.05 抬到 0.136）→ 黑位发灰、画面像蒙了一层白雾；
2. 中高频细节大面积塌陷（发丝、皮肤纹理、布料织纹都消失）→ 只剩柔和的大色块；
3. 边缘对比掉 25%~40% → 轮廓"洇开"，正是水彩在纸上扩散的观感。

三者叠加，就是老板看到的「印在水彩纸上」。

---

## 三、关键证据：三组生成参数的逐字段比对

从 PNG 内嵌 SDCPP 参数中直接读出（同版本 LocModel 的历史记录）：

| 字段 | Qwen-Image 2.0（Q2_K） | **Qwen-Image 2.1（Q3_K）** | Z-Image Turbo（Q4_K） | FLUX.2 klein |
|---|---|---|---|---|
| scheduler | `discrete`（界面显示 Normal） | **`flux`** | `discrete` | `flux2` |
| flow_shift | **`3.0`（显式传入）** | **缺失，未传** | 无 | 无 |
| sample_method | euler | euler | euler | euler |
| sample_steps | 20 | 20 | 8 | — |
| guidance.txt_cfg | 2.5 | 6.0 | 1.0 | 1.0 |
| guidance.distilled_guidance | 3.5 | 3.5（Qwen 也照填） | 3.5 | 3.5 |
| 尺寸 | 1344×768 | 1344×768 | 1344×768 | 1152×896 |

**两个异常点**：
- **① `flow_shift` 在 2.0 上是 3.0，在 2.1 上直接没写。** Qwen-Image 2.0/2.1 都是 flow-matching 架构，shift 决定噪声调度曲线的形态。缺省值与训练分布不匹配时，采样轨迹会偏离，表现为"去噪不足"——残余噪声抬升暗部、细节来不及成形，与第二部分测到的现象完全对应。
- **② `scheduler` 从 `discrete` 变成了 `flux`。** 官方 stable-diffusion.cpp 的 Qwen-Image 2.1 示例命令是 `--steps 20 --cfg-scale 6.0 --sampling-method euler`，**不带 `--flow-shift`、不指定 scheduler**，即走引擎默认调度。LocModel 这里主动套了一层 flux 调度，属于额外引入的偏移。
- 附带提一句：`distilled_guidance: 3.5` 也被一并填给 Qwen（这是 Z-Image / FLUX dev 需要的蒸馏引导值），说明 LocModel 的引导参数面板是全局默认、未按架构区分。

**注意**：`cfg 6.0 / 20 步 / euler` 本身**不是**错设——这正是 leejet / unsloth 官方 README 给 base GGUF 的示例参数。所以问题不在"面板参数填错"，而在"底层调度参数缺项与错映射"。

---

## 四、量化档位：官方样图用的是 Q4_K_M

直接解析本机三个 GGUF 的张量类型构成：

| 模型文件 | 体积 | 张量类型构成 | 敏感层保护 |
|---|---|---|---|
| `qwen_image_2.1-Q3_K.gguf` | 3.05 GB | Q3_K 98.09%（3.44 bpw）+ BF16 1.91% | 仅 norm / txt_in / proj_out 保 BF16；32 层 DiT 全部 Q3_K |
| `qwen_image_2.1-Q4_K.gguf` | 3.91 GB | Q4_K 98.09%（4.5 bpw）+ BF16 1.91% | 同上 |
| `z_image_turbo-Q4_K.gguf` | 3.60 GB | Q4_K 88.16% + **Q8_0 11.63%** + **F32（t_embedder / adaLN_modulation 全保）** | 时间步嵌入与调制层整层 F32 |

两点要害：
1. **Q3_K 是官方明确的下限档**。unsloth / leejet 仓库所有示例图都是 Q4_K_M 去噪器 + Q4_K_M 文本编码器渲染的，并注明"再小就 Q3 起步"。老板这张正好踩在下限上，而对照组 Z-Image 用的是 Q4_K。
2. **z-image 的 GGUF 是"混合精度"的**（关键层上采样到 F32/Q8_0），而 qwen 这两个文件是均匀量化。这就是为什么同体积下 Z-Image 看起来更"实"。

量化误差在 DiT 上的典型视觉症状恰好是：高频纹理化开成柔和块面、皮肤失去微结构、暗部出现灰雾——与实测吻合。

---

## 五、其他加重因素

| 因素 | 现状 | 影响 |
|---|---|---|
| 分辨率 | 1344×768，仅 1.03 MP，比例 1.75:1 | Qwen-Image 2.1 原生 2K 训练（官方预设 2048×2048、2752×1536），1 MP 属于"舒适区下限"；同面积下每边像素少，面部/发丝像素密度不足 |
| 负向词 | **空字符串**，但 cfg=6.0 | cfg 6 时负向分支是生效的，喂空字符串等于让模型对比"什么都没有"，容易抬灰、过曝、细节劣化 |
| 提示词 | 全中文、纯情绪描述（"甜美""温柔""体贴"） | 无任何摄影/材质语汇，Qwen3-VL 编码器得不到"皮肤纹理、发丝、景深、光线类型"这些可落地指令；Z-Image Turbo 蒸馏了审美先验，短提示也能自锐 |
| 文本编码器 | `Qwen3VL-8B-Instruct-Q4_K_M`（均匀量化） | 官方推荐 UD-Q4_K_XL（动态量化，敏感层上采样）；均匀量化会损失语义精度与细节词编码 |

---

## 六、排除项（省得白折腾）

- **不是 VAE 配错**：用的是 `qwen_image_2.1_vae_bf16.safetensors`，与 LocModel 内置模型表一致；2.1 的 VAE 与前代 Qwen-Image、Wan2.2 VAE 不通用，这里没串。
- **不是"Qwen 2.1 本身做不到真实感"**：官方这一代主打就是"人像光影 + 细腻纹理"。
- **不是模型文件损坏**：张量数 265、32 层 DiT、BF16 敏感层齐备，结构完整。
- **不是纸张滤镜/后期**：频谱无纸纹周期峰。

---

## 七、解决方案

### 第 1 档：立刻能做，改动最小（预期提升最大）

| 项 | 从 | 改为 |
|---|---|---|
| 去噪器 | qwen_image_2.1-**Q3_K** | **qwen_image_2.1-Q4_K**（首选，3.91 GB）；显存允许上 Q5_K / Q6_K |
| 尺寸 | 1344×768 | **1024×1024** 或 **1536×864 / 1664×928**（必须是 32 的倍数） |
| 步数 | 20 | **30~40**（官方 pipeline 用 40~50，20 步是速度档） |
| CFG | 6.0 | **4.0**（配合下方负向词；若不想给负向词则保持 6.0） |
| 负向词 | 空 | **模糊、低分辨率、水彩、插画、平涂、过曝、灰雾、塑料感、五官变形、多余手指** |
| sampler | euler | euler（不变） |

### 第 2 档：提示词改造（把"情绪"换成"可执行的物理描述"）

中文可写，但英文命中率更高。建议模板：

```
photorealistic portrait of two Chinese young women, natural casual outfits,
both looking at camera, gentle warm expressions,
85mm lens, f/1.8, shallow depth of field, soft window light from left,
visible skin texture, individual hair strands, subtle film grain,
true-to-life color, Kodak Portra 400, high detail, sharp focus
```

要点：Qwen-Image 2.1 的 Qwen3-VL 编码器对**具体器材、光线方向、材质名词**响应明显好于抽象情绪词。老板原提示词里的"非常自然性感""充分体现温柔体贴"属于情绪指令，模型只能理解成"氛围柔和"——这也正是画面偏柔偏灰的推手之一。

### 第 3 档：调度参数（LocModel 当前界面未暴露，需外部工具验证）

LocModel 的 GUI 未提供 `scheduler` / `flow_shift` 选项（exe 内仅存在字段名，无 discrete/karras 等选项串）。所以第 1 档改完若仍有"灰雾"，就只能：

1. 用 **stable-diffusion.cpp 官方 sd-cli** 按官方命令跑（不指定 `--flow-shift`、不指定 scheduler）：
   ```
   sd-cli --diffusion-model qwen-image-2.1-Q4_K.gguf ^
          --vae qwen_image_2.1_vae_bf16.safetensors ^
          --llm Qwen3-VL-8B-Instruct-UD-Q4_K_XL.gguf ^
          -p "<提示词>" --steps 20 --cfg-scale 6.0 --sampling-method euler ^
          -W 1024 -H 1024 --offload-to-cpu -o out.png
   ```
2. 或用 **ComfyUI + Qwen-Image 2.1 官方模板**（steps 25/40、cfg 1、euler、simple、1 MP），走原生 guidance 路径。
3. 或把"2.1 路径缺少 flow_shift、scheduler 被映射为 flux"作为反馈提给 LocModel 作者。

---

## 八、建议的验证实验（可代跑）

固定同一 seed 与提示词，单变量对照，20 步为基线：

| 组 | 模型 | 尺寸 | 步数 | CFG | 负向词 | 验证目的 |
|---|---|---|---|---|---|---|
| A | Q3_K | 1344×768 | 20 | 6.0 | 空 | 复现基线（=老板图1） |
| B | **Q4_K** | 1344×768 | 20 | 6.0 | 空 | **只改量化** |
| C | Q3_K | **1024×1024** | 20 | 6.0 | 空 | 只改分辨率 |
| D | **Q4_K** | 1024×1024 | **30** | **4.0** | **有** | 综合方案 |

判读规则：
- B 明显变锐 → 主因是量化；
- B 仍发灰、C 略好转 → 主因是调度/shift 缺项（需走第 3 档）；
- D 达标 → 按 D 固化为默认参数。

---

## 附：一手证据索引

- 图1 原始参数：`C:\Users\2013083\Pictures\LocModel\LocModel_1794292797336_1.png`（1344×768，1868.6 KB，Steps 20 / Euler / Schedule: flux / CFG 6.0 / seed 1090474679 / 108.87 s）
- 图2 原始参数：`C:\Users\2013083\Pictures\LocModel\LocModel_1794222470797_1.png`（1344×768，Steps 8 / Euler / Schedule: Normal / CFG 1.0 / seed 6083011）
- Qwen-Image 2.0 历史对照：`LocModel_1793946146079_1.png`（flow_shift 3.0 / scheduler discrete / CFG 2.5）
- 模型文件：`F:\AI-Models\image\qwen-image-2.1\` 与 `F:\AI-Models\image\z-image-turbo\`
*（内容由AI生成，仅供参考）*
