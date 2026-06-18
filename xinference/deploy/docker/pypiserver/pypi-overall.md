# PyPI Offline Server — 全局方案

## 目标

在**无网络**的服务器上运行 xinference，当 xinference 为不同模型/引擎创建 virtualenv 时，能从**本地 pypiserver** 下载所有需要的 Python 包。

## 背景分析

参见 [pypi-plan.md](pypi-plan.md)：

- 118 个唯一条目（17 个占位符 + 101 个具体包）分布在 6 个 JSON 文件中
- 5 阶段解析流水线：展开 → CUDA 自动配置 → 合并 → 过滤 → 安装
- 架构差异：显式判断仅 sgl_kernel，其余依赖 pip 自动选择
- `#system_xxx#` 占位符由 xoscar 外部库解析

核心挑战：xinference 的包需求是**动态的**——不同引擎 × 不同 CUDA 版本 × 不同架构 × 不同 Python 版本 × 不同模型，产生的包集合完全不同。

---

## 第一阶段：确定覆盖范围（Scope Definition）

### 需要明确的变量

| 变量 | 可能选项 | 对包列表的影响 |
|------|---------|--------------|
| 目标架构 | amd64 / arm64 / 两者 | 每个 C 扩展包 × 2 的 wheel |
| CUDA 版本 | cu128 / cu129 / cu130 / 无 GPU | torch/vllm/sglang wheel 完全不同 |
| Python 版本 | 3.10 / 3.11 / 3.12 / 3.13 | 部分 wheel 限制 cp310/cp311/... |
| 引擎范围 | vllm / sglang / transformers / mlx / diffusers / llama.cpp / sentence_transformers | 决定占位符展开后的包集合 |
| 操作系统 | Linux only / 含 macOS（mlx 引擎仅 macOS） | mlx-lm 仅 macOS |

### 关键决策 1：覆盖范围

> **建议**：以目标服务器的实际配置为准（如 amd64 + CUDA 13.0 + Python 3.12），否则组合爆炸（3 CUDA × 2 架构 × 4 Python ≈ 24 套环境）。

### 输出物

一份明确的覆盖矩阵表，例如：

| 维度 | 覆盖值 |
|------|-------|
| 架构 | amd64 |
| CUDA | cu130 |
| Python | 3.12 |
| 引擎 | vllm, sglang, transformers, sentence_transformers |
| OS | Linux |

---

## 第二阶段：确定完整包列表（Complete Package Enumeration）

### 2.1 展开占位符

对每个目标引擎，执行 `expand_engine_dependency_placeholders()` 的等价逻辑：

| 占位符 | 引擎 | 展开结果 |
|--------|------|---------|
| `#transformers_dependencies#` | Transformers | `transformers>=4.53.3`, `accelerate>=0.28.0` |
| `#vllm_dependencies#` | vllm | `vllm>=0.11.2` |
| `#sglang_dependencies#` | sglang | `pybase64`, `zmq`, `partial_json_parser`, `sentencepiece`, `dill`, `ninja`, `numpy>=2.4.1`, `sglang>=0.5.6`, sgl_kernel（带条件） |
| `#sentence_transformers_dependencies#` | sentence_transformers | `sentence_transformers`, `einops`, `transformers>=4.53.3`, `accelerate>=0.28.0` |
| `#diffusers_dependencies#` | diffusers | `diffusers>=0.32.0`, `huggingface-hub<1.0` |
| `#mlx_dependencies#` | MLX | `mlx-lm>=0.24.0` |
| `#llama_cpp_dependencies#` | llama.cpp | `xllamacpp>=0.2.6` |

### 2.2 按目标环境过滤条件表达式

对每个带条件的条目，用目标环境参数求值：

- `; #engine# == "vllm"` → 当前引擎匹配则保留，不匹配则移除
- `; cuda_version >= "13.0"` → CUDA 13.0 匹配则保留
- `; platform_machine == "x86_64"` → amd64 匹配则保留

匹配的包去掉条件部分，不匹配的直接移除。最终得到一个**无条件**的包列表。

### 2.3 处理 `#system_xxx#` 占位符

这些占位符由 xoscar 解析为**与系统已安装版本匹配**的实际包。需要在打包时预先确定目标服务器上的版本：

| 占位符 | 解析结果示例 |
|--------|------------|
| `#system_torch#` | `torch==2.5.0+cu130` |
| `#system_torchaudio#` | `torchaudio==2.5.0+cu130` |
| `#system_torchvision#` | `torchvision==0.20.0+cu130` |
| `#system_torchcodec#` | `torchcodec==0.1.0+cu130` |
| `#system_numpy#` | `numpy==1.26.4` |
| `#system_pandas#` | `pandas==2.2.0` |

> **关键问题**：离线服务器上这些包是**已预装在系统 Python 中**，还是也需要从 pypiserver 安装？
>
> **建议**：即使系统已安装，也全部放入 pypiserver——`#system_xxx#` 的语义是"安装与系统相同版本的包到 virtualenv 中"，virtualenv 内 `pip install` 仍需从 pypiserver 获取。

### 2.4 发现传递依赖（Transitive Dependencies）

**这是最容易被低估的一步**。上述分析只覆盖了 xinference JSON 中**显式列出的直接依赖**。但 `pip install transformers` 会连带安装 `numpy`, `packaging`, `filelock`, `pyyaml`, `regex`, `requests`, `tokenizers`, `tqdm`, `safetensors`, `huggingface-hub` 等十几个传递依赖。

**方案对比**：

| 方案 | 做法 | 优点 | 缺点 |
|------|------|------|------|
| A: `pip download` | 在有网环境对每个包执行 `pip download pkg -d wheels/` | ✅ 自动解析传递依赖 | 需要同架构 + 同 Python 版本的参考机器 |
| B: `pip-compile` | 对每组依赖生成 `requirements.in`，`pip-compile` 锁定 | 产生确定性锁定文件 | 需要额外工具；对 URL/git 依赖支持有限 |
| C: 实际安装 + `pip freeze` | 创建 virtualenv，安装所有直接依赖，`pip freeze > locked.txt` | 最准确 | 需要每个引擎组合实际做一次安装 |
| D: 静态分析 | 解析每个包的 `setup.cfg`/`pyproject.toml` | 不需要安装 | 不准确（版本范围、条件依赖、extras） |

> **推荐方案 A**：在有网络的参考机器上（与目标服务器同架构、同 Python 版本），对每组引擎的最终包列表执行 `pip download --dest wheels/`，自动收集所有传递依赖。

### 关键决策 2：传递依赖是否放入镜像？

> **必须放入**。离线环境下 `pip install` 无 PyPI 可回退，任何缺失的传递依赖都会导致安装失败。

### 输出物

- 每个引擎组合的最终锁定包列表（直接依赖 + 传递依赖）
- wheel 文件命名清单

---

## 第三阶段：处理特殊来源的包（Special Sources）

### 3.1 Git 仓库安装（`git+https://...`）

JSON 中存在以下 git 来源包：

- `funasr @ git+https://...FunASR@b25472b...`
- `transformers @ git+https://...transformers@82a06db...`
- `git+https://github.com/huggingface/diffusers`

**处理**：

1. clone 仓库，checkout 指定 commit
2. 用 `pip wheel` 构建为 wheel 文件
3. 将构建好的 wheel 放入 pypiserver
4. 修改 xinference JSON 或代码：将 `git+https://` 引用替换为 `package==version`

### 3.2 直接 URL wheel

JSON 和 Python 代码中的 sgl_kernel wheel 是**直接指向特定文件的 URL**（不是 PEP 503 索引）：

```
https://github.com/sgl-project/whl/releases/download/v0.3.20/
  sgl_kernel-0.3.20+cu130-cp310-abi3-manylinux2014_x86_64.whl
```

**处理**：

1. 直接下载这些 wheel 文件
2. 放入 pypiserver
3. **修改代码**：将 URL 替换为 `sgl_kernel==0.3.20+cu130`，否则 pip 仍会尝试访问外网 URL（`--index-url` 对直接 URL 无效）

### 3.3 外部索引的包（extra_index_url）

vllm 从 `https://wheels.vllm.ai/` 下载，sglang/torch 从 `https://download.pytorch.org/whl/` 下载。这些包在 PyPI 上可能不存在或版本不同。

**处理**：

1. 从这些外部索引下载所有需要的 wheel
2. 统一放入 pypiserver

### 关键决策 3：如何让 pip 找到本地 pypiserver 上的包

> **推荐**：所有包统一放入一个 pypiserver，去掉 xinference 中的 `extra_index_url` 配置（或指向 local pypiserver）。不推荐让 local pypiserver 模拟多个上游索引。

---

## 第四阶段：构建 Wheel 仓库（Wheel Repository）

### 4.1 下载所有 wheel

对每组（引擎, CUDA, 架构, Python 版本），在有网参考机器上执行：

```bash
# 创建临时 virtualenv
python3.12 -m venv /tmp/venv

# 下载所有直接 + 传递依赖的 wheel
pip download \
  --dest wheels/ \
  --platform manylinux2014_x86_64 \
  --python-version 312 \
  --only-binary=:all: \
  -r requirements.txt

# 需要从特殊索引获取的包
pip download \
  --dest wheels/ \
  --extra-index-url https://download.pytorch.org/whl/cu130 \
  -r requirements-cuda.txt
```

### 4.2 构建缺失的 wheel

对于只有源码分发的包（sdist-only，无预编译 wheel），需要在目标架构上编译：

```bash
pip wheel --wheel-dir wheels/ <package>
```

### 4.3 目录结构

```
wheels/
├── common/            # 纯 Python 包（跨架构，py3-none-any）
│   ├── transformers-4.53.3-py3-none-any.whl
│   ├── accelerate-0.28.0-py3-none-any.whl
│   └── ...
├── cu130/
│   ├── amd64/         # CUDA 13.0 + x86_64
│   │   ├── torch-2.5.0+cu130-cp312-cp312-manylinux2014_x86_64.whl
│   │   ├── vllm-0.11.2-cp312-cp312-manylinux2014_x86_64.whl
│   │   └── ...
│   └── arm64/         # CUDA 13.0 + aarch64（如需要）
│       └── ...
└── cpu/               # 无 GPU 版本（如需要）
    └── ...
```

> pypiserver 也可以直接从扁平目录服务所有 wheel（pip 根据 wheel tag 自动选择），分层结构仅便于管理。

---

## 第五阶段：选择和部署 PyPI Server

### 5.1 技术选型

| 方案 | 特点 | 适用性 |
|------|------|--------|
| **pypiserver** | 轻量，纯静态文件服务 + 自动 PEP 503 索引 | ✅ 推荐：简单、离线场景 |
| devpi | 功能全（缓存代理、多索引、ACL） | 多级索引 / 团队协作 |
| bandersnatch | PyPI 全量镜像 | ❌ 全量 TB 级别，不合适 |
| 自建静态 HTTP + 手动索引 | 完全可控 | 灵活但维护成本高 |

> **推荐 pypiserver**：`pip install pypiserver && pypi-server run -p 8080 /data/packages/`

### 5.2 启动命令

```bash
pypi-server run \
  --port 8080 \
  --overwrite \
  /data/packages/
```

### 5.3 Docker 化

```dockerfile
FROM python:3.12
COPY wheels/ /data/packages/
RUN pip install pypiserver
EXPOSE 8080
CMD ["pypi-server", "run", "-p", "8080", "--overwrite", "/data/packages/"]
```

---

## 第六阶段：修改 Xinference 适配离线环境

**这是整个方案中最关键的一步**——即使 pypiserver 有所需的包，xinference 也必须知道去那里找。

### 6.1 需要修改的位置

| 文件 | 行号 | 当前行为 | 需要改为 |
|------|------|---------|---------|
| `virtual_env_manager.py` | 66-74 | `ENGINE_VIRTUALENV_EXTRA_INDEX_URLS` 指向外网 | 指向 local pypiserver |
| `virtual_env_manager.py` | 83-87 | `PYTORCH_CUDA_WHEEL_URLS` 指向 download.pytorch.org | 指向 local pypiserver |
| `virtual_env_manager.py` | 36-37 | sglang 依赖中包含 GitHub Releases 的直接 URL | 改为 `sgl_kernel==0.3.21+cu130` |
| `llm_family.json` | 多处 | sgl_kernel GitHub Releases URL | 改为 `sgl_kernel==0.3.20+cu130` |
| `worker.py` | 1783-1788 | `get_cuda_version()` 依赖系统 CUDA | 在离线环境下仍需返回正确值 |
| `worker.py` | 1790-1795 | `is_cuda_compatible()` 不匹配时清除 index | 离线模式下跳过检查或改为检查 pypiserver 可用性 |
| xoscar（外部库） | — | `install_packages()` 解析 `#system_xxx#` 并 pip install | 确保 pip 从 local pypiserver 获取 |

### 6.2 方案 A：pip 全局配置（最小侵入）

不修改 xinference 代码，利用 pip 配置层级：

```ini
# ~/.pip/pip.conf 或 /etc/pip.conf
[global]
index-url = http://local-pypiserver:8080/simple/
```

或环境变量：

```bash
export PIP_INDEX_URL=http://local-pypiserver:8080/simple/
export PIP_NO_INDEX=true      # 禁用默认 PyPI
```

> **局限**：JSON 和 Python 代码中的**直接 URL**（如 `https://github.com/.../sgl_kernel-xxx.whl`）不受 `index-url` 影响——pip 仍会尝试访问外网。这些 URL 必须单独处理。

### 6.3 方案 B：修改 xinference 代码（彻底）

添加配置点 `XINFERENCE_PYPI_SERVER_URL`：

1. 当设置此变量时：
   - `ENGINE_VIRTUALENV_EXTRA_INDEX_URLS` → 替换为 `[local_server/simple/]`
   - `PYTORCH_CUDA_WHEEL_URLS` → 替换为 `[local_server/simple/]`
   - sgl_kernel 的直接 URL → 替换为 `sgl_kernel==0.3.21+cu130`（走索引）
2. `is_cuda_compatible()` 在离线模式下跳过兼容性检查（或改为检查 pypiserver 中是否有所需 CUDA 版本的包）
3. JSON 中的 GitHub URL 统一处理：添加预处理步骤，检测到直接 URL 时转换为包名引用

### 6.4 `#system_xxx#` 的离线兼容

xoscar 的 `install_packages()` 解析 `#system_xxx#` 时，需要确保：

1. 系统 Python 中已安装对应的包（如 `torch`, `numpy`）
2. `importlib.metadata.version(pkg)` 能返回正确版本
3. local pypiserver 中存在该版本的 wheel

---

## 第七阶段：验证与测试

### 7.1 单引擎 pip install 验证

对每个引擎，在离线容器中模拟 virtualenv 创建：

```bash
docker run --network none --rm -it test-image

python -m venv /tmp/test-venv
source /tmp/test-venv/bin/activate
pip install \
  --index-url http://localhost:8080/simple/ \
  --no-deps \
  vllm>=0.11.2 transformers>=4.53.3 ...
```

验证点：
- 所有直接依赖安装成功
- 传递依赖无缺失错误
- 无外网连接尝试

### 7.2 端到端验证

1. 启动 pypiserver 容器（生产模式）
2. 启动 xinference worker 容器（配置 `PIP_INDEX_URL`）
3. 实际 launch 模型（如 `qwen2.5:0.5b`）
4. 验证：
   - virtualenv 创建日志无错误
   - 所有包从 local pypiserver 下载（非外网）
   - 模型加载成功
   - 推理正常

### 7.3 多架构验证

如果覆盖多种架构，在对应架构机器（或 QEMU 模拟）上分别验证。

---

## 全局阶段总览

```
Phase 1: Scope Definition
  └── 确定目标架构 / CUDA / Python / 引擎覆盖矩阵

Phase 2: Complete Package List
  ├── 2.1 展开 #xxx_dependencies# 占位符
  ├── 2.2 按目标环境过滤条件表达式
  ├── 2.3 处理 #system_xxx#（确定系统包版本）
  └── 2.4 发现传递依赖（pip download 自动收集）

Phase 3: Special Source Handling
  ├── 3.1 Git 仓库 → pip wheel 构建为 wheel
  ├── 3.2 直接 URL → 下载 + 改代码引用为包名
  └── 3.3 外部索引 → 下载到本地 pypiserver

Phase 4: Wheel Repository
  ├── 4.1 pip download 下载所有 wheel（含传递依赖）
  ├── 4.2 编译缺失的 sdist 包
  └── 4.3 组织目录结构（按架构/CUDA 分层）

Phase 5: PyPI Server Setup
  ├── 5.1 选型 pypiserver
  ├── 5.2 PEP 503 索引自动生成
  └── 5.3 Docker 镜像化

Phase 6: Xinference Adaptation
  ├── 6.1 添加 XINFERENCE_PYPI_SERVER_URL 配置点
  ├── 6.2 替换 extra_index_url + wheel URL
  └── 6.3 #system_xxx# + is_cuda_compatible 离线兼容

Phase 7: Validation
  ├── 7.1 单引擎 pip install 验证
  ├── 7.2 端到端模型 launch 验证
  └── 7.3 多架构验证（如需要）
```

---

## 关键风险与未决问题

| # | 风险 | 严重程度 | 建议 |
|---|------|---------|------|
| 1 | **传递依赖遗漏**导致离线 install 失败 | 🔴 高 | 用 `pip download` 自动收集，不要手动列举 |
| 2 | **JSON 中的硬编码 URL**（sgl_kernel wheel）绕过 index 配置 | 🔴 高 | 必须修改代码 + JSON，或用 nginx 拦截重定向 |
| 3 | `#system_xxx#` 依赖系统已安装包的版本——离线服务器上是否预装？ | 🟡 中 | 预先在系统 Python 中装好 torch/numpy；同时放入 pypiserver |
| 4 | vllm/sglang 版本更新频繁——镜像多久更新？ | 🟡 中 | 设计增量更新机制（只下载新增/变化的包） |
| 5 | 不同引擎要求同一包的不同版本（如 transformers==4.57.1 vs >=4.53.3） | 🟡 中 | pypiserver 保留所有版本；每个引擎独立 virtualenv 互不冲突 |
| 6 | xoscar `install_packages()` 在离线环境下未经验证 | 🟡 中 | 需要 audit xoscar 的实现 |
| 7 | macOS mlx 引擎的包是否需要纳入（仅限 macOS） | 🟢 低 | 如果目标服务器是 Linux-only，可排除 mlx |
| 8 | `is_cuda_compatible()` 在无 GPU / 离线条件下清空 extra_index_url | 🟡 中 | 离线模式下需修改此行为 |
