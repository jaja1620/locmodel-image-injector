---
AIGC:
    Label: "1"
    ContentProducer: 001191440300708461136T1XGW3
    ProduceID: 798659b1f59514cb753b383c1f423b09_7abedeaddce911f1bff2525400cd780f
    ReservedCode1: EYcHeYivjFhmCdvDDVivmQxAc9u9hWE8kZnrlKB0yAysetPYsB5dUAI+eod1BdAtsn5tAffGLUpnlUaSIagicAfR2ri16UrnCd5AnBXJnp4PQ6tuDHK/ZHfFWKqueWkhylrkzrqwKW0u8zvujvID6I02iuzWf8Xg3cF0isYU1VggnCTxYAh0EKJj5uU=
    ContentPropagator: 001191440300708461136T1XGW3
    PropagateID: 798659b1f59514cb753b383c1f423b09_7abedeaddce911f1bff2525400cd780f
    ReservedCode2: EYcHeYivjFhmCdvDDVivmQxAc9u9hWE8kZnrlKB0yAysetPYsB5dUAI+eod1BdAtsn5tAffGLUpnlUaSIagicAfR2ri16UrnCd5AnBXJnp4PQ6tuDHK/ZHfFWKqueWkhylrkzrqwKW0u8zvujvID6I02iuzWf8Xg3cF0isYU1VggnCTxYAh0EKJj5uU=
---

# Qwen-Image 2.1 通用参数注入器 —— 需求分析与方案建议

> 分析日期：2026-11-10
> 分析对象：stable-diffusion.cpp `master-1050-58bb3c1` / LocModel 内置 sd-server
> 结论先说：**可行，且实现成本远低于预期——单个 HTML 文件即可闭环，不需要后端、不需要打包、不需要改 LocModel**

---

## 一、可行性结论（已实测验证，非推断）

| 验证项 | 实测结果 | 对方案的意义 |
|---|---|---|
| 是否提供参数化 HTTP 接口 | `POST /sdcpp/v1/img_gen`，请求体即完整参数字典 | 注入点存在，无需改引擎 |
| 是否支持跨域（CORS） | `Access-Control-Allow-Origin` 回显任意来源，`Allow-Methods: *`，`Allow-Headers: *`，OPTIONS 预检返回 204 | **纯静态 HTML 可直连引擎**，这是方案成立的关键前提 |
| 是否提供参数契约 | `GET /sdcpp/v1/capabilities` 返回含全部字段的 `defaults` 模板 | GUI 可自动生成，不用硬编码字段名 |
| 是否提供模型规格库 | `GET /sdcpp/v1/recipes` 返回 158 KB、约 50 条 recipe（含 `qwen-image-21`） | 可按模型动态裁剪参数面板，实现"通用" |
| 是否可局域网访问 | `-l/--listen-ip`（默认 127.0.0.1）可设 0.0.0.0 | 手机/平板浏览器可当遥控器 |
| 出图是否确定性 | 同参数同 seed 重跑 MD5 一致（已验证） | 预设可精确复现，注入器可信 |
| 出图是否内嵌参数 | 存在 `--disable-image-metadata`，说明默认内嵌 SDCPP 参数 | 可从历史图反查参数、一键回填 |

**结论**：注入器不需要拦截、不需要 Hook、不需要代理 LocModel 的进程间通信——引擎本身就是标准 HTTP 服务，直接对话即可。

---

## 二、核心洞察：为什么这个工具真正有价值

### 2.1 三层默认值存在系统性错位（这是画质问题的制度性根因）

`sd-server --help` 显示：`--cfg-scale <float> (default: 7.0)`

但 `GET /sdcpp/v1/recipes` 中 `qwen-image-21` 的定义是：

```json
"defaults": {"cfg_scale": 1, "height": 1024, "sampling_method": "euler", "steps": 20, "width": 1024}
```

即：

| 层级 | 来源 | txt_cfg 实际取值 | 性质 |
|---|---|---|---|
| L1 引擎二进制默认 | 命令行未指定时的兜底 | **7.0** | 通用默认，对 2.1 是错的 |
| L2 模型 recipe 默认 | `recipes[] → qwen-image-21 → defaults` | **1.0** | 官方正确值 |
| L3 LocModel GUI 实际下发 | 实测历史出图 | **6.0** | 两边都没对齐 |

**关键点**：`runtime_recipe.id` 在命令行加载时是 `@command-line`（"all features"，全宽松模式），此时**没有任何 recipe 约束来纠正 7.0**。LocModel GUI 若直接读 `capabilities.defaults.sample_params.guidance.txt_cfg`（=7.0），或自行写死 6.0，就会长期以错误 CFG 出图——这正是此前诊断中"最脏最糊、耗时 2.3 倍"的来源。

**因此注入器的第一价值不是"能改参数"，而是"始终按 recipe 下发正确默认值"。** 界面上的"恢复默认"必须明确指向 L2（recipe 默认），而不是 L1（引擎默认），否则会一键恢复成错误配置。

### 2.2 recipe 默认值 ≠ 请求体字段名（存在命名错位，必须做映射层）

| recipe `defaults` 中的键 | 请求体 `sample_params` 中的键 |
|---|---|
| `cfg_scale` | `guidance.txt_cfg` |
| `sampling_method` | `sample_method` |
| `steps` | `sample_steps` |
| `scheduler` | `scheduler`（同名） |
| `flow_shift` | `flow_shift`（同名） |
| `custom_sigmas` | `custom_sigmas`（同名） |

这层错位很可能就是 LocModel GUI 放弃 recipe 默认值、转而使用全局默认的技术原因。注入器必须内置这张映射表。

### 2.3 "默认"与"推荐"是两件事，界面上必须分开

以本次实验结论为例：

| 参数 | recipe 默认（L2） | 本项目推荐（L3，建筑效果图取向） |
|---|---|---|
| txt_cfg | 1.0 | 1.0（一致） |
| scheduler | 未指定 → auto | `default`（高光最准）；需更强边缘时 `discrete` + flow_shift 3.0 |
| 负向词 | 空 | 「模糊，水彩纸纹，灰雾，低对比，过曝，噪点，涂抹感，失真」 |
| 量化档 | — | Q3_K（Q4_K 慢 44% 无收益，Q6_K 慢 141 s） |

所以界面需要两个按钮：**「恢复官方默认」** 与 **「恢复我的推荐」**，语义不同，不能合并。

---

## 三、功能需求清单

### P0（无此不可用）

| 编号 | 功能 | 说明 |
|---|---|---|
| F1 | 引擎自动发现 | 依次探测 `127.0.0.1:8790`（LocModel 托管实例）、`8899`（实验实例）、用户自定义端口；显示每个端口的 `state`、已加载模型、`runtime_recipe.id` |
| F2 | 参数契约自动加载 | 拉取 `/capabilities` 得到完整字段树与默认值，拉取 `/recipes` 得到 recipe 规格 |
| F3 | 推荐值 / 自定义值双输入 | 每个参数一行：左侧下拉/滑杆给推荐值，右侧可直接键入精确数值；两者双向同步 |
| F4 | 恢复默认（分层） | ①恢复官方 recipe 默认；②恢复内置推荐预设；③全部清空重置 |
| F5 | 一键注入并生成 | 组装请求 → `POST /img_gen` → 轮询 `GET /jobs/<id>` → 展示进度与结果图 → 支持下载 |
| F6 | 参数校验与自动纠偏 | 尺寸按 `size_multiple` 对齐（命令行为 8，recipe 为 32）；提示非法枚举值；提示被 recipe 标记为不支持的参数 |
| F7 | 预设保存/加载 | 存 localStorage，支持导出/导入 JSON，多套预设命名管理 |

### P1（显著提升实用性）

| 编号 | 功能 | 说明 |
|---|---|---|
| F8 | 出图回读参数 | 上传/拖入已有 PNG，解析内嵌 SDCPP 参数（latin1 解码后转 utf-8），一键回填到面板 |
| F9 | 参数变更影响提示 | 内置经验规则：改 txt_cfg 提示耗时与饱和度变化、改 flow_shift 提示与 scheduler 的联动、量化档不在运行时参数内需换模型文件 |
| F10 | 生成历史与对比 | 保留本次会话出图缩略图 + 参数快照，支持 A/B 并排对比 |
| F11 | 时长预测 | 基于已记录的历史耗时与参数估算预计耗时，超时预警 |
| F12 | 提示词模板库 | 建筑类常用提示词片段库，可组合插入 |

### P2（可选扩展）

| 编号 | 功能 | 说明 |
|---|---|---|
| F13 | 局域网模式 | 引擎以 `--listen-ip 0.0.0.0` 启动后，手机/平板浏览器打开同一 HTML 即可出图 |
| F14 | LoRA / 参考图 / 图生图 | 契约中已有 `lora`、`ref_images`、`img2img`、`mask_image` 字段，可后续展开 |
| F15 | Hires 二次放大 | 契约已有 `hires` 子树 |
| F16 | 批量网格出图 | `batch_count` + 多 seed 网格，用于方案比选 |

---

## 四、界面结构草案

```
┌─────────────────────────────────────────────────────────────┐
│ [●] 引擎 127.0.0.1:8790  ready  Qwen-Image 2.1  recipe:—    │ ← 状态条
│     端口[8790 ▾] [重新探测]  [QLoRA/模型信息]                │
├──────────────────────────┬──────────────────────────────────┤
│ 左：参数面板（60%）      │ 右：出图区（40%）                │
│                          │                                  │
│ ▸ 预设  [建筑效果图 ▾]   │  ┌────────────────────────┐      │
│   [恢复官方默认]         │  │                        │      │
│   [恢复我的推荐]         │  │      进度/结果图        │      │
│   [全部重置]             │  │                        │      │
│                          │  └────────────────────────┘      │
│ ▸ 基础                   │  [下载] [回填参数] [加入对比]    │
│   正向提示词 [____]      │                                  │
│   负向提示词 [____]      │  ── 生成历史 ──                  │
│   尺寸 1344 × 768        │  ▫ ▫ ▫ ▫                         │
│   批量 1   种子 [随机□]  │                                  │
│                          │                                  │
│ ▸ 采样（核心）           │                                  │
│   txt_cfg   [1.0] ⚠推荐  │                                  │
│   dg        [3.5]        │                                  │
│   步数      [20]         │                                  │
│   采样器    [euler ▾]    │                                  │
│   调度器    [default ▾]  │                                  │
│   flow_shift [auto □]    │                                  │
│   eta       [0.0]        │                                  │
│                          │                                  │
│ ▸ 高级 ▸ 折叠            │                                  │
│   slg / VAE tiling /     │                                  │
│   qwen_layers / 输出格式 │                                  │
├──────────────────────────┴──────────────────────────────────┤
│              [ 一键注入并生成 ]   预计 ~50 s                │ ← 主按钮
└─────────────────────────────────────────────────────────────┘
```

交互要点：
- 每个参数行右侧的小灰字显示 recipe 默认值与当前值的关系（一致/偏离），偏离时高亮
- `txt_cfg` 这类"引擎默认错、recipe 正确"的字段，加固定警示图标
- 主按钮在执行中变为「取消（cancel_generating）」
- 开机自动探测 + 失败时给出明确的排查文案（引擎未启动 / 端口不对 / 模型未加载）

---

## 五、架构选型对比

| 方案 | 三端通用 | 依赖 | 开发量 | 局域网手机访问 | 结论 |
|---|---|---|---|---|---|
| **A. 单文件 HTML（原生 JS）** | Windows / Ubuntu / 安卓浏览器全通 | 无 | 小 | 天然支持 | **推荐** |
| B. 本地 Python 服务 + 前端 | 需装 Python | 有 | 中 | 需另配 | 与"免装 Python"目标冲突 |
| C. PyQt/Tkinter 桌面程序 | 仅桌面，需打包 | 需分发 exe/apk | 大 | 不支持 | 过重 |
| D. 代理 LocModel 端口劫持 | 可拦截 GUI 请求 | 需改 settings.json | 中 | — | 脆弱，端口/版本一变即失效 |

选 A 的决定性依据：**实测 CORS 完全开放**。既然浏览器能直接跨域调 8790，就不需要任何中间层。这同时满足"Windows / Ubuntu / 安卓三端分发、免装 Python"的既有约束——同一个 HTML 文件，桌面双击打开是工具，平板浏览器打开是遥控器。

---

## 六、数据模型

```json
{
  "schema": "img2.1-injector/1",
  "presets": [
    {
      "name": "建筑效果图-标准",
      "tags": ["qwen-image-21", "建筑", "高保真"],
      "recipe_hint": "qwen-image-21",
      "params": {
        "width": 1344, "height": 768,
        "sample_params": {
          "sample_method": "euler",
          "scheduler": "default",
          "sample_steps": 20,
          "flow_shift": null,
          "eta": 0.0,
          "guidance": {"txt_cfg": 1.0, "distilled_guidance": 3.5, "img_cfg": null}
        },
        "negative_prompt": "模糊，水彩纸纹，灰雾，低对比，过曝，噪点，涂抹感，失真"
      },
      "notes": "cfg=1.0 相对 6.0 饱和度降约 50%（0.298→0.149），暗部 p1 抬至 0.087",
      "measured_seconds": 50
    }
  ]
}
```

预设字段设计要留 `recipe_hint`：加载时若与当前引擎模型不匹配，弹出提示但不阻断（通用性优先）。

---

## 七、风险与规避

| 风险 | 影响 | 规避 |
|---|---|---|
| 端口漂移（8686 / 8790 / 8899） | 连不上 | 多端口并发探测 + 手动输入 + 记住上次成功端口 |
| LocModel 重启后端口变化 | 断连 | 每次生成前轻量 ping `state`，失败自动重探 |
| 引擎未加载模型（state≠ready） | 请求被拒 | 状态条红色阻断，提示"请先在 LocModel 加载模型"或给出启动命令 |
| 命令行实例无 recipe 约束（`@command-line`） | 非法参数不报错、静默出错图 | 前端按 recipe 表自行校验，越界时警告 |
| 尺寸非 size_multiple 倍数 | 引擎报错或画质异常 | 输入即对齐并提示（32 或 8 视 recipe） |
| 大图 base64 内存占用（2048² PNG ≈ 15 MB → b64 20 MB） | 浏览器卡顿 | 用 Blob URL 而非 data URI，预览降采样 |
| 引擎串行队列，无并发 | 连续点击堆积 | 按钮锁 + 队列状态显示 |
| **监听 0.0.0.0 后无鉴权** | 局域网任意设备可调用引擎、消耗算力 | 默认仅 127.0.0.1；开启局域网模式时明确提示风险，并建议只在受信网络使用 |
| 参数写错导致耗时暴涨（如 dg=0 实测 726 s） | 卡死等待 | 危险组合前置警告 + 预计耗时显示 |

---

## 八、实施计划

| 阶段 | 内容 | 交付 |
|---|---|---|
| S1 | 引擎探测 + 契约加载 + 参数面板自动生成 + 一键生成 | 可用的 v0.1 单文件 |
| S2 | 预设库（推荐/官方默认分层）+ 保存导出 + 校验纠偏 | v0.5 |
| S3 | 出图回读参数、历史对比、提示词库 | v1.0 |
| S4 | 局域网手机端适配（响应式 + 触控优化） | v1.2 |

S1 预计一次性可完成，S2/S3 为增量迭代。

---

## 九、待确认决策点

1. **是否需要手机/平板访问？** 需要则引擎须以 `--listen-ip 0.0.0.0` 启动，且需接受无鉴权的暴露风险（仅受信网络）。
2. **是否需要与 LocModel GUI 联动？** 若不联动，注入器是独立出图通道，LocModel 的历史图库不会收录；若需收录，可把输出目录指向 LocModel 的图片目录（此改动涉及写入其配置，需单独确认）。
3. **参数面板默认展开范围？** 建议只展开"基础 + 采样"两组，其余折叠，避免信息过载。
4. **是否内置量化档切换？** 量化档属于模型加载参数（换 GGUF 文件），不在 `sample_params` 内，注入器只能提示、不能运行时切换。需确认是否接受"仅提示"。
5. **是否要多套模型通用？** 若要覆盖 Z-Image / Anima 等，面板需完全由 recipe 驱动而非硬编码；这会增加约 20% 工作量，但换来真正的"通用"。

---

## 十、一句话建议

先做 **A 方案的单文件 HTML、v0.1 只覆盖 Qwen-Image 2.1 的核心采样参数**，跑通"探测→加载契约→双输入→注入→出图"闭环；确认好用后再补预设库与手机端。核心设计上必须坚持一点：**"恢复默认"指向 recipe 默认（txt_cfg=1.0），绝不指向引擎默认（7.0）**，否则这个工具会帮倒忙。
*（内容由AI生成，仅供参考）*
