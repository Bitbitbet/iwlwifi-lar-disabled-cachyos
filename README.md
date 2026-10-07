# 注：这个仓库完全由deepseek v4.1 flash生成，我看不懂也不想看。Use at your own risk

# 关闭 iwlwifi LAR（让 `iw reg set` 生效）— CachyOS 内核适配版

针对 **linux-cachyos**（内核 release `7.2.8-2-cachyos`，网卡 Intel 700 系 CNVi 走 `iwlmvm` 路径）。

## 这个目录里有什么

| 文件 | 说明 |
|---|---|
| `PKGBUILD` | 已适配 CachyOS 的打包脚本（AUR 原版只能用于 Arch 官方 `linux` 包） |
| `iwlwifi-lar-disabled-cachyos.install` | 安装后自动 `depmod` |
| `0001-iwlwifi-add-lar_disable.patch`  |
| `0002-iwlwifi-lar_disable-ignore-stale-nvm-no-ir.patch` |
| `iwlwifi-lar.conf` | `options iwlwifi lar_disable=1` |

## 为什么 5GHz 热点一开始起不来（0002 修复的原因）

只打 AUR 的 0001（`lar_disable=1`）之后，`iw reg set CN` 能生效（法规域确实变成了 CN，5GHz 最大功率由世界域的 20 dBm 变成 22 dBm），
但 `iw phy` 里 5GHz 所有信道仍然是 `no IR`，AP 起不来。链路是这样的：

1. `iwl_get_channel_flags()`（`iwl-nvm-parse.c`）在 NVM 没设 `NVM_CHANNEL_ACTIVE` / `NVM_CHANNEL_IBSS` 时给信道加 `IEEE80211_CHAN_NO_IR`；
   而 `iwl-nvm-parse.h` 注释写明这两位在 LAR 模式下**已废弃**——Intel 新 NVM 根本不再为 5GHz 设置它们。
2. LAR 启用时驱动走 `channel->flags = 0`（不做任何限制，交给固件），所以从来没人踩到这个坑；
   0001 把 LAR 关掉后，驱动改走 `iwl_get_channel_flags()`，这些**过期位就生效了** → 5GHz 全变 `orig_flags |= NO_IR`。
3. cfg80211 的 `handle_channel_single_rule()`（`net/wireless/reg.c`）算的是
   `chan->flags = chan->orig_flags | ... | map_regdom_flags(reg_rule->flags)` —— **NO_IR 是"或"进去的，任何法规域都清不掉**，
   所以 `iw reg set CN` 对 NO_IR 完全无效。
4. 本内核 `CONFIG_CFG80211_REG_RELAX_NO_IR` 未开启（即使开了，`cfg80211_ir_permissive_chan()` 也只豁免
   `P2P-GO/STATION/P2P-CLIENT`，**不含 `NL80211_IFTYPE_AP`**），而 NM 热点用的是 `mode=ap` → 5GHz AP 被 cfg80211 直接拒绝。
5. 2.4GHz 没事，因为 NVM 对 2.4GHz 仍然设置了 `ACTIVE`/`IBSS`。

0002 的做法：`lar_disable=1` 时只清掉那个过期的 `NO_IR`（顺带 `IR_CONCURRENT`），其余 NVM 限制（`RADAR`/DFS、`INDOOR_ONLY`、带宽限制）全部保留，
让 cfg80211 按你设置的法规域决定。

## 前置条件（很重要）

```bash
sudo pacman -S wireless-regdb
```

没有 `regulatory.db` 的话，内核无法把 `CN`/`US` 这种国家码解析成法规域，`iw reg set` 会静默不生效（本机当前 `/lib/firmware/regulatory.db` 不存在，`iw reg get` 显示 `country 00` 世界域）。
前提是 `CONFIG_CFG80211_REQUIRE_SIGNED_REGDB=y` + `CONFIG_CFG80211_USE_KERNEL_REGDB_KEYS=y`，而 `wireless-regdb`（core 仓库，2026.09.03-1）提供的正是签名过的 regdb。

## 安装

```bash
makepkg -si          # 在本目录执行
```

## 生效与验证

重启是推荐做法（也可以先 `systemctl stop NetworkManager` 再 `modprobe -r iwlmvm iwlwifi && modprobe iwlwifi`）。

```bash
# 1) 确认加载的是我们打的模块（应指向 updates/iwlwifi-lar-disabled-cachyos/）
modinfo -F filename iwlwifi

# 2) 确认参数生效（应为 Y）
cat /sys/module/iwlwifi/parameters/lar_disable

# 3) 设置法规域
iw reg get                  # 看当前
sudo iw reg set CN          # 换成你自己所在的国家码
iw reg get                  # 应显示 country CN
```

## 为什么 5GHz 热点的 `channel=0`（自动）在本机无法工作

这是 **NetworkManager 自身的限制**，不是驱动能修的：

1. `nm-device-wifi.c` 的 `ensure_hotspot_frequency()` 里，5GHz 的候选频率是**写死的 U-NII-1**：
   `freqs_a[] = {5180, 5200, 5220, 5240, 0}` —— 也就是只考虑 **36/40/44/48**，从不考虑 149–165；
   而且 `find_frequency()` 返回 0 时还有 `freq = rnd_freqs[0];` 的盲目回退。
2. `nm-wifi-utils-nl80211.c` 的 `wifi_nl80211_find_freq()` 在 AP 模式下会跳过 `no_ir` 信道；
   0005 之后 36–48 都是 `no_ir` → 全部被跳过 → 回退到 36 → cfg80211 以 "no IR" 拒绝 START_AP
   → wpa_supplicant 报 `Failed to start AP functionality`（此时还没到驱动，所以 dmesg 是干净的）。
3. 根子上：这颗 AX211 的固件在 **CN** 法规域下把 36–48 标成 `INDOOR_ONLY` 且没有 `ACTIVE`
   （不允许 active 发射），所以固件根本不会在 36–48 上起 AP；而 149–165 才是 `VALID IBSS ACTIVE`。

结论与建议：

- **`band=a` 时请显式指定信道**：`149 / 153 / 157 / 161 / 165`（非 DFS 且固件允许）。
  ```bash
  sudo nmcli connection modify <连接名> 802-11-wireless.channel 149
  ```
- 想要"自动选信道"，就用 `band=bg`（2.4GHz）：2.4GHz 在固件 CN 域里是 `ACTIVE`，NM 的自动选择也只在 2.4GHz 里挑，正常工作。
- 如果一定要 `band=a` + `channel=0`，唯一的办法是让**固件**认为 U-NII-1 可主动发射，即把
  `/etc/modprobe.d/iwlwifi-lar.conf` 里的 `lar_country=CN` 改成 `lar_country=US`（固件的表里 36–48 是 ACTIVE）。
  代价：**固件会按 US 的功率/DFS 规则工作**（cfg80211 侧仍是 `iw reg set CN`），属于法规上的不一致用法。
- 或者给 NetworkManager 打补丁，把 `freqs_a[]` 加上 `5745/5765/5785/5805/5825`。

0005 仍然有价值：它把"固件断言崩设备"变成了"cfg80211 干净拒绝"，因此 dmesg 不再刷固件错误，
并且任何显式选 36–48 的尝试也不会再把固件打崩。

## 注意事项

- 只有 `iwlwifi.ko` + `iwlmvm.ko` **必须成对替换**（补丁改了 `iwlwifi.ko` 导出给 `iwlmvm.ko` 的 `iwl_parse_nvm_data()` 签名，且本内核未开 `CONFIG_MODVERSIONS`，符号 CRC 不会兜底）。包内已同时包含两者，并附带 `iwlmld.ko`/`iwldvm.ko`。
- 本补丁只改 **mvm** 路径。如果你的机器用的是 MLD 设备（BE200/BE202 等 `iwlmld` 驱动），`lar_disable` 不会起作用。你这台是 AX211 走 mvm，没问题。
- 模块未签名；Secure Boot 已关闭且 `MODULE_SIG_FORCE` 未开，所以能加载，只会让内核带上 out-of-tree taint（你本来就有 nvidia 的 taint）。
- 关闭 LAR 后收发功率/信道由你手动设置的法规域决定，**请遵守当地无线电法规**。
- 卸载包会自动 `depmod`，重启后恢复内核自带的 iwlwifi。