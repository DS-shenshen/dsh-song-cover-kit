# DSH 唱歌语音安装教程（第一版）

这份说明讲四件事：怎么装、怎么配置、怎么加模型、以及版权与免责。
如果你只想快速跑一遍，直接看 [README.md](README.md) 也行。

---

## 一、安装

### 1. 基础环境

| 东西 | 要求 |
|---|---|
| 操作系统 | Windows 10/11、Linux、macOS 都能跑（本文命令以 Windows 为例） |
| Python | **3.10 / 3.11 最稳**；3.12 需要 `compat/fairseq`（见第 5 步） |
| ffmpeg | 必须装，且能直接调用 `ffmpeg`（或稍后在配置里写绝对路径） |
| 磁盘 | 模型 + 依赖大概 5–10GB 起（torch、demucs 权重、你自己的 RVC 模型） |
| 显卡 | 可选。纯 CPU 能跑，慢；有 NVIDIA GPU 会快很多 |

### 2. 拉代码

```bash
git clone <你的仓库地址>
cd song-cover-kit
```

或者直接把压缩包解压。

### 3. 建虚拟环境、装依赖

```bash
python -m venv .venv
# Windows PowerShell:
.\.venv\Scripts\Activate.ps1
# Linux/macOS:
source .venv/bin/activate

pip install -r requirements.txt
```

CPU 用户建议先单独装 CPU 版 torch（体积小、不容易装错）：

```bash
pip install torch torchaudio --index-url https://download.pytorch.org/whl/cpu
```

GPU 用户按 https://pytorch.org 选对应 CUDA 版本。

### 4. 准备 rvc-python 的基础模型

`rvc-python` 推理需要两个基础模型：

- `hubert_base.pt`
- `rmvpe.pt`

放到：

```
<python安装目录>/Lib/site-packages/rvc_python/base_model/
```

（首次运行有时会自动下载；下不动就手动放。文件可从 rvc-python 项目说明或 RVC 原项目获取。）

### 5. Python 3.12 的 fairseq 兼容层（3.10/3.11 可跳过）

`rvc-python 0.1.5` 会 `import fairseq`，而 fairseq 在 Python 3.12 上没有轮子。
本仓库带了最小兼容层：

```bash
# Windows（把 <python> 换成本机路径）
copy compat\fairseq <python>\Lib\site-packages\ -r
# 或临时加 PYTHONPATH（PowerShell）：
$env:PYTHONPATH = "$PWD\compat"
```

### 6. 装 ffmpeg（如果还没有）

Windows：去 ffmpeg 官网下载 release，解压后把 `bin` 目录加进 PATH；
或者装好后在 `config.json` 里把 `ffmpeg` 写成 `ffmpeg.exe` 的绝对路径。

### 7. 自检

```bash
python cover.py --help
```

看到参数列表就说明依赖装好了。

---

## 二、配置

### 1. 复制配置模板

```bash
copy config.example.json config.json     # Windows
cp config.example.json config.json       # Linux/macOS
```

`config.json` 各项：

| 字段 | 说明 | 默认 |
|---|---|---|
| `ffmpeg` | ffmpeg 命令或绝对路径 | `ffmpeg` |
| `voices_file` | 声线表路径（见下） | `voices.json` |
| `demucs_model` | demucs 模型名 | `htdemucs` |
| `work_dir` | 中间文件目录 | `work` |
| `out_dir` | 成品输出目录 | `out` |
| `device` | `cpu:0` 或 `cuda:0` | `cpu:0` |
| `hf_endpoint` | 下载 demucs 权重用的镜像 | `https://hf-mirror.com` |

**相对路径按 `config.json` 所在目录计算**，不是按你运行命令的目录。

### 2. 声线表 voices.json

```json
{
  "我的声线": {
    "pth": "D:/models/my_voice/my_voice.pth",
    "index": "D:/models/my_voice/my_voice.index"
  }
}
```

- 键名（`我的声线`）就是运行时 `--voice` 传的名字，随便起。
- `index` 可以留空字符串——没有索引也能换声。
- **index 路径不要带中文**：rvc-python 内部用 faiss，读中文路径会崩，放纯英文目录。

### 3. 音源准备

本工具**不内置歌曲下载**，需要你先准备好本地音频文件。拿音源的推荐顺序
（按音质和可用性）：

1. **酷狗、网易云这类音乐平台**（优先，音质/伴奏质量一般最好）；
2. 抖音；
3. B 站（兜底，找现场/翻唱/冷门版本时用）。

注意：从各平台拿音频请遵守它们的服务条款，只处理你有权使用的素材；
不要拿它做公开发布或商用的侵权内容。

### 4. 运行

```bash
python cover.py --input "D:/songs/song.mp3" --voice "我的声线"
```

输出：

- `out/<歌名>_<声线>_full.mp3` —— 整首
- `out/<歌名>_<声线>_clip60s.mp3` —— 从人声最响处截的 60 秒试听

---

## 三、怎么添加模型

本仓库**不附带任何声线模型**。你要换谁的声音，就自己放一个 RVC 模型进来。

### 1. 模型长什么样

一个 RVC 声线模型通常是两个文件：

- `<名字>.pth` —— 模型本体
- `<名字>.index` —— 特征索引（可选，能提高贴合度）

### 2. 放哪儿

建议放成：

```
D:/models/<声线名>/<声线名>.pth
D:/models/<声线名>/<声线名>.index
```

路径随便，但**整个路径尽量用英文**（中文路径对 faiss、部分 Python 包都不友好）。

### 3. 加进 voices.json

```json
{
  "voice_a": {
    "pth": "D:/models/voice_a/voice_a.pth",
    "index": "D:/models/voice_a/voice_a.index"
  }
}
```

### 4. 验证

```bash
python cover.py --input "D:/songs/短音频.mp3" --voice "voice_a" --clip-seconds 15
```

先拿 20–30 秒的短音频试，确认能出结果再跑整首，省时间。

### 5. 声音不像怎么办

- 换更匹配的模型（性别、音域接近的）；
- 调 `--index-rate`（0.3–0.7 之间试）；
- 调 `--rms`（0.25–0.5）；
- 男女声互转时 `--f0` 常用 ±12（半音）。

---

## 四、训练几轮

**本仓库只做推理，不包含训练脚本。** 训练 RVC 模型请用 RVC 官方 / WebUI。
下面只是经验值，最终以你耳机里听到的为准。

### RVC v2（本工具用的就是这种）

| 项目 | 参考值 |
|---|---|
| 干净干声素材 | 10–60 分钟（3–10 秒切一段，先做去混响/去和声） |
| 总轮数（epoch） | 常见 **100–300 轮**；素材少于 10 分钟从 **50–100 轮**开始 |
| 保存间隔 | 每 10 轮存一份权重，方便挑 |
| 索引 | 训完生成 `.index`，再放进 `voices.json` |
| 判断标准 | A/B 试听：清楚、稳、不跑调；**不是轮数越多越好**，过头会过拟合（发尖、气声怪、跑调） |

### GPT-SoVITS（另一种路线，做「说话」更像）

不是本工具需要的格式，但如果你也想做说话声线，参考量级：

| 项目 | 参考值 |
|---|---|
| 干净语音素材 | 30 分钟–2 小时 |
| GPT 训练 | **15–30 轮** |
| SoVITS 训练 | **10–20 轮** |
| 判断标准 | 同样靠试听；参考音频要干净、无混响 |

> 训练很吃时间和显存；先用小数据跑几十轮听效果，再决定要不要加。

---

## 五、语音模型侵权（重要）

说清楚：

1. **本仓库不提供任何模型。** 你用谁的声线、从哪拿的模型，是你自己的选择和行为。
2. **声音也有权利。** 拿真实人物（尤其是公众人物、你身边的人）的声音训模型、公开发布、
   商用，可能侵犯声音权、肖像权、名誉权，也可能违反平台规则甚至法律。
3. **别人的模型不一定能随便用。** 网上流传的 RVC 模型很多是别人训练的，可能带
   作者声明或许可证；用之前先看许可，别默认「能下载就能用」。
4. **别传播。** 别把别人的声线模型、别人声音的翻唱成品打包分享、上传到公开平台。
5. **做自己的声音最稳。** 拿自己录音、自己授权的素材训模型，风险最低。

一句话：**本项目只提供工具，不提供授权。** 用之前先问自己：这个声音，我有权用吗？

---

## 六、免责条款

- 本仓库代码按 **MIT 许可**「按现状」（AS IS）提供，不附带任何明示或暗示的担保，
  包括但不限于可用性、稳定性、效果、不侵权。
- 作者不对使用本项目产生的任何直接或间接后果负责，包括但不限于：
  生成的音频、模型效果、第三方依赖变化、平台封禁、法律纠纷。
- 使用者需自行确保：所使用的音频素材、模型权重、发布行为**合法合规**，
  并遵守所在地区法律与各平台的服务条款。
- 本项目与任何音乐平台、模型作者、艺人、游戏公司均无关联，也不代表其立场。
- 如果你不同意以上条款，请不要使用本仓库。

---

## 七、常见问题

| 问题 | 处理 |
|---|---|
| `ffmpeg not found` | 装 ffmpeg 并加 PATH，或在 `config.json` 写绝对路径 |
| `ModuleNotFoundError: fairseq` | Python 3.12 没轮子，用 `compat/fairseq`（见安装第 5 步） |
| `faiss` 报错 / 崩 | index 路径改纯英文 |
| 声音发闷、不像 | 换模型、调 `--index-rate`、`--f0` |
| 第一次跑很慢 | demucs 在下载权重；之后会快 |
| 内存不够 | 用短一点的音频，或换更大内存的机器 |
| 输出没声音/破音 | 检查输入的伴奏/人声是否正常；`--rms` 调低一点 |
