# Billie NetHunter kernel continuation

Target backup: OnePlus Nord N10 5G (BE2028 / OnePlusN10TMO), stock OxygenOS
`10.5.8.BE88CB`. The only official OnePlus Android 10 source branch ends at a
commit labeled `10.5.7.BE89BA`; that is not an exact release label match.

The full config fragment is layered over `vendor/lito-perf_defconfig`. It keeps
the stock `4.19.81-perf+` release identity, enables the Kali general/network/USB
baseline, and adds only the user's observed external Wi-Fi adapter: MediaTek
USB ID `0e8d:7610` (`mt76x0u`).

`billie-nethunter-wifi-first.config` is the conservative first-boot profile.
It deliberately keeps `CONFIG_SYSVIPC=n` because enabling it changes
`task_struct` and 7,659 exported symbol CRCs in this tree. It also keeps the
legacy `CONFIG_CFG80211_WEXT` interface and `CONFIG_MAC80211_MESH` disabled.
Mesh increases a public network header cache and changes 2,198 more exported
CRCs. MT7610U monitor and injection use nl80211/mac80211 and need none of these
three options. These omissions preserve compatibility with the stock internal
WLAN module as far as the reference CRC comparison can prove. Actual stock
`wlan.ko` still must pass the included audit before flashing.

The MT7610U/mac80211 driver stack is built as signed modules because Qualcomm's
linker layout cannot grow the built-in text section past its next fixed 2 MiB
boundary. The firmware itself remains embedded in `Image`, so no
vendor-partition firmware install is required. The firmware input expected by
the build is `/lib/firmware/mediatek/mt7610u.bin`, SHA-256:

`5a4268e9021bb587426ba624b425f1e660bfc82cd63b36ad3ce6fb9ce6751760`

The first bring-up build disables `CONFIG_MODULE_SIG_FORCE`. This is deliberate:
OnePlus's private module-signing key is unavailable, and changing the kernel's
generated certificate while keeping forced enforcement could reject stock
OxygenOS vendor modules such as the internal WLAN module. Module versioning and
the stock release string remain enabled; runtime testing is still required to
confirm vendor-module CRC compatibility.

The kernel source also needs Kali's official
`patches/4.19/add-wifi-injection-4.19.patch`. The exact reviewed patch used here
has SHA-256:

`f794aa68f1e4ee4540ac7062e220033442a325994c68e8e56618b4a790d42c5c`

Modern host binutils also require the standard one-line DTC `yylloc` fix. This
tree generates the lexer from `scripts/dtc/dtc-lexer.l` instead of keeping the
`dtc-lexer.lex.c_shipped` file targeted by Kali's generic patch, so the lab's
`fix-yylloc-billie.patch` applies the equivalent declaration change at the
generator source.

OnePlus's final HE-operation header update omitted the matching mac80211 field
extraction update. Enabling mac80211 therefore exposed an undefined offset and
incorrect left shifts in `mlme.c`. `fix-mac80211-he-operation-fields.patch`
ports the equivalent masked right-shift semantics from upstream Linux 5.4.

The generated AnyKernel package is currently an abort-gated engineering draft.
It replaces only `Image`, includes only newly enabled modules in a Magisk
systemless overlay, and never packages loose development DTB/DTBO files. Stock
boot DTB, DTBO, ramdisk, command line, Magisk state, and active-slot metadata
are preserved at flash time. The release gate must not be removed until the
source mismatch and stock-module ABI audit are resolved.

