# 指定网关 · luci-app-device-gateway

在 OpenWrt / ImmortalWrt 的 LuCI 中，按设备分配固定 IPv4、默认网关和 DNS。设备规则直接在「网络 → 指定网关」里编辑，插件负责维护 dnsmasq 的 DHCP 标签。

## 适合什么场景

| 场景 | 设备规则 |
| --- | --- |
| 只有电视或手机需要走旁路由 | 选择该设备的 MAC 和固定 IP，再选择旁路由网关；其他设备继续使用原来的 DHCP 设置 |
| 家里有两台旁路由，不同设备分别使用 | 把两台旁路由存成网关预设，在设备行里分别选择 |
| NAS、打印机只需要固定地址 | 网关和 DNS 都选择默认或继承 |
| 单台设备需要不同的 DNS | 在「指定 DNS」单独选择常用 DNS，或输入自定义 IPv4 DNS，多个地址用逗号分隔 |

适用于由主路由上的 dnsmasq 提供 DHCPv4 的网络。旁路由必须在设备所在的网段内可达，并已配置转发。插件只下发网络参数，旁路由的转发、防火墙和代理由旁路由自身负责。IPv6 默认路由由 RA 等机制管理，独立于本插件的 IPv4 规则。

## 安装与使用

1. 从 [GitHub Releases](https://github.com/huladabang/luci-app-device-gateway/releases/latest) 下载 `.ipk`。支持 `opkg` 的固件可在软件包页面上传安装，或将安装包放进路由器的 `/tmp` 后执行：

   ```sh
   opkg install /tmp/luci-app-device-gateway_1.0.0-r1_all.ipk
   ```

2. 打开「网络 → 指定网关」。需要旁路由时，点击网关预设表格里的「添加」，填好名称和 IPv4 地址，再点「添加网关」。设备规则也可直接选择默认网关。
3. 添加设备，核对当前 Wi-Fi 使用的 MAC 和固定 IP，按需选择网关与 DNS，然后点「保存并应用」。如果同一 MAC 已有系统静态租约，请沿用它的 IP。
4. 让设备重新获取 DHCP 租约，然后在设备上核对实际收到的 IPv4 网关和 DNS。固定私有 MAC 可以使用，轮换 MAC 的设备需先设为固定模式。

正式版沿用在 TP-Link TL-7DR7299、ImmortalWrt 24.10-SNAPSHOT、LuCI 24.10 和 dnsmasq 上验证的 r20 功能。IPK 架构为 `all`，不含按 CPU 架构编译的程序。其他固件需具备 LuCI、dnsmasq DHCPv4、`flock` 和 `opkg`。使用 `apk` 的固件不适用此 Release 的 IPK。

## 实现原理

LuCI 把预设和设备规则保存在 `/etc/config/device-gateway`。提交后，OpenWrt 的 procd 配置变更事件触发插件同步脚本，脚本按 MAC 为 dnsmasq 生成静态租约和设备标签：

- **固定 IP** 由 DHCP 静态租约提供。已有系统租约的 MAC 在地址相同时会复用原租约；冲突会报错。
- **指定网关** 通过该设备标签下发 DHCPv4 Option 3。选择继承时没有专属 Option 3。
- **指定 DNS** 通过该设备标签下发 DHCPv4 Option 6。选择继承时没有专属 Option 6。

脚本先在临时目录生成并校验规则，再更新 `/etc/config/dhcp`。规则有变化时重启 dnsmasq 并检查 DHCP 监听，失败时尝试恢复上一份配置。插件用 `dg_owner` 标记自己生成的条目，关闭或卸载时清理受管规则并保留原有静态租约。页面底部的「DHCP 同步记录」用于查看路由器侧结果；客户端何时采用新参数取决于 DHCP 续租。

## 从源码构建 IPK

在 macOS 或提供 `bsdtar` 兼容 `tar` 的环境执行：

```sh
./build-ipk.sh
```

产物位于 `dist/`。如需在 OpenWrt 源码树内构建，将本目录放在 `package/luci-app-device-gateway`，在 `make menuconfig` 中选择该插件，再运行 `make package/luci-app-device-gateway/compile V=s`。
