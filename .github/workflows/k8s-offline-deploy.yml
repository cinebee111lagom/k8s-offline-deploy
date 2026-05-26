#!/usr/bin/env bash
# =============================================================================
# K8S 离线安装脚本
# 运行环境：国内云服务器（断网/受限网络）
# 依赖：离线包已解压到 WORK_DIR，或通过 --bundle 参数指定压缩包路径
# =============================================================================

set -euo pipefail

# ============================================================
# 颜色输出
# ============================================================
RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'
BLUE='\033[0;34m'; CYAN='\033[0;36m'; NC='\033[0m'

log_info()    { echo -e "${GREEN}[INFO]${NC}  $*"; }
log_warn()    { echo -e "${YELLOW}[WARN]${NC}  $*"; }
log_error()   { echo -e "${RED}[ERROR]${NC} $*" >&2; }
log_step()    { echo -e "\n${CYAN}========== $* ==========${NC}"; }
log_success() { echo -e "${GREEN}✅ $*${NC}"; }

# ============================================================
# 参数解析
# ============================================================
BUNDLE_PATH=""
WORK_DIR="/tmp/k8s-offline"
SKIP_UNPACK=false
POD_NETWORK_CIDR="10.244.0.0/16"
SERVICE_CIDR="10.96.0.0/12"
API_SERVER_IP=""    # 留空则自动检测本机 IP

while [[ $# -gt 0 ]]; do
  case $1 in
    --bundle)    BUNDLE_PATH="$2"; shift 2 ;;
    --work-dir)  WORK_DIR="$2"; shift 2 ;;
    --skip-unpack) SKIP_UNPACK=true; shift ;;
    --pod-cidr)  POD_NETWORK_CIDR="$2"; shift 2 ;;
    --svc-cidr)  SERVICE_CIDR="$2"; shift 2 ;;
    --api-ip)    API_SERVER_IP="$2"; shift 2 ;;
    *) log_warn "未知参数: $1"; shift ;;
  esac
done

# ============================================================
# 全局变量（从离线包 bundle.env 中读取）
# ============================================================
K8S_VERSION=""
CNI_PLUGIN=""
ARCH=""

# ============================================================
# 前置检查
# ============================================================
preflight_check() {
  log_step "前置环境检查"

  # 必须是 root
  if [[ $EUID -ne 0 ]]; then
    log_error "请以 root 用户运行此脚本"
    exit 1
  fi
  log_info "✓ 当前用户: root"

  # 检测系统类型
  if command -v apt-get &>/dev/null; then
    PKG_MANAGER="apt"
    log_info "✓ 包管理器: apt (Debian/Ubuntu)"
  elif command -v yum &>/dev/null; then
    PKG_MANAGER="yum"
    log_info "✓ 包管理器: yum (RHEL/CentOS)"
  elif command -v dnf &>/dev/null; then
    PKG_MANAGER="dnf"
    log_info "✓ 包管理器: dnf (Fedora/Rocky)"
  else
    log_error "不支持的系统：未找到 apt/yum/dnf"
    exit 1
  fi

  # 检测架构
  SYSTEM_ARCH=$(uname -m)
  case $SYSTEM_ARCH in
    x86_64)  SYSTEM_ARCH="amd64" ;;
    aarch64) SYSTEM_ARCH="arm64" ;;
    *) log_error "不支持的架构: $SYSTEM_ARCH"; exit 1 ;;
  esac
  log_info "✓ 系统架构: $SYSTEM_ARCH"

  # 检测内存（至少 2G）
  TOTAL_MEM_KB=$(grep MemTotal /proc/meminfo | awk '{print $2}')
  TOTAL_MEM_GB=$((TOTAL_MEM_KB / 1024 / 1024))
  if [[ $TOTAL_MEM_GB -lt 2 ]]; then
    log_warn "内存不足 2GB ($TOTAL_MEM_GB GB)，kubeadm 可能报错，将使用 --ignore-preflight-errors=NumCPU,Mem"
  fi
  log_info "✓ 内存: ${TOTAL_MEM_GB} GB"

  # 检测 CPU（至少 2 核）
  CPU_CORES=$(nproc)
  log_info "✓ CPU 核数: $CPU_CORES"

  # 获取本机 IP（用于 kubeadm init）
  if [[ -z "$API_SERVER_IP" ]]; then
    API_SERVER_IP=$(ip route get 1.1.1.1 2>/dev/null | awk '{print $7; exit}' || \
                    hostname -I 2>/dev/null | awk '{print $1}')
  fi
  log_info "✓ API Server IP: $API_SERVER_IP"
}

# ============================================================
# 解压离线包
# ============================================================
unpack_bundle() {
  log_step "解压离线包"

  if [[ "$SKIP_UNPACK" == "true" ]]; then
    log_info "跳过解压（--skip-unpack）"
  elif [[ -n "$BUNDLE_PATH" ]]; then
    if [[ ! -f "$BUNDLE_PATH" ]]; then
      log_error "找不到离线包: $BUNDLE_PATH"
      exit 1
    fi
    log_info "解压 $BUNDLE_PATH 到 /tmp ..."
    mkdir -p "$WORK_DIR"
    tar -xzf "$BUNDLE_PATH" -C /tmp/ 2>&1 | tail -5
    log_success "解压完成"
  else
    log_error "未指定 --bundle 参数且未设置 --skip-unpack"
    exit 1
  fi

  # 读取 bundle.env
  if [[ -f "$WORK_DIR/bundle.env" ]]; then
    # 清理可能的前导空格
    source <(sed 's/^[[:space:]]*//' "$WORK_DIR/bundle.env")
    log_info "离线包信息:"
    log_info "  K8S 版本: $K8S_VERSION"
    log_info "  架构:     $ARCH"
    log_info "  CNI 插件: $CNI_PLUGIN"
    log_info "  构建时间: $BUILD_TIME"
  else
    log_error "离线包不完整：缺少 bundle.env"
    exit 1
  fi
}

# ============================================================
# 系统内核参数优化
# ============================================================
configure_kernel() {
  log_step "配置系统内核参数"

  # 关闭 swap（K8S 强要求）
  log_info "关闭 swap..."
  swapoff -a
  sed -i '/\bswap\b/d' /etc/fstab
  log_success "swap 已关闭"

  # 加载内核模块
  cat > /etc/modules-load.d/k8s.conf << 'EOF'
overlay
br_netfilter
EOF
  modprobe overlay
  modprobe br_netfilter
  log_success "内核模块已加载: overlay, br_netfilter"

  # 内核网络参数
  cat > /etc/sysctl.d/k8s.conf << 'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
vm.overcommit_memory                = 1
vm.panic_on_oom                     = 0
kernel.panic                        = 10
kernel.panic_on_oops                = 1
EOF
  sysctl --system -q
  log_success "内核网络参数已应用"

  # 关闭 SELinux（如有）
  if command -v getenforce &>/dev/null; then
    setenforce 0 2>/dev/null || true
    sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config 2>/dev/null || true
    log_success "SELinux 已设为 permissive"
  fi

  # 关闭防火墙（如有，生产环境请按需配置规则）
  if systemctl is-active --quiet firewalld 2>/dev/null; then
    systemctl stop firewalld && systemctl disable firewalld
    log_warn "firewalld 已关闭（生产环境请手动配置放行规则）"
  fi
  if systemctl is-active --quiet ufw 2>/dev/null; then
    ufw disable 2>/dev/null || true
    log_warn "ufw 已关闭（生产环境请手动配置放行规则）"
  fi
}

# ============================================================
# 本地安装系统依赖包（socat / conntrack / ipset）
# ============================================================
install_system_deps() {
  log_step "本地安装系统依赖包"

  PACKAGES_DIR="$WORK_DIR/packages"

  if [[ -d "$PACKAGES_DIR" ]] && ls "$PACKAGES_DIR"/*.deb &>/dev/null 2>&1; then
    log_info "发现 .deb 包，使用 dpkg 离线安装..."
    dpkg -i --force-all "$PACKAGES_DIR"/*.deb 2>&1 || \
      log_warn "部分包安装失败（可能已安装），继续..."
    log_success ".deb 系统依赖包安装完成"

  elif [[ -d "$PACKAGES_DIR" ]] && ls "$PACKAGES_DIR"/*.rpm &>/dev/null 2>&1; then
    log_info "发现 .rpm 包，使用 rpm 离线安装..."
    rpm -ivh --nodeps "$PACKAGES_DIR"/*.rpm 2>&1 || \
      log_warn "部分包安装失败（可能已安装），继续..."
    log_success ".rpm 系统依赖包安装完成"
  else
    log_warn "未找到离线系统依赖包，尝试在线安装（网络可能不可用）..."
    case $PKG_MANAGER in
      apt) apt-get install -y socat conntrack ipset ipvsadm 2>/dev/null || \
           log_warn "在线安装失败，继续（kubeadm 预检可能报错）" ;;
      yum|dnf) $PKG_MANAGER install -y socat conntrack ipset ipvsadm 2>/dev/null || \
               log_warn "在线安装失败，继续（kubeadm 预检可能报错）" ;;
    esac
  fi

  # 验证关键工具
  for tool in socat conntrack; do
    if command -v $tool &>/dev/null; then
      log_success "$tool ✓"
    else
      log_warn "$tool 未安装，kubeadm 预检可能警告（可用 --ignore-preflight-errors 跳过）"
    fi
  done
}

# ============================================================
# 安装 containerd
# ============================================================
install_containerd() {
  log_step "安装 containerd 容器运行时"

  BIN_DIR="$WORK_DIR/bin"

  if [[ -f "$BIN_DIR/containerd.tar.gz" ]]; then
    log_info "从离线包安装 containerd..."
    tar -xzf "$BIN_DIR/containerd.tar.gz" -C /usr/local/
    log_success "containerd 解压到 /usr/local/bin/"
  else
    log_error "找不到 $BIN_DIR/containerd.tar.gz"
    exit 1
  fi

  # 安装 runc
  if [[ -f "$BIN_DIR/runc" ]]; then
    install -m 755 "$BIN_DIR/runc" /usr/local/sbin/runc
    log_success "runc 安装完成"
  fi

  # 安装 crictl
  if [[ -f "$BIN_DIR/crictl.tar.gz" ]]; then
    tar -xzf "$BIN_DIR/crictl.tar.gz" -C /usr/local/bin/
    log_success "crictl 安装完成"
  fi

  # 创建 containerd 数据目录
  mkdir -p /etc/containerd /var/lib/containerd /run/containerd

  # 生成默认配置并修改关键参数
  log_info "生成并修改 containerd 配置..."
  containerd config default > /etc/containerd/config.toml

  # ==============================================================
  # 关键修改 1：修改 pause 镜像地址
  # 防止 kubeadm init 时 containerd 去外网拉取 pause 镜像
  # ==============================================================
  PAUSE_TAG=$(docker images --format "{{.Repository}}:{{.Tag}}" 2>/dev/null | \
              grep "pause" | head -1 || \
              ls "$WORK_DIR/images/" | grep pause | head -1 | sed 's/.tar$//' | sed 's/_/\//;s/_/:/;s/_/\//g')

  # 优先使用 registry.k8s.io/pause 形式（kubeadm 期望的格式）
  PAUSE_IMAGE="registry.k8s.io/pause:3.9"
  sed -i "s|sandbox_image = .*|sandbox_image = \"${PAUSE_IMAGE}\"|" \
    /etc/containerd/config.toml

  # ==============================================================
  # 关键修改 2：使用 systemd cgroup 驱动（与 kubeadm 对齐）
  # ==============================================================
  sed -i 's|SystemdCgroup = false|SystemdCgroup = true|' \
    /etc/containerd/config.toml

  log_success "containerd 配置修改完成（pause: $PAUSE_IMAGE, cgroup: systemd）"

  # 安装 containerd.service（官方 systemd 服务文件）
  cat > /etc/systemd/system/containerd.service << 'EOF'
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd
Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF

  systemctl daemon-reload
  systemctl enable containerd
  systemctl start containerd

  # 验证
  sleep 3
  if systemctl is-active --quiet containerd; then
    log_success "containerd 服务启动成功"
    containerd --version
  else
    log_error "containerd 服务启动失败"
    journalctl -xe -u containerd --no-pager | tail -20
    exit 1
  fi
}

# ============================================================
# 导入所有离线镜像
# ============================================================
import_images() {
  log_step "导入离线容器镜像"

  IMAGES_DIR="$WORK_DIR/images"
  if [[ ! -d "$IMAGES_DIR" ]]; then
    log_error "找不到镜像目录: $IMAGES_DIR"
    exit 1
  fi

  IMAGE_COUNT=$(ls "$IMAGES_DIR"/*.tar 2>/dev/null | wc -l)
  log_info "发现 $IMAGE_COUNT 个镜像包..."

  SUCCESS=0
  FAILED=0

  for tar_file in "$IMAGES_DIR"/*.tar; do
    [[ -f "$tar_file" ]] || continue
    
    image_name=$(basename "$tar_file" .tar)
    log_info "导入: $image_name ..."
    
    if ctr --namespace k8s.io images import "$tar_file" 2>&1; then
      log_success "✓ $image_name"
      SUCCESS=$((SUCCESS + 1))
    else
      log_warn "✗ $image_name 导入失败（将尝试继续）"
      FAILED=$((FAILED + 1))
    fi
  done

  echo ""
  log_info "镜像导入结果: 成功 $SUCCESS，失败 $FAILED，共 $IMAGE_COUNT"

  # 列出已导入镜像
  log_info "已导入镜像列表:"
  ctr --namespace k8s.io images list 2>/dev/null | awk '{print "  " $1}' || true

  if [[ $FAILED -gt 0 ]]; then
    log_warn "有 $FAILED 个镜像导入失败，如果是 CNI 插件镜像可暂时忽略"
  fi
}

# ============================================================
# 安装 K8S 核心二进制（kubeadm / kubectl / kubelet）
# ============================================================
install_k8s_binaries() {
  log_step "安装 K8S 核心二进制文件"

  BIN_DIR="$WORK_DIR/bin"

  for binary in kubeadm kubectl kubelet; do
    if [[ -f "$BIN_DIR/$binary" ]]; then
      install -m 755 "$BIN_DIR/$binary" /usr/local/bin/
      log_success "$binary → /usr/local/bin/$binary"
    else
      log_error "找不到 $BIN_DIR/$binary"
      exit 1
    fi
  done

  # 安装 crictl 配置
  cat > /etc/crictl.yaml << 'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 30
debug: false
EOF

  # 配置 kubelet systemd 服务
  mkdir -p /etc/systemd/system/kubelet.service.d

  if [[ -f "$BIN_DIR/kubelet.service" ]]; then
    cp "$BIN_DIR/kubelet.service" /etc/systemd/system/kubelet.service
    # 修正二进制路径（官方模板可能写 /usr/bin，我们装在 /usr/local/bin）
    sed -i 's|/usr/bin/kubelet|/usr/local/bin/kubelet|g' \
      /etc/systemd/system/kubelet.service
  else
    # 兜底：手写 kubelet.service
    cat > /etc/systemd/system/kubelet.service << 'EOF'
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/local/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
  fi

  if [[ -f "$BIN_DIR/10-kubeadm.conf" ]]; then
    cp "$BIN_DIR/10-kubeadm.conf" \
      /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
  fi

  systemctl daemon-reload
  systemctl enable kubelet
  # kubelet 此时启动会失败（需等 kubeadm init），这是正常的
  log_success "kubelet service 已注册"

  # 验证版本
  log_info "二进制版本确认:"
  kubeadm version 2>/dev/null && true
  kubectl version --client 2>/dev/null && true
}

# ============================================================
# kubeadm init（离线初始化）
# ============================================================
init_cluster() {
  log_step "初始化 K8S 集群（离线模式）"

  # 检查是否已初始化
  if kubectl get nodes &>/dev/null 2>&1; then
    log_warn "集群似乎已存在，跳过初始化"
    log_warn "如需重新初始化，请先执行: kubeadm reset -f"
    return 0
  fi

  log_info "API Server 地址: $API_SERVER_IP"
  log_info "Pod 网络 CIDR:   $POD_NETWORK_CIDR"
  log_info "Service CIDR:    $SERVICE_CIDR"
  log_info "K8S 版本:        $K8S_VERSION"

  # 生成 kubeadm 配置文件（避免命令行参数过多）
  cat > /tmp/kubeadm-config.yaml << EOF
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "${API_SERVER_IP}"
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
  imagePullPolicy: Never   # 关键：禁止拉取镜像，只用本地
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: "${K8S_VERSION}"
networking:
  podSubnet: "${POD_NETWORK_CIDR}"
  serviceSubnet: "${SERVICE_CIDR}"
controllerManager:
  extraArgs:
    allocate-node-cidrs: "true"
    cluster-cidr: "${POD_NETWORK_CIDR}"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF

  log_info "执行 kubeadm init（忽略网络、系统预检错误）..."

  # --ignore-preflight-errors 兜底应对断网环境的各种预检失败
  kubeadm init \
    --config /tmp/kubeadm-config.yaml \
    --ignore-preflight-errors=NumCPU,Mem,Swap,FileContent--proc-sys-net-bridge-bridge-nf-call-iptables,DirAvailable--etc-kubernetes-manifests,ImagePull \
    --skip-phases=preflight/SystemVerification \
    2>&1 | tee /tmp/kubeadm-init.log

  INIT_EXIT=${PIPESTATUS[0]}

  if [[ $INIT_EXIT -ne 0 ]]; then
    log_error "kubeadm init 失败，退出码: $INIT_EXIT"
    log_error "请查看详细日志: /tmp/kubeadm-init.log"
    tail -30 /tmp/kubeadm-init.log >&2
    exit $INIT_EXIT
  fi

  log_success "kubeadm init 完成！"
}

# ============================================================
# 配置 kubectl 访问
# ============================================================
configure_kubectl() {
  log_step "配置 kubectl 访问凭证"

  # root 用户
  mkdir -p /root/.kube
  cp -f /etc/kubernetes/admin.conf /root/.kube/config
  log_success "kubectl 配置完成 (/root/.kube/config)"

  # 如有普通用户，也配置一份
  SUDO_USER_HOME=$(eval echo ~${SUDO_USER:-} 2>/dev/null || true)
  if [[ -n "${SUDO_USER:-}" && -d "$SUDO_USER_HOME" ]]; then
    mkdir -p "$SUDO_USER_HOME/.kube"
    cp -f /etc/kubernetes/admin.conf "$SUDO_USER_HOME/.kube/config"
    chown "${SUDO_USER}:${SUDO_USER}" "$SUDO_USER_HOME/.kube/config"
    log_success "kubectl 配置已同步到 $SUDO_USER"
  fi

  # 等待 API Server 就绪
  log_info "等待 API Server 就绪..."
  for i in $(seq 1 30); do
    if kubectl get nodes &>/dev/null 2>&1; then
      log_success "API Server 已就绪"
      break
    fi
    echo -n "."
    sleep 5
  done
  echo ""
}

# ============================================================
# 去除 Master 节点 taint（单节点场景）
# ============================================================
untaint_master() {
  log_step "去除 Master 节点污点（允许调度 Pod）"

  NODE_NAME=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
  
  if [[ -z "$NODE_NAME" ]]; then
    log_warn "无法获取节点名称，跳过去污点"
    return 0
  fi

  kubectl taint nodes "$NODE_NAME" \
    node-role.kubernetes.io/control-plane:NoSchedule- \
    2>/dev/null || \
  kubectl taint nodes "$NODE_NAME" \
    node-role.kubernetes.io/master:NoSchedule- \
    2>/dev/null || \
  log_warn "污点已移除或不存在"

  log_success "节点 $NODE_NAME 污点已去除（可调度 Pod）"
}

# ============================================================
# 安装 CNI 网络插件（离线应用 YAML）
# ============================================================
install_cni() {
  log_step "安装 CNI 网络插件: ${CNI_PLUGIN}"

  CNI_DIR="$WORK_DIR/cni"

  case "$CNI_PLUGIN" in
    calico)
      if [[ -f "$CNI_DIR/calico.yaml" ]]; then
        log_info "应用 Calico 配置..."
        # 修改 calico.yaml 中的 CIDR 为实际配置
        sed -i "s|192.168.0.0/16|${POD_NETWORK_CIDR}|g" "$CNI_DIR/calico.yaml"
        kubectl apply -f "$CNI_DIR/calico.yaml"
        log_success "Calico 配置已应用"
      else
        log_error "找不到 $CNI_DIR/calico.yaml"
        exit 1
      fi
      ;;
    flannel)
      if [[ -f "$CNI_DIR/flannel.yaml" ]]; then
        log_info "应用 Flannel 配置..."
        sed -i "s|10.244.0.0/16|${POD_NETWORK_CIDR}|g" "$CNI_DIR/flannel.yaml"
        kubectl apply -f "$CNI_DIR/flannel.yaml"
        log_success "Flannel 配置已应用"
      else
        log_error "找不到 $CNI_DIR/flannel.yaml"
        exit 1
      fi
      ;;
    none)
      log_warn "未安装 CNI 插件，节点网络不可用，请手动安装"
      ;;
  esac
}

# ============================================================
# 验证集群状态
# ============================================================
verify_cluster() {
  log_step "验证集群安装结果"

  log_info "等待节点 Ready（最多 3 分钟）..."
  for i in $(seq 1 36); do
    STATUS=$(kubectl get nodes --no-headers 2>/dev/null | awk '{print $2}' | head -1)
    if [[ "$STATUS" == "Ready" ]]; then
      log_success "节点状态: Ready ✓"
      break
    fi
    echo -n "  等待中... ($STATUS) "
    sleep 5
  done

  echo ""
  log_info "======== 集群节点状态 ========"
  kubectl get nodes -o wide 2>/dev/null || log_warn "无法获取节点状态"

  echo ""
  log_info "======== 系统 Pod 状态 ========"
  kubectl get pods -n kube-system 2>/dev/null || log_warn "无法获取 Pod 状态"

  echo ""
  log_info "======== 集群信息 ========"
  kubectl cluster-info 2>/dev/null || true

  # 输出加入命令（用于 Worker 节点加入）
  echo ""
  log_info "======== Worker 节点加入命令 ========"
  kubeadm token create --print-join-command 2>/dev/null || \
    log_warn "无法生成 join 命令"
}

# ============================================================
# 主流程
# ============================================================
main() {
  echo -e "${CYAN}"
  cat << 'BANNER'
  ██╗  ██╗ █████╗ ███████╗    ██╗███╗   ██╗███████╗████████╗ █████╗ ██╗     ██╗
  ██║ ██╔╝██╔══██╗██╔════╝    ██║████╗  ██║██╔════╝╚══██╔══╝██╔══██╗██║     ██║
  █████╔╝ ╚█████╔╝███████╗    ██║██╔██╗ ██║███████╗   ██║   ███████║██║     ██║
  ██╔═██╗ ██╔══██╗╚════██║    ██║██║╚██╗██║╚════██║   ██║   ██╔══██║██║     ██║
  ██║  ██╗╚█████╔╝███████║    ██║██║ ╚████║███████║   ██║   ██║  ██║███████╗███████╗
  ╚═╝  ╚═╝ ╚════╝ ╚══════╝    ╚═╝╚═╝  ╚═══╝╚══════╝   ╚═╝   ╚═╝  ╚═╝╚══════╝╚══════╝
                          离线安装脚本 v1.0 | by GitHub Actions
BANNER
  echo -e "${NC}"

  log_info "开始时间: $(date '+%Y-%m-%d %H:%M:%S')"
  log_info "工作目录: $WORK_DIR"

  preflight_check
  unpack_bundle
  configure_kernel
  install_system_deps
  install_containerd
  import_images
  install_k8s_binaries
  init_cluster
  configure_kubectl
  untaint_master
  install_cni
  verify_cluster

  echo ""
  log_success "============================================="
  log_success "🎉 K8S 离线安装完成！"
  log_success "  K8S 版本: $K8S_VERSION"
  log_success "  API Server: https://${API_SERVER_IP}:6443"
  log_success "  kubeconfig: /root/.kube/config"
  log_success "============================================="
  log_info "结束时间: $(date '+%Y-%m-%d %H:%M:%S')"
}

main "$@"
