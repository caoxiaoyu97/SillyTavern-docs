# SillyTavern（酒馆）本地全模块沉浸式部署与调优全指南

> 本指南针对消费级中高端配置（特别适配 **RTX 4070 Super 12G 显存 + 32G 内存** 架构）量身定制，涵盖 **文字大模型（Kobold.cpp）+ 无审查动漫生图引擎（ComfyUI）+ 状态好感度追踪（ScenePulseN）+ 一键场景插画联动** 的完整落地部署、作者官方推荐参数调优与踩坑避坑指南。

---

## 目录
- [一、 整体系统架构与工作流设计](#一-整体系统架构与工作流设计)
- [二、 硬件规划与 12G 显存分配策略](#二-硬件规划与-12g-显存分配策略)
- [三、 软件与模型下载清单（含国内高速镜像）](#三-软件与模型下载清单含国内高速镜像)
- [四、 核心重点：文字模型参数微调与采样配置（Sao10K 官方推荐）](#四-核心重点文字模型参数微调与采样配置sao10k-官方推荐)
- [五、 ScenePulseN 状态与好感度仪表盘配置](#五-scenepulsen-状态与好感度仪表盘配置)
- [六、 ComfyUI 生图联动与“一键场景插图”配置](#六-comfyui-生图联动与一键场景插图配置)
- [七、 常见问题与技术避坑指南（FAQ）](#七-常见问题与技术避坑指南faq)

---

## 一、 整体系统架构与工作流设计

本工作流采用 **“前端交互 + 双后端动力引擎”** 架构，将文字推理与生图渲染完全解耦：

```
                           ┌───────────────────────────────┐
                           │    SillyTavern 前端交互界面   │
                           │  (角色卡 / 聊天记录 / 仪表盘) │
                           └───────────────┬───────────────┘
                                           │
                 ┌─────────────────────────┴─────────────────────────┐
                 ▼                                                   ▼
┌─────────────────────────────────┐                 ┌─────────────────────────────────┐
│     文字后端: Kobold.cpp        │                 │       生图后端: ComfyUI         │
│  (端口: 127.0.0.1:5001)         │                 │    (端口: 127.0.0.1:8188)       │
├─────────────────────────────────┤                 ├─────────────────────────────────┤
│ • 模型: L3-8B-Stheno-v3.2       │                 │ • 模型: AutismMix SDXL Confetti │
│ • 显存占用: ~5.5GB ~ 6.0GB      │                 │ • 显存占用: ~5.5GB (按需调度)   │
│ • 特性: 零审查、长文本记忆平移  │                 │ • 特性: 零审查、Pony架构、二次元│
└─────────────────────────────────┘                 └─────────────────────────────────┘
```

1. **SillyTavern（酒馆）**：前端主面板，负责渲染对话、管理角色卡世界观、展示角色好感度与属性。
2. **Kobold.cpp**：文字推理动力源，读取 `Stheno-8B` GGUF 量化文件，调动 CUDA 核心进行极速文本生成。
3. **ComfyUI**：生图动力源，读取 `AutismMix SDXL`，负责在剧情推进时生成高精度的场景插画与角色立绘。
4. **ScenePulseN 插件**：注入追踪提示词，让文字模型在后台偷偷输出结构化数据，前端实时更新好感度、心情与环境天气。

---

## 二、 硬件规划与 12G 显存分配策略

针对 **RTX 4070 Super 12G 显存 + 32G 内存**，如何做到**图文同时运行且绝不爆显存**：

| 运行模块 | 规格版本 | 显存占用 (VRAM) | 内存占用 (RAM) | 说明 |
| :--- | :--- | :--- | :--- | :--- |
| **系统桌面与底噪** | Windows 基础底噪 | ~0.8 GB | ~4.0 GB | 系统基础开销 |
| **Kobold.cpp + Stheno** | `Q5_K_M` 量化 (5.7GB) | **~5.5 GB** | ~2.0 GB | 100% 装入显存，保证 60~80 字/秒极速响应 |
| **ComfyUI + AutismMix** | SDXL Pony 架构 | **~5.5 GB** | ~4.0 GB | 依靠 ComfyUI 动态内存调度，出图时激活 |
| **总计峰值占用** | - | **~11.8 GB** | **~10.0 GB** | **完美压在 12G 显存红线以内，绝无溢出卡死** |

---

## 三、 软件与模型下载清单（含国内高速镜像）

### 1. 核心软件与扩展

| 组件 | 官方仓库 / 下载源 | 说明 |
| :--- | :--- | :--- |
| **SillyTavern 酒馆** | [GitHub: SillyTavern](https://github.com/SillyTavern/SillyTavern) | 前端核心本体 |
| **Kobold.cpp 推理器** | [GitHub: LostRuins/koboldcpp Releases](https://github.com/LostRuins/koboldcpp/releases) | 下载 `koboldcpp.exe` 单文件便携版 |
| **ComfyUI 生图引擎** | [GitHub: ComfyUI Windows Portable](https://github.com/comfyanonymous/ComfyUI/releases/latest/download/ComfyUI_windows_portable_nvidia.7z) | 官方免配置 Windows 便携整合包 |
| **ScenePulseN 扩展** | [GitHub: l1kiru/SillyTavern-ScenePulseN](https://github.com/l1kiru/SillyTavern-ScenePulseN) | 好感度/状态栏/环境天气追踪插件 |

---

### 2. 权重模型文件

#### ① 文字模型：`L3-8B-Stheno-v3.2`（零审查酒馆神作）
* **国内高速下载地址**：[HF-Mirror 镜像站直达](https://hf-mirror.com/bartowski/L3-8B-Stheno-v3.2-GGUF)
* **目标下载文件**：`L3-8B-Stheno-v3.2-Q5_K_M.gguf`（大小约 5.73 GB）
* **国际官方地址**：[Hugging Face 原作者仓库](https://huggingface.co/Sao10K/L3-8B-Stheno-v3.2)

#### ② 生图模型：`AutismMix SDXL Confetti`（零审查 Pony 架构二次元天花板）
* **国内高速下载地址**：[HF-Mirror 镜像站直达 (lyliaEngine搬运版)](https://hf-mirror.com/lyliaEngine/autismmixSDXL_autismmixConfetti/tree/main)
* **目标下载文件**：`autismmixSDXL_autismmixConfetti.safetensors`（大小约 6.46 GB）
* **存放位置**：放置于 `ComfyUI_windows_portable\ComfyUI\models\checkpoints\` 目录下。

---

## 四、 核心重点：文字模型参数微调与采样配置（Sao10K 官方推荐）

很多人使用 Stheno 或 Llama-3 衍生模型时觉得“模型偶尔复读、说话呆板或者突然乱码”，**根本原因在于没有配置原作者专属的采样超参数**。

请在 SillyTavern 的 **AI 响应格式与采样设置（Generation Settings）** 中，严格按照以下参数进行配置：

### 1. 上下文与提示词模板（Instruct Template）
* **Context Template（模板）**：必须选择 **`Llama-3-Instruct`**
* **System Prompt 角色引导格式**：
  ```
  <|begin_of_text|><|start_header_id|>system<|end_header_id|>
  {{system_prompt}}<|eot_id|>
  <|start_header_id|>user<|end_header_id|>
  {{user_input}}<|eot_id|>
  <|start_header_id|>assistant<|end_header_id|>
  ```

---

### 2. 官方精调采样参数（Sampler Parameters）

| 参数项 (Parameter) | 推荐设定值 | 技术原理解析 |
| :--- | :--- | :--- |
| **Temperature（温度）** | **`1.12` ~ `1.22`** | **核心关键**：普通模型一般设 0.7，但 Stheno 针对高温度进行了文学对齐。高温度赋予它极强的情感感知、生动的修辞和丰富的词汇量，绝不生硬。 |
| **Min-P** | **`0.075`** | **高温度的防翻车保险栓**：动态裁剪掉概率低于最高项 7.5% 的离谱词汇，彻底解决高温度带来的胡言乱语。比传统的 Top-P 体验更好。 |
| **Top-K** | **`50`** | 限制在候选概率最高的前 50 个词汇中挑选。 |
| **Repetition Penalty（重复惩罚）** | **`1.10`** | 消除长轮次对话中的“复读机”倾向，避免不断重复上一句的神态描写。 |
| **Presence Penalty** | **`0.00`** | 保持默认即可。 |
| **Context Length（上下文长度）** | **`8192` ~ `16384`** | 12G 显存最佳平衡点，既拥有充沛的记忆空间，又不会抢占过多生图显存。 |

---

### 3. Kobold.cpp 启动参数建议
在启动 `koboldcpp.exe` 时：
* **GPU Layers (全卸载到 GPU)**：拉满（全部层送入 GPU，约 33 层）；
* **Context Shift（智能上下文平移）**：**务必勾选开启**（长篇对话复用 KV Cache，出字永远秒回）；
* **Flash Attention**：**勾选开启**（大幅节省上下文显存并提速）。

---

## 五、 ScenePulseN 状态与好感度仪表盘配置

`ScenePulseN` 是把酒馆变成 Galgame 的灵魂组件。

### 1. 安装步骤
1. 打开 SillyTavern，点击右上角 **扩展管理（Extensions）** -> **Install Extension**；
2. 粘贴仓库地址：`https://github.com/l1kiru/SillyTavern-ScenePulseN`；
3. 点击安装并刷新网页。*(若之前安装过原版 ScenePulse，请先禁用或卸载以防冲突)*。

### 2. 功能配置
* **好感度追踪（Affection & Trust）**：聊天时侧边栏会动态弹出 `好感度 +5`、`信任度上升` 等数值变化浮窗；
* **内心独白与心境（Inner Thoughts）**：展示角色“嘴上说的”与“心里真实想的”；
* **环境氛围与场景转换（Weather & Atmosphere）**：根据剧情自动渲染阴雨、黄昏、微风等动态光效遮罩。

---

## 六、 ComfyUI 生图联动与“一键场景插图”配置

### 1. ComfyUI 启动与接口配置
1. 打开 ComfyUI 根目录，双击 **`run_nvidia_gpu.bat`**；
2. 确保它在后台运行并监听默认地址：`http://127.0.0.1:8188`。

### 2. SillyTavern 连接生图引擎
1. 在酒馆中点击右上角 **扩展（Extensions）** -> **图像生成（Image Generation）**；
2. **API Source（接口来源）**：下拉选择 **`ComfyUI`**；
3. **Server URL（服务器地址）**：填入 `http://127.0.0.1:8188/`，点击 **Connect** 变为绿色连接状态；
4. **模型选择**：选择你下载的 `autismmixSDXL_autismmixConfetti.safetensors`。

### 3. 如何在聊天中“一键场景生图”？
* **方法 A（鼠标一键触发）**：
  把鼠标悬停在 AI 刚刚回复的消息上，点击消息下方的 **魔法棒（Wand Menu）** 图标，在弹出菜单中点击 **`The Whole Story`（生成整段场景剧情插图）**。文字模型会在后台自动总结当前场景英文词汇，ComfyUI 渲染后在 3~5 秒内自动把插画贴在该条对话下方！
* **方法 B（指令快捷触发）**：
  在打字框输入 `/sd scene` 按回车，即可瞬间触发当前场景插画生成。

---

## 七、 常见问题与技术避坑指南（FAQ）

### Q1: 为什么搜索到的生图模型（如 AutismMix）全是 2024 年的？会不会过时？
* **解答**：**完全不会！** 文字模型（LLM）需要频繁追新，但 Stable Diffusion 的 Pony/SDXL 架构讲究**“终极生态稳定”**。
* 社区中成千上万的角色 LoRA、服装包全都是基于 2024 年固定的底模训练的。底模保持不变，才能保证所有动漫角色的服装、脸模生态 100% 兼容。目前 AutismMix 依然是无审查二次元立绘的画质天花板。

### Q2: 为什么之前尝试 35B-A3B 模型写代码觉得很“拉跨”，而且电脑风扇狂转？
* **解答**：`A3B` 代表“每一步只有 30 亿活跃参数（Active 3B）”。虽然它的总知识库有 35B，但单个 token 的逻辑计算深度只有 3B 级别，面对复杂的工程重构与代码嵌套必然会出现幻觉。
* 加上 24GB 权重把 12G 显存和 32G 内存挤得满满当当，PCIe 总线频繁交换数据导致电脑发烫。在本地部署中，**纯显存跑满的纯稠密 14B 模型（如 Qwen2.5-Coder-14B）实际代码战斗力远超轻量 MoE**。

### Q3: 运行中提示显存溢出（Out of Memory / CUDA OOM）怎么办？
1. 确保没有同时打开大型 3D 游戏占用显存；
2. 在 ComfyUI 中，生成分辨率建议保持在 `832x1216` 或 `1024x1024`（SDXL 原生最佳比例），切勿直接出未经切片的 4K 原图；
3. 检查 Kobold.cpp 上下文长度是否超过了 16384，日常聊天建议锁在 8192 或 16384。

---
*Created for SillyTavern Community by caoxiaoyu97*
