patches/deliver/ — 17.15.0 交付补丁（相对官方 Frida tag 17.15.0）

与 17.14.1 同源，补丁 0 冲突套用。含新增 lib/base/obfuscate.vala、src/topatch.py。
需 NDK r29。

应用：
  1) git clone --recurse-submodules -b 17.15.0 https://github.com/frida/frida frida17.15.0
  2) cd frida17.15.0 && git apply --exclude=releng patches/deliver/superrepo.patch
  3) (cd subprojects/frida-core && git apply ../../patches/deliver/frida-core.patch)
  4) (cd subprojects/frida-core && git apply ../../patches/deliver/frida-core-gadget-custom-linker.patch)
  5) (cd subprojects/frida-gum  && git apply ../../patches/deliver/frida-gum.patch)
  6) export ANDROID_NDK_ROOT=<NDK r29>; ./tools/build-android-all.sh
  7) python tools/verify-patch.py dist-android

注意：第 4 步（frida-core-gadget-custom-linker.patch）必须在第 3 步之后应用，
它改的是 gadget.vala 的 detect_location，与 frida-core.patch 无重叠 hunk。

--- frida-core-gadget-custom-linker.patch 说明 ---

目的：让 Gadget 能被「自定义链接器」（把 .so 直接 mmap 进内存、不把模块注册进
系统 soinfo 链表的加载器，例如 KernelSU/APatch 生态里常见的 CSOLoader）加载起来。

未打该补丁时的现象：Gadget 构造函数执行到一半 abort，tombstone 里是
  Frida:ERROR:../subprojects/frida-core/lib/gadget/gadget.vala:744:
    frida_gadget_detect_location: assertion failed: (our_range != null)

根因：gum 在 Android 上枚举模块走 gum_enumerate_soinfo()，即 bionic 的 soinfo
链表（solist_get_head()）；而自定义链接器的设计前提恰恰是「不进 soinfo」，
两边直接冲突 —— gum 永远看不到 Gadget，our_range 恒为 null。
又因为该断言在 detect_location() 里、早于 interaction 解析，改 interaction 模式
（listen/script/connect）都绕不过去。

修法：enumerate_modules 找不到自己时，回退到 Gum.Process.enumerate_ranges()
（它读 /proc/self/maps，不依赖 soinfo），并从 RangeDetails.file.path 取回自身路径；
两条都没命中才退化为 1 字节占位 range（只求不再 abort）。

顺带修好一件事：拿到 our_path 后，Gadget 的 .config 定位重新可用 ——
load_config() 会由路径推导 <stem>.config.so，而模块预置的暂存配置路径正好是
<app files>/.org.chromium.<hex>.config.so，与 maps 里那条容器路径推导结果吻合，
所以配好 Gadget 后无需额外改动。

对系统链接器加载的常规用法零影响：那条路径下 enumerate_modules 能命中，
回退分支不会执行。
