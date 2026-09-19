# ImmortalWrt Docker rootfs for 网心云 OEC (RK3566)

用 GitHub Actions 云编译 ImmortalWrt **armsr/armv8**（通用 aarch64）的
`rootfs.tar.gz`，`docker import` 后即可在 OEC 上以容器方式跑 ImmortalWrt。

---

## 一、为什么选 armsr/armv8 而不是 rockchip

| 项 | 说明 |
|---|---|
| **容器共用宿主内核** | Docker 容器不启动自己的内核，用的是 OEC 宿主（Armbian）的内核。rootfs 里的 `kmod-*` 内核模块**对容器无效**，靠宿主内核提供。 |
| **不需要板级 dtb/u-boot** | dtb 与 u-boot 是引导阶段的事，容器里完全用不到。 |
| **不需要 WiFi 驱动** | 正是你的要求。容器内没有 PCIe 直通，WiFi 由宿主管理。 |
| **指令集匹配** | OEC 是 RK3566 = ARMv8-A aarch64；armsr/armv8 同为 aarch64，二进制完全兼容。 |
| **官方产物** | ImmortalWrt 官方 `armsr/armv8` 就发布 `rootfs.tar.gz`，本就是给容器/VM 用的。 |

> 结论：**整机固件（rockchip）≠ 容器 rootfs**。容器方案选 armsr/armv8 是正解。

---

## 二、产物与获取

云编译完成后，在 GitHub 仓库 **Actions → 最新 run → Artifacts** 下载：

- `immortalwrt-docker-rootfs`（含 `immortalwrt-armsr-armv8-generic-rootfs.tar.gz`）

仓库：`https://github.com/sy3tem/ImmortalWrt-docker`

产物文件名形如：
```
immortalwrt-armsr-armv8-generic-rootfs.tar.gz
```

---

## 三、在 OEC 上导入并运行

### 0. 前置：确认宿主内核能力

容器共用宿主内核，OpenClash 的透明代理需要宿主的这些模块。**先在 OEC 上检查**：

```bash
# 必需：TUN（透明代理 / tun 模式）
ls /dev/net/tun && echo "TUN OK"

# 必需：nftables 相关（OpenClash 默认 nft 模式）
sudo modprobe nft_tproxy 2>/dev/null; lsmod | grep -E "nft_tproxy|nft_socket|tun"
sudo nft list tables >/dev/null 2>&1 && echo "nft OK"

# 可选：inet-diag（部分模式需要）
sudo modprobe inet_diag 2>/dev/null
```

若 `/dev/net/tun` 不存在，在宿主上执行：
```bash
sudo modprobe tun
echo tun | sudo tee -a /etc/modules   # 开机自动加载
```

### 1. 导入镜像

```bash
# 把 tar.gz 传到 OEC，然后：
docker import immortalwrt-armsr-armv8-generic-rootfs.tar.gz immortalwrt:oec
docker images | grep immortalwrt
```

### 2. 首次运行（关键参数）

OpenWrt 用 procd 作为 init，容器里必须让它当 PID 1：

```bash
docker run -d \
  --name immortalwrt \
  --restart unless-stopped \
  --privileged \
  --network host \
  -v /lib/modules:/lib/modules:ro \
  -v /dev/net/tun:/dev/net/tun \
  --cap-add NET_ADMIN --cap-add NET_RAW --cap-add SYS_MODULE \
  immortalwrt:oec /sbin/init
```

参数说明：

| 参数 | 作用 |
|---|---|
| `--privileged` | OpenClash 要操作 nftables/路由表/TUN，需要较高权限。 |
| `--network host` | 透明代理必须共用宿主网络栈，否则无法代理宿主流量。 |
| `-v /lib/modules:/lib/modules:ro` | 让容器能看到**宿主**内核模块（容器内 kmod 包用不上）。 |
| `-v /dev/net/tun:/dev/net/tun` | TUN 设备。 |
| `/sbin/init` | 启动 OpenWrt 的 procd，拉起全部服务。 |

### 3. 进入容器 / 访问 LuCI

```bash
docker exec -it immortalwrt /bin/sh
```

LuCI 默认监听容器内的 `192.168.1.1:80`（`--network host` 下即宿主该地址）。
**建议改掉默认 LAN 地址**，避免和 OEC 宿主网段冲突：

```bash
docker exec -it immortalwrt sh -c "
  uci set network.lan.ipaddr='192.168.100.1'
  uci commit network
  /etc/init.d/network restart
"
```

然后浏览器打开 `http://<OEC-IP>`，默认无密码（首次登录请立即设置）。

### 4. 若容器启动即退出（preinit 问题）

OpenWrt 的 `/sbin/init` 会走 preinit → `80_mount_root`，它会尝试挂载根分区。
容器里没有独立的块设备，个别宿主环境下会卡住或退出。排查与对策：

```bash
# 看容器日志
docker logs immortalwrt

# 若卡在 preinit / mount_root，给容器加上 proc/sys 与 pid 命名空间
docker rm -f immortalwrt
docker run -d --name immortalwrt --restart unless-stopped \
  --privileged --network host \
  -v /lib/modules:/lib/modules:ro \
  -v /dev/net/tun:/dev/net/tun \
  --cap-add NET_ADMIN --cap-add NET_RAW --cap-add SYS_MODULE \
  --tmpfs /tmp --tmpfs /run \
  immortalwrt:oec /sbin/init
```

仍不行时，退一步只跑关键服务（不启动完整 procd 流程）：

```bash
docker run -d --name immortalwrt --privileged --network host \
  immortalwrt:oec /bin/sh -c "/etc/init.d/openclash start; sleep infinity"
```

> 注：ImmortalWrt 的 `/sbin/init` 由 **procd** 包提供（已确认在 rootfs 内），
> 这是让它作为 PID 1 正常工作的正确入口。

---

## 四、OpenClash 使用

1. LuCI → **服务 → OpenClash**
2. 首次进入会提示下载**内核**（Clash/Mihomo）与**规则库**——容器需能联网。
3. 「配置文件订阅」添加你的订阅链接 → 更新 → 启动。
4. 「运行模式」建议：
   - **Fake-IP (TUN)** 或 **TProxy**（需宿主内核支持，见第三节第 0 步）
   - 「绕过中国大陆」按需开启

> OpenClash 的依赖（`kmod-tun`、`kmod-nft-tproxy`、`kmod-inet-diag`、`ruby`、
> `dnsmasq-full` 等）已编入 rootfs；但**内核模块实际由宿主提供**，
> 所以宿主内核必须带 `tun` / `nft_tproxy`。

---

## 五、已知限制

1. **容器内 `kmod-*` 无效**。rootfs 里的 kmod 包只是依赖占位，真正的模块来自宿主内核。
   若宿主缺 `nft_tproxy`，OpenClash 请改用「Redir 模式」（走 iptables/nft 重定向，兼容性更好）。
2. **`/etc/config/network` 的物理接口名**取决于宿主。容器内看不到 OEC 的 `eth0/eth1`，
   透明代理场景一般无需配置接口，交给宿主转发即可。
3. **不要在容器内启用 DHCP/DNS 抢宿主**。若只想做透明代理，建议关闭容器内
   `dnsmasq` 的 53 端口监听，或在 LuCI 里把 DHCP 关掉，避免与宿主冲突。
4. **重启后配置丢失**：容器重建即回到初始状态。需要持久化请挂载卷：   ```bash
   -v /opt/imm-etc:/etc -v /opt/imm-root:/root
   ```
   （更稳妥的做法是只挂载 `/etc/config` 与 `/etc/openclash`。）

---

## 六、自己重新编译

仓库里两个文件即可复现：

- `config/imm-docker-defconfig` —— 编译配置（armsr/armv8 + TARGZ + OpenClash + LuCI）
- `.github/workflows/build-rootfs.yml` —— 云编译流程

手动触发：**Actions → Build ImmortalWrt Docker rootfs → Run workflow**

可调参数：
- `imm_ref`：ImmortalWrt 源码分支/tag（默认 `master`，即最新）
- `proxy_plugin`：`luci-app-openclash` / `luci-app-passwall` / `luci-app-homeproxy` / `none`
- `strip_nic_drivers`：是否剔除 armsr 默认的 28 个物理网卡驱动（默认 `true`）

### 关于精简（容器环境专优化）

这套配置**刻意去掉了整机固件才需要、容器里纯属累赘**的东西：

| 剔除项 | 原因 |
|---|---|
| `docker` / `dockerd` / `docker-compose` / `luci-app-dockerman` | 容器里再跑 docker 是 DinD，无意义；`dockerd` 还是 Go 编译，占整个构建 20~30 分钟 |
| `parted` / `lsblk` / `block-mount` / `kmod-fs-ext4` / `kmod-usb-storage` | 容器内没有独立块设备，分区/格式化/挂载全由宿主完成 |
| **28 个物理网卡驱动** | armsr 的 `DEVICE_PACKAGES` 默认带入 `kmod-e1000e`/`vmxnet3`/`dwmac-rockchip`/`mvneta`/`kmod-sfp`/`kmod-phy-*` 等；容器共用宿主网络栈，一个都用不到。工作流会 patch 掉 `target/linux/armsr/image/Makefile` |
| WiFi（`kmod-mt7915e` 等） | 按要求不带 |

保留的都是容器里真正会用到的：LuCI + OpenClash 全套依赖（`nftables`/`iptables-nft`/`ipset`/
`dnsmasq-full`/`ruby`/`kmod-tun` 等）、`ttyd` 终端、以及 `curl`/`jq`/`nmap`/`tcpdump` 等排障工具。

本地复现（Linux/WSL，需 ≥8GB 内存、≥30GB 磁盘）：
```bash
git clone --depth 1 https://github.com/immortalwrt/immortalwrt.git imm
cd imm
./scripts/feeds update -a && ./scripts/feeds install -a

# 剔除容器用不上的物理网卡驱动（与云编译同一逻辑）
python3 - target/linux/armsr/image/Makefile <<'PY'
import sys
p = sys.argv[1]
lines = open(p, 'r', newline='').read().split('\n')
out, i = [], 0
while i < len(lines):
    line = lines[i]
    if line.lstrip().startswith('DEVICE_PACKAGES +='):
        while i < len(lines):
            cur = lines[i]; i += 1
            if not cur.rstrip('\r').rstrip().endswith('\\'): break
        out.append('\t# DEVICE_PACKAGES removed for container builds')
        continue
    out.append(line); i += 1
open(p, 'w', newline='').write('\n'.join(out))
PY

cp ../config/imm-docker-defconfig .config
make defconfig
make -j$(nproc) download
make -j$(nproc)
ls -lh bin/targets/armsr/armv8/*rootfs.tar.gz
```

---

## 七、与之前 RT28 整机固件的区别

| | RT28 整机固件 | 本项目（OEC docker rootfs） |
|---|---|---|
| 目标 | `rockchip/armv8` | `armsr/armv8` |
| 产物 | `squashfs-sysupgrade.img.gz` | `rootfs.tar.gz` |
| 用途 | 烧写 eMMC/SD 直接启动 | `docker import` 跑容器 |
| 内核 | 自带（含板级 dtb/驱动） | **用宿主的** |
| u-boot | 需要 | 不需要 |
| WiFi | 需要 kmod | 不需要 |
