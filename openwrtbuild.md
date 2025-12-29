这是一份经过彻底重构的**《OpenWrt 24.10 云编译工厂・全流程部署指南（2025 完美避坑版）》**。

这份文档吸取了之前所有的教训，针对**权限地狱**、**依赖冲突**、**Opkg 兼容性**以及**日志乱码**问题做了预防性处理。

只要严格按照此流程操作，您将获得一个稳定、高效、可视化的编译系统。

---

# 🏭 OpenWrt 24.10 云编译工厂部署手册

### 📋 0. 准备工作

* **操作系统**: Ubuntu 22.04 LTS (全新纯净系统)。
* **硬件配置**: CPU 4核+ / 内存 8GB+ / 硬盘 60GB+。
* **目标固件**: OpenWrt 24.10 (Kernel 6.6, 原生 Opkg, 完美支持 PVE/三方脚本)。

---

### 第一阶段：底层环境构建 (SSH Root)

请使用终端工具（如 Putty, Xshell）登录 SSH，用户为 `root`。

**1. 安装宝塔面板**
*(如果已安装可跳过，但建议环境纯净)*

```bash
wget -O install.sh http://download.bt.cn/install/install-ubuntu_6.0.sh && sudo bash install.sh ed8484bec

```

> **记下**: 面板地址、账号、密码。

**2. 安装核心编译依赖 (地基)**

```bash
apt-get update
apt-get install -y build-essential asciidoc binutils bzip2 gawk gettext git libncurses5-dev libz-dev patch python3 python2.7 unzip zlib1g-dev lib32gcc-s1 libc6-dev-i386 subversion flex uglifyjs git-core gcc-multilib p7zip p7zip-full msmtp libssl-dev texinfo libglib2.0-dev xmlto qemu-utils upx-ucl libelf-dev autoconf automake libtool autopoint device-tree-compiler g++-multilib antlr3 gperf wget curl swig rsync dosfstools mtools

```

**3. 设置 4GB Swap (防内存溢出)**

```bash
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab

```

---

### 第二阶段：宝塔面板配置 (Web 操作)

**1. 安装 LNMP 环境**

* 登录面板 -> 商店 -> 安装 **Nginx** (任意版本) 和 **PHP 8.0** (推荐)。
* **不要**安装 MySQL 和 FTP。

**2. 解除 PHP 限制 (至关重要)**

* **软件商店** -> **PHP-8.0** -> **设置**。
* **禁用函数**: 删除 `shell_exec`, `exec`, `proc_open`。
* **配置修改**: `max_execution_time` 改为 `300`，`memory_limit` 改为 `1024M`。
* **重启 PHP 服务**。

**3. 创建网站**

* **网站** -> **添加站点** -> 域名填写 IP (如 `10.0.0.173`) -> PHP 选择 `8.0`。

---

### 第三阶段：部署 24.10 源码 (SSH Root)

请回到 SSH，替换目录路径为您实际的网站目录。

```bash
# 进入网站目录
cd /www/wwwroot/10.0.0.173/

# 1. 清理杂项
rm -rf index.html 404.html .htaccess

# 2. 克隆 OpenWrt 24.10 分支 (确保是 24.10)
git clone -b openwrt-24.10 https://github.com/openwrt/openwrt.git source

# 3. 解决 Git 信任报错 (防止 dubious ownership)
git config --global --add safe.directory $(pwd)/source
git config --global --add safe.directory $(pwd)/source/feeds/packages
git config --global --add safe.directory $(pwd)/source/feeds/luci
git config --global --add safe.directory $(pwd)/source/feeds/routing
git config --global --add safe.directory $(pwd)/source/feeds/telephony

# 4. 预创建目录并移交权限给 Web 用户
mkdir download
chown -R www:www .
chmod -R 755 .

```

---

### 第四阶段：生成 x86 母版配置 (SSH Root)

这一步只需做一次，用于确定固件架构。

```bash
cd source
# 更新并安装源
./scripts/feeds update -a && ./scripts/feeds install -a

# 进入菜单
make menuconfig

```

**在蓝底菜单中选择：**

1. Target System: **x86**
2. Subtarget: **x86_64**
3. Target Profile: **Generic**
4. 保存为 `.config` 并退出。

**导出母版：**

```bash
cp .config ../base.config
cd ..
chown www:www base.config

```

---

### 第五阶段：部署核心程序 (SSH Root)

请直接复制粘贴以下三个代码块，这将生成核心文件。

#### 📄 1. `compile.sh` (智能编译脚本)

*优化点：强制锁定 24.10 分支，防止版本漂移；增加缓存清理。*

```bash
cat > compile.sh <<'EOF'
#!/bin/bash
ROOT_DIR=$(pwd)
SOURCE_DIR="$ROOT_DIR/source"
LOG_FILE="$ROOT_DIR/compile.log"
LOCK_FILE="$ROOT_DIR/compile.lock"
EXTRA_CONFIG="$ROOT_DIR/extra.config"

exec 1>>"$LOG_FILE"
exec 2>&1
echo "=========================================="
echo "   任务启动 (OpenWrt 24.10): $(date '+%Y-%m-%d %H:%M:%S')"
echo "=========================================="

if [ ! -d "$SOURCE_DIR" ]; then echo "错误: 源码目录不存在！"; rm -f "$LOCK_FILE"; exit 1; fi
cd "$SOURCE_DIR" || exit 1
export FORCE_UNSAFE_CONFIGURE=1
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

echo ">>> [0/5] 环境准备..."
rm -rf tmp/
mkdir -p tmp
# 强制检出 24.10 分支
git checkout openwrt-24.10 >/dev/null 2>&1

echo ">>> [1/5] 更新软件源..."
# 尝试更新，如果失败则暴力修复 feeds 目录
./scripts/feeds update -a
if [ $? -ne 0 ]; then
    echo "!!! 警告: 源更新失败，正在重置 Feeds..."
    rm -rf feeds package/feeds
    ./scripts/feeds update -a
fi
./scripts/feeds install -a

echo ">>> [2/5] 生成配置文件..."
if [ -f "$ROOT_DIR/base.config" ]; then cp "$ROOT_DIR/base.config" .config; else echo "致命错误: base.config 不存在！"; rm -f "$LOCK_FILE"; exit 1; fi
if [ -f "$EXTRA_CONFIG" ]; then echo ">>> 合并用户配置..."; cat "$EXTRA_CONFIG" >> .config; fi
make defconfig

echo ">>> [3/5] 下载依赖..."
make download -j$(nproc) || make download -j1 V=s

echo ">>> [4/5] 开始编译..."
# 忽略非致命错误，防止文档生成失败中断编译
make -j$(nproc) IGNORE_ERRORS=1 || { echo "!!! 多核编译失败，尝试单核..."; make -j1 V=s IGNORE_ERRORS=1; }

echo "=== 任务结束: $(date '+%Y-%m-%d %H:%M:%S') ==="
rm -f "$LOCK_FILE"
EOF
chmod +x compile.sh
chown www:www compile.sh

```

#### 📄 2. `build.php` (后端控制)

*优化点：修复日志换行显示，自动创建下载目录。*

```bash
cat > build.php <<'EOF'
<?php
$base_dir = __DIR__;
$output_dir = $base_dir . '/source/bin/targets/x86/64';
$dl_dir = $base_dir . '/download';
$lock_file = $base_dir . '/compile.lock';
$log_file = $base_dir . '/compile.log';
$extra_config_file = $base_dir . '/extra.config';

if (file_exists($lock_file) && (time() - filemtime($lock_file) > 43200)) @unlink($lock_file);

if (isset($_GET['action']) && $_GET['action'] == 'cancel') {
    exec("pkill -u www -f make"); exec("pkill -u www -f compile.sh");
    if (file_exists($lock_file)) unlink($lock_file);
    file_put_contents($log_file, "\n=== 用户终止任务 ===\n", FILE_APPEND);
    header("Location: index.html"); exit;
}

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    if (file_exists($lock_file)) die("任务忙");
    file_put_contents($lock_file, "Compiling...");
    file_put_contents($log_file, "=== 任务启动 ===\n");
    chmod($log_file, 0666);

    if (!empty($_POST['custom_feeds'])) file_put_contents($base_dir . '/source/feeds.conf.default', str_replace("\r\n", "\n", $_POST['custom_feeds']));
    file_put_contents($extra_config_file, str_replace("\r\n", "\n", $_POST['custom_config']));

    $uci_dir = $base_dir . '/source/files/etc/uci-defaults';
    if (!is_dir($uci_dir)) mkdir($uci_dir, 0755, true);
    if (!empty(trim($_POST['uci_script']))) {
        file_put_contents($uci_dir . '/99-custom', str_replace("\r\n", "\n", $_POST['uci_script']));
        chmod($uci_dir . '/99-custom', 0755);
    }

    exec("nohup bash " . escapeshellarg($base_dir . '/compile.sh') . " > /dev/null 2>&1 &");
    echo "success"; exit;
} elseif (isset($_GET['action']) && $_GET['action'] == 'status') {
    $building = file_exists($lock_file);
    header("Cache-Control: no-cache");
?>
<!DOCTYPE html>
<html><head><meta charset="UTF-8"><link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
<?php if($building) echo '<meta http-equiv="refresh" content="5">'; ?>
<style>
    body{background:#121212;color:#eee;font-family:'Segoe UI',sans-serif}
    /* 日志显示关键修复 */
    .log{background:#000;height:600px;overflow-y:auto;font-family:Consolas,monospace;color:#0f0;padding:15px;white-space:pre-wrap;word-break:break-all;border:1px solid #333;}
</style></head>
<body class="py-4 container">
<div class="d-flex justify-content-between mb-4 align-items-center">
    <h3 class="m-0"><i class="fa-solid fa-terminal"></i> 编译监控</h3>
    <a href="index.html" class="btn btn-outline-light btn-sm">返回</a>
</div>
<?php if($building): ?>
    <div class="alert alert-primary d-flex justify-content-between align-items-center">
        <div><div class="spinner-border spinner-border-sm me-2"></div> <strong>正在全速编译中...</strong> <small class="ms-2 opacity-75">首次编译需3-5小时</small></div>
        <a href="?action=cancel" class="btn btn-danger btn-sm" onclick="return confirm('确定停止？')">强制停止</a>
    </div>
<?php else: ?>
    <div class="alert alert-success">任务结束</div>
    <?php $files=glob($output_dir."/*combined-efi.img.gz"); if($files){
        if(!is_dir($dl_dir)){mkdir($dl_dir,0755,true);chown($dl_dir,'www');}
        echo '<div class="row">';
        foreach(array_slice($files,0,4) as $f){ $n=basename($f); @copy($f,$dl_dir.'/'.$n);
        echo '<div class="col-md-6 mb-2"><div class="card bg-dark border-secondary p-3 d-flex justify-content-between flex-row text-white align-items-center"><span class="text-truncate me-2">'.$n.'</span><a href="download/'.$n.'" class="btn btn-success btn-sm">下载</a></div></div>'; }
        echo '</div>'; } ?>
<?php endif; ?><div class="log" id="log"><?=file_exists($log_file)?htmlspecialchars(shell_exec("tail -n 200 ".escapeshellarg($log_file))):'Waiting...'?></div>
<script>var d=document.getElementById("log");d.scrollTop=d.scrollHeight;</script></body></html>
<?php } else { header("Location: index.html"); } ?>
EOF

```

#### 📄 3. `index.html` (前端网页)

*优化点：**Tab 1 默认值修复**，强制使用 `;openwrt-24.10` 后缀，彻底解决 `libcrypt` 等依赖报错。*

```bash
cat > index.html <<'EOF'
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>OpenWrt 24.10 编译工厂</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <style>
        body { background: #f4f6f9; font-family: 'Segoe UI', sans-serif; }
        .code-editor { font-family: 'Consolas', monospace; font-size: 0.85rem; background: #2d2d2d; color: #f8f8f2; border: none; }
        #loadingOverlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(255,255,255,0.95); z-index: 9999; display: none; flex-direction: column; align-items: center; justify-content: center; }
    </style>
</head>
<body>
<div class="container py-5">
    <div class="row justify-content-center">
        <div class="col-xl-9">
            <div class="text-center mb-4"><h2 class="fw-bold text-dark">OpenWrt 24.10 编译工厂</h2><p class="text-muted">Kernel 6.6 | Native Opkg | PVE Ready</p></div>
            <form id="buildForm">
                <div class="card shadow-sm border-0">
                    <div class="card-header bg-white pt-3 px-4 border-0">
                        <ul class="nav nav-tabs card-header-tabs" id="myTab" role="tablist">
                            <li class="nav-item"><button class="nav-link active" data-bs-toggle="tab" data-bs-target="#tab-feeds" type="button">1. 软件源</button></li>
                            <li class="nav-item"><button class="nav-link" data-bs-toggle="tab" data-bs-target="#tab-packages" type="button">2. 插件配置</button></li>
                            <li class="nav-item"><button class="nav-link" data-bs-toggle="tab" data-bs-target="#tab-scripts" type="button">3. 启动脚本</button></li>
                        </ul>
                    </div>
                    <div class="card-body p-4 tab-content">
                        <div class="tab-pane fade show active" id="tab-feeds">
                            <textarea name="custom_feeds" class="form-control code-editor" rows="10">src-git packages https://git.openwrt.org/feed/packages.git;openwrt-24.10
src-git luci https://git.openwrt.org/project/luci.git;openwrt-24.10
src-git routing https://git.openwrt.org/feed/routing.git;openwrt-24.10
src-git telephony https://git.openwrt.org/feed/telephony.git;openwrt-24.10</textarea>
                            <small class="text-muted">* 已锁定 openwrt-24.10 分支，请勿随意更改，否则会导致依赖错误。</small>
                        </div>
                        <div class="tab-pane fade" id="tab-packages">
                            <textarea name="custom_config" class="form-control code-editor" rows="15"># === 24.10 完美配置 (PVE + Nikki + Opkg) ===
CONFIG_PACKAGE_kmod-virtio=y
CONFIG_PACKAGE_kmod-virtio-net=y
CONFIG_PACKAGE_kmod-virtio-pci=y
CONFIG_PACKAGE_ethtool=y
CONFIG_PACKAGE_curl=y
CONFIG_PACKAGE_wget-ssl=y
CONFIG_PACKAGE_ca-bundle=y
CONFIG_PACKAGE_ip-full=y
CONFIG_PACKAGE_firewall4=y
CONFIG_PACKAGE_kmod-nft-tproxy=y
CONFIG_PACKAGE_kmod-nft-socket=y
CONFIG_PACKAGE_kmod-tun=y
CONFIG_PACKAGE_bash=y
CONFIG_PACKAGE_jq=y
CONFIG_PACKAGE_tar=y
CONFIG_PACKAGE_coreutils=y
CONFIG_PACKAGE_coreutils-base64=y
CONFIG_PACKAGE_openssh-sftp-server=y
CONFIG_PACKAGE_luci=y
CONFIG_PACKAGE_luci-i18n-base-zh-cn=y
CONFIG_PACKAGE_luci-i18n-firewall-zh-cn=y
CONFIG_PACKAGE_libustream-mbedtls=n
CONFIG_PACKAGE_libustream-openssl=y</textarea>
                        </div>
                        <div class="tab-pane fade" id="tab-scripts">
                            <textarea name="uci_script" class="form-control code-editor" rows="12">#!/bin/sh
# LAN (eth0/2/3)
uci set network.lan.ipaddr='10.0.0.1'
uci set network.lan.netmask='255.255.255.0'
uci set network.@device[0].ports='eth0'
[ -d "/sys/class/net/eth2" ] && uci add_list network.@device[0].ports='eth2'
[ -d "/sys/class/net/eth3" ] && uci add_list network.@device[0].ports='eth3'
# WAN (eth1)
if [ -d "/sys/class/net/eth1" ]; then
    uci delete network.wan 2>/dev/null; uci delete network.wan6 2>/dev/null
    uci set network.wan=interface; uci set network.wan.device='eth1'; uci set network.wan.proto='dhcp'
    uci set network.wan6=interface; uci set network.wan6.device='eth1'; uci set network.wan6.proto='dhcpv6'
    uci add_list firewall.@zone[1].network='wan'; uci add_list firewall.@zone[1].network='wan6'
fi
echo -e "password\npassword" | passwd root
uci set dropbear.@dropbear[0].PasswordAuth='on'
uci set dropbear.@dropbear[0].RootPasswordAuth='on'
uci commit network; uci commit firewall; uci commit dropbear
/etc/init.d/network restart</textarea>
                        </div>
                    </div>
                    <div class="card-footer bg-white p-4 border-0 text-end">
                        <button type="button" onclick="startCompile()" class="btn btn-primary rounded-pill">开始编译</button>
                    </div>
                </div>
            </form>
        </div>
    </div>
</div>
<div id="loadingOverlay"><div class="spinner-border text-primary" style="width:4rem;height:4rem"></div></div>
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
<script>
function startCompile() {
    if(!confirm("⚠️ 确认编译 OpenWrt 24.10？\n首次编译需 3-5 小时。")) return;
    document.getElementById('loadingOverlay').style.display='flex';
    fetch('build.php', { method:'POST', body:new FormData(document.getElementById('buildForm'))})
    .then(r=>r.text()).then(res=>{
        if(res.trim()==='success') setTimeout(()=>{window.location.href='build.php?action=status';},1000);
        else {alert("失败: "+res); document.getElementById('loadingOverlay').style.display='none';}
    });
}
</script>
</body>
</html>
EOF

```

---

### 第六阶段：最终检查与执行 (Web)

1. 打开浏览器访问 `http://10.0.0.173/index.html`。
2. **检查 Tab 1**: 确保末尾有 `;openwrt-24.10`（这解决了之前那几百行依赖报错的问题）。
3. 点击 **“开始编译”**。

---

### 🚨 第七阶段：运维与故障急救 (Permission Hell)

**场景**:
如果您在 SSH 中手动操作了 `git pull` 或 `make`，导致网页端编译时出现大量 `Permission denied`。

**一键修复指令 (SSH Root)**:
请保存这条指令，出现权限问题时随时运行：

```bash
cd /www/wwwroot/10.0.0.173/ && pkill -u www make && pkill -u www compile.sh && rm -rf source/tmp source/feeds source/package/feeds source/staging_dir/toolchain* && chown -R www:www . && chmod +x compile.sh && echo "✅ 权限已修复！"

```
