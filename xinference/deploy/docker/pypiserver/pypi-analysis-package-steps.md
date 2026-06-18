# PyPI Package Analysis Plan

分析项目中所有需要的 Python package name list。

---

## 步骤 1：定位 model 目录下的 JSON 配置文件

**目标**：找出 `xinference/model/` 下所有类似 `llm_family.json`、包含 `virtualenv.packages` 结构的 JSON 文件。

**结果**：找到 6 个 JSON 文件，全部包含 `virtualenv.packages`：

| # | 文件路径 | virtualenv 出现次数 |
|---|---------|-------------------|
| 1 | `xinference/model/llm/llm_family.json` | ~140 |
| 2 | `xinference/model/embedding/model_spec.json` | 40 |
| 3 | `xinference/model/image/model_spec.json` | 33 |
| 4 | `xinference/model/rerank/model_spec.json` | 14 |
| 5 | `xinference/model/audio/model_spec.json` | 13 |
| 6 | `xinference/model/video/model_spec.json` | 8 |

这些文件中的 `virtualenv.packages` 使用了占位符（placeholder）引用，如 `#transformers_dependencies#`、`#system_numpy#` 等，需要后续步骤解析这些占位符对应的实际包名。

---

## 步骤 2：提取所有 virtualenv.packages 条目，合并去重

**目标**：对 6 个 JSON 文件，提取所有 `virtualenv.packages[]` 条目，合并并去重。

**结果**：共得到 **118 个唯一条目**。分为两类：

### A. 占位符引用（placeholder，以 `#` 包裹）

共 17 个：

| # | 占位符 | 含义 |
|---|--------|------|
| 1 | `#diffusers_dependencies# ; #engine# == "diffusers"` | diffusers 相关依赖 |
| 2 | `#llama_cpp_dependencies# ; #engine# == "llama.cpp"` | llama.cpp 相关依赖 |
| 3 | `#mlx_dependencies# ; #engine# == "MLX"` | MLX 相关依赖 |
| 4 | `#sentence_transformers_dependencies# ; #engine# == "sentence_transformers"` | sentence-transformers 依赖 |
| 5 | `#sglang_dependencies# ; #engine# == "sglang"` | sglang 依赖 |
| 6 | `#sglang_dependencies# ; #engine# == \"sglang\"` | sglang 依赖（转义变体） |
| 7 | `#system_numpy#` | numpy（无条件） |
| 8 | `#system_numpy# ; #engine# == "Transformers"` | numpy（Transformers 引擎） |
| 9 | `#system_numpy# ; #engine# == "vllm"` | numpy（vllm 引擎） |
| 10 | `#system_pandas#` | pandas |
| 11 | `#system_torch#` | torch（无条件） |
| 12 | `#system_torch# ; #engine# == "Transformers"` | torch（Transformers 引擎） |
| 13 | `#system_torch# ; #engine# == "sentence_transformers"` | torch（sentence_transformers 引擎） |
| 14 | `#system_torchaudio#` | torchaudio |
| 15 | `#system_torchvision# ; #engine# == "sentence_transformers"` | torchvision |
| 16 | `#transformers_dependencies# ; #engine# == "Transformers"` | transformers 相关依赖 |
| 17 | `#vllm_dependencies# ; #engine# == "vllm"` | vllm 相关依赖 |

### B. 具体包名（concrete package specs）

共 101 个，按字母排序：

| # | Package Spec | 来源文件 |
|---|-------------|---------|
| 1 | `ChatTTS==0.2.4` | audio |
| 2 | `FlagEmbedding ; #engine# == "flag"` | embedding |
| 3 | `HyperPyYAML` | audio |
| 4 | `Pillow` | embedding, image |
| 5 | `PyMuPDF` | image |
| 6 | `accelerate>=0.28.0` | embedding |
| 7 | `accelerate>=0.28.0 ; #engine# == "Transformers"` | llm |
| 8 | `accelerate>=0.28.0 ; #engine# == "sentence_transformers"` | embedding |
| 9 | `accelerate>=0.34.2 ; #engine# == "Transformers"` | image |
| 10 | `addict` | image |
| 11 | `argbind` | audio |
| 12 | `conformer` | audio |
| 13 | `deepspeed==0.12.3` | image |
| 14 | `diffusers==0.29.0` | audio |
| 15 | `diffusers==0.35.1` | video, image |
| 16 | `diffusers==0.35.1 ; #engine# == "diffusers"` | image |
| 17 | `diffusers>=0.33.0` | video |
| 18 | `easydict` | image |
| 19 | `einops` | embedding, image |
| 20 | `einops ; #engine# == "sentence_transformers"` | embedding |
| 21 | `einops-exts==0.0.4` | image |
| 22 | `einops==0.6.1` | image |
| 23 | `en_core_web_sm@https://...` | audio |
| 24 | `en_core_web_trf@https://...` | audio |
| 25 | `flatten_dict` | audio |
| 26 | `ftfy` | video |
| 27 | `funasr @ git+https://...FunASR@b25472b...` | audio |
| 28 | `gdown` | audio |
| 29 | `git+https://github.com/huggingface/diffusers` | image |
| 30 | `https://...sgl_kernel-0.3.20+cu130...aarch64.whl` | llm |
| 31 | `https://...sgl_kernel-0.3.20+cu130...x86_64.whl` | llm |
| 32 | `httpx==0.24.0` | image |
| 33 | `huggingface-hub<1.0 ; #engine# == "diffusers"` | image |
| 34 | `hydra-core>=1.3.2` | audio |
| 35 | `imageio` | video |
| 36 | `imageio-ffmpeg` | video |
| 37 | `img2pdf` | image |
| 38 | `inflect` | audio |
| 39 | `json5` | audio |
| 40 | `julius` | audio |
| 41 | `librosa` | audio |
| 42 | `lightning>=2.0.0` | audio |
| 43 | `matplotlib` | audio |
| 44 | `mineru-vl-utils[transformers] ; #engine# == "Transformers"` | llm |
| 45 | `mlx-audio==0.2.3` | audio |
| 46 | `mlx-lm==0.24.1` | audio |
| 47 | `mlx-lm>=0.25.2 ; #engine# == "MLX"` | llm |
| 48 | `mlx-vlm ; #engine# == "mlx"` | image |
| 49 | `mlx-vlm>=0.4.4 ; #engine# == "MLX"` | llm |
| 50 | `mlx_dependencies ; #engine# == "MLX"` | llm |
| 51 | `munch` | audio |
| 52 | `onnxruntime>=1.16.0` | audio |
| 53 | `peft==0.4.0` | image |
| 54 | `peft>=0.17.0` | image |
| 55 | `pillow==11.3.0` | image |
| 56 | `pyarrow` | audio |
| 57 | `pyworld>=0.3.4` | audio |
| 58 | `qwen-asr` | audio |
| 59 | `qwen-tts` | audio |
| 60 | `qwen-vl-utils` | llm, embedding |
| 61 | `qwen-vl-utils==0.0.11` | embedding |
| 62 | `qwen-vl-utils>=0.0.14` | embedding |
| 63 | `qwen_omni_utils` | llm |
| 64 | `qwen_omni_utils ; #engine# == "vllm"` | llm |
| 65 | `randomname` | audio |
| 66 | `sentence_transformers` | embedding |
| 67 | `sentence_transformers ; #engine# == "sentence_transformers"` | embedding |
| 68 | `sentencepiece==0.1.99` | image |
| 69 | `sgl_kernel ; #engine# == "sglang" and cuda_version < "13.0"` | llm |
| 70 | `sglang_dependencies ; #engine# == "sglang"` | llm |
| 71 | `soundfile` | llm |
| 72 | `tensorboard` | audio |
| 73 | `tiktoken` | audio |
| 74 | `tiktoken==0.6.0` | image |
| 75 | `timm==0.6.13` | image |
| 76 | `tokenizers` | image |
| 77 | `transformers @ git+https://...transformers@82a06db...` | image |
| 78 | `transformers==4.47.1 ; #engine# == "transformers"` | image |
| 79 | `transformers==4.51.3` | embedding, audio |
| 80 | `transformers==4.52.0` | embedding |
| 81 | `transformers==4.52.0 ; #engine# == "sentence_transformers"` | embedding |
| 82 | `transformers==4.52.1` | audio |
| 83 | `transformers==4.53.2` | audio |
| 84 | `transformers==4.55.0` | image |
| 85 | `transformers==4.57.1 ; #engine# == "Transformers"` | llm |
| 86 | `transformers==4.57.1 ; #engine# == "sentence_transformers"` | embedding |
| 87 | `transformers==4.57.6 ; #engine# == "vllm"` | llm |
| 88 | `transformers>=4.45.0 ; #engine# == "Transformers"` | llm |
| 89 | `transformers>=4.51.0` | image |
| 90 | `transformers>=4.51.3` | image |
| 91 | `transformers>=4.53.2 ; #engine# == "Transformers"` | llm |
| 92 | `transformers>=4.57.0 ; #engine# == "sentence_transformers"` | embedding |
| 93 | `transformers>=5.2.0 ; #engine# == "Transformers"` | llm |
| 94 | `transformers>=5.5.0 ; #engine# == "vllm"` | llm |
| 95 | `transformers_dependencies ; #engine# == "Transformers"` | llm |
| 96 | `vllm==0.21.0 ; #engine# == "vllm"` | llm |
| 97 | `vllm>=0.17.0 ; #engine# == "vllm"` | llm |
| 98 | `vllm>=0.19.0 ; #engine# == "vllm"` | llm |
| 99 | `vllm_dependencies ; #engine# == "vllm"` | llm |
| 100 | `wetext==0.0.9` | audio |
| 101 | `xformers` | embedding |

### 统计

- 占位符引用：17 个
- 具体包名条目：101 个
- 合并去重后总计：**118 个唯一条目**

### 说明

- `;` 之后的部分是条件表达式（如 `#engine# == "vllm"`），用于在运行时按引擎条件过滤
- 带 `#` 包裹的占位符（如 `#transformers_dependencies#`）需要进一步解析为实际包名——这将在后续步骤中处理
- 部分条目为 URL（如 sgl_kernel wheel 链接、git repo 地址），这些也是有效的 pip install 目标

---

## 步骤 3：解析占位符引用 → 实际包名

**目标**：理解 `#transformers_dependencies#`、`#system_numpy#` 等占位符如何解析为实际的 pip package name。

**结果**：占位符分为两类，解析方式不同：

### A. `#xxx_dependencies#` — 引擎依赖宏

定义在 [virtual_env_manager.py:26-64](xinference/core/virtual_env_manager.py#L26-L64) 的 `ENGINE_VIRTUALENV_PACKAGES` 字典中。

展开逻辑在 [virtual_env_manager.py:182-207](xinference/core/virtual_env_manager.py#L182-L207) 的 `expand_engine_dependency_placeholders()`：

1. 去掉 `#` 包裹和 `_dependencies` 后缀，提取引擎名（如 `transformers`、`vllm`）
2. 处理异常映射：`llama_cpp` → `llama.cpp`
3. 仅当引擎名匹配当前 `model_engine` 时才展开
4. 展开为预定义的包列表

| 占位符（去掉 `#`）               | 展开后的 pip 包                                                                                           |
|---------------------------------|----------------------------------------------------------------------------------------------------------|
| `transformers_dependencies`     | `transformers>=4.53.3`, `accelerate>=0.28.0`                                                             |
| `vllm_dependencies`             | `vllm>=0.11.2`                                                                                           |
| `sglang_dependencies`           | `pybase64`, `zmq`, `partial_json_parser`, `sentencepiece`, `dill`, `ninja`, `numpy>=2.4.1`, `sglang>=0.5.6`, `sgl_kernel` (带条件) |
| `sentence_transformers_dependencies` | `sentence_transformers`, `einops`, `transformers>=4.53.3`, `accelerate>=0.28.0`                      |
| `diffusers_dependencies`        | `diffusers>=0.32.0`, `huggingface-hub<1.0`                                                               |
| `mlx_dependencies`              | `mlx-lm>=0.24.0`                                                                                         |
| `llama_cpp_dependencies`        | `xllamacpp>=0.2.6`                                                                                       |

### B. `#system_xxx#` — 系统包占位符

定义在 [utils.py:355-361](xinference/core/utils.py#L355-L361) 的 `system_markers` 集合中。

这些占位符**不会被展开**为固定版本。它们被保留原样穿过 `filter_virtualenv_packages_by_markers()`，最终由 **xoscar 外部库**的 `install_packages()` 解析为**匹配系统已安装版本**的实际包名：

| 占位符               | 解析结果                   |
|---------------------|--------------------------|
| `#system_torch#`     | `torch`（匹配系统版本）       |
| `#system_torchaudio#` | `torchaudio`（匹配系统版本）  |
| `#system_torchvision#` | `torchvision`（匹配系统版本） |
| `#system_torchcodec#` | `torchcodec`（匹配系统版本）  |
| `#system_numpy#`     | `numpy`（匹配系统版本）       |
| `#system_pandas#`    | `pandas`（匹配系统版本）       |

### C. 完整解析流水线

JSON 文件（`virtualenv.packages`）中的条目经过 5 个阶段处理：

```
JSON virtualenv.packages
        │
        ▼
[1] expand_engine_dependency_placeholders()
    解析 #xxx_dependencies# → 展开为实际包列表
        │
        ▼
[2] PyTorch CUDA auto-configuration
    检测系统 PyTorch CUDA 版本，配置 extra_index_url
        │
        ▼
[3] merge_virtual_env_packages()
    合并基础包 + 用户额外包（同名覆盖）
        │
        ▼
[4] filter_virtualenv_packages_by_markers()
    按 engine / cuda_version / platform 条件过滤
    保留 #system_*# 占位符
        │
        ▼
[5] xoscar install_packages()
    将 #system_xxx# 解析为系统版本匹配的实际包名
    pip install 到隔离 virtualenv
```

### 涉及的核心脚本

| 文件 | 函数 / 变量 | 作用 |
|------|------------|------|
| [virtual_env_manager.py](xinference/core/virtual_env_manager.py) | `ENGINE_VIRTUALENV_PACKAGES` (L26) | 定义引擎依赖映射表 |
| [virtual_env_manager.py](xinference/core/virtual_env_manager.py) | `expand_engine_dependency_placeholders()` (L182) | 展开 `#xxx_dependencies#` |
| [utils.py](xinference/core/utils.py) | `filter_virtualenv_packages_by_markers()` (L342) | 条件过滤 + 保留 `#system_*#` |
| [utils.py](xinference/core/utils.py) | `merge_virtual_env_packages()` (L254) | 合并去重 |
| [worker.py](xinference/core/worker.py) | `_prepare_virtual_env()` (L1687) | 主流程编排 |
| xoscar（外部库）     | `install_packages()`          | 解析 `#system_xxx#` 并 pip install |

### 统计

- `#xxx_dependencies#` 引擎宏：**7 个**（含 `llama_cpp` → `llama.cpp` 映射）
- `#system_xxx#` 系统占位符：**6 个**（`torch`, `torchaudio`, `torchvision`, `torchcodec`, `numpy`, `pandas`）
- 注意：`#system_pandas#` 只在 `system_markers` 中被识别保留，不在解析集合中

---

## 步骤 4：不同系统架构（amd64 / arm64）的包下载差异

**目标**：分析在 amd64 和 arm64 架构下，下载的 Python 包是否不同，以及现有代码中如何处理架构差异。

**结果**：不同架构下载的包**会不同**，但显式架构判断逻辑非常有限——大部分依赖 pip 的 wheel tag 自动匹配机制。

### A. 显式架构判断（xinference 代码层面）

#### A1. `filter_virtualenv_packages_by_markers()` — `platform_machine` 条件过滤

定义在 [utils.py:387-395](xinference/core/utils.py#L387-L395)，使用 Python 的 `platform.machine()` 判断当前系统架构：

```
platform_machine == "x86_64"  →  非 x86_64 机器上跳过该包
platform_machine == "aarch64" →  非 aarch64 机器上跳过该包
```

只支持 `==` 运算符，不支持 `!=` 或其他比较。

#### A2. `ENGINE_VIRTUALENV_PACKAGES` — sglang kernel wheel 二选一

定义在 [virtual_env_manager.py:36-38](xinference/core/virtual_env_manager.py#L36-L38)，sglang 引擎针对 CUDA 13.0 提供了两个不同架构的预编译 wheel URL：

| 条件                                                       | wheel 文件                                                  |
|-----------------------------------------------------------|------------------------------------------------------------|
| `cuda_version == "13.0" and platform_machine == "x86_64"`  | `sgl_kernel-0.3.21+cu130-cp310-abi3-manylinux2014_x86_64.whl`  |
| `cuda_version == "13.0" and platform_machine == "aarch64"` | `sgl_kernel-0.3.21+cu130-cp310-abi3-manylinux2014_aarch64.whl` |
| `cuda_version < "13.0"`                                    | `sgl_kernel`（通用包名，pip 自行解析）                           |

`llm_family.json` 中存在相同模式的条目（v0.3.20 版本），JSON 配置与 Python 代码默认值重复定义。

### B. 隐式架构处理（pip 层面）

对于**绝大多数包**，xinference 不做显式架构判断，而是依赖 pip 安装时自动选择：

#### B1. 包名不带架构信息

如 `vllm>=0.11.2`、`transformers>=4.53.3`、`numpy>=2.4.1` 等，pip 根据当前平台的 wheel tag 自动匹配：

| 平台       | pip 匹配的 wheel tag 示例              |
|-----------|--------------------------------------|
| Linux x86_64 | `manylinux2014_x86_64` / `manylinux_2_28_x86_64` |
| Linux aarch64 | `manylinux2014_aarch64` / `manylinux_2_28_aarch64` |
| macOS arm64   | `arm64` / `universal2`                            |

#### B2. `extra_index_url` 是 PEP 503 索引

`virtual_env_manager.py` 中配置的额外索引 URL：

| 引擎     | extra_index_url                                |
|---------|------------------------------------------------|
| vllm    | `https://wheels.vllm.ai/0.19.0/cu130`          |
| vllm    | `https://download.pytorch.org/whl/cu130`       |
| sglang  | `https://download.pytorch.org/whl/cu130`       |

这些是 PEP 503 simple index，索引页面列出**所有架构**的 wheel。pip 根据本机 `platform.machine()` 自动选择匹配的 wheel——**xinference 不需要显式关心架构**。

#### B3. PyTorch CUDA wheel 自动配置

[worker.py:1737-1781](xinference/core/worker.py#L1737-L1781)：当包列表包含 `#system_torch#` 等占位符时，检测系统已安装的 PyTorch CUDA 版本（从 `torch` 的 version string 如 `2.5.0+cu130` 中提取 CUDA 后缀），自动配置对应的 `extra_index_url`。

`PYTORCH_CUDA_WHEEL_URLS` 映射表 ([virtual_env_manager.py:83-87](xinference/core/virtual_env_manager.py#L83-L87))：

| CUDA 后缀 | URL                                   |
|----------|---------------------------------------|
| cu130    | `https://download.pytorch.org/whl/cu130` |
| cu129    | `https://download.pytorch.org/whl/cu129` |
| cu128    | `https://download.pytorch.org/whl/cu128` |

> **注意**：这是 CUDA 版本判断，不是架构判断。但不同架构 + 相同 CUDA 版本会从同一索引下载**不同**的 wheel（索引同时包含 x86_64 和 aarch64 wheel）。

### C. CUDA 版本兼容性检查

[worker.py:1783-1799](xinference/core/worker.py#L1783-L1799) 在过滤前检测系统 CUDA 版本：

```python
from xoscar.virtualenv.platform import get_cuda_version
cuda_version = get_cuda_version()
```

通过 `is_cuda_compatible()` 检查 `extra_index_url` 中的 CUDA 版本是否与系统 CUDA 匹配。不匹配则**清除 extra_index_url 和 index_strategy**，退回到默认 PyPI 索引。

此检查与架构无关——仅比较 CUDA 版本字符串。

### D. 不随架构变化的包

以下纯 Python 包（wheel tag 为 `py3-none-any`）在所有架构上**下载完全相同的 wheel**：

- `transformers`, `diffusers`, `accelerate`, `huggingface-hub`
- `sentence_transformers`, `einops`, `tokenizers`, `tiktoken`
- `pybase64`, `zmq`, `partial_json_parser`, `dill`, `ninja`
- `sentencepiece`, `Pillow`（有平台 wheel 但 API 相同）
- `peft`, `ftfy`, `httpx`, `PyMuPDF`, `Pillow`
- JSON 中所有 `transformers==X.Y.Z`、`diffusers==X.Y.Z` 等纯 Python 条目

### E. 按包分类总结

| 包                         | amd64 vs arm64 下载不同? | 判断方式                              |
|---------------------------|------------------------|-------------------------------------|
| `sgl_kernel` (CUDA 13.0)  | ✅ 不同 wheel URL        | `platform_machine` 显式判断 + 条件过滤     |
| `torch` / `torchaudio` / `torchvision` / `torchcodec` | ✅ 不同 wheel   | pip 自动 + `extra_index_url` CUDA 检测 |
| `vllm`                    | ✅ 不同 wheel            | pip 自动从 `wheels.vllm.ai` 索引选      |
| `sglang`                  | ✅ 不同 wheel            | pip 自动从 `download.pytorch.org` 选   |
| `numpy`, `onnxruntime` 等 C 扩展 | ✅ 不同 wheel       | pip 自动根据 wheel tag 匹配             |
| `xllamacpp`               | ✅ 不同 wheel            | pip 自动                              |
| `transformers`, `diffusers` 等纯 Python | ❌ 相同 wheel   | `py3-none-any` tag                   |

### F. 注意事项

1. **`platform_machine` 只支持 `==`**：[utils.py:387-395](xinference/core/utils.py#L387-L395) 的过滤逻辑使用 `in` 字符串匹配，只处理 `"x86_64"` 和 `"aarch64"` 两种值。无法表达 `platform_machine != "x86_64"` 这类逻辑。

2. **sglang JSON 与 Python 代码重复定义**：`llm_family.json` 中 sglang 模型的 `virtualenv.packages` 直接写入了 sgl_kernel wheel URL + `platform_machine` 条件，而 `ENGINE_VIRTUALENV_PACKAGES["sglang"]` 也定义了相同内容。JSON 中的条目会先被处理，然后在 `expand_engine_dependency_placeholders()` 中再次展开——两边的版本号可能不一致（JSON 中是 v0.3.20，Python 代码中是 v0.3.21）。

3. **macOS / Windows 无架构判断**：现有 `platform_machine` 判断仅区分 `x86_64` 和 `aarch64`，适用于 Linux。macOS 上 `platform.machine()` 返回 `arm64`（Apple Silicon）或 `x86_64`（Intel），但代码中**没有 `arm64` 分支**——意味着带 `platform_machine == "aarch64"` 条件的包在 Apple Silicon Mac 上会被过滤掉。Windows 同理无特殊处理。

4. **PyPI 镜像服务器需同时缓存多架构 wheel**：构建 PyPI 镜像时，同一个包名 + 版本号可能对应多个不同架构的 wheel 文件。例如 `torch-2.5.0+cu130` 可能同时有 `manylinux2014_x86_64.whl` 和 `manylinux2014_aarch64.whl`。镜像服务器需要保留**全部架构**的 wheel，否则某些架构的 pip install 会失败。

5. **`extra_index_url` 中的索引 URL 必须对所有目标架构可用**：`https://download.pytorch.org/whl/cu130` 同时提供 x86_64 和 aarch64 wheel，但 `https://wheels.vllm.ai/0.19.0/cu130` 需要确认是否提供 aarch64 wheel。如果某个索引不包含某架构的 wheel，pip 会回退到 PyPI（可能找不到匹配的 CUDA 版本）。

6. **`is_cuda_compatible()` 可能导致丢失 extra_index_url**：如果系统 CUDA 版本与引擎默认的 extra_index_url 中的 CUDA 版本不匹配，会清空 extra_index_url。无 GPU 环境（`cuda_version = None`）也会触发清空。此时所有包走默认 PyPI 索引，可能安装 CPU-only 版本。

---

