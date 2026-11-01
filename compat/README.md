# fairseq 兼容层（仅 Python 3.12 需要）

`rvc-python 0.1.5` 的推理路径会 `import fairseq`，而 fairseq 在 Python 3.12 上没有可用的轮子。
这里是一个最小兼容实现，只覆盖 rvc-python 推理真正用到的那部分：

- `fairseq.checkpoint_utils.load_model_ensemble_and_task()`
- 把 fairseq 格式的 `hubert_base.pt` 权重映射到 `torchaudio.models.hubert_base()`

## 用法

任选其一：

```bash
# 1) 拷进 site-packages
cp -r compat/fairseq <python>/Lib/site-packages/

# 2) 临时加到 PYTHONPATH
# Windows PowerShell:
$env:PYTHONPATH = "<仓库目录>\compat"
# Linux/macOS:
export PYTHONPATH="<仓库目录>/compat"
```

## 注意

- 如果 Python 3.10/3.11 能正常装 fairseq，就不需要这个兼容层。
- 它只实现了推理需要的最小切片，不要拿它当完整 fairseq 用。
- 依赖 torchaudio（`pip install torchaudio`）。
