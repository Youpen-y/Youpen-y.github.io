---
title: "Qwen3.5 4B上的小实验"
date: 2026-09-20T19:31:56+08:00
draft: false
tags: ["ai", "llama"]
math: true
comments: true
---
### 一、从 HF safetensors 到 gguf 的格式转换
llama.cpp 提供了从 safetensors 格式模型文件转换为 gguf 格式模型文件的工具 [`convert_hf_to_gguf.py`](https://github.com/ggml-org/llama.cpp/blob/master/convert_hf_to_gguf.py) ，但是直接使用起来不太方便，最好的方式是为其创建一个 Wrapper 脚本，在脚本中指定运行虚拟环境并放置在系统路径下，这样就可以直接使用 `convert_hf_to_gguf` 命令了。

[Qwen3.5 4B](https://huggingface.co/Qwen/Qwen3.5-4B) 是一款小型视觉-语言多模态模型，权重分为三部分：
- 主语言模型（`model.language_model.*`）：32层混合线性注意力/全注意力
- 视觉编码器（`model.visual.*`）：24层 ViT + merger，把图像/视频 patch 编码并投影为 2560 维视觉 token 注入语言模型
- MTP（Multi-Token Prediction, 多 token 预测）头（`mtp.*`）：单层多 token 预测模块，训练时提供多步预测监督，推理时用于猜测解码加速

转换流程：先切换至模型目录。

$$a_1)$$导出主语言模型（含 MTP）
```bash
convert_hf_to_gguf . \
    --outfile ./qwen3.5-4b-{ftype}.gguf \
    --outtype bf16
```
$$a_2)$$导出视觉编码器
```bash
convert_hf_to_gguf . \
    --outfile ./mmproj-qwen3.5-4b-{ftype}.gguf
    --outtype bf16 \
    --mmproj
```

使用 `llama-cli` 运行命令：
```bash
llama-cli \
    -m qwen3.5-4b-bf16.gguf \
    --mmproj mmproj-qwen3.5-4b-bf16.gguf \
    -ngl 99
```

独立导出主语言模型与 MTP 头。

$$b_1)$$导出主语言模型（不含 MTP）
```bash
convert_hf_to_gguf . \
    --outfile ./qwen3.5-4b-{ftype}.gguf \
    --outtype bf16 \
    --no-nextn
```

$$b_2)$$导出视觉编码器，同$$a_2$$

$$b_3)$$导出 MTP 头
```bash
convert_hf_to_gguf . \
    --outfile ./mtp-qwen3.5-4b-{ftype}.gguf \
    --outtype bf16 \
    --mtp
```
独立导出主语言模型与MTP头的好处一是主语言模型文件会更小（实测 bf16 格式主模型文件从 8.1G 降到了 7.9G）；二是 MTP 头可采用更激进的量化方式或切换其他 MTP 头。


使用 `llama-server` 运行命令：
```bash
llama-server \
    -m qwen3.5-4b-bf16.gguf \
    --mmproj mmproj-qwen3.5-4b-bf16.gguf \
    --model-draft mtp-qwen3.5-4b-bf16.gguf \
    -ngl 99
```

### 二、Qwen3.5 4B 模型的量化与评测
llama.cpp 提供了 `llama-quantize` 和 `llama-bench` 用于 gguf 格式模型文件量化与评测。实验在 MINIS FORUM X1 Pro-370 上进行，无独立显卡，使用 AMD Radeon 890M Graphics, gfx1150 (0x1150) 核显。

量化命令：
```bash
llama-quantize ./qwen3.5-4b-bf16.gguf qwen3.5-4b-1-Q8_0.gguf Q8_0
```

评测命令：
```bash
llama-bench -m ./qwen3.5-4b-bf16.gguf
```

| 量化格式 | Qwen3.5 4B 量化后大小 | pp512 test | tg128 test |
|---|---|---|---|
| BF16(with MTP) | 8.06G | 523.23 ± 18.49 | 8.90 ± 0.34 |
| BF16(no MTP) | 7.84G | 561.92 ± 9.32 | 8.85 ± 0.23 |
| Q8_0 | 4.16G | 582.34 ± 4.17 | 13.61 ± 0.39 |
| Q6_K | 3.22G | 428.51 ± 2.03 | 18.18 ± 0.19 |
| Q4_K_M | 2.51G | 557.70 ± 2.34 | 23.59 ± 0.05 |
| Q4_K_S | 2.38G | 591.58 ± 2.45 | 24.27 ± 0.18 |


Q8_0, Q6_K等均在 BF16(no MTP)基础上量化。