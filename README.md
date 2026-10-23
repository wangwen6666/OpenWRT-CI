# MY-NOTE · 亚瑟 owrt 固件构建说明

> 本仓库是 `VIKINGYFY/OpenWRT-CI` 的个人 Fork，用于编译 **京东云亚瑟（RE-SS-01 / AX1800 Pro，IPQ6000）** 的 `owrt` 固件（OpenWrt 主线 + ImmortalWrt 软件层 + 满血 PPE 开源硬件加速）。
>
> **这是本人自定义说明，上游没有此文件，点 Sync fork 同步上游时不会被覆盖**，可长期保留。
> 要让 GitHub 仓库首页（项目首页）显示本内容，需把本文件内容**也粘进 `README.md`**——因为 GitHub 只渲染 README.md。README.md 会在 Sync fork 时被上游覆盖，之后从本文件重新粘一次即可。

**目标固件规格**

| 项目 | 值 |
|---|---|
| 编译配置 | `IPQ60XX-WIFI-YES`（带 WiFi） |
| 主题 | argon |
| 后台地址 | 192.168.5.1 |
| 主机名 | OpenWrt |
| WiFi SSID | JDC_AX1800 |
| 网页/SSH 账号 | root（密码见构建默认值） |
| 内置插件 | OpenClash / PassWall / PassWall2 / SSR-Plus（**只启用一个**）、mosdns、ddnsto、quickfile、ttyd、upnp、easytier、netspeedtest、diskman |

---

## 一、本 Fork 相对上游的自定义改动

### 1. `Config/PRIVATE.txt`（插件清单，整文件）

关键点：插件全部写在这里，会被 `Settings.sh` 自动并入 `.config`。

```txt
# ============================================================
# PRIVATE.txt —— 私人插件清单
# 格式：CONFIG_PACKAGE_<包名>=y   （依赖会自动补齐）
# ============================================================

# ---------- 科学上网（4 款，平时只启用 1 个） ----------
CONFIG_PACKAGE_luci-app-openclash=y
CONFIG_PACKAGE_luci-app-passwall=y
CONFIG_PACKAGE_luci-app-passwall2=y
CONFIG_PACKAGE_luci-app-ssr-plus=y      # 依赖下方 helloworld 源

# ---------- DNS ----------
CONFIG_PACKAGE_luci-app-mosdns=y

# ---------- 工具 ----------
CONFIG_PACKAGE_luci-app-ttyd=y
CONFIG_PACKAGE_luci-app-quickfile=y     # 悟空 quickfile 文件管理器（前端）
CONFIG_PACKAGE_quickfile=y              # quickfile 后端
CONFIG_PACKAGE_luci-app-upnp=y
CONFIG_PACKAGE_luci-app-easytier=y
CONFIG_PACKAGE_luci-app-ddnsto=y        # DDNSTO 远程控制
CONFIG_PACKAGE_luci-app-netspeedtest=y
CONFIG_PACKAGE_luci-app-diskman=y
```

> 注：`luci-app-filetransfer` 在本编译源中不存在（会被 kconfig 静默丢弃），已改用悟空 quickfile。

### 2. `Scripts/Packages.sh`（新增/替换 3 个外部源）

```bash
# SSR-Plus 来源（helloworld 的默认分支是 dev，不是 master）
UPDATE_PACKAGE "helloworld" "fw876/helloworld" "dev"

# DDNSTO 来源（从 kenzok8 大仓库按目录提取 ddnsto + luci-app-ddnsto）
UPDATE_PACKAGE "ddnsto" "kenzok8/openwrt-packages" "master" "pkg"

# 悟空 quickfile（替换上游 sbwml 版，两者包名相同会撞车）
UPDATE_PACKAGE "quickfile" "wkccd/quickfile" "v2026.05.18"
```

### 3. `Scripts/Settings.sh`（可选：注入 root 密码）

上游 `WRT_PW` 变量实际未被脚本引用，如需固定密码，可在文件末尾追加 uci-defaults 脚本：

```bash
mkdir -p ./files/etc/uci-defaults
cat > ./files/etc/uci-defaults/99-root-password <<'EOF'
#!/bin/sh
echo 'root:root' | chpasswd 2>/dev/null
EOF
chmod +x ./files/etc/uci-defaults/99-root-password
```

### 4. `.github/workflows/QCA-ALL.yml`（构建默认值 + 矩阵）

```yaml
    with:
      WRT_THEME: argon
      WRT_NAME: OpenWrt
      WRT_SSID: JDC_AX1800
      WRT_WORD: 12345678
      WRT_IP: 192.168.5.1
      WRT_PW: 无
```

矩阵只保留亚瑟，避免白编其它机型：

```yaml
        CONFIG: [IPQ60XX-WIFI-YES]
```

---

## 二、构建流程

1. **Actions** → 选 **QCA-ALL** → 右上 **Run workflow**
2. `PACKAGE` 留空；`TEST` 勾选 = 只生成 `.config` 不编译（校验用）；不勾 = 全量编译
3. **先跑一次 TEST**，去 Releases 下载 `Config-*.txt`，确认关键行都是 `=y`：

```bash
grep -E "luci-app-(openclash|passwall|passwall2|ssr-plus|ddnsto|quickfile)|^CONFIG_PACKAGE_quickfile=" Config-IPQ60XX-*.txt
```

4. 校验通过后，**取消勾选 TEST** 再跑一次 = 全量编译（约 1~1.5 小时）

> ⚠️ TEST 全绿不等于插件都进了固件：`make defconfig` 会**静默丢弃**不认识的 `CONFIG_PACKAGE_*`，必须人工核对 `.config`。

---

## 三、Releases 产物与刷机

`Package Firmware` 步骤会把 `bin/targets/` 下所有文件（去掉 buildinfo/json/sha256sums/packages）上传，因此会同时出现两种镜像：

| 场景 | 用哪个 |
|---|---|
| 从原厂固件首次刷入 | **factory**（`*-squashfs-factory.bin` / `.ubi`），走 U-Boot / 恢复模式 |
| 已是 OpenWrt 系，升级 | **sysupgrade**（`*-squashfs-sysupgrade.bin`），LuCI 系统→备份/升级，保留配置 |

---

## 四、已知风险

1. **quickfile 后端联网下载**：`quickfile` 的 Makefile 在**编译时**从 `r2.cooluc.com` 拉预编译包（哈希已固定）。该域名不通会导致整个编译失败。
   - 兜底：刷机后手动装 `wkccd/quickfile` Release `v2026.05.18` 里的 `aarch64_cortex-a53` 包（`luci-app-quickfile` + `quickfile` + `luci-i18n-quickfile-zh-cn`）。
2. **luci-nginx**：quickfile 前端依赖 `luci-nginx`，会额外引入，属正常。
3. **4 款科学上网不要同时启用**：DNS/透明代理规则会互相抢，只开一个，其余保持关闭（不启用即不会开机自启）。

---

## 五、⚠️ Sync fork 会覆盖的自定义

本仓库直接修改了上游作者文件，点 **Sync fork** 同步上游会**整文件覆盖**，以下内容将丢失、需重新加回：

- `Config/PRIVATE.txt` 全部插件行
- `Scripts/Packages.sh`：helloworld / ddnsto / quickfile(wkccd) 三行
- `Scripts/Settings.sh`：root 密码 uci-defaults 段
- `.github/workflows/QCA-ALL.yml`：`with:` 默认值与矩阵
- `README.md`：首页说明（Sync 后会变回上游广告页，从本 MY-NOTE 重新粘）

建议同步后照本 MY-NOTE 逐项恢复，或为自定义内容单独保留一份备份。

---

## 六、编译额度（公开仓库）

| 项目 | 限制 |
|---|---|
| 分钟数 | public 仓库免费、不限时长 |
| 单 job 时长 | 硬上限 6 小时 |
| 机器 | 默认 2 核 Linux runner |
| 并发 | 免费约 20 个 job |
| Artifact/Release 存储 | 每仓库免费 500MB（固件较大，旧 Release 及时清理） |
