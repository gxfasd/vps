#!/bin/bash

#===============================================================================
# qBittorrent 4.3.9 + libtorrent 1.2.15 全自动安装脚本
# 支持: Ubuntu/Debian/CentOS/AlmaLinux
# 架构: x86_64/arm64/aarch64
#===============================================================================

set -e

# 颜色定义
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

# 配置变量
QB_VERSION="4.3.9"
LT_VERSION="1.2.15"
QB_USER="qbittorrent"
WEB_USER="admin"
WEB_PASS='@Ggg123456789'
DOWNLOAD_DIR="/home/down"
QB_PORT="8080"
INSTALL_DIR="/opt/qbittorrent"
CONFIG_DIR="/home/${QB_USER}/.config/qBittorrent"

# 日志函数
log_info() { echo -e "${GREEN}[INFO]${NC} $1"; }
log_warn() { echo -e "${YELLOW}[WARN]${NC} $1"; }
log_error() { echo -e "${RED}[ERROR]${NC} $1"; }
log_step() { echo -e "${BLUE}[STEP]${NC} $1"; }

# 检查root权限
check_root() {
    if [[ $EUID -ne 0 ]]; then
        log_error "此脚本需要root权限运行"
        exit 1
    fi
}

# 检测系统类型
detect_os() {
    if [ -f /etc/os-release ]; then
        . /etc/os-release
        OS_ID="${ID}"
        OS_VERSION="${VERSION_ID}"
    elif [ -f /etc/redhat-release ]; then
        OS_ID="centos"
        OS_VERSION=$(rpm -q --queryformat '%{VERSION}' centos-release 2>/dev/null || echo "7")
    else
        log_error "无法检测操作系统类型"
        exit 1
    fi
    
    case "${OS_ID}" in
        ubuntu|debian)
            PKG_MANAGER="apt"
            ;;
        centos|rhel|almalinux|rocky)
            PKG_MANAGER="yum"
            if command -v dnf &>/dev/null; then
                PKG_MANAGER="dnf"
            fi
            ;;
        *)
            log_error "不支持的操作系统: ${OS_ID}"
            exit 1
            ;;
    esac
    
    log_info "检测到系统: ${OS_ID} ${OS_VERSION}"
    log_info "包管理器: ${PKG_MANAGER}"
}

# 检测系统架构
detect_arch() {
    ARCH=$(uname -m)
    case "${ARCH}" in
        x86_64|amd64)
            ARCH_TYPE="x86_64"
            ;;
        aarch64|arm64)
            ARCH_TYPE="aarch64"
            ;;
        armv7l|armhf)
            ARCH_TYPE="armv7"
            ;;
        *)
            log_error "不支持的架构: ${ARCH}"
            exit 1
            ;;
    esac
    log_info "检测到架构: ${ARCH_TYPE}"
}

# 安装依赖 - Debian/Ubuntu
install_deps_debian() {
    log_step "更新包列表..."
    apt update -y
    
    log_step "安装编译依赖..."
    apt install -y \
        build-essential \
        pkg-config \
        automake \
        libtool \
        git \
        zlib1g-dev \
        libssl-dev \
        libgeoip-dev \
        libboost-dev \
        libboost-system-dev \
        libboost-chrono-dev \
        libboost-random-dev \
        libqt5svg5-dev \
        qtbase5-dev \
        qttools5-dev \
        qttools5-dev-tools \
        python3 \
        cmake \
        ninja-build \
        curl \
        wget
}

# 安装依赖 - CentOS/AlmaLinux
install_deps_rhel() {
    log_step "安装EPEL仓库..."
    ${PKG_MANAGER} install -y epel-release
    
    log_step "安装编译依赖..."
    ${PKG_MANAGER} groupinstall -y "Development Tools"
    ${PKG_MANAGER} install -y \
        gcc-c++ \
        cmake \
        ninja-build \
        git \
        openssl-devel \
        boost-devel \
        qt5-qtbase-devel \
        qt5-qttools-devel \
        qt5-qtsvg-devel \
        qt5-linguist \
        python3 \
        curl \
        wget \
        zlib-devel \
        GeoIP-devel
}

# 安装依赖
install_dependencies() {
    log_step "安装编译依赖..."
    
    case "${PKG_MANAGER}" in
        apt)
            install_deps_debian
            ;;
        yum|dnf)
            install_deps_rhel
            ;;
    esac
}

# 编译安装libtorrent
install_libtorrent() {
    log_step "下载并编译 libtorrent-rasterbar ${LT_VERSION}..."
    
    cd /tmp
    rm -rf libtorrent-rasterbar-${LT_VERSION}
    
    wget -q --show-progress "https://github.com/arvidn/libtorrent/releases/download/v${LT_VERSION}/libtorrent-rasterbar-${LT_VERSION}.tar.gz" \
        -O libtorrent-rasterbar-${LT_VERSION}.tar.gz || {
        # 备用下载地址
        wget -q --show-progress "https://github.com/arvidn/libtorrent/releases/download/libtorrent-${LT_VERSION}/libtorrent-rasterbar-${LT_VERSION}.tar.gz" \
            -O libtorrent-rasterbar-${LT_VERSION}.tar.gz
    }
    
    tar -xzf libtorrent-rasterbar-${LT_VERSION}.tar.gz
    cd libtorrent-rasterbar-${LT_VERSION}
    
    mkdir -p build && cd build
    
    cmake -G Ninja \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_INSTALL_PREFIX=/usr/local \
        -DCMAKE_CXX_STANDARD=17 \
        ..
    
    ninja -j$(nproc)
    ninja install
    
    # 更新库缓存
    if [ -f /etc/ld.so.conf ]; then
        echo "/usr/local/lib" > /etc/ld.so.conf.d/libtorrent.conf
        echo "/usr/local/lib64" >> /etc/ld.so.conf.d/libtorrent.conf
        ldconfig
    fi
    
    log_info "libtorrent ${LT_VERSION} 安装完成"
}

# 编译安装qBittorrent
install_qbittorrent() {
    log_step "下载并编译 qBittorrent ${QB_VERSION}..."
    
    cd /tmp
    rm -rf qBittorrent-release-${QB_VERSION}
    
    wget -q --show-progress "https://github.com/qbittorrent/qBittorrent/archive/refs/tags/release-${QB_VERSION}.tar.gz" \
        -O qBittorrent-${QB_VERSION}.tar.gz
    
    tar -xzf qBittorrent-${QB_VERSION}.tar.gz
    cd qBittorrent-release-${QB_VERSION}
    
    mkdir -p build && cd build
    
    # 设置PKG_CONFIG_PATH
    export PKG_CONFIG_PATH="/usr/local/lib/pkgconfig:/usr/local/lib64/pkgconfig:${PKG_CONFIG_PATH}"
    
    cmake -G Ninja \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR} \
        -DGUI=OFF \
        -DWEBUI=ON \
        -DCMAKE_CXX_STANDARD=17 \
        ..
    
    ninja -j$(nproc)
    ninja install
    
    # 创建符号链接
    ln -sf ${INSTALL_DIR}/bin/qbittorrent-nox /usr/local/bin/qbittorrent-nox
    
    log_info "qBittorrent ${QB_VERSION} 安装完成"
}

# 创建用户和目录
create_user_and_dirs() {
    log_step "创建用户和目录..."
    
    # 创建qbittorrent用户
    if ! id "${QB_USER}" &>/dev/null; then
        useradd -r -m -s /bin/bash ${QB_USER}
        log_info "创建用户: ${QB_USER}"
    fi
    
    # 创建下载目录
    mkdir -p ${DOWNLOAD_DIR}
    chown -R ${QB_USER}:${QB_USER} ${DOWNLOAD_DIR}
    chmod 755 ${DOWNLOAD_DIR}
    
    # 创建配置目录
    mkdir -p ${CONFIG_DIR}
    chown -R ${QB_USER}:${QB_USER} /home/${QB_USER}/.config
}

# 生成密码哈希 (PBKDF2)
generate_password_hash() {
    local password="$1"
    # qBittorrent 4.2+ 使用PBKDF2-SHA512
    python3 << EOF
import hashlib
import os
import base64

password = "${password}"
salt = os.urandom(16)
iterations = 100000
dk = hashlib.pbkdf2_hmac('sha512', password.encode(), salt, iterations, dklen=64)
result = "@ByteArray(" + base64.b64encode(salt).decode() + ":" + base64.b64encode(dk).decode() + ")"
print(result)
EOF
}

# 配置qBittorrent
configure_qbittorrent() {
    log_step "配置 qBittorrent..."
    
    # 生成密码哈希
    PASS_HASH=$(generate_password_hash "${WEB_PASS}")
    
    # 创建配置文件
    cat > ${CONFIG_DIR}/qBittorrent.conf << EOF
[Application]
FileLogger\Enabled=true
FileLogger\Path=${CONFIG_DIR}/logs

[BitTorrent]
Session\AddExtensionToIncompleteFiles=true
Session\AlternativeGlobalDLSpeedLimit=0
Session\AlternativeGlobalUPSpeedLimit=0
Session\AnonymousModeEnabled=true
Session\DHTEnabled=false
Session\DefaultSavePath=${DOWNLOAD_DIR}
Session\Encryption=1
Session\GlobalMaxRatio=-1
Session\IgnoreLimitsOnLAN=false
Session\IncludeOverheadInLimits=false
Session\LSDEnabled=false
Session\MaxActiveDownloads=10
Session\MaxActiveTorrents=100
Session\MaxActiveUploads=100
Session\PeXEnabled=false
Session\Preallocation=true
Session\uTPRateLimited=false

[Core]
AutoDeleteAddedTorrentFile=Never

[Meta]
MigrationVersion=4

[Network]
PortForwardingEnabled=true

[Preferences]
Advanced\IgnoreSSLErrors=true
Advanced\trackerPort=9000
Connection\PortRangeMin=45000
Downloads\PreAllocation=true
Downloads\SavePath=${DOWNLOAD_DIR}
Downloads\TempPath=${DOWNLOAD_DIR}/temp
Downloads\TempPathEnabled=false
DynDNS\Enabled=false
General\Locale=zh_CN
WebUI\Address=*
WebUI\AlternativeUIEnabled=false
WebUI\AuthSubnetWhitelistEnabled=false
WebUI\CSRFProtection=true
WebUI\ClickjackingProtection=true
WebUI\Enabled=true
WebUI\HTTPS\Enabled=false
WebUI\HostHeaderValidation=true
WebUI\LocalHostAuth=true
WebUI\Password_PBKDF2=${PASS_HASH}
WebUI\Port=${QB_PORT}
WebUI\SecureCookie=true
WebUI\ServerDomains=*
WebUI\SessionTimeout=3600
WebUI\UseUPnP=false
WebUI\Username=${WEB_USER}
EOF

    # 设置权限
    chown -R ${QB_USER}:${QB_USER} ${CONFIG_DIR}
    chmod 600 ${CONFIG_DIR}/qBittorrent.conf
    
    log_info "配置文件已创建: ${CONFIG_DIR}/qBittorrent.conf"
}

# 创建systemd服务
create_systemd_service() {
    log_step "创建 systemd 服务..."
    
    cat > /etc/systemd/system/qbittorrent.service << EOF
[Unit]
Description=qBittorrent-nox Daemon Service
Documentation=man:qbittorrent-nox(1)
Wants=network-online.target
After=network-online.target nss-lookup.target

[Service]
Type=simple
User=${QB_USER}
Group=${QB_USER}
UMask=002
Environment="LD_LIBRARY_PATH=/usr/local/lib:/usr/local/lib64"
ExecStart=/usr/local/bin/qbittorrent-nox --webui-port=${QB_PORT}
Restart=on-failure
RestartSec=5
TimeoutStopSec=infinity
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

    systemctl daemon-reload
    systemctl enable qbittorrent
    systemctl start qbittorrent
    
    log_info "systemd 服务已创建并启动"
}

# 配置防火墙
configure_firewall() {
    log_step "配置防火墙..."
    
    # firewalld (CentOS/AlmaLinux)
    if command -v firewall-cmd &>/dev/null; then
        firewall-cmd --permanent --add-port=${QB_PORT}/tcp 2>/dev/null || true
        firewall-cmd --permanent --add-port=45000-45100/tcp 2>/dev/null || true
        firewall-cmd --permanent --add-port=45000-45100/udp 2>/dev/null || true
        firewall-cmd --reload 2>/dev/null || true
        log_info "firewalld 规则已添加"
    fi
    
    # ufw (Ubuntu/Debian)
    if command -v ufw &>/dev/null; then
        ufw allow ${QB_PORT}/tcp 2>/dev/null || true
        ufw allow 45000:45100/tcp 2>/dev/null || true
        ufw allow 45000:45100/udp 2>/dev/null || true
        log_info "ufw 规则已添加"
    fi
    
    # iptables
    if command -v iptables &>/dev/null; then
        iptables -I INPUT -p tcp --dport ${QB_PORT} -j ACCEPT 2>/dev/null || true
    fi
}

# 清理临时文件
cleanup() {
    log_step "清理临时文件..."
    rm -rf /tmp/libtorrent-rasterbar-${LT_VERSION}*
    rm -rf /tmp/qBittorrent-*
    log_info "清理完成"
}

# 显示安装信息
show_install_info() {
    # 获取服务器IP
    SERVER_IP=$(hostname -I | awk '{print $1}')
    
    echo ""
    echo "============================================================"
    echo -e "${GREEN}qBittorrent ${QB_VERSION} 安装完成!${NC}"
    echo "============================================================"
    echo ""
    echo -e "Web UI 地址: ${BLUE}http://${SERVER_IP}:${QB_PORT}${NC}"
    echo -e "用户名: ${YELLOW}${WEB_USER}${NC}"
    echo -e "密码: ${YELLOW}${WEB_PASS}${NC}"
    echo ""
    echo -e "下载目录: ${BLUE}${DOWNLOAD_DIR}${NC}"
    echo -e "配置文件: ${BLUE}${CONFIG_DIR}/qBittorrent.conf${NC}"
    echo ""
    echo "服务管理命令:"
    echo "  启动: systemctl start qbittorrent"
    echo "  停止: systemctl stop qbittorrent"
    echo "  重启: systemctl restart qbittorrent"
    echo "  状态: systemctl status qbittorrent"
    echo "  日志: journalctl -u qbittorrent -f"
    echo ""
    echo "============================================================"
    echo ""
    
    # 检查服务状态
    sleep 3
    if systemctl is-active --quiet qbittorrent; then
        log_info "qBittorrent 服务运行正常"
    else
        log_warn "qBittorrent 服务可能未正常启动，请检查日志"
        journalctl -u qbittorrent -n 20 --no-pager
    fi
}

# 主函数
main() {
    echo ""
    echo "============================================================"
    echo "  qBittorrent ${QB_VERSION} + libtorrent ${LT_VERSION} 安装脚本"
    echo "============================================================"
    echo ""
    
    check_root
    detect_os
    detect_arch
    
    install_dependencies
    install_libtorrent
    install_qbittorrent
    create_user_and_dirs
    configure_qbittorrent
    create_systemd_service
    configure_firewall
    cleanup
    show_install_info
}

# 执行主函数
main "$@"

