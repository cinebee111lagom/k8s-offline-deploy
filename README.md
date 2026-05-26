# K8S 离线安装套件

> 海外构建 → 跨境传输 → 国内断网部署的全自动三阶段流水线

## 架构概览

```
[ GitHub Actions (海外) ]
        │
        ├── 下载 kubeadm/kubectl/kubelet 二进制
        ├── 拉取所有 K8S 核心镜像（apiserver/etcd/coredns 等）
        ├── 下载 containerd/runc/crictl
        ├── 下载 CNI 插件镜像（Calico/Flannel）
        ├── 下载系统依赖 .deb/.rpm 包（socat/conntrack）
        └── 打包 → k8s-offline-{version}-{arch}.tar.gz
                        │
              ┌─────────┴─────────┐
              │ SCP 直传           │ OSS 中转
              │ (带宽够用时)       │ (带宽受限时)
              └─────────┬─────────┘
                        ▼
        [ 国内云服务器 (断网环境) ]
                        │
                        ├── 解压离线包
                        ├── 配置系统内核参数
                        ├── 离线安装 containerd
                        ├── 导入所有镜像（ctr import）
                        ├── 安装 K8S 二进制
                        └── kubeadm init（--imagePullPolicy=Never）
```

## 快速开始

### 第一步：Fork 本仓库并配置 Secrets

在 GitHub 仓库的 **Settings → Secrets and variables → Actions** 中添加以下 Secret：

| Secret 名称 | 说明 | 示例 |
|---|---|---|
| `SSH_PRIVATE_KEY` | 用于登录国内服务器的 SSH 私钥（完整内容） | `-----BEGIN OPENSSH PRIVATE KEY-----...` |
| `SSH_USER` | SSH 登录用户名（需有 sudo/root 权限） | `root` 或 `ubuntu` |
| `OSS_ENDPOINT` | （OSS 模式）阿里云 OSS 接入点 | `oss-cn-hangzhou.aliyuncs.com` |
| `OSS_ACCESS_KEY_ID` | （OSS 模式）AccessKey ID | `LTAI5t...` |
| `OSS_ACCESS_KEY_SECRET` | （OSS 模式）AccessKey Secret | `xxxxxxxx` |
| `OSS_BUCKET` | （OSS 模式）存储桶名称 | `my-k8s-bucket` |

### 第二步：配置目标服务器

编辑 `hosts.txt`，每行填写一个国内服务器的公网 IP：

```
1.2.3.4
5.6.7.8
```

确保 SSH 公钥已添加到所有服务器的 `~/.ssh/authorized_keys`。

### 第三步：触发工作流

在 GitHub Actions 页面点击 **"Run workflow"**，填写参数：

| 参数 | 说明 | 默认值 |
|---|---|---|
| `k8s_version` | Kubernetes 版本 | `v1.28.2` |
| `arch` | 目标服务器架构 | `amd64` |
| `cni_plugin` | 网络插件 | `calico` |
| `transfer_method` | 传输方式 | `scp` |
| `install_after_transfer` | 传输后自动安装 | `true` |

---

## 传输方式选择

### 方式一：SCP 直传（推荐带宽 ≥ 10Mbps）

直接从 GitHub Actions 节点 SCP 到国内服务器，使用 `rsync` 支持断点续传。

**估算传输时间：**
- 离线包约 1.5～2 GB
- 1 Mbps 带宽：约 200 分钟
- 10 Mbps 带宽：约 25 分钟
- 100 Mbps 带宽：约 3 分钟

> 💡 **技巧**：国内云服务器通常支持"按量付费"带宽，部署前临时调至 100Mbps，完成后降回，费用约 2～5 元。

### 方式二：OSS 中转（带宽受限 / 多台服务器）

1. Actions 将包上传到**阿里云 OSS 国内存储桶**
2. 生成 1 小时有效临时链接
3. SSH 通知各服务器并行从 OSS 内网高速下载（国内服务器与 OSS 同地域走内网免流量费）

适合场景：多台服务器部署、GitHub→国内线路不稳定。

---

## 离线包目录结构

```
k8s-offline/
├── bundle.env                          # 版本元信息（脚本自动读取）
├── install.sh                          # 安装入口脚本
├── bin/
│   ├── kubeadm                         # K8S 核心工具
│   ├── kubectl
│   ├── kubelet
│   ├── kubelet.service                 # systemd 服务文件
│   ├── 10-kubeadm.conf
│   ├── containerd.tar.gz               # 容器运行时
│   ├── runc                            # 低级容器运行时
│   └── crictl.tar.gz                   # 容器调试工具
├── images/
│   ├── registry.k8s.io_kube-apiserver_v1.28.2.tar
│   ├── registry.k8s.io_kube-controller-manager_v1.28.2.tar
│   ├── registry.k8s.io_kube-scheduler_v1.28.2.tar
│   ├── registry.k8s.io_kube-proxy_v1.28.2.tar
│   ├── registry.k8s.io_etcd_3.5.9-0.tar
│   ├── registry.k8s.io_coredns_v1.10.1.tar
│   ├── registry.k8s.io_pause_3.9.tar   # ⚠️ 关键：sandbox 镜像
│   └── docker.io_calico_*.tar          # CNI 插件镜像
├── cni/
│   └── calico.yaml                     # CNI 配置（已内嵌）
└── packages/
    ├── socat_*.deb                     # 系统依赖（离线 .deb）
    ├── conntrack_*.deb
    └── ipset_*.deb
```

---

## 常见问题排查

### ❌ kubeadm init 报 "ImagePull" 错误

**原因**：containerd 仍在尝试拉取外网镜像。

**解决**：检查 `/etc/containerd/config.toml` 中的 `sandbox_image` 是否已改为本地 tag，并重启 containerd：
```bash
systemctl restart containerd
kubeadm init --config /tmp/kubeadm-config.yaml --ignore-preflight-errors=all
```

### ❌ 节点一直 NotReady

**原因**：CNI 网络插件未就绪。

**检查**：
```bash
kubectl get pods -n kube-system | grep -E "calico|flannel|coredns"
kubectl describe pod -n kube-system <pod-name>
```

### ❌ SCP 传输超时（GitHub Actions 6h 限制）

**方案**：切换为 OSS 中转模式，或临时提升服务器带宽。

### ❌ socat/conntrack 缺失导致 kubeadm 预检失败

**解决**：安装脚本会自动从 `packages/` 目录离线安装，若仍失败可跳过：
```bash
bash install.sh --bundle xxx.tar.gz 2>&1 | grep -v "preflight"
```

---

## 手动单步安装

如需在已传输文件的服务器上单独执行安装：

```bash
# 解压（如尚未解压）
tar -xzf k8s-offline-v1.28.2-amd64.tar.gz -C /tmp/

# 执行安装（自定义参数）
bash /tmp/k8s-offline/install.sh \
  --skip-unpack \
  --pod-cidr 10.244.0.0/16 \
  --svc-cidr 10.96.0.0/12 \
  --api-ip 192.168.1.10
```

---

## Worker 节点加入

Master 初始化完成后，安装日志末尾会输出 `kubeadm join` 命令。

在 Worker 节点上同样传输并解压离线包后，**只需执行前半部分（不含 kubeadm init）**，然后运行输出的 join 命令。
