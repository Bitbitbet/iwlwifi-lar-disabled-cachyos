# Maintainer: 适配 CachyOS 内核（原包: TenkyuChimata/iwlwifi-lar-patched）
# 本 PKGBUILD 与 AUR 原版的区别（原版只能用于 Arch 官方 linux 包）：
#   1. 基于 CachyOS 官方内核源码 tarball 编译，而不是 kernel.org vanilla
#      —— 保留 CachyOS 对 iwlwifi 的自有改动（mld/ 下的两处固件错误保护）
#   2. 依赖改为 linux-cachyos / linux-cachyos-headers
#   3. 模块目录改为 7.2.8-2-cachyos（CachyOS 的 localversion 带 -cachyos 后缀）
#   4. 编译必须带 LLVM=1：CachyOS 内核是 Clang+ThinLTO 构建的，用 gcc 会因为
#      clang 专有标志（-fsplit-lto-unit / -mllvm 等）报错

pkgname=iwlwifi-lar-disabled-cachyos

_pkgbase=linux-cachyos
_kver=7.2.8
_kernel_pkgrel=2
_cachyos_src_pkgrel=1

_kernelpkgver="${_kver}-${_kernel_pkgrel}"          # 7.2.8-2
_kernelrelease="${_kernelpkgver}-cachyos"            # 7.2.8-2-cachyos
_srctag="cachyos-${_kver}-${_cachyos_src_pkgrel}"    # cachyos-7.2.8-1

pkgver=$_kver
pkgrel=2
pkgdesc="Intel iwlwifi family modules with lar_disable support, built for ${_pkgbase} ${_kernelpkgver}"
arch=('x86_64')
url="https://github.com/Bitbitbet/iwlwifi-lar-disabled-cachyos"
license=('GPL-2.0-only')

depends=(
  "${_pkgbase}=${_kernelpkgver}"
  "${_pkgbase}-headers=${_kernelpkgver}"
  'zstd'
)
optdepends=(
  'mkinitcpio: rebuild initramfs if your initramfs contains iwlwifi'
)
makedepends=(
  'bc'
  'kmod'
  'patch'
)

source=(
  "https://github.com/CachyOS/linux/releases/download/${_srctag}/${_srctag}.tar.gz"
  '0001-iwlwifi-add-lar_disable.patch'
  '0002-iwlwifi-lar_disable-ignore-stale-nvm-no-ir.patch'
  '0003-iwlwifi-lar_disable-program-fw-lar-regdomain.patch'
  '0004-iwlwifi-lar_disable-push-country-to-fw.patch'
  '0005-iwlwifi-lar_disable-sync-chanflags-with-fw.patch'
  '0006-iwlwifi-lar_disable-activate-unii1-in-fw.patch'
  'iwlwifi-lar.conf'
)
sha256sums=('f848df42bd4ac31a4918188c6723660d3c6dd7591b63a6419123dcbc23ecf328'
            'da2ab52ccdef2b93088c9e0c56bc1c166bf748d021b529cb2af2ff6c5d9e85cc'
            '1ddec120797ae6707e2c61928412e6a14f012a3df1811299ee2ff07e88143ccb'
            '4ffe1142ba2cb9de04be2af54561c8789dafc91d2a4264dda8ded8bf40d3e347'
            '28eee80046bc32777f94ac0e2a5bb7422d373d6e8b084df7ea85ec3f611bbe25'
            'fca8c7cff8e677098b8db32a23f09b522bc2c25d64a168e4db44cf2ab12117b1'
            'cb6f79e6c4b2364151f97a0db57848cc3b6fa1e36f16184b4a1185d14fe72105'
            'c2958abe2634f4dfc75c2ca97ed842f66db43b89102250d639d71dabc7662a8c')

install="${pkgname}.install"
options=(!debug)

prepare() {
  cd "${srcdir}/${_srctag}"

  # 0001: AUR 原补丁（关闭 LAR），在 CachyOS 7.2.8 源码上干净应用
  patch -Np1 -i "${srcdir}/0001-iwlwifi-add-lar_disable.patch"

  # 0002: 修 0001 的副作用 —— lar_disable 后不能再信任 LAR 时代已废弃的
  #       NVM 每信道标志，否则 5GHz 全被标成 NO_IR，AP 永远起不来
  patch -Np1 -i "${srcdir}/0002-iwlwifi-lar_disable-ignore-stale-nvm-no-ir.patch"

  # 0003: 修 FW 侧 —— lar_disable 时仍然给固件编好 LAR 法规域
  #       （否则固件在 5GHz 起 AP/IBSS 时会 NMI_INTERRUPT_UMAC_FATAL）
  patch -Np1 -i "${srcdir}/0003-iwlwifi-lar_disable-program-fw-lar-regdomain.patch"

  # 0004: 把国家码灌进固件自己的 MCC（固件没有 MCC 就在 5GHz 起 AP 时崩）
  patch -Np1 -i "${srcdir}/0004-iwlwifi-lar_disable-push-country-to-fw.patch"

  # 0005: 按固件的裁决把 no IR 写回信道标志（flags + orig_flags），
  #       这样 hostapd 自动选信道时会跳过固件不认的 36-48，直接落到 149
  patch -Np1 -i "${srcdir}/0005-iwlwifi-lar_disable-sync-chanflags-with-fw.patch"

  # 0006: 让固件自己在 CN 等法规域下把 UNII-1（36-48）当成可用信道。
  #       这些信道固件标的是 INDOOR_ONLY 且没有 ACTIVE（= 不允许主动发射），
  #       OEM 本来要通过 ACPI _DSM(8) 的 ACTIVATE_5G2_IN_WW 位来激活；
  #       BIOS 没给这一位时由驱动补上，这样固件与 cfg80211 的判定一致，
  #       NetworkManager 自动选信道（它只会在 36-48 里挑）才有得选。
  #       若固件对 CN 域不认这一位，0005 仍会照旧把 36-48 标成 no IR，
  #       行为退回 0005 的状态（只是多一条 dmesg 信息）。
  patch -Np1 -i "${srcdir}/0006-iwlwifi-lar_disable-activate-unii1-in-fw.patch"
}

build() {
  local builddir="/usr/lib/modules/${_kernelrelease}/build"
  local srcsubdir="${srcdir}/${_srctag}/drivers/net/wireless/intel/iwlwifi"

  [[ -d "${builddir}" ]] || {
    echo "ERROR: 缺少内核构建目录 ${builddir}"
    echo "请先安装 ${_pkgbase}-headers=${_kernelpkgver}"
    return 1
  }

  cd "${srcsubdir}"

  # makepkg 会导出 CFLAGS/LDFLAGS，内核构建不需要它们
  unset CFLAGS CXXFLAGS LDFLAGS

  export KBUILD_BUILD_USER='builder'
  export KBUILD_BUILD_HOST='arch'
  export KBUILD_BUILD_VERSION='1'
  export KBUILD_BUILD_TIMESTAMP='1970-01-01'

  # LLVM=1 必须：内核用 Clang+ThinLTO 构建
  make -C "${builddir}" \
    M="$PWD" \
    LLVM=1 LLVM_IAS=1 \
    KBUILD_BUILD_USER="${KBUILD_BUILD_USER}" \
    KBUILD_BUILD_HOST="${KBUILD_BUILD_HOST}" \
    KBUILD_BUILD_VERSION="${KBUILD_BUILD_VERSION}" \
    KBUILD_BUILD_TIMESTAMP="${KBUILD_BUILD_TIMESTAMP}" \
    modules -j"$(nproc)"
}

package() {
  local moddir="${pkgdir}/usr/lib/modules/${_kernelrelease}/updates/${pkgname}"
  local srcsubdir="${srcdir}/${_srctag}/drivers/net/wireless/intel/iwlwifi"

  install -dm755 "${moddir}"

  # 注意：iwlwifi.ko 与 iwlmvm.ko 必须成对替换。
  # 本补丁修改了 iwlwifi.ko 导出给 iwlmvm.ko 的 iwl_parse_nvm_data() 签名
  # （多了一个 bool 参数），且内核未开 CONFIG_MODVERSIONS，符号 CRC 不会兜底。
  install -m644 "${srcsubdir}/iwlwifi.ko" "${moddir}/iwlwifi.ko"
  install -m644 "${srcsubdir}/mvm/iwlmvm.ko" "${moddir}/iwlmvm.ko"

  [[ -f "${srcsubdir}/dvm/iwldvm.ko" ]] && \
    install -m644 "${srcsubdir}/dvm/iwldvm.ko" "${moddir}/iwldvm.ko"
  [[ -f "${srcsubdir}/mld/iwlmld.ko" ]] && \
    install -m644 "${srcsubdir}/mld/iwlmld.ko" "${moddir}/iwlmld.ko"

  strip --strip-debug "${moddir}"/*.ko || true

  install -Dm644 "${srcdir}/iwlwifi-lar.conf" \
    "${pkgdir}/etc/modprobe.d/iwlwifi-lar.conf"

  install -Dm644 /dev/stdin \
    "${pkgdir}/usr/share/${pkgname}/kernel-release" <<EOF
${_kernelrelease}
EOF
}
