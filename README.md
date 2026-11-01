# song-cover-kit · 本地翻唱流水线

把你手里的歌（或任何带人声的音频）换成指定 RVC 声线，输出整首歌 + 试听片段。
全程在本机跑，不上传音频、不依赖在线服务。

> 本仓库**只包含代码**，不包含任何模型权重或音频素材。
> RVC 模型、基础模型（HuBERT / RMVPE）、demucs 权重都需要你自己获取，并遵守各自的许可。
> 详见 [docs/MODELS-AND-LICENSES.md](docs/MODELS-AND-LICENSES.md)。
>
> 安装、配置、添加模型、训练轮数、侵权与免责的完整说明见 **[DSH唱歌语音安装教程（第一版）.md](DSH唱歌语音安装教程（第一版）.md)**。

## 它做什么

输入一首歌：

1. ffmpeg 转成 wav；
2. demucs 拆出人声和伴奏；
3. RVC 把人声换成你的声线；
4. 混音；
5. 输出 `<歌名>_<声线>_full.mp3` 和从人声最响处截的 `<歌名>_<声线>_clip60s.mp3`。

CPU 能跑（慢），有 GPU 会快很多。

## 不包含什么

- 不含模型权重（RVC `.pth` / `.index`、HuBERT、RMVPE、demucs 权重）。
- 不含 B 站下载、QQ / 聊天机器人自动发消息等集成——那部分属于私人自用环境，故意没有放。
- 不含任何歌曲 / 人声素材。

## 目录

```
cover.py             主流程
rvc_full.py          单段 RVC 换声
rvc_mix.py           人声 + 伴奏混音
config.example.json  配置模板（复制成 config.json 使用）
voices.example.json  声线表模板（复制成 voices.json，填自己的模型路径）
compat/fairseq/      Python 3.12 下 rvc-python 的 fairseq 兼容层（可选）
docs/                模型与版权声明
```

## 环境要求

- Python 3.10 / 3.11 推荐（3.12 需要 `compat/fairseq`，见下）
- ffmpeg：命令行能直接调用，或在 `config.json` 里写绝对路径
- torch + torchaudio（CPU 版即可）
- demucs、rvc-python

```bash
python -m venv .venv
# Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

GPU 用户按 PyTorch 官网装对应 CUDA 版；CPU 用户可以先：

```bash
pip install torch torchaudio --index-url https://download.pytorch.org/whl/cpu
```

### rvc-python 的基础模型

`rvc-python` 需要 `hubert_base.pt` 和 `rmvpe.pt`，放在
`<python>/Lib/site-packages/rvc_python/base_model/` 下（首次运行有时会自动下载；下不动就手动放）。
文件可从 rvc-python 项目说明或 RVC 原项目获取，注意许可。

### Python 3.12（fairseq 没有轮子）

`rvc-python 0.1.5` 会 `import fairseq`。Python 3.12 没有 fairseq 轮子，会报 `ModuleNotFoundError`。
本仓库带了一个最小兼容层 `compat/fairseq/`（只实现 rvc-python 推理用到的那一小块）：

```bash
# 任选其一
cp -r compat/fairseq <python>/Lib/site-packages/
# 或临时：set PYTHONPATH=<仓库目录>\compat
```

## 准备声线

```bash
cp voices.example.json voices.json
```

`voices.json`：

```json
{
  "my_voice": {
    "pth": "D:/models/my_voice.pth",
    "index": "D:/models/my_voice.index"
  }
}
```

`index` 可以留空字符串（没有 index 也能换声）。
注意：`rvc-python` 内部用 faiss 读 index，**路径不要带中文**（中文路径会崩），放纯英文目录。

> 示例里只有占位名（`my_voice` / `another_voice`）。
> **不要把任何具体声线模型的名字、路径写进代码或示例再发布**——本仓库不内置声线列表，
> 使用者想用什么模型，自己往 `voices.json` 里加就行。

## 运行

```bash
python cover.py --input "D:/songs/song.mp3" --voice my_voice
```

输出在 `out/`：

- `song_my_voice_full.mp3` —— 整首
- `song_my_voice_clip60s.mp3` —— 从人声最响处截的 60 秒片段

常用参数：

| 参数 | 说明 | 默认 |
|---|---|---|
| `--name` | 歌名（输出文件名用） | 输入文件名 |
| `--voice` | 声线（`voices.json` 的键） | 第一个 |
| `--config` | 配置文件 | 仓库目录下的 `config.json`（没有就用默认值） |
| `--clip-seconds` | 片段长度，0 = 不切 | 60 |
| `--f0` | 升降调（半音） | 0 |
| `--rms` | RVC rms_mix_rate | 0.35 |
| `--index-rate` | RVC index_rate | 0.5 |
| `--device` | `cpu:0` / `cuda:0` | `cpu:0` |
| `--keep-work` | 保留中间文件 | 关 |

## 配置文件

`config.example.json` → 复制成 `config.json`：

```json
{
  "ffmpeg": "ffmpeg",
  "voices_file": "voices.json",
  "demucs_model": "htdemucs",
  "work_dir": "work",
  "out_dir": "out",
  "device": "cpu:0",
  "hf_endpoint": "https://hf-mirror.com"
}
```

相对路径按 `config.json` 所在目录计算。`hf_endpoint` 用于国内下载 demucs 权重，不需要可删。

## 性能参考（纯 CPU）

- demucs 拆 3 分钟歌：2–6 分钟
- RVC 换 3 分钟人声：2–5 分钟
- 整首流程约 10 分钟量级；GPU 快很多

## 常见问题

- **ffmpeg not found**：装 ffmpeg 并加 PATH，或在 `config.json` 里写绝对路径。
- **fairseq 报错**：见上面「Python 3.12」一节。
- **faiss 报错**：index 路径改成纯英文。
- **声音不像**：换更匹配的模型、调 `--index-rate`；男女声互转常用 `--f0 ±12`。
- **第一次跑很慢**：demucs 正在下载模型。

## 声明

- 本仓库只提供代码（MIT）。**不提供、不附带任何模型权重或歌曲素材。**
- RVC 模型、demucs、HuBERT、RMVPE 等各有各的许可，使用前请自行确认。
- 请只对你有权使用的音频做处理；不要用于侵权、冒充他人或违法用途。
- 模型效果与法律风险由使用者自行承担。

## License

MIT，见 [LICENSE](LICENSE)。
