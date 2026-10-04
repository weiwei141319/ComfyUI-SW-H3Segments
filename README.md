# ComfyUI-SW-H3Segments

[English](README_EN.md) | 简体中文

海螺 H3（MiniMax H3）长视频节点包：**一条提示词分多段生成，在 latent 域续接，最后一次 VAE 解码出片**。

H3 单段上限约 7 秒，本包把它扩展到 **15 ~ 35 秒**，并且成片**自带声音**（H3 是 audio + video 联合生成，语音由模型从提示词自动产出，无需后期对轨）。

---

## 目录

- [两个模块，该用哪个](#两个模块该用哪个)
- [安装](#安装)
- [快速开始](#快速开始)
- [节点一：SW_H3MultiPrompt（一体机）](#节点一sw_h3multiprompt一体机)
  - [输入参数全表](#输入参数全表)
  - [参考输入：图 / 视频 / 音频](#参考输入图--视频--音频)
  - [输出](#输出)
  - [提示词里的引用语法](#提示词里的引用语法)
  - [报告怎么读](#报告怎么读)
- [调参指南（重点）](#调参指南重点)
- [常见问题排查](#常见问题排查)
- [节点二：分段三节点（手工搭循环用）](#节点二分段三节点手工搭循环用)
- [原理：帧网格与 latent 续接](#原理帧网格与-latent-续接)
- [已知约束](#已知约束)

---

## 两个模块，该用哪个

| 模块 | 节点 | 适用场景 |
|---|---|---|
| **一体机**（推荐） | `SW_H3MultiPrompt` | 多段**不同提示词**。一个节点搞定：填几段提示词就跑几段，**零循环节点**。 |
| **分段三节点** | `SW_H3_SegPlan` / `SW_H3_SegBridge` / `SW_H3_SegConcat` | 需要用官方 `StartLoop` / `EndLoop` 手工搭循环、或想在循环内插入自定义节点的场景。 |

> 绝大多数人直接用**一体机**。分段三节点是底层积木，保留给需要精细控制的场景。

---

## 安装

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/weiwei141319/ComfyUI-SW-H3Segments.git
```

重启 ComfyUI 即可，无 pip 依赖、无需额外模型。

**环境要求**

- ComfyUI 需支持 `comfy_api.latest`（`io.ComfyNode` / `io.Autogrow.Input`）——较新版本
- 模型：H3 UNET + Qwen3-VL CLIP + H3 video VAE + H3 audio VAE
- 可选：`ComfyUI-SW-Sampler`（提供 `SW_loadvideo` 等 SW 系列配套节点）

节点目录：`SW/H3Segments`

---

## 快速开始

最小接线（成片带声音）：

```
UNETLoader(H3)  ────────────────► model
CLIPLoader(Qwen3-VL) ───────────► clip
VAELoader(H3 video VAE) ────────► vae
VAELoader(H3 audio VAE) ────────► audio_vae      ← ★ 接上才有声音
LoadImage ×N ───────────────────► ref_images

SW_H3MultiPrompt.video ──► CreateVideo.images
SW_H3MultiPrompt.audio ──► CreateVideo.audio
CreateVideo ──► SaveVideo / PreviewImage
```

1. 在 `prompt_1` ~ `prompt_5` 里填提示词（**填几个跑几段**）
2. 参考图连到 `ref_images`（会自动长出 `ref_image_0`、`ref_image_1`…）
3. 提示词里用 `<Picture 1>`、`<Picture 2>` 引用参考图
4. 跑

---

## 节点一：SW_H3MultiPrompt（一体机）

**一个节点干完**：算帧数 → 逐段编译条件（**参考图只编码一次**）→ 卸载 CLIP 省显存 → 逐段采样并在 latent 域续接 → latent 拼接 → **一次性 VAE 解码**。

### 段数怎么定

节点输入在图加载时就固定，节点内部拿不到「哪个口连了线」，所以判定规则是「**哪个提示词口填了非空文本**」：

| 情况 | 段数 |
|---|---|
| `prompt_1` 非空 | 1 段 |
| `prompt_1..4` 非空 | 4 段 |
| 遇到第一个空口就停 | 后面填了也不算（报告里会警告） |
| `cond_N` 已连外部 CONDITIONING | 同样算一段，且**优先于** `prompt_N` |

`segment_count` 可手动锁定（`"1"`~`"5"`），锁定时忽略空口。

> ⚠️ **auto 数出 < 3 段会直接报错**。本节点面向 15 秒以上成片，2 段（14.4 s）达不到。
> 只想试效果 → 把 `segment_count` 手动设成 `"1"` 或 `"2"`。

### 输入参数全表

#### 模型与条件

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `model` | MODEL | — | H3 权重（可接 LoRA 之后的 model） |
| `clip` | CLIP | — | Qwen3-VL 文本编码器。**节点编完条件会自动卸载它**（约 16.5 GB） |
| `vae` | VAE | — | H3 **video** VAE，最后一次性解码 |
| `audio_vae` | VAE | — | H3 **audio** VAE。★ **不接则成片静音** |
| `cond_1..5` | CONDITIONING | — | 可选。外接条件时连这里，连了就跳过 `prompt_N` 的内部编码 |
| `prompt_1..5` | STRING | — | 段提示词。**`prompt_1` 必填** |

#### 参考输入

| 参数 | 上限 | 说明 |
|---|---|---|
| `ref_images` | **9 张** | 提示词用 `<Picture 1>` ~ `<Picture 9>` |
| `ref_videos` | **3 个** | 帧序列（IMAGE），提示词用 `<Video 1>` ~ `<Video 3>` |
| `ref_video_audios` | **3 条** | 与 `ref_video_N` 同号的配套音轨，需 `audio_vae` |
| `ref_audios` | **3 条** | 独立参考音频，提示词用 `<Audio 1>` ~ `<Audio 3>`，需 `audio_vae` |

> 这些都是 **Autogrow 动态槽位**：界面上往 `ref_image_3`、`ref_image_4`… 一路连，槽位自动长出来，插件按连了几个吃几个，**改张数不用改插件**。
>
> ⚠️ **参考视频与独立参考音频建议二选一**：参考视频自带音轨（`ref_video_audios`），再叠独立 `ref_audios` 就是两条音频 latent 同时喂进 DiT，音色会打架。

#### 画布与时长

| 参数 | 默认 | 说明 |
|---|---|---|
| `width` / `height` | 1344 / 768 | 画布尺寸（32 的倍数）。竖版填 `768 × 1344` |
| `segment_seconds` | 7.0 | 每段目标秒数，自动吸附到 `5+17n` 网格（7 s → 175 帧 = 7.29 s） |
| `anchor_frames` | `"5"` | 段间锚定帧数。**保持 5**：保证拼接后总帧数仍落在网格上 |
| `裁剪到秒数` | 0.0 | **0 = 不裁**。填具体秒数见下方说明 |

**`裁剪到秒数` 是「上限」语义**（不会把短片拉长）：

- 优先让**最后一段只生成「补足剩余时长」的长度**，而不是先满段再砍尾 —— 这样末段对白完整保留（修复「最后一段语音说不全」）
- 视频与音频**同步裁剪**，不会音视频错位
- 若前 N−1 段已超出目标（如 5 段硬塞 26 秒），回退「砍尾」并在报告里警告

#### 采样

| 参数 | 默认 | 说明 |
|---|---|---|
| `seed` | 0 | 起始种子 |
| `递增种子` | `True` | 第 2 段 `seed+1`、第 3 段 `seed+2`… 段间噪声不同，过渡更自然。**推荐保持 `True`** |
| `steps` | 20 | 采样步数。★ 见[调参指南](#调参指南重点) |
| `cfg` | 1.0 | 官方推荐（不做 CFG）。**不是**泛紫根因，见[调参指南](#调参指南重点) |
| `sampler_name` / `scheduler` | `euler` / **`beta`** | ★ `scheduler` 必须用 `beta`，用 `simple` 会画面泛紫 |
| `denoise` | 1.0 | 从噪声全采样 |

#### 条件细节

| 参数 | 默认 | 说明 |
|---|---|---|
| `ref_image_size` | `match` | `match` = 按成片面积等比缩放参考图（快，推荐）；`max` = 2048 短边（一致性最好但慢很多） |
| `visual_strength` | 0.999 | 参考图 latent 保真度。参考图颜色正确时建议 `1.0` |
| `audio_strength` | 1.0 | 参考音频 latent 保真度 |
| `内置sigma_shift` | `True` | 内部打 sigma shift（等价官方 `MiniMaxH3SigmaShift`）。★ **外面已接该节点时这里要关掉** |
| `shift_video` / `shift_audio` | 12.0 / 3.0 | 官方默认值，一般不动 |
| `内部解码` | `True` | `False` = 只出 LATENT，自己接官方 `VAE Decode` |

### 输出

节点有 **8 个输出**：

| # | 名称 | 类型 | 说明 |
|---|---|---|---|
| 1 | `video` | IMAGE | 成片帧序列，直接接 `CreateVideo` |
| 2 | `latent` | LATENT | 合并后的完整 latent（可二次加工） |
| 3 | `段数` | INT | 实际跑了几个段 |
| 4 | `成片帧数` | INT | |
| 5 | `成片秒数` | FLOAT | |
| 6 | `每段帧数` | INT | |
| 7 | `报告` | STRING | 详细运行报告，接 `ShowText` / 看终端 |
| 8 | `audio` | AUDIO | 成片音轨，接 `CreateVideo.audio` |

### 提示词里的引用语法

参考资产在提示词里用标签引用，**编号严格按接口顺序**（不是文件名顺序）：

```
<Picture 1>   第一个 ref_image（ref_image_0）
<Picture 2>   第二个 ref_image（ref_image_1）
...
<Video 1>     第一个 ref_video
<Audio 1>     第一条 ref_audio
```

典型写法：

```
<subject_definitions>
  <Subject 1> shown in <Picture 1>
  <Subject 2> shown in <Picture 2>
</subject_definitions>

<background>
  <Picture 3>
</background>
```

> 插件**没有** `negative_prompt` 输入。H3 的否定走提示词文本里的 `negative_prompt:` 段（H3 提示词格式的第 7 个键），那是**模型层的文本否定**，不是 ComfyUI 的负向 conditioning，照样有效。

### 报告怎么读

节点会把报告打到终端，接 `ShowText` 也能看。关键行：

```
段数    : 4 段（来源：auto 自动判定）
每段    : 175 帧 / 7.2917 秒（7.00s 请求 → 吸附到 5+17n 网格）
锚定    : 5 帧 = 2 token（第 2 段起开头复刻上段末帧，拼接时丢弃）
末段    : 补足到 107 帧（目标 26 秒；末段对白完整保留，不再砍尾）   ← 或「砍尾回退 ⚠️」
合并后  : 152 token → 515 帧 / 21.4583 秒
网格校验: ✅ 5c+2，与训练网格一致
参考资产: 图 3 张 / 视频 0 个（含音轨 0） / 独立音频 0 条（只编码 1 次，4 段共用）
采样    : 6 步 × 4 段，euler / beta，cfg=1.00，seed=1234567890（逐段 +1）
显存    : CLIP 编码器已在采样前卸载（已卸）
★ 音频流：已解码并随 video 输出    ← 若显示「未解码」说明没接 audio_vae
```

看到 `末段：砍尾回退 ⚠️` → 减段数或调大 `裁剪到秒数`。

---

## 调参指南（重点）

### ★ 画面泛紫：根因是调度器，不是 CFG

> ⚠️ **本节结论已修订。** 早期版本把泛紫归因于 `cfg` 与 VAE 时间块边界，实测均被推翻。
> 真机 A/B 验证：**`scheduler` 从 `simple` 换成官方 `beta` 后泛紫彻底消失。**

H3 官方采样链路是这样的（`RandomNoise → BasicGuider → KSamplerSelect → BasicScheduler → SamplerCustomAdvanced`）：

- `BasicGuider` → `Guider_Basic`，只接 `model + conditioning`，**结构上没有 negative**，
  且 `CFGGuider.__init__` 里`self.cfg = 1.0` 写死 —— 官方**压根不开 CFG**
- `BasicScheduler` 用的是 **`beta`**

而 `simple` 在高噪声段跨步明显更大。按 H3 flow-matching曲线验算 6 步的 sigma：

| 步 | `simple` | `beta`（官方） | simple 相对偏差 |
|---|---|---|---|
| 1 | 0.167 | 0.092 | **噪声多 82%** |
| 2 | 0.334 | 0.276 | 多 21% |
| 3 | 0.501 | 0.500 | 0 |
| 4 | 0.667 | 0.725 | 少 8% |
| 5 | 0.834 | 0.909 | 少 8% |

蒸馏 LoRA（4 步 turbo 之类）的输出本已偏离原始分布，再让第一步跨掉82% 的噪声区间，
latent 容易冲出分布 → **低饱和区（肤色、天空、沥青）色度向零塌陷 → 泛紫**。
这解释了为什么它表现为「平滑低纹理区最明显」。

**做法：`scheduler` 保持 `beta`（本节点默认值）。这是根治项。**

#### `cfg` 该怎么设

`cfg` **不是**泛紫根因，但会放大症状。插件的负样本是 `_zero_out(cond)`（**空条件**，
不是负向提示词），代入 `comfy/samplers.py`的混合公式：

```
cfg_result = uncond + (cond − uncond) × cfg
            = 0    + (cond − 0)      × cfg
            = cond × cfg
```

也就是说 `cfg < 1.0` **每一步都把去噪信号整体衰减**，本质是衰减而非引导。

| cfg | 行为 | 建议 |
|---|---|---|
| **1.0** | `samplers.py` 主动剔除 uncond，只前向 cond，**省一半算力** | ✅ **官方推荐，用这个** |
| 0.9 / 0.8 | 每步信号衰减 10% / 20%，能压住症状 | 应急可用，别当根治 |
| < 0.7 | 发灰、人物掉相似度、运镜变钝 | ❌ |

> 早期版本建议「降到 0.8」，那是在根因未找到时的**症状压制**，不是正解。
> 换成 `beta` 后可以回到 `cfg = 1.0`。

#### 已排除的因素

以下都经实测排除，**不要在它们身上浪费时间**：

| 嫌疑 | 结论 | 证据 |
|---|---|---|
| **bit 深度 / 色彩空间** | 无关 | `auto + sRGB` 在源码里就等于 8bit；ffprobe 实测紫片与正常片 `pix_fmt`/`color_space` 逐字段一致 |
| **分辨率** | 无关 | 同为 544×960 的成片有成批紫、有成批清；跨 640×1152 / 768×1344 紫峰落在同一批帧 |
| **固定种子** | 无关 | 单次采样工作流用 `randomize` 也一样出紫 |
| **VAE 时间块边界** | 非主因 | 官方解码已有 5 帧交叠混合；加大到 13 帧会引入画面跳动（已回滚） |
| **逐段解码 / 拼接方式** | 无关 | 紫峰精确落在解码块边界上，与段边界（帧 158/311/464）差 32 帧 |
| **int8 VAE** | 无效 | 实测无改善 |
| **`ColorTransfer` 消紫** | 反效果 | mkl_lab 把色调拉向绿茵场，整片发绿（已移除该节点） |

### `steps`：匹配你挂的 LoRA

节点默认 `20`，但那是给全量模型设计的。挂了 turbo / 加速 LoRA 时 20 步是过采样。

| 配置 | 建议步数 |
|---|---|
| 全量模型 | 15 ~ 25 |
| **4 步 turbo LoRA** | **6 ~ 8** |
| 8 步加速 LoRA | 10 ~ 12 |

### `visual_strength`

默认 `0.999`。参考图颜色正确时建议 **`1.0`**（更贴参考图）。参考图有色偏时调低，给模型自由度。

### 其他

| 项 | 建议 |
|---|---|
| `sigma shift` | 12.0 / 3.0 是官方默认，一般不动 |
| VAE 精度 | 若 VAE 只有 fp16，色偏会更明显。ComfyUI 有 `--fp32-vae` 开关，但显存占用翻倍，慎用 |
| 分辨率 | 先跑 `960×544` 验证构图，再上 `768×1344` |
| 段数 | 每段 7 秒最稳。段太长（>10 s）单段质量下降，段太短接缝变多 |

---

## 常见问题排查

| 症状 | 原因 | 处理 |
|---|---|---|
| **成片没声音** | 没接 `audio_vae` | 接上 H3 **audio** VAE。语音由 H3 从提示词自动生成，不需要参考音频 |
| **最后一段话说不全** | 裁剪时砍了末段尾部 | 用 `裁剪到秒数` 的「末段补足」语义；报告里看是否出现「砍尾回退」 |
| **画面泛紫** | `scheduler` 用了 `simple`（高噪声段跨步过大，蒸馏 LoRA 下 latent 冲出分布） | **`scheduler` 改成 `beta`**（节点默认值），`cfg` 回到 1.0 |
| **前几秒人物脸超大** | 参考图是竖版大头照 + 成片横版，构图被放大 | 用全身正视参考图；成片画幅与参考图一致；提示词加长焦/中景约束 |
| **`No input node found for id [x] slot [y] ref_images.ref_image_3`** | 旧工作流 JSON 与新的 Autogrow 动态槽位冲突 | **重新生成工作流**。增减参考图后务必重建 JSON，不要手工改旧文件 |
| **CLIP 显存爆（16.5 GB）** | 编码器没卸载 | 节点内部已自动卸载（看报告「显存」行）。若外部还有节点连 `CLIPLoader`，确保它们在卸载前跑完 |
| **参考音频没生效** | 没接 `audio_vae` | 缺 `audio_vae` 时**静默失效、不报错**，很阴 |
| **`cfg` 显示 `NaN`，报「值类型错误」** | 手写 `widgets_values` 时漏了 `control_after_generate` 占位 | `KSampler.seed` 的 `widgets_values` 必须是 **7** 个：`[seed, "fixed", steps, cfg, sampler, scheduler, denoise]`。那个 `"fixed"` 不在 `INPUT_TYPES` 里，按 schema 数不到 → 整列左移 |

---

## 节点二：分段三节点（手工搭循环用）

底层积木，配合官方 `StartLoop` / `EndLoop` 使用。

| 节点 | 位置 | 作用 |
|---|---|---|
| `SW_H3_SegPlan` | 循环外 | 把「每段 7 秒 × 段数」编译成参数（单段帧数 / 段数 / 锚定帧数 / 宽高） |
| `SW_H3_SegBridge` | **循环体内** | 取上一段 latent 尾部 token 注入本段条件（等价官方 AddGuide，但喂 latent 不喂像素） |
| `SW_H3_SegConcat` | 循环外 | N 段 latent 沿时间维拼成一个，接官方 `VAE Decode` |

### 接线要点

1. `SW_H3_SegPlan` 的**单段帧数 → `MiniMaxH3ReferenceToVideo.length`**，宽高也接过去，保证单一数据源
2. **`MiniMaxH3ReferenceToVideo` 放循环外**（只跑一次）。它的条件被 `SegBridge` 每轮派生并注入 keyframe，「每段都带参考图」效果不变
3. `EndLoop` 必须打开 **`accumulate`**，`output_value` 和 `next_iteration_value` **都接 `KSampler` 输出 latent**
4. `StartLoop.max_iteration` 与 `SW_H3_SegPlan.segment_count` **必须一致**
5. 官方 `VAE Decode` 只解视频流（`nodes.py` 里 `if latent.is_nested: latent = latent.unbind()[0]`）

### 负向提示词：cfg=1.0 下它本来就不生效

同上，`cfg=1.0` 时官方把负样本置 `None`。所以该工作流用官方 `ConditioningZeroOut`（复制正样本并清零）接负样本口，**零 CLIP 依赖**，也没有「负样本必须先跑完」的时序坑。

> 若把 `cfg` 调到 1 以上，负样本就会真的参与计算，那时写具体负面词才有意义。

---

## 原理：帧网格与 latent 续接

**官方常数**（逐条来自源码）：

```
comfy/ldm/minimax/model.py:30        FRAME_PER_TOKEN = (1, 4, 4, 4, 4)   每 5 个 token = 17 帧
comfy_extras/nodes_minimax_h3.py:37  align_frame_count:  合法帧数 = 5 + 17n
comfy_extras/nodes_minimax_h3.py:43  video_latent_t = ((f-5)//17)*5 + 2
```

**拼接算术**：第 1 段保留全部 52 token（175 帧），第 2 段起丢掉开头 2 个锚定 token（170 帧）。
总 token = `50N + 2`，总帧数 = `170N + 5`，**天然落在 `5+17n` 网格上**。

**段数与时长**（`segment_seconds = 7.0`）：

| 段数 | 单段帧 | 总帧 | 总时长 | 总 token |
|---|---|---|---|---|
| 1 | 175 | 175 | 7.29 s | 52 |
| 2 | 175 | 345 | 14.38 s | 102 |
| 3 | 175 | 515 | 21.46 s | 152 |
| 4 | 175 | 685 | 28.54 s | 202 |
| 5 | 175 | 855 | 35.62 s | 252 |

**三个和传统做法不同的地方**：

1. **参考图只编码一次** —— 传统做法每个 H3 节点各自 `vae.encode(参考图)`，N 段 = N 次重复编码
2. **CLIP 只卸载一次** —— N 段全编完 → 卸一次 → 开采，省约 16.5 GB
3. **段间续接在 latent 域且不经过 VAE** —— 取上段尾部 2 token（5 帧）注入本段 `frame_idx=0` 的 `minimax_keyframes`，与官方 `MiniMaxH3AddGuide` 等价但吃 latent 不吃像素

**VAE 一次性解码**：H3 的 video VAE 自带时间分块解码（`comfy/ldm/minimax/vae.py` `comfy_has_chunked_io=True`，5 token 一块、2 token 重叠、线性混合），任意长度都能一次解完。

> **关于「块边界缺上下文」**：VAE 自带 5 帧交叠混合（`blend()`），实测块边界并非泛紫主因——
> 紫峰精确落在块边界上与段边界无关，且加大混合区（`token_drop3→1`）会引入画面跳动，
> 故保持官方默认参数。**泛紫的根因在采样层的 `scheduler`，见[调参指南](#调参指南重点)。**

---

## 已知约束

- **一次性解码 855 帧**时输出 buffer 落在 CPU 内存：1344×768 fp32 ≈ 10.6 GB，960×544 ≈ 5.4 GB。长视频建议先跑低分辨率
- **参考视频至少需要 5 帧**（约 0.2 s @24fps），帧数会被吸附到 `n % 17 == 5`
- **段间一致性**靠 latent 尾部注入 + 参考图条件双重保证；后段仍可能轻微漂移，可缩短段长或加参考图
- **音频 latent** 按总帧数精确校正（1 帧 = 5/3 个音频 latent），无累积漂移
- 参考视频尺寸会经 `_adapt_canvas` 适配到 32 的倍数且不超过 `768×1344`

---

## 许可证

保留所有权利。未经作者书面许可，不得复制、分发或用于商业用途。
