# PonWrt 多机型云编译

这个仓库用于给多个 Airoha/PonWrt 设备编译固件。当前默认机型是 Nokia
`XG-040G-TF`，目标 profile 为：

```text
nokia_xg-040g-tf-ubi
```

基础源码使用 [pbs05/ponwrt](https://github.com/pbs05/ponwrt)，它包含
AN7581 的 PON 驱动、用户态控制工具和 LuCI PON 页面。工作流不会把任何一台设备的
`bosa`、`ri` 或 `romfile` 备份提交到仓库。

## GitHub 云编译

1. 进入对应机型的 `configs/<model>.build.env` 修改源码分支、附加包、是否打包全量
   kmod/软件包、是否启用 ccache 和是否发布 Release：

   ```text
   PONWRT_REF="master"
   EXTRA_PACKAGES="luci-app-sqm kmod-tun"
   EXCLUDE_KMODS="kmod-ipt-rtpengine"
   INCLUDE_ALL_KMODS="true"
   INCLUDE_ALL_PACKAGES="true"
   USE_CCACHE="true"
   PUBLISH_RELEASE="true"
   ```

   `<model>.build.env` 还定义了机型名称、目标 Kconfig、产物目录和必须的软件包。
   例如当前配置文件是 `configs/xg-040g-tf.build.env`，配套的 OpenWrt 配置是
   `configs/xg-040g-tf.config`。新增机型时复制这两个文件并修改其中的
   `MODEL_ID`、`TARGET_CONFIG_SYMBOL`、`TARGET_SUBDIR`、`ARTIFACT_PREFIX` 和
   `REQUIRED_PACKAGES`。

   删除基础包时在包名前加 `-`，例如 `EXTRA_PACKAGES="-luci-app-upnp"`。
   `EXCLUDE_KMODS` 用于从全量 kmod 扩展中排除当前 PonWrt 内核树无法编译的模块；
   它只接受 `kmod-*` 包名，并会写入最终 `.config` 作为显式禁用项。
2. 推送配置文件到 `main` 会触发一次检查；只有发现 `PONWRT_REF`、普通插件
   feeds、PON feeds 或机型配置指纹变化时才编译。工作流每天北京时间 02:00
   也会执行同样的检查；成功构建后会记录按机型区分的状态。
3. 手动运行时选择 `model`（配置文件前缀）和 `force_build`；其他构建参数继续由
   对应的 `configs/<model>.build.env` 管理。
4. 编译完成后，在运行页面底部下载 `<model>-ponwrt-*` Artifact。先核对
   `SHA256SUMS`，再使用其中的固件文件。
5. 手动运行工作流时，在 `model` 输入中填写配置文件前缀，例如 `xg-040g-tf`；
   定时和推送触发默认使用 `xg-040g-tf`。`PUBLISH_RELEASE="true"` 时，工作流会按“机型
   + PonWrt 提交短 SHA + 配置指纹 + feed
   指纹”创建不可覆盖的 GitHub Release；每次上游提交、feed 或配置发生变化都会
   形成独立归档，旧 Release 不会被替换。成功创建的 Release 会显式标记为
   GitHub `latest`，说明同时提供中文和 English 两部分。设为 `false` 时只保留
   Actions Artifact。

工作流不会自动刷写设备，也不会把 PON 身份数据写入产物。

将 `INCLUDE_ALL_KMODS="true"` 时，CI 会启用 `CONFIG_ALL_KMODS=y`，额外生成
`<artifact-prefix>-kmods.zip`。压缩包里包含本次构建生成的全部目标兼容 `kmod-*` 包，以及匹配的
kernel/firmware 伴随包、`.config`、清单和校验文件。

这里的“全部”是“全部可编译 kmod”。例如当前 PonWrt 6.18 内核树缺少
`linux/netfilter/xt_RTPENGINE.h`，所以默认通过 `EXCLUDE_KMODS` 排除
`kmod-ipt-rtpengine`；这不会影响 XG-040G-TF 所需的 PON 驱动和用户态组件。

这个压缩包不是通用驱动仓库，只能用于同一 PonWrt 提交、同一内核配置和同一设备固件。
不要把其他 OpenWrt/ImmortalWrt 版本的 kmod 混进来；升级到新的固件后，应重新生成对应
版本的 kmod 压缩包。`CONFIG_ALL_KMODS` 会显著增加编译时间、磁盘占用和 Artifact 大小。
固件目录中的 `SHA256SUMS` 会覆盖固件、配置文件和这个 ZIP；解压 ZIP 后，可在压缩包根目录
执行 `sha256sum -c SHA256SUMS` 校验其中的包、清单和构建配置。

启用 `INCLUDE_ALL_PACKAGES="true"` 时，产物还会包含
`<artifact-prefix>-packages.tar.gz`，里面是本次构建生成的全部 `.apk/.ipk` 包、包索引元数据、
构建配置和公钥（若配置了签名密钥）。压缩包内含 `SHA256SUMS`，可在解压目录执行
`sha256sum -c SHA256SUMS`；它和 kmod 压缩包一样，只能用于匹配的 PonWrt 提交、feed
指纹、内核和设备配置。

工作流默认启用 ccache，并在下载失败时自动清理不完整文件后重试；编译失败时依次
尝试并行、单线程和单线程详细模式。成功构建后会删除 `main` 分支旧的同项目缓存，
保留最近两代；不同分支的缓存不会互相删除。

如需稳定的 APK 包签名，可在 GitHub Secrets 中设置 `PONWRT_APK_PRIVATE_KEY`。
未设置时使用构建系统默认密钥。

工作流会在成功后由 `github-actions[bot]` 向 `main` 写入按机型区分的
`ci/last-built-ponwrt-state-<model>.env`；如果仓库分支保护禁止该机器人推送，状态记录会失败，
后续每日检查可能重复编译，需要为 Actions 允许这一次状态提交。

## 上游插件兼容性

PonWrt 的 `feeds.conf.default` 仍然使用 ImmortalWrt 的 `packages`、`luci` 和 OpenWrt 的
`routing`、`telephony`、`video` feeds，同时额外加入 `pon_drivers`、`pon_userspace` 两个
PON feed。工作流每次构建都会执行 `feeds update -a`、把这些 feed 固定到检测阶段记录的
精确 commit，再执行 `feeds install -a`，所以普通 LuCI 插件和用户态程序可以通过
`EXTRA_PACKAGES` 加入固件。若上游修改 `feeds.conf.default`，同步更新
`configs/xg-040g-tf.build.env` 中的 `PONWRT_FEEDS`。

需要区分三类包：

- **纯用户态 / LuCI 包**：通常可以直接同步并编译，例如常规 DNS、代理、监控类插件；
- **`kmod-*` 内核模块**：必须在同一份 PonWrt 源码、同一个 6.18 内核和同一份配置中重编，
  不能拿别的 OpenWrt/ImmortalWrt 固件的模块直接安装；
- **依赖 `opkg` 的旧插件**：PonWrt 使用 `apk`，插件脚本如果硬编码 `opkg`、`.ipk` 下载或
  `/etc/opkg`，需要先适配后再用。

PonWrt 的 AN7581 默认配置还会关闭 `luci-app-iptv`，因此本仓库也不强制内置它；确实需要
时再把 `luci-app-iptv` 放进 `EXTRA_PACKAGES`，让 CI 验证它在当前 feed 组合下能否编译。

工作流会在 `make defconfig` 后验证附加包仍然存在，把所有 feed 的 commit、PonWrt
提交和配置指纹写入 `build-manifest.txt`，并在成功后更新
`ci/last-built-ponwrt-state.env`。
这能避免上游插件改名、依赖不满足或被 Kconfig 静默移除后仍产出错误固件。

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

产物在 `openwrt/bin/targets/<target-subdir>/`，当前 XG-040G-TF 为
`openwrt/bin/targets/airoha/an7581/`。

## 风险边界

刷机前保留可用的整片备份、USB-TTL 和原厂回退路径。请只在自己拥有或获授权的设备与线路
上使用原厂身份数据，不要复制他人的 SN、MAC 或 PON 认证信息。
