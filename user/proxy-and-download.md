# 在集群上访问外网资源

本文档介绍如何在 xshixun 集群的 Pod 中通过代理访问外网，包括下载 HuggingFace 模型、pip 安装 Python 包、apt-get 安装系统包、npm/wget/curl/git clone 等常见场景。

## 背景

xshixun 集群的 Pod **无法直接访问外网**。所有外网请求都需要通过 IIISConnect 代理完成。IIISConnect 代理会自动将常用资源（HuggingFace、PyPI、GitHub、Ubuntu APT 等）路由到国内高速镜像，无需手动配置镜像源。

## 基础代理配置

### 方法一：在 Deployment/Pod YAML 中配置（推荐）

在你的 Pod 或 Deployment 的 `env` 中加入以下环境变量：

```yaml
env:
# 代理配置（所有外网访问都需要）
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
# HuggingFace 专用配置（如果需要下载 HF 模型）
- name: HF_ENDPOINT
  value: "https://hf-mirror.com"
- name: HF_HUB_DISABLE_XET
  value: "1"
- name: HF_HUB_DISABLE_TELEMETRY
  value: "1"
```

### 方法二：在 Pod 内临时设置

```bash
# 基础代理（所有场景都需要）
export HTTP_PROXY=http://iiisconnect-gateway.weixu.svc.cluster.local:3128
export HTTPS_PROXY=http://iiisconnect-gateway.weixu.svc.cluster.local:3128
export http_proxy=$HTTP_PROXY
export https_proxy=$HTTPS_PROXY
export no_proxy=localhost,127.0.0.1,cluster.local

# HuggingFace 专用（下载 HF 模型时需要）
export HF_ENDPOINT=https://hf-mirror.com
export HF_HUB_DISABLE_XET=1
```

> **提示**：可以把上述 export 命令加到 `~/.bashrc` 中，这样每次进入 Pod 时会自动生效。

---

## 各场景使用方法

### pip install

设置了 `HTTP_PROXY`/`HTTPS_PROXY` 后，pip 会自动走代理。代理会自动将 PyPI 请求路由到清华 TUNA 镜像。

```bash
# 自动走代理（环境变量已设置的情况下）
pip install torch transformers

# 显式指定代理
pip install torch --proxy http://iiisconnect-gateway.weixu.svc.cluster.local:3128

# 如果遇到 SSL 问题，可以同时指定镜像源
pip install torch -i https://pypi.tuna.tsinghua.edu.cn/simple --trusted-host pypi.tuna.tsinghua.edu.cn
```

### apt-get（Ubuntu/Debian）

apt-get 也会读取 `http_proxy`/`https_proxy` 环境变量。代理会自动将 Ubuntu 源路由到清华 TUNA 镜像。

```bash
# 环境变量已设置时，直接使用
apt-get update && apt-get install -y vim curl

# 或者通过 apt 配置文件设置代理（持久化）
echo 'Acquire::http::Proxy "http://iiisconnect-gateway.weixu.svc.cluster.local:3128";' > /etc/apt/apt.conf.d/proxy.conf
echo 'Acquire::https::Proxy "http://iiisconnect-gateway.weixu.svc.cluster.local:3128";' >> /etc/apt/apt.conf.d/proxy.conf
apt-get update && apt-get install -y vim
```

### wget

wget 会自动使用 `http_proxy`/`https_proxy` 环境变量。

```bash
# 自动走代理
wget https://example.com/file.tar.gz

# 显式指定代理
wget -e http_proxy=http://iiisconnect-gateway.weixu.svc.cluster.local:3128 \
     -e https_proxy=http://iiisconnect-gateway.weixu.svc.cluster.local:3128 \
     https://example.com/file.tar.gz
```

### curl

curl 会自动使用 `HTTP_PROXY`/`HTTPS_PROXY` 环境变量，也可以用 `-x` 参数指定。

```bash
# 自动走代理
curl -O https://example.com/file.tar.gz

# 显式指定代理
curl -x http://iiisconnect-gateway.weixu.svc.cluster.local:3128 \
     -O https://example.com/file.tar.gz
```

### git clone

git 会使用 `http_proxy`/`https_proxy` 环境变量。代理会自动将 GitHub releases/archives 路由到 ghproxy 镜像。

```bash
# 自动走代理
git clone https://github.com/user/repo.git

# 或者配置 git 全局代理
git config --global http.proxy http://iiisconnect-gateway.weixu.svc.cluster.local:3128
git config --global https.proxy http://iiisconnect-gateway.weixu.svc.cluster.local:3128
```

### npm install

npm 会使用 `HTTP_PROXY`/`HTTPS_PROXY` 环境变量。代理会自动路由到 npmmirror。

```bash
# 自动走代理
npm install

# 或者显式设置
npm config set proxy http://iiisconnect-gateway.weixu.svc.cluster.local:3128
npm config set https-proxy http://iiisconnect-gateway.weixu.svc.cluster.local:3128
```

### conda install

conda 需要在配置文件中设置代理：

```bash
# 通过环境变量（conda 4.x+ 支持）
# HTTP_PROXY/HTTPS_PROXY 已经设置的情况下，conda 会自动使用

# 或者在 .condarc 中配置
cat >> ~/.condarc << 'EOF'
proxy_servers:
  http: http://iiisconnect-gateway.weixu.svc.cluster.local:3128
  https: http://iiisconnect-gateway.weixu.svc.cluster.local:3128
EOF

conda install numpy
```

---

## HuggingFace 模型下载

### 环境变量说明

除了基础代理配置外，下载 HuggingFace 模型还需要额外设置：

| 变量 | 值 | 说明 |
|------|---|------|
| `HF_ENDPOINT` | `https://hf-mirror.com` | 使用 hf-mirror.com 国内镜像替代 huggingface.co |
| `HF_HUB_DISABLE_XET` | `1` | **必须设置**。禁用 xet 存储协议，否则大文件会重定向到 CloudFront CDN，从国内无法访问 |
| `HF_HUB_DISABLE_TELEMETRY` | `1` | 可选。禁用遥测，避免不必要的网络请求 |

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
    allow_patterns=["*.json", "*.txt"],
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

### 命令行下载

```bash
# 先设置环境变量（见上文"基础代理配置"和"HuggingFace 环境变量"）
huggingface-cli download fishaudio/s2-pro --cache-dir /data/models
```

---

## 自动镜像加速

IIISConnect 代理会自动将以下域名路由到国内高速镜像，用户无需手动配置镜像源：

| 原始域名 | 自动路由到 | 加速内容 |
|---------|-----------|---------|
| `huggingface.co` | `hf-mirror.com` | HuggingFace 模型/数据集 |
| `pypi.org` / `files.pythonhosted.org` | 清华 TUNA PyPI | pip install |
| `github.com`（releases/archives） | ghproxy | GitHub 发布文件 |
| `registry.npmjs.org` | npmmirror | npm 包 |
| `archive.ubuntu.com` | 清华 TUNA mirrors | apt-get |

这意味着你用 `pip install`、`apt-get install`、`npm install` 时，不需要手动切换镜像源，代理会自动完成。

---

## 性能参考

| 场景 | 速度 |
|------|------|
| pip install（常见包） | 通常 < 30 秒 |
| apt-get install | 通常 < 10 秒 |
| HuggingFace 小文件（config.json 等） | ~1-3 秒/文件 |
| HuggingFace 中等文件（12 MB） | ~2-5 秒 |
| HuggingFace 大文件（1.8 GB） | ~7 MB/s（约 4 分钟） |
| 同一文件第二次下载（代理缓存命中） | ~900 MB/s（秒级） |

---

## 常见问题

### Q: HuggingFace 下载报错 `ConnectTimeout` 或 `SSL handshake timed out`

**A:** 检查是否设置了 `HF_HUB_DISABLE_XET=1`。如果没设置，大文件会被重定向到 CloudFront CDN，从集群无法访问。

### Q: HuggingFace 下载报错 `Distant resource does not seem to be on huggingface.co`

**A:** 确认 `HF_ENDPOINT` 设置为 `https://hf-mirror.com`。

### Q: 下载报错 `ProxyError` 或 `Connection refused`

**A:** 检查 IIISConnect gateway 是否正常运行：
```bash
curl -s http://iiisconnect-gateway.weixu.svc.cluster.local:8000/health
```
应返回 `{"status":"ok","agent_connected":true,...}`。如果 gateway 不可用，请联系管理员。

### Q: 需要下载私有 HuggingFace 模型

**A:** 设置 `HF_TOKEN` 环境变量：
```bash
export HF_TOKEN=hf_xxxxx
```

### Q: 某些网站访问超时

**A:** 代理只能加速**国内可达**的域名。如果目标网站被防火墙封锁（如 google.com），代理也无法访问。代理主要解决的是集群 Pod 无外网访问的问题，以及通过国内镜像加速常用资源下载。

### Q: 不想全局设置代理

**A:** 可以只对单条命令设置代理，不影响其他操作：
```bash
# 只对这一条命令生效
HTTP_PROXY=http://iiisconnect-gateway.weixu.svc.cluster.local:3128 \
HTTPS_PROXY=http://iiisconnect-gateway.weixu.svc.cluster.local:3128 \
pip install torch

# 或用 env 命令
env HTTP_PROXY=http://iiisconnect-gateway.weixu.svc.cluster.local:3128 \
    HTTPS_PROXY=http://iiisconnect-gateway.weixu.svc.cluster.local:3128 \
    wget https://example.com/file.tar.gz
```
