# dsh-multimodal-video

DeepSeek Harness 的多模态 host 插件（视频理解增强版）：给文本模型补全视觉、生图与**视频时序理解**能力，不改动 harness 源码。

> 本仓库是 [`dsh-multimodal`](https://github.com/SCT192221/dsh-multimodal) 的增强分支：在稳定版全部能力之上，新增 `analyze_video` 视频理解工具。

## 功能

- **vision** — 识图 / OCR / 图表理解 / 多图比较 / 视觉问答。省略 `images` 时自动读取本会话最近一次粘贴或发送的图片（跨轮可读）。
- **generate_image** — 文生图 / 参考图编辑（P图）。省略 `references` 时自动用本会话最近一次图片作参考。
  - **参考图比例匹配**：有参考图且未显式传 `WIDTHxHEIGHT` 时，读参考图宽高比，从豆包 Seedream 该档（2K/3K）官方推荐像素表里按对数距离选最近的 `WIDTHxHEIGHT` 传 API——新图严格匹配参考图比例，不再固定 2K 出方图。
  - **超限自动预览**：生成的图片超过 harness 附件准入限制（新版默认单边 2000px / 字节数上限）时，自动等比缩放为预览图（名称带 `-preview`）内联展示，原始分辨率文件照常写入 `<workspace>/imgs/` 并在 `files` 里给出路径。
- **analyze_video** — 视频时序理解（本插件核心增强）。自动提取「增强帧包」一次性送视觉模型做跨帧分析，含**运动峰值自动定位与原始分辨率裁剪放大帧组**（复刻网页动效/特效的关键），详见下文[视频理解增强](#视频理解增强增强帧包)。
- **show_image** — 把本地图片文件（脚本生成的图表、截图等）直接展示在对话输出里。传 `path`（单张）或 `paths`（多张列表）。对纯文本模型也可用（无 `read_image` 的 image-capable 闸）。超限图片同样自动缩放为预览展示。
- **llm/stream text-only 适配** — 纯文本模型收到带图消息时，在 adapter 边界把图片替换为内联提示（`[本条消息含 N 张图片，请调用 vision 工具]`），仅改模型请求、不碰 durable 日志，图片在浏览器照常显示。

## 工具

| 工具 | 用途 | 关键参数 |
|------|------|----------|
| `vision` | 识图/问答 | `prompt`（必填）、`images`（可省，自动读最近图）、`detail`、`max_tokens` |
| `generate_image` | 生图/改图 | `prompt`（必填）、`references`（可省，自动用最近图）、`size`（默认 2K）、`count`（1-4） |
| `analyze_video` | 视频时序分析 | `path`（必填，本地视频绝对路径）、`prompt`（可省）、`grid_frames`（默认 16，>30s 自动 24）、`detail_segments`（运动区锚点帧数，默认 4）、`detail_peaks`（运动峰值放大处数，默认 5，0 禁用）、`detail_frames`（每峰值帧数，默认 4）、`scene_threshold`（默认 0.3）、`max_tokens`（默认 4096） |
| `show_image` | 展示本地图 | `path`（单张，本地绝对路径）、`paths`（多张列表，二选一或同时） |

## 视频理解增强：增强帧包

逐帧调用视觉模型有三个结构性缺陷：每帧都是无上下文的静态图，模型看不到「变化」；均匀采样必然漏掉运动峰值与过渡过程；碎片化调用无法跨帧推理。录屏场景还有第四个：**页面滚动的运动能量淹没一切**，模型极易把滚动速度当成动效。`analyze_video` 的解法是把**时间维度编码进空间图像 + 数值证据**，一次调用让模型看到完整演变：

**第一步：全局运动补偿（滚动分离）**。对相邻采样帧做两级块匹配运动估计（含半像素细化），得到每个时刻的整画面位移向量（= 滚动/镜头平移速度）。扣除全局运动后的**残差**才是真正的局部动效信号——事件检测、区域定位全部基于残差，滚动不可能再伪装成动效。测得的滚动时段与速度、每个效果事件的精确时间/时长/区域占比，全部作为**数值证据**直接写进分析提示，模型不再目测。

| 增强帧 | 编码了什么 | 怎么读 |
|--------|-----------|--------|
| **contact-sheet 接触图** | 整条时间线。N 帧均匀采样拼成网格（行优先、时间递增），每格左上角带时间戳水印 | 把握整体结构与时间线 |
| **diff-sheet 差分图** | 运动本身（未扣滚动）。同一采样序列的相邻帧差分（高对比增强），亮度 = 变化幅度 | 判断「什么在动、动得多强」 |
| **detail-anchor 锚点帧** | 每个局部变化区的全幅中点帧 | 定位 zoom 放大帧在整页中的位置 |
| **detail-zoom 事件放大帧**（核心） | 单次效果事件的像素级细节。在残差峰值处（= 特效发生瞬间）检测运动区域包围盒，裁剪出**原始分辨率**局部画面并按滚动位移对齐（区域锁定在效果发生处），以约 0.2s 间隔连拍数帧，标着与证据同名的事件编号（E1、E2…） | 帧间内容差异 = 特效逐步演变；文字/颜色/边框等像素细节以此为准；时长直接用证据数值 |
| **scene 关键帧** | 镜头切换瞬间（已排除滚动段误检） | 转场时刻的完整画面 |

**为什么这套组合是复刻的关键**：网页动效通常只持续 300ms–1s，均匀采样（16 帧覆盖 30s ≈ 每 1.9s 一帧）采不到演变中间态；滚动不分离时检测到的「运动区」全是滚动本身。补偿后：滚动速度精确测量（px/s）、效果事件的时间/时长数值化、0.2s 间隔 × 原始分辨率裁剪推进到单次特效内部。引导提示强制模型：时长用证据数据、滚动不是动效、缓动从相邻帧位移进度推断、色值/布局量化输出。

- **依赖 ffmpeg**（仅此工具需要，装在 PATH 即可；vision/generate_image/show_image 完全不受影响）。未安装时调用会报错并给出安装指引。
- **零额外配置**：复用视觉通道（同一模型/端点/API Key）。
- 帧提取参数可调：`grid_frames` 4–24（超过 30s 的视频自动升到 24），`detail_segments` 0–8（默认 4，设 0 禁用锚点帧），`detail_peaks` 0–12（默认 5，设 0 禁用放大帧），`detail_frames` 每事件 2–8 帧（默认 4，覆盖事件前后约 0.6s），`scene_threshold` 0.05–0.9（越小越敏感）。整包帧数有预算上限（约 30 张），超出时自动舍弃能量较低的事件。运动扫描覆盖前 240 秒。

## 安装

file:// 挂载（host 唯一支持的安装方式）：

把 `index.mjs` + `multimodal-helper.cjs` 放到 `~/.dsh/plugins/dsh-multimodal-video/`，在 web profile 的 `~/.dsh/profiles/web/cordis.patch.yml` 的 `- insert:` 数组加：

```yaml
- id: multimodal-host
  name: file:///C:/Users/<you>/.dsh/plugins/dsh-multimodal-video/index.mjs
```

重启 `dsh web` 生效。

> 不走 pnpm 依赖安装是有意的：运行时配置 `global-multimodal-config.json` 写在插件目录，放 `~/.dsh/plugins/` 才不会被 profile 重装清掉。host 侧的服务依赖由 `index.mjs` 的 `export const inject` 声明，无需 package.json 元数据。

## 必需的 harness 补丁：modality guard 豁免

官方 harness 的 api-proxy 有模态守卫：纯文本模型的会话里粘贴/发送图片会被直接拒绝（提示「当前模型不支持图片，请切换支持图片的模型」），图片进不了会话，vision 也就读不到它。要启用「贴图 → vision 自动识别」链路，需给 harness 源码树打上本仓库附带的小补丁：

```sh
# 在 harness 源码树根（packages/ 的上级）执行
git apply <本仓库路径>/patches/apiproxy-modality-guard.patch
```

补丁做的事：api-proxy 在图片准入守卫（两处：prompt 提交与模型切换）追加 `ctx.get('globalMultimodal').visionEnabled()` 检查——本插件挂载且视觉通道启用时放行图片进入会话；图片随后由插件的 llm/stream 适配替换为文字提示，不会真的发给文本模型。守卫其余行为不变（未挂载本插件时与原版完全一致）。

- **不打补丁的降级行为**：贴图被 harness 拒绝；`vision` 仍可通过 `images` 参数识别本地路径 / URL / data URI 的图片；文生图、`analyze_video` 与 `show_image` 完全不受影响。
- 补丁基于上游 master（2026-08-20）验证。harness 升级后若 `git apply` 因上下文漂移失败，手动改法：在 `packages/host/apiproxy/src/api-proxy.ts` 找到两处 `inputModalities !== undefined && !…includes('image')` 守卫条件，各追加 `&& !multimodalVisionActive(ctx)`，再把 patch 文件里的 `multimodalVisionActive` 函数复制到该文件顶部。

## 配置

- **凭据**：视觉与生图两个通道各一个 API Key，写在 `~/.dsh/.credentials.yaml`：
  - `DSH_VISION_API_KEY` — 视觉模型（analyze_video 也走此通道）
  - `DSH_GENERATION_API_KEY` — 生图模型
- **模型/端点**：两个通道的模型 ID 与 Base URL 均可配置，支持任何 OpenAI 兼容端点（Gemini、GPT、豆包等），**不预置任何默认值**——首次使用需先在设置页（或配置文件）填写，填好前工具调用会提示未配置。运行时配置写在插件目录的 `global-multimodal-config.json`（首次运行自动生成空配置）。
- 从 dsh-multimodal 迁移：把旧插件目录的 `global-multimodal-config.json` 拷到新插件目录即可，凭据在 `~/.dsh/.credentials.yaml` 全局共享，无需改动。
- 本开源包**不含** `global-multimodal-config.json` 与凭据。

## 依赖

- **helper 子进程**：`multimodal-helper.cjs` 由 `index.mjs` 用 node 子进程调用，向所配置的 OpenAI 兼容端点发 `chat/completions`（vision / analyze_video 的最终分析）与 `images/generations`（generate_image）请求；超限缩图用 sharp 完成（从 dsh 安装树解析，rc.8+ 的 `attachment-local` 自带该依赖，用户无需安装）；视频帧提取用 ffmpeg（需装在 PATH，仅 `analyze_video` 需要）。API 请求仅用 Node 内置 `fetch`。
- **配套 client 插件**：同仓库的 [`dsh-multimodal-client`](../dsh-multimodal-client/) 渲染 `generate_image`/`show_image` 的工具卡内联图与 turn-tail 图集，并提供「设置 → 多模态」设置页。可选：未安装时 `vision`/`analyze_video` 的文本结果不受影响，工具结果中的图片以通用卡片显示（附件引用完整，模型可正常引用），配置需手编文件。

## 已知问题

- 参考图尺寸解析覆盖 PNG/JPEG/GIF；WebP 参考图读不到尺寸时 fallback 回档位（保持原行为）。
- `analyze_video` 的运动扫描以 4fps 粒度：短于 250ms 的极端瞬态（单帧闪光）可能检测不到；慢速滚动（低于约 45px/s）不会被判为滚动，其残差可能放大事件区域（区域信息仅供参考，时间/时长不受影响）；事件超过 `detail_peaks` 上限或整包帧数超预算时自动舍弃能量较低者。
- 运动估计假设背景有纹理（真实网页/实景均满足）：纯色无纹理背景上的单一运动物体可能被整体当作全局运动。
- 视频帧提取质量依赖 ffmpeg 的解码能力；不支持的编码会直接报错而不是静默降级。

## harness 附件尺寸上限与你的选择

新版 harness（rc.8 起）的附件存储默认限制**单边 2000px**（`attachment-local` 服务的 `maxImageDimension`，引入原因是模型路由会拒绝多图请求中超 2000px 的历史图片；旧版 harness 无此限制）。本插件默认生成 2K/3K 尺寸，必然超过该上限。

**零配置即可内联预览**：超限图片会自动等比缩放到限内（名称带 `-preview` 后缀）入库内联展示，原始分辨率文件完整保留，**功能不会坏**：

- **generate_image**：生成照常成功、原图照常写入 `<workspace>/imgs/`；超限的那张以预览图内联展示，对应原图路径列在结果 `files` 里（预览原图在前、未内联的在后）。找不到 sharp 或缩放失败时退回纯路径交付并附配置提示——工具不会失败，模型也不会因此重试。
- **show_image**：超限图片同样自动缩放为预览展示，原路径见 `paths`；缩放不可用时才报错并给出实际尺寸、上限值与配置方法。
- **analyze_video** 的增强帧不进附件存储（直接作为 API 请求内容送视觉模型），不受此限制影响。
- 贴图本身仍受 harness 层限制（不受本插件控制）：单边超 2000px 的图会被拒入会话。
- 缩图由 helper 子进程用 sharp 完成，sharp 从 dsh 安装树自动解析（rc.8+ 的 `attachment-local` 自带该依赖，npm/pnpm/源码启动均覆盖）；旧版 harness 无限制时这条路径根本不会触发，行为与从前一致。

想看内联原图？两种方案按需取舍：

### 方案 A：调大 harness 上限（看 2K/3K 原图）

在 `~/.dsh/settings.yaml` 追加后重启 `dsh web`：

```yaml
attachment-local:
  maxImageDimension: 4096   # 3K 档最大单边 4704，需全覆盖可设 4800
```

- ✅ 2K/3K 原图直接在工具卡与图集里内联展示，视觉质量无损，不产生预览图
- ✅ 贴图也放宽（≤4096px 的截图可直接进会话）
- ⚠️ 该上限存在的原始原因是保护模型路由：会话历史里**同时携带多张**大图时，可能被模型路由拒绝。生图工作流（单图/少图）一般无碍；如果你常贴多张大图同时识别，建议保持默认并选方案 B

### 方案 B：不动配置（零配置，自动缩放预览）

- ✅ 无需改 harness 任何配置，装完即用；超限图片自动以预览内联展示，不再是干巴巴的文件路径
- ✅ 原始分辨率文件完整保存在 `<workspace>/imgs/`（或 `show_image` 的原路径），随时可取用
- ⚠️ 内联的是缩到 2000px 内的 JPEG 预览，细节有损；要无损原图请用方案 A
- 适合：不想动配置、以识图出图为主、偶尔需要放大看原图的场景

## License

MIT
