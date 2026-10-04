# SW 海螺H3长视频超级进化版（SW_H3Ultra）

一个节点干完除「模型输入 / 素材输入 / 创建视频」之外的所有事。

节点 ID：`SW_H3Ultra`　　菜单分类：`SW/H3Segments`

---

## 一句话接线

```
UNETLoader(H3) ─► LoRA ─► SageAttn ─────────────► model
CLIPLoader(Qwen3-VL) ─────────────────────────► clip
VAELoader(H3 video VAE) ──────────────────────► vae
VAELoader(H3 audio VAE) ──────────────────────► audio_vae
LoadImage ×N ─────────────────────────────────► ref_images.ref_image_N
LoadAudio（整首歌，可选）──────────────────────► song
文本节点 ×N ──────────────────────────────────► prompts.prompt_N

SW_H3Ultra.video ──► CreateVideo.images
SW_H3Ultra.audio ──► CreateVideo.audio
CreateVideo ──────► SaveVideo
```

---

## 和普通版 SW_H3MultiPrompt 的区别

| | 普通版 | 超级进化版 |
|---|---|---|
| 提示词口 | 固定 5 个 widget | **Autogrow，1~12 段随接随长** |
| 段数上限 | 5 段（~35 秒） | **12 段（~85 秒）** |
| 外部整首歌驱动 | ✗ | **逐段切片注入 + 成片音轨替换** |
| 参考接力回灌 | ✗ | **上段画面回灌成下一段的新参考图** |
| 参考图编码模式 | 只算一次 | 一次 / 每段重编 |
| 锚定帧数 | 固定 5 帧 | 5 / 22 帧 |
| 成片音轨 | 只能 H3 自产 | H3 / 外部歌曲 / 混合 |

段长对照（每段 7 秒、锚定 5 帧）：

| 段数 | 成片 |
|---|---|
| 3 | 21.46 秒 |
| 4 | 28.54 秒 |
| 5 | 35.63 秒 |
| 8 | 56.88 秒 |
| 12 | 85.21 秒 |

---

## ★ 关键答疑：参考图能不能多次编码？

能，但**重复编码同一张图没有任何意义**。理由两条，都能在官方源码里查证：

**① `vae.encode()` 是确定性的** —— 重跑 4 次得到逐位相同的 tensor。

**② 参考 token 的 RoPE 位置与「第几段」无关**

```
comfy/ldm/minimax/model.py:353  PackedLayout.__init__
    cursor = float(text_len)
    for blk in refs:
        cursor += _ref_t_span(blk)
    ...
    n_video = latent_t * frame_rows
    pos.append(_video_grid(latent_t, frame, cursor))   # target 起点
```

参考图 token 的位置只由 `text_len + refs 的 span` 决定，target 长度和段号都不参与
→ 复用和重编在模型眼里完全等价。

**顺带澄清一件事**：CLIP 视觉塔那一路本来就**每段都跑 N 次**（每段 prompt 不同、
`<Picture>` 标签的视觉也要重过一遍 Qwen3-VL）。从来就没被优化掉。
真正共用只有 VAE latent 那一份。

> 节点里仍保留「参考图编码 = 每段重编」选项，但报告会明写
> 「结果与一次编码逐位相同，仅供验证」——你可以自己跑一遍确认。

---

## ★ 那后几段画质劣化该怎么治：参考接力

真正丢的不是「参考图被看过几次」，而是**后续段看不到上一段实际生成的样子**：
段间只有 5 帧（2 个 latent token）的首帧锚定拉着，而每段还有各自的提示词在牵引
画面，第 3、4 段的细节和肤色会顺着各自的去噪轨迹漂开。

`参考接力` 把上一段末端画面**直接变成下一段的新 ref_img token**：

```python
# PackedLayout 的 image 分支以单帧为一个 spatial group，
# 所以 latent 切片必须是 [B, 24, 1, H/16, W/16]
{"kind": "image", "latent_h": H//16, "latent_w": W//16,
 "latent": video[:, :, -1:, :, :]}
```

| 模式 | 做什么 | 成本 |
|---|---|---|
| `off` | 不回灌 | 0 |
| `latent`（默认，推荐） | 纯 latent 切片，**不跑 VAE、不跑 CLIP** | 几乎为 0 |
| `full` | 额外把末端 decode 回像素，让 Qwen 视觉塔也看见 | 每段多一次单帧解码；**CLIP 必须常驻（约 16.5 GB）** |

`接力窗口` 控制回灌最近几段，默认 1（开销恒定）。调到 3、4 会让参考 token 变多、
序列变长。

另外两招：
- `anchor_frames = 22`（7 token 而不是 2）——段间拉得更紧，代价是每段少 17 帧有效画面
- `ref_image_size = max`（2048 短边）——参考图本来的分辨率更高，一致性更好，但慢很多

---

## 唱歌 MV：外部整首歌怎么用

1. `LoadAudio` 接到 `song`
2. `音轨来源` 选 `外部歌曲`（默认）
3. 节点会按每段的起点秒自动切片：

```
第 1 段  0.00s 起 → 切 7.29s
第 2 段  7.08s 起 → 切 7.29s    ← 拼接时会丢掉 5 帧锚定，所以起点不是整 7.29
...
```

切片既当该段的条件（对口型），也可直接当最终成片音轨。

尾部不够长会**补静音**，不会像 `AudioConcat` 那样从头绕回去。

`音轨来源` 三档：
- `H3生成`：用模型自产的音（不接 song 时只能这个）
- `外部歌曲`：成片音轨直接换成原曲（唱歌 MV 推荐）
- `混合`：两条按 `歌曲音量` 混合

---

## 推荐参数（已实测）

| 参数 | 值 | 说明 |
|---|---|---|
| `steps` | 6 | 配 turbo LoRA 的 4 步模型时 |
| `cfg` | 1.0 | 官方推荐（不做 CFG）。**偏紫的根因不是 cfg，是 scheduler** |
| `sampler / scheduler` | euler / **beta** | ★ 必须用 `beta`；用 `simple` 会画面泛紫 |
| `参考接力` | latent | 画质连续性，几乎免费 |
| `anchor_frames` | 5 | 想更稳可以试 22 |
| `width / height` | 768 / 1344 | 竖版 |
| `裁剪到秒数` | 26.0 | 优先让末段少生成而不是砍尾 |

---

## 注意

- **提示词口是 Autogrow，必须外接文本节点**（`PrimitiveStringMultiline` 之类）。
  ComfyUI 的 Autogrow 会强制 `force_input=True`（`_io.py:1092`），
  所以面板里没有输入框可以打字。
- 段数按「从第 1 口起连续非空」判定，中间断号只认前面的（报告里会警告）。
- 参考接力 = `full` 时 `卸载CLIP` 会被自动忽略，跑之前留意显存。
- 每段都会同时用到「首帧锚定」和「参考接力」，两者作用不同不要互相替代：
  锚定管**时间连贯**（首尾接得上），接力管**细节不丢**（人物不发胖不换脸）。
  一起开最稳。
