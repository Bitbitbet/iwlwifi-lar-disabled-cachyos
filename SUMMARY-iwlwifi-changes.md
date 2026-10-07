# iwlwifi 改动汇总（lar_disable + 5GHz 热点）

目标：关闭 Intel iwlwifi 的 LAR（Location Aware Regulatory），让 `iw reg set` 真正生效，
从而可以在 5GHz 上开热点。

平台：**linux-cachyos 7.2.8-2**（内核 release `7.2.8-2-cachyos`，Clang 23 + ThinLTO），
网卡 Intel 700 系 CNVi `8086:7a70`（AX211，走 **iwlmvm** / mvm 路径）。
源码基线：CachyOS 官方内核源码 `cachyos-7.2.8-1`（= PKGBUILD 里 `source=` 的那个 release asset，
已用 GitHub blob SHA 逐文件验证与 git tag 完全一致，294 文件 0 差异）。

补丁共 5 个，全部由 `PKGBUILD` 的 `prepare()` 按序应用。0001 是 AUR 原补丁未作任何修改，
0002–0005 是本次新增。

---

## 0001 — AUR 原补丁：加入 `lar_disable` 模块参数（未修改）

| 文件 | 位置 | 改动 |
|---|---|---|
| `iwl-drv.c` | 模块参数区 | 新增 `module_param_named(lar_disable, …, bool, 0444)` + `MODULE_PARM_DESC` |
| `iwl-modparams.h` | `struct iwl_mod_params` | 新增 `bool lar_disable;` |
| `iwl-nvm-parse.c` | `iwl_parse_nvm_data()` | 函数增加 `bool lar_fw_supported` 形参；`sbands_flags \|= IWL_NVM_SBANDS_FLAGS_LAR` 的条件由 `lar_enabled && fw_has_capa(LAR)` 改为 `lar_fw_supported && lar_enabled` |
| `iwl-nvm-parse.c` | `iwl_nvm_get_info()` | 新增 `lar_fw_supported = !lar_disable && fw_has_capa(LAR_SUPPORT)`；`nvm->lar_enabled` 只在该条件下置 true |
| `iwl-nvm-parse.h` | 原型/注释 | 同步形参 |
| `mvm/mvm.h` | `iwl_mvm_is_lar_supported()` | 函数开头 `if (iwlwifi_mod_params.lar_disable) return false;` |
| `mvm/nvm.c` | `iwl_nvm_init()` | 计算 `lar_enabled = !lar_disable && fw_has_capa(LAR_SUPPORT)` 传给 `iwl_parse_nvm_data()` |

作用：`lar_disable=1` 时驱动整体不启用 LAR（wiphy 不标 self-managed，运行期 LAR 同步全部跳过）。

---

## 0002 — 别再用 LAR 时代已废弃的 NVM 每信道标志

| 文件 | 位置 | 改动 |
|---|---|---|
| `iwl-nvm-parse.c` | `iwl_nvm_init_band()` | 算完 `channel->flags` 后，若 `lar_disable` 则清掉 `IEEE80211_CHAN_NO_IR \| IEEE80211_CHAN_IR_CONCURRENT` |

原因：`iwl_get_channel_flags()` 在 NVM 缺 `NVM_CHANNEL_ACTIVE`/`NVM_CHANNEL_IBSS` 时给信道加 `NO_IR`，
而这两个位按 `iwl-nvm-parse.h` 的注释在 LAR 模式下**已废弃**（Intel 新 NVM 不再为 5GHz 设置）。
LAR 开着时驱动走 `channel->flags = 0` 分支所以从不暴露；关掉 LAR 后这些过期位生效，
而 cfg80211 的 `handle_channel_single_rule()` 是 `chan->flags = chan->orig_flags | …`（**或**运算），
`NO_IR` 一旦进入 `orig_flags` 就再也清不掉 → 任何法规域都无法在 5GHz 起 AP。

---

## 0003 — 关 LAR 也要给固件编好它自己的 LAR 法规域

| 文件 | 位置 | 改动 |
|---|---|---|
| `mvm/ops.c` | NVM-from-FW 路径（`iwl_run_init_mvm_ucode()` 之后） | 原来 `if (!ret && iwl_mvm_is_lar_supported(mvm)) { …SELF_MANAGED…; iwl_mvm_init_mcc(); }`，改为 lar_disable 时**也调用 `iwl_mvm_init_mcc()`，但不设** `REGULATORY_WIPHY_SELF_MANAGED` |
| `mvm/nvm.c` | `iwl_mvm_init_mcc()` | 不再对 lar_disable 提前 `return 0`；lar_disable 时跳过 `iwl_mvm_init_fw_regd()`，并跳过最后的 `regulatory_set_wiphy_regd_sync()`（固件的域不回灌 cfg80211） |
| `mvm/nvm.c` | `iwl_mvm_update_mcc()` | `WARN_ON_ONCE(!iwl_mvm_is_lar_supported())` 放宽为：仅非 lar_disable 时才 WARN 并返回 `-EOPNOTSUPP` |
| `mvm/nvm.c` | `iwl_mvm_rx_chub_update_mcc()` | lar_disable 时不 WARN，改为打印固件上报的 MCC 后返回（忽略，不影响 cfg80211） |

原因：`iwl_mvm_init_mcc()` 是"把 MCC/法规域写进固件"的唯一途径。lar_disable 让它整条链路被跳过，
固件处于"法规状态未初始化"，一旦要求它在 5GHz 上起 AP/IBSS 就 **`NMI_INTERRUPT_UMAC_FATAL`
（`FW error in SYNC CMD LINK_CONFIG_CMD`）**并复位设备。驱动代码里自己也有注释：
"Disallow scans that might crash the FW while the LAR regdomain is not set."

---

## 0004 — 新增 `lar_country`，把国家码灌进固件

| 文件 | 位置 | 改动 |
|---|---|---|
| `iwl-modparams.h` | `struct iwl_mod_params` | 新增 `char *lar_country;` |
| `iwl-drv.c` | 模块参数区 | 新增 `module_param_named(lar_country, …, charp, 0444)` + `MODULE_PARM_DESC` |
| `mvm/nvm.c` | `iwl_mvm_init_mcc()` | lar_disable 且 `lar_country` 是合法两位大写字母时，用 `iwl_mvm_get_regdomain(wiphy, lar_country, MCC_SOURCE_BIOS, NULL)` 把它作为 MCC 送给固件（优先于 ACPI/BIOS MCC） |

原因：驱动**看不到** `iw reg set` 设的 alpha2，而固件必须有真实 MCC 才允许 5GHz AP；
实测这台机器的 ACPI 里没有 MCC，固件默认 MCC 是 `00`（世界域，全部信道 "No traffic"）。
配合 modprobe.d：`options iwlwifi lar_disable=1 lar_country=CN`
（`lar_country` 应与 `iw reg set` 的国家码保持一致）。

---

## 0005 — 让 cfg80211 的信道标志与固件的实际裁决一致

| 文件 | 位置 | 改动 |
|---|---|---|
| `mvm/nvm.c` | 新增静态函数 `iwl_mvm_lar_disable_sync_chanflags()`，在 `iwl_mvm_init_mcc()` 的 lar_disable 分支里调用 | 遍历 `wiphy->bands[*]->channels`，查固件返回的法规域 rule：该频率带 `NL80211_RRF_NO_IR_ALL` 就给信道**同时**打上 `IEEE80211_CHAN_NO_IR`（写 `flags` 立即生效 + 写 `orig_flags` 使其在后续每次法规更新后依然保持），否则清掉 `NO_IR` |

原因：固件在 CN 域下把 **36–48 标成 `INDOOR_ONLY` 且没有 `ACTIVE`**（不允许主动发射），
只认 **149/153/157/161/165**（`VALID IBSS ACTIVE`）。而 hostapd/wpa_supplicant 的
`hostapd_get_hw_features()` 会把 `NO_IR` 信道直接标成 `HOSTAPD_CHAN_DISABLED`。
把固件的裁决映射回 cfg80211 后：
- 自动选信道不会再挑到固件拒绝的信道（不会再把固件打崩）；
- 显式选 36–48 会在 cfg80211 层被干净拒绝，dmesg 不再出现固件错误；
- 5GHz 客户端连 36–48 的 AP **不受影响**（`NO_IR` 只禁止 beacon/主动扫描）。

---

## 打包侧（非 iwlwifi 代码）

- `PKGBUILD`：改用 CachyOS 官方源码 tarball（`cachyos-7.2.8-1`，保留 CachyOS 对 iwlwifi 的两处自有改动
  `mld/mac80211.c`、`mld/tx.c`）；依赖 `linux-cachyos` / `linux-cachyos-headers`；
  `_kernelrelease=7.2.8-2-cachyos`；编译必须带 **`LLVM=1 LLVM_IAS=1`**（内核是 Clang+ThinLTO 构建的）。
- 模块装到 `/usr/lib/modules/7.2.8-2-cachyos/updates/iwlwifi-lar-disabled-cachyos/`，
  `iwlwifi.ko` 与 `iwlmvm.ko` **必须成对替换**（0001 改了 `iwlwifi.ko` 导出给 `iwlmvm.ko` 的
  `iwl_parse_nvm_data()` 签名，而本内核未开 `CONFIG_MODVERSIONS`，没有 symbol CRC 兜底）。
- `.install`：安装/升级/卸载后自动 `depmod -a 7.2.8-2-cachyos`；`/etc/modprobe.d/iwlwifi-lar.conf`
  写入 `options iwlwifi lar_disable=1 lar_country=CN`。
- 前置依赖：`wireless-regdb`（否则内核无法把 `CN` 解析成法规域，`iw reg set` 静默不生效）。

---

## 验证记录（都在本次会话里实测过）

| 项目 | 结果 |
|---|---|
| 5 个补丁在 `cachyos-7.2.8-1` 源码上 `patch -Np1` | 全部干净应用 |
| `LLVM=1` 编译 | 0 error，生成 iwlwifi/iwlmvm/iwlmld/iwldvm 四个 .ko |
| vermagic | `7.2.8-2-cachyos SMP preempt mod_unload`，与运行内核一致 |
| 模块参数 | `lar_disable`、`lar_country` 均存在 |
| 导出符号集合 | stock 149 / 补丁版 149，完全一致 |
| 未定义符号集合（决定能否加载） | iwlwifi 250/250、iwlmvm 342/342、iwlmld 285/285、iwldvm 176/176，**0 差异** |
| srcversion（源码指纹） | iwlmvm / iwlmld / iwldvm 与内核自带模块**完全一致** |
| CachyOS vs vanilla 的 iwlwifi | 307 个文件里只有 `mld/mac80211.c`、`mld/tx.c` 不同（固件错误保护，与 LAR 无关） |

实际效果（用户现场日志验证）：`iw phy` 的 5GHz `no IR` 消失 → 固件 MCC 变成 `CN`
（`iwl_mvm_update_mcc MCC response status: 0x1. new MCC: 0x434e ('CN') n_chans: 110`）→
`channel=149` 热点正常；`channel=0` 因 NetworkManager 只考虑 U-NII-1 而失败，但已变成
**cfg80211 干净拒绝**，不再产生固件错误（0005 的目的达成）。