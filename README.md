# XG-040G-TF PonWrt 云编译

这个仓库只负责给 Nokia `XG-040G-TF` 编译 PonWrt 固件，目标 profile 固定为：

```text
nokia_xg-040g-tf-ubi
```

基础源码使用 [pbs05/ponwrt](https://github.com/pbs05/ponwrt)，它包含
AN7581 的 PON 驱动、用户态控制工具和 LuCI PON 页面。工作流不会把任何一台设备的
`bosa`、`ri` 或 `romfile` 备份提交到仓库。

## GitHub 云编译

1. 把本仓库推到自己的 GitHub 仓库，进入 `Settings -> Actions -> General`，允许 Actions 运行。
2. 打开 `Actions -> Build PonWrt for XG-040G-TF -> Run workflow`。
3. `ponwrt_ref` 默认填 `master`；需要可复现构建时填一个提交 SHA。
4. 需要附加软件包时，在 `extra_packages` 填空格分隔的包名，例如：

   ```text
   luci-app-sqm kmod-tun
   ```

   删除基础包时在包名前加 `-`，例如 `-luci-app-upnp`。
5. 编译完成后，在运行页面底部下载 `xg-040g-tf-ponwrt-*` Artifact。先核对
   `SHA256SUMS`，再使用其中的固件文件。
6. 需要把产物发布到仓库 Release 时，将 `publish_release` 设为 `true`；默认只上传
   Actions Artifact，避免测试构建自动创建 Release。

工作流默认只上传 Artifact，不自动创建 Release，也不会自动刷写设备。

将 `include_all_kmods` 设为 `true` 时，CI 会启用 `CONFIG_ALL_KMODS=y`，额外生成
`xg-040g-kmods.zip`。压缩包里包含本次构建生成的全部目标兼容 `kmod-*` 包，以及匹配的
kernel/firmware 伴随包、`.config`、清单和校验文件。

这个压缩包不是通用驱动仓库，只能用于同一 PonWrt 提交、同一内核配置和同一设备固件。
不要把其他 OpenWrt/ImmortalWrt 版本的 kmod 混进来；升级到新的固件后，应重新生成对应
版本的 kmod 压缩包。`CONFIG_ALL_KMODS` 会显著增加编译时间、磁盘占用和 Artifact 大小。
固件目录中的 `SHA256SUMS` 会覆盖固件、配置文件和这个 ZIP；解压 ZIP 后，可在压缩包根目录
执行 `sha256sum -c SHA256SUMS` 校验其中的包、清单和构建配置。

## 上游插件兼容性

PonWrt 的 `feeds.conf.default` 仍然使用 ImmortalWrt 的 `packages`、`luci` 和 OpenWrt 的
`routing`、`telephony`、`video` feeds，同时额外加入 `pon_drivers`、`pon_userspace` 两个
PON feed。工作流每次构建都会执行 `feeds update -a`、`feeds install -a`，所以普通 LuCI
插件和用户态程序可以通过 `extra_packages` 加入固件。

需要区分三类包：

- **纯用户态 / LuCI 包**：通常可以直接同步并编译，例如常规 DNS、代理、监控类插件；
- **`kmod-*` 内核模块**：必须在同一份 PonWrt 源码、同一个 6.18 内核和同一份配置中重编，
  不能拿别的 OpenWrt/ImmortalWrt 固件的模块直接安装；
- **依赖 `opkg` 的旧插件**：PonWrt 使用 `apk`，插件脚本如果硬编码 `opkg`、`.ipk` 下载或
  `/etc/opkg`，需要先适配后再用。

PonWrt 的 AN7581 默认配置还会关闭 `luci-app-iptv`，因此本仓库也不强制内置它；确实需要
时再把 `luci-app-iptv` 放进 `extra_packages`，让 CI 验证它在当前 feed 组合下能否编译。

工作流会在 `make defconfig` 后验证你输入的附加包仍然存在，并把所有 feed 的 commit 写入
`build-manifest.txt`。这能避免上游插件改名、依赖不满足或被 Kconfig 静默移除后仍产出错误
固件。

## PON 数据必须先备份

固件和 PON 身份数据是两回事。刷机前必须通过串口或原厂系统备份整片 flash，并至少
单独保存以下逐机数据：

- `romfile`：SN、PON 认证等原厂信息；
- `bosa`：光模块校准；
- `ri`：MAC、序列号等板级身份数据。

具体 MTD 编号以设备上的 `/proc/mtd` 为准，不要把网上其他批次的编号直接套用。备份文件
不要放进这个公开 Git 仓库，也不要把别人的身份数据写入自己的设备。

## 刷机和恢复 PON

这个 profile 是 UBI 布局，不能把它当成普通家用路由器固件直接刷到任意设备：

- 需要按 AN758x/PonWrt 的 U-Boot 迁移流程准备引导程序和 Web 恢复环境；
- 首次迁移时准备 USB-TTL 串口，断电或写入中断可能需要串口救砖；
- 刷入后，在 U-Boot Web 或 LuCI 的“网络 -> PON -> 配置 -> PON board data”中恢复自己
  备份的 `bosa` 和 `ri`；
- `romfile` 的恢复按你的 U-Boot/原厂迁移流程处理，不能用 `bosa` 或 `ri` 文件代替；
- 先确认光猫属于 XG-040G-TF，不能把 MD、MF 或其他 AN7581 设备的引导文件混用。

PON 注册仍取决于运营商侧认证、原厂身份数据是否完整，以及你使用的光模块/线路模式。
本仓库只提供构建流程，不保证任意线路可以注册。

## 本地复现

本地需要 Linux（或 WSL2）和 OpenWrt/ImmortalWrt 编译依赖。最接近 CI 的流程是：

```sh
git clone https://github.com/pbs05/ponwrt.git openwrt
cd openwrt
./scripts/feeds update -a
./scripts/feeds install -a
cp ../configs/xg-040g-tf.config .config
make defconfig
make -j"$(nproc)" world
```

产物在 `openwrt/bin/targets/airoha/an7581/`。

## 风险边界

刷机前保留可用的整片备份、USB-TTL 和原厂回退路径。请只在自己拥有或获授权的设备与线路
上使用原厂身份数据，不要复制他人的 SN、MAC 或 PON 认证信息。
