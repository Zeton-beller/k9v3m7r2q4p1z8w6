# Switch CN NRO 构建完整修改日志 (BUILD_CHANGELOG.md)

本文档记录 Switch CN NRO 构建优化与问题修复的所有历史修改与方案，防止后续上下文丢失或重复尝试。

---

## 📅 历史核心修改记录

### 1. StormLib 64位 API 屏蔽 (`FileStream.cpp` & `StormPort.h`)
* **问题**：Switch devkitA64 / Newlib 缺失 `ftruncate64` 及相关 64位 POSIX 文件接口。
* **解决**：在 CI 步骤中拦截 `FileStream.cpp`，将 `ftruncate64` 调用桩化返回 `0`（因 NRO 资源包加载为 Read-only 模式，截断补丁安全）。

### 2. ImGui 桌面后端与 Fast/OpenGL 后端屏蔽 (`Gui.cpp` & backend masking)
* **问题**：Switch 平台缺少桌面 OpenGL3 / SDL2 链接符号 (`ImGui_ImplOpenGL3_Init` 等)。
* **解决**：
  * 使用 `#ifndef __SWITCH__` 屏蔽所有桌面 `imgui_impl_opengl3*` 和 `imgui_impl_sdl2*` 源码；
  * 在 `Gui.cpp` 顶部注入 Stub 宏，将其重定义为 NOP (`(true)` / `((void)0)`).

### 3. devkitPro Newlib `__assert` 声明冲突 (`functions.h`)
* **问题**：N64 遗留头文件中的 `void __assert` 声明与 devkitA64 `<assert.h>` 中的原型冲突。
* **解决**：直接在 `soh/include/functions.h` 源码中追加 `&& !defined(__aarch64__)` 保护条件，从源头禁止该旧声明在 Switch (ARM64) 平台上编译。

### 4. 音频解码第三方库依赖重构 (`switch.yml` & `soh/CMakeLists.txt`)
* **问题**：`soh/mixer.c` 依赖 `opusfile.h`，而 `dkp-pacman -Sy` 会触发 devkitPro 官方服务器 HTTP 403 阻断，且默认没有导入头文件包含路径。
* **解决**：
  * 移除 `dkp-pacman` 联网下载，改为在 `switch.yml` 中直接使用 `Switch.cmake` 交叉编译官方 GitHub 的 `ogg`、`opus`、`opusfile`（启用 `-DOP_DISABLE_HTTP=ON` 屏蔽 OpenSSL 依赖）；
  * 在 `soh/CMakeLists.txt` 的 `NintendoSwitch` 分支追加 `${DEVKITPRO}/portlibs/switch/include/opus` 搜索目录。

### 5. 碰撞盒 `quad` 标识符重命名闭环 (`z64collision_check.h` & CI patch)
* **问题**：`quad` 标识符与系统底层定义冲突；此前只改了业务代码，未改头文件导致 59% 处报无成员错误。
* **解决**：
  * 在 `soh/include/z64collision_check.h` 源码中将 `ColliderQuadDim` / `ColliderQuadDimInit` 的结构体成员重命名为 `quad_t[4]`；
  * CI 脚本使用正则 `(\bdim|\bsrc|\bdest)([\.\->]+)quad\[` 精确替换所有 4 种访问结构，实现 0 副作用 100% 匹配。

### 6. 100% 链接期 Vorbis 音频库与桌面解包存根防护 (`CMakeLists.txt` & `switch.yml`)
* **问题**：编译 100% 通过后，链接 ELF 阶段抛出 `ov_open_callbacks` 未定义与 `waitpid`/`execvp`/`pipe`/`zapd_report` 符号缺失。
* **解决**：
  * 在 `soh/CMakeLists.txt` 中增加 `-lvorbisfile -lvorbis` 依赖链接；
  * 在 `switch.yml` 中为 `Extract.cpp` 注入 `zapd_report` Weak Stub 存根，并给 `portable-file-dialogs.h` 注入 `defined(__SWITCH__)` 屏蔽规则。

### 8. 101 条链接符号全量归类平定 (`CMakeLists.txt` & `switch.yml`)
* **问题**：静态库链接依赖顺序颠倒引发 88 条 `oggpack_*` 报错；`imgui.cpp` 与 `Fast3dWindow.cpp` 残留 13 条 POSIX 与桌面虚函数表报错。
* **解决**：
  * 将 `soh/CMakeLists.txt` 中的静态库顺序更正为被依赖倒序：`-lvorbisfile -lvorbis -lopusfile -lopus -logg`；
  * 使用 Python 正则在 `switch.yml` 中无缝平定 `imgui.cpp` 的 `Platform_OpenInShell` 和 `Fast3dWindow.cpp` 的 `GfxRenderingAPIOGL`。

---

*最新更新时间：2026-07-24 (Checkpoint-2 全量 100% 平定)*



