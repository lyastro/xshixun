# 在集群上下载 HuggingFace 模型

本文档介绍如何在 xshixun 集群的 Pod 中使用 `huggingface_hub` 下载 HuggingFace 模型。

## 背景

xshixun 集群的 Pod **无法直接访问外网**，需要通过代理才能下载模型。本方案通过 IIISConnect 代理 + hf-mirror.com 国内镜像实现高速下载，无需额外安装任何工具。

## 快速开始

### 方法一：在 Deployment/Pod YAML 中配置环境变量

在你的 Pod 或 Deployment 的 `env` 中加入以下环境变量：

```yaml
env:
# HuggingFace 镜像配置
- name: HF_ENDPOINT
  value: "https://hf-mirror.com"
- name: HF_HUB_DISABLE_XET
  value: "1"
- name: HF_HUB_DISABLE_TELEMETRY
  value: "1"
# 代理配置
- name: HTTP_PROXY
  value: "http://iiisconnect-gateway.weixu.svc.cluster.local:3128"
- name: HTTPS_PROXY
  value: "http://iiisconnect-gateway.weixu.svc.cluster.local:3128"
- name: http_proxy
  value: "http://iiisconnect-gateway.weixu.svc.cluster.local:3128"
- name: https_proxy
  value: "http://iiisconnect-gateway.weixu.svc.cluster.local:3128"
- name: no_proxy
  value: "localhost,127.0.0.1,cluster.local"
```

### 方法二：在 Pod 内临时设置

如果你已经有一个运行中的 Pod，可以在 shell 中临时设置：

```bash
export HF_ENDPOINT=https://hf-mirror.com
export HF_HUB_DISABLE_XET=1
export HF_HUB_DISABLE_TELEMETRY=1
export HTTP_PROXY=http://iiisconnect-gateway.weixu.svc.cluster.local:3128
export HTTPS_PROXY=http://iiisconnect-gateway.weixu.svc.cluster.local:3128
export no_proxy=localhost,127.0.0.1,cluster.local
```

## 使用示例

### Python 代码下载模型

```python
import os

# 设置环境变量（如果没有在 Pod YAML 中配置）
os.environ["HF_ENDPOINT"] = "https://hf-mirror.com"
os.environ["HF_HUB_DISABLE_XET"] = "1"
os.environ["HTTP_PROXY"] = "http://iiisconnect-gateway.weixu.svc.cluster.local:3128"
os.environ["HTTPS_PROXY"] = "http://iiisconnect-gateway.weixu.svc.cluster.local:3128"

from huggingface_hub import snapshot_download, hf_hub_download

# 下载整个模型仓库
path = snapshot_download("fishaudio/s2-pro", cache_dir="/data/models")

# 只下载单个文件
path = hf_hub_download("fishaudio/s2-pro", "config.json", cache_dir="/data/models")

# 只下载部分文件（按模式匹配）
path = snapshot_download(
    "fishaudio/s2-pro",
    cache_dir="/data/models",
    allow_patterns=["*.json", "*.txt"],  # 只下载 json 和 txt 文件
)
```

### 使用 transformers 加载模型

```python
import os
os.environ["HF_ENDPOINT"] = "https://hf-mirror.com"
os.environ["HF_HUB_DISABLE_XET"] = "1"
os.environ["HTTP_PROXY"] = "http://iiisconnect-gateway.weixu.svc.cluster.local:3128"
os.environ["HTTPS_PROXY"] = "http://iiisconnect-gateway.weixu.svc.cluster.local:3128"

from transformers import AutoModel, AutoTokenizer

model = AutoModel.from_pretrained("bert-base-uncased", cache_dir="/data/models")
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased", cache_dir="/data/models")
```

### 命令行下载（huggingface-cli）

```bash
# 先设置环境变量（见上文）
huggingface-cli download fishaudio/s2-pro --cache-dir /data/models
```

## 环境变量说明

| 变量 | 值 | 说明 |
|------|---|------|
| `HF_ENDPOINT` | `https://hf-mirror.com` | 使用 hf-mirror.com 国内镜像替代 huggingface.co |
| `HF_HUB_DISABLE_XET` | `1` | **必须设置**。禁用 HuggingFace 新的 xet 存储协议，否则大文件会重定向到 `cas-bridge.xethub.hf.co`（CloudFront CDN），从国内无法访问 |
| `HF_HUB_DISABLE_TELEMETRY` | `1` | 可选。禁用遥测，避免不必要的网络请求 |
| `HTTP_PROXY` / `HTTPS_PROXY` | `http://iiisconnect-gateway.weixu.svc.cluster.local:3128` | IIISConnect 代理地址，Pod 通过此代理访问外网 |
| `no_proxy` | `localhost,127.0.0.1,cluster.local` | 不走代理的地址 |

## 性能参考

| 场景 | 速度 |
|------|------|
| 小文件（config.json 等） | ~1-3 秒/文件 |
| 中等文件（tokenizer.json, 12 MB） | ~2-5 秒 |
| 大文件（model shards, 1.8 GB） | ~7 MB/s（约 4 分钟） |
| 同一文件第二次下载（代理缓存命中） | ~900 MB/s（秒级） |

## 常见问题

### Q: 下载报错 `ConnectTimeout` 或 `SSL handshake timed out`

**A:** 检查是否设置了 `HF_HUB_DISABLE_XET=1`。如果没设置，大文件会被重定向到 CloudFront CDN，从集群无法访问。

### Q: 下载报错 `Distant resource does not seem to be on huggingface.co`

**A:** 确认 `HF_ENDPOINT` 设置为 `https://hf-mirror.com`（不是 `https://huggingface.co`）。

### Q: 下载报错 `ProxyError` 或 `Connection refused`

**A:** 检查 IIISConnect gateway 是否正常运行：
```bash
curl -s http://iiisconnect-gateway.weixu.svc.cluster.local:8000/health
```
应返回 `{"status":"ok","agent_connected":true,...}`。

### Q: 需要下载私有模型（需要 HuggingFace token）

**A:** 在环境变量中设置 `HF_TOKEN`：
```bash
export HF_TOKEN=hf_xxxxx
```
或在代码中：
```python
hf_hub_download("private/model", "file.bin", token="hf_xxxxx")
```

### Q: pip install 也需要代理吗？

**A:** 是的。设置了 `HTTP_PROXY`/`HTTPS_PROXY` 后，pip 会自动走代理。也可以显式指定：
```bash
pip install torch --proxy http://iiisconnect-gateway.weixu.svc.cluster.local:3128
```
