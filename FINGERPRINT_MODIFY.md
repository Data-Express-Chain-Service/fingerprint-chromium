# Fingerprint Chromium 指纹修改总览

> 本文档汇总了在 `patches/extra/fingerprint/` 以及 `patches/extra/ungoogled-chromium/` 中与浏览器指纹相关的全部 Chromium 源码修改。每一项都给出了对应的 Patch 文件、被修改的 Chromium 源文件以及修改的具体行为，便于与代码对齐核对。注解采用 `patch → file::function()` 的形式精准标注落点。

## 目录

- [核心机制](#核心机制)
- [命令行开关（switches）](#命令行开关switches)
- [身份标识指纹](#身份标识指纹)
- [硬件信息指纹](#硬件信息指纹)
- [Canvas / WebGL 指纹](#canvas--webgl-指纹)
- [DOM 几何指纹](#dom-几何指纹)
- [音频指纹](#音频指纹)
- [字体指纹](#字体指纹)
- [时区指纹](#时区指纹)
- [自动化 / Headless 检测规避](#自动化--headless-检测规避)
- [Shadow DOM 扩展](#shadow-dom-扩展)
- [TLS / 网络层指纹](#tls--网络层指纹)
- [总结](#总结)

---

## 核心机制

项目以一个单一的 **fingerprint seed**（由命令行 `--fingerprint=<seed>` 传入）驱动所有维度的随机化：

- 维度噪声计算方式：`std::hash<std::string>(seed + "维度标识")` → 得到确定性的 `uint32_t` 哈希值。
- 将哈希值再归一化为 `[-0.5, 0.5]`、`[0, 1)` 或 `(0, limit]` 等区间，施加到相应维度上。
- 所有 renderer 子进程通过 `RenderProcessHostImpl::PropagateBrowserCommandLineToRenderer` 继承这些开关。
- 注意：`--fingerprint-hardware-concurrency` 与 `--fingerprint` 在 `navigator_concurrent_hardware.cc` 中是直接调用 `std::stoul` / `std::stoull` 把 seed 当成整数，并非 hash；`((seed % 13) + 4) * 2`。

核心文件：
- `components/ungoogled/ungoogled_switches.cc` / `.h`（开关定义）
- `content/browser/renderer_host/render_process_host_impl.cc`（开关传播，`PropagateBrowserCommandLineToRenderer` 列表，约第 3416 行）

---

## 命令行开关（switches）

**Patch:** `patches/extra/fingerprint/000-add-fingerprint-switches.patch`（基础 8 个）；`patches/extra/fingerprint/018-timezone.patch` 追加 `kFingerprintTimezone`

新增在 `components/ungoogled/ungoogled_switches.cc` / `.h`：

| 开关常量                        | 命令行参数                        | 用途                                       |
|---------------------------------|-----------------------------------|--------------------------------------------|
| `kFingerprint`                  | `--fingerprint`                   | 通用 seed，驱动绝大部分维度                |
| `kFingerprintBrand`             | `--fingerprint-brand`             | 伪装浏览器品牌（Chrome/Edge/Opera/Vivaldi） |
| `kFingerprintBrandVersion`      | `--fingerprint-brand-version`     | 指定品牌版本号                             |
| `kFingerprintGpuVendor`         | `--fingerprint-gpu-vendor`        | （开关存在，当前 GPU 伪造路径未直接使用）  |
| `kFingerprintGpuRenderer`       | `--fingerprint-gpu-renderer`      | （开关存在，当前 GPU 伪造路径未直接使用）  |
| `kFingerprintHardwareConcurrency` | `--fingerprint-hardware-concurrency` | 直接指定 `navigator.hardwareConcurrency`    |
| `kFingerprintPlatform`          | `--fingerprint-platform`          | 伪装平台（windows/linux/macos）            |
| `kFingerprintPlatformVersion`   | `--fingerprint-platform-version`  | 伪装平台版本                               |
| `kFingerprintTimezone`          | `--timezone`                      | 覆盖系统时区（**注意命令行名为 `timezone`**） |

Renderer 传播：`000-add-fingerprint-switches.patch → content/browser/renderer_host/render_process_host_impl.cc::RenderProcessHostImpl::PropagateBrowserCommandLineToRenderer()`（在开关继承数组中追加上述 8 个）；`018-timezone.patch` 再追加 `kFingerprintTimezone`。

---

## 身份标识指纹

### User-Agent / 品牌版本 / Client Hints

**Patch:** `patches/extra/fingerprint/002-user-agent-fingerprint.patch`、`patches/extra/fingerprint/019-user-agent-2.patch`

文件与函数落点：
- `002 → components/embedder_support/user_agent_utils.cc::GetUserAgentInternal()`：删除 `kHeadless` 前缀插入；末尾 `user_agent += blink::GetUserAgentFingerprintBrandInfo();`
- `002 → components/embedder_support/user_agent_utils.cc::GetUserAgentPlatform()` / `::GetUnifiedPlatform()`：命中 `kFingerprintPlatform` 时返回固定平台串
  - `windows` → `""` / `"Windows NT 10.0; Win64; x64"`
  - `linux` → `"X11; "` / `"X11; Linux x86_64"`
  - `macos` → `"Macintosh; "` / `"Macintosh; Intel Mac OS X 10_15_7"`
- `002 → third_party/blink/common/user_agent/user_agent_metadata.cc::UpdateUserAgentMetadataFingerprint()` 与 `::GetUserAgentFingerprintBrandInfo()`（019 对 `UpdateUserAgentMetadataFingerprint()` 全面重写，见下）
- `002 → third_party/blink/public/common/user_agent/user_agent_metadata.h`：声明上述两个函数
- `002 → third_party/blink/renderer/core/execution_context/navigator_base.cc::GetReducedNavigatorPlatform()`：按 `kFingerprintPlatform` 返回 `Win32` / `Linux x86_64` / `MacIntel`
- `002 → third_party/blink/renderer/core/frame/navigator.cc::Navigator::platform()`：同上
- `002 → third_party/blink/renderer/core/frame/navigator_ua.cc::NavigatorUA::userAgentData()`：先取 metadata 再调用 `UpdateUserAgentMetadataFingerprint`
- `002 → third_party/blink/renderer/core/loader/frame_fetch_context.cc::FrameFetchContext::AddClientHintsIfNecessary()`：同上
- `002 → content/browser/client_hints/client_hints.cc::UpdateNavigationRequestClientUaHeadersImpl()`：同上

行为：
- `--fingerprint-brand=Chrome|Edge|Opera|Vivaldi` 决定主品牌。`UpdateUserAgentMetadataFingerprint` 内置的默认版本：
  - `Chrome` → 保留 `metadata->full_version`（并同步 Chromium 版本）
  - `Edge` → `136.0.3240.92`
  - `Opera` → `120.0.5516.0`
  - `Vivaldi` → `7.4.3684.38`
- UA 字符串末尾追加（`GetUserAgentFingerprintBrandInfo()` 内置默认，**与上一套不同**）：
  - `Chrome` → 不追加
  - `Edge` → ` Edg/136.0.0.0`
  - `Opera` → ` OPR/120.0.0.0`
  - `Vivaldi` → ` Vivaldi/7.4.3684.38`
- Client Hints 的 `brand_version_list` / `brand_full_version_list` 被重建为固定顺序：`Chromium → 主品牌 → Not.A/Brand`。
- **019 补丁的关键改动**：`not_a_brand` 与 `chromium` 的名称/版本不再硬编码为 `Not.A/Brand` / `"99"` / `"99.0.0.0"`，而是**遍历原始 `brand_full_version_list`**，按 `brand.find("Not") && brand.find("Brand")` 以及 `brand == "Chromium"` 提取并复用已有条目（主版本号通过 `substr(0, '.')` 切分）。同时提取出 `default_*_version` 作为局部常量整理（逻辑未变）。
- `--fingerprint-platform=windows|linux|macos` 同时影响：
  - UA 字符串中的平台片段（`GetUserAgentPlatform()` / `GetUnifiedPlatform()`）
  - `navigator.platform`（`Navigator::platform()` 与 `GetReducedNavigatorPlatform()` 返回 `Win32` / `Linux x86_64` / `MacIntel`）
  - `UpdateUserAgentMetadataFingerprint()` 中 `metadata->platform` / `metadata->platform_version`：`Windows` / `19.0.0`、`Linux` / `6.11.0`、`macOS` / `15.2.0`（macOS 时 `architecture` 额外设为 `arm`）

### Client Hints 全局关闭

**Patch:** `patches/extra/ungoogled-chromium/add-flag-to-remove-client-hints.patch`
**Flag:** `--remove-client-hints`（feature `blink::features::kRemoveClientHints`，默认 disabled）

文件与函数落点：
- `content/browser/client_hints/client_hints.cc::UpdateNavigationRequestClientUaHeadersImpl()` 顶部 `return`
- `content/browser/client_hints/client_hints.cc::AddRequestClientHintsHeaders()` 顶部 `return`
- `third_party/blink/renderer/core/frame/navigator_ua.cc::NavigatorUA::userAgentData()` 直接返回未填充的 `ua_data`
- `third_party/blink/renderer/core/loader/frame_fetch_context.cc::FrameFetchContext::AddClientHintsIfNecessary()` 顶部 `return`
- `third_party/blink/common/features.cc` / `features.h`：声明 `kRemoveClientHints`

启用后：所有 `Sec-CH-*` 头不被发送，`navigator.userAgentData` 返回空对象。

### Reduced System Info

**Patch:** `patches/extra/ungoogled-chromium/add-flag-to-reduce-system-info.patch`
**Flag:** `--reduced-system-info`（feature `blink::features::kReducedSystemInfo`，默认 disabled）

- `components/embedder_support/user_agent_utils.cc::ShouldReduceUserAgentMinorVersion()` / `::ShouldSendUserAgentUnifiedPlatform()`：启用时直接返回 `true`
- `components/embedder_support/user_agent_utils.cc::GetUserAgentMetadata(const PrefService*, bool)`：启用时强制 `only_low_entropy_ch = true`
- `third_party/blink/renderer/core/execution_context/navigator_base.cc::NavigatorBase::platform()`：启用时回退到 `GetReducedNavigatorPlatform()`
- `third_party/blink/renderer/core/execution_context/navigator_base.cc::NavigatorBase::hardwareConcurrency()`：启用时固定返回 `2`
- `third_party/blink/common/features.cc` / `features.h`：声明 `kReducedSystemInfo`

### HTTP Accept Header

**Patch:** `patches/extra/ungoogled-chromium/add-flag-to-change-http-accept-header.patch`
**Flag:** `--http-accept-header=<value>`（纯命令行 switch，非 feature；`ORIGIN_LIST_VALUE_TYPE`）

- `content/public/browser/frame_accept_header.cc::FrameAcceptHeaderValue()`：开关命中时直接返回自定义值。
- `components/webui/flags/flags_state.cc::GetCombinedOriginListValue()` / `::FlagsState::SetOriginListFlag()`：对 `http-accept-header` 不做 origin 合并，保留用户原值。

### Referrer 定制

**Patch:** `patches/extra/ungoogled-chromium/add-flags-for-referrer-customization.patch`
**Flag:** `--remove-referrers`（复用已有 `features::kNoReferrers`）/ `--remove-cross-origin-referrers`（新增 `network::features::kNoCrossOriginReferrers`）/ `--minimal-referrers`（新增 `network::features::kMinimalReferrers`）

- 新增 `services/network/public/cpp/referrer_sanitizer.cc` / `.h`，提供两个重载 `sanitize_referrer()`（分别支持 `net::ReferrerPolicy` 与 `network::mojom::ReferrerPolicy`）
- 将原先的 `if (!enable_referrers) { ... }` 替换为 `referrer_sanitizer::sanitize_referrer(...)` 调用，改动的文件与函数：
  - `content/browser/renderer_host/navigation_request.cc::AddAdditionalRequestHeaders()`
  - `content/renderer/render_frame_impl.cc::RenderFrameImpl::FinalizeRequestInt(...)`（在文件中名为 `FinalizeRequestInternal` 系列，patch 截断行后缀 `Int`）
  - `services/network/network_service_network_delegate.cc::NetworkServiceNetworkDelegate::MaybeTruncateReferrer()`
  - `third_party/blink/renderer/modules/service_worker/web_service_worker_fetch_context_impl.cc`（finalize 路径）
  - `third_party/blink/renderer/platform/loader/fetch/url_loader/dedicated_or_shared_worker_fetch_context_impl.cc`（finalize 路径）
- `services/network/public/cpp/features.cc` / `.h`：声明 `kMinimalReferrers` / `kNoCrossOriginReferrers`

---

## 硬件信息指纹

### hardwareConcurrency / deviceMemory

**Patch:** `patches/extra/fingerprint/005-hardware-concurrency-fingerprint.patch`

- `005 → third_party/blink/renderer/core/frame/navigator_concurrent_hardware.cc::NavigatorConcurrentHardware::hardwareConcurrency()`：
  - 若显式指定 `--fingerprint-hardware-concurrency` → `std::stoul(value)` 直接返回
  - 否则若指定 `--fingerprint` → `std::stoull(seed)`, 返回 `((seed % 13) + 4) * 2` → 取值集合 `{8,10,12,…,32}`（偶数，最小 8 最大 32）
  - 否则回退到 `base::SysInfo::NumberOfProcessors()`
- `005 → third_party/blink/renderer/core/frame/navigator_device_memory.cc::NavigatorDeviceMemory::deviceMemory()`：恒定返回 `8`（原 `ApproximatedDeviceMemory::GetApproximatedDeviceMemory()` 调用被注释）。

### GPU（WebGL VENDOR / RENDERER）

**Patch:** `patches/extra/fingerprint/011-gpu-info.patch`

新增源文件：
- `third_party/blink/renderer/modules/webgl/gpu_info.cc` / `.h` — 内置 **57** 条 NVIDIA GeForce RTX 30/40/50 系列条目（含 Device ID），以及 **11** 条 macOS Apple M 系列型号（M1, M1 Pro, M2, M2 Max, M2 Pro, M3, M3 Max, M3 Pro, M4, M4 Max, M4 Pro）。
  - 导出 `GetGpuCount()`、`GetGpuInfo(index)`、`GetMacosGpuString(index)`、`GetLinuxGpuString(model)`、`GetWindowsGpuInfo(index)`、`GetWindowsGpuInfoList()`；并导出常量 `kMacosGpuModelCount`、`kWindowsVendorString="NVIDIA"`、`kLinuxVendorString="NVIDIA Corporation"`、`kWindowsGpuSuffix="Direct3D11 vs_5_0 ps_5_0, D3D11"`、`kLinuxGpuSuffix="/PCIe/SSE2, OpenGL 4.5.0"`。
- `third_party/blink/renderer/modules/webgl/gpu_fingerprint.cc` / `.h` — `GetGLVendorStringForFingerprint()` / `GetGLRendererStringForFingerprint()`；
  - 平台取自 `--fingerprint-platform`（默认 `windows`），fingerprint 通过 `base::StringToUint` 解析（非 hash）；
  - Renderer 下标：Windows/Linux `fingerprint % GetGpuCount()`，macOS `fingerprint % kMacosGpuModelCount`。

修改源文件：
- `011 → third_party/blink/renderer/modules/webgl/BUILD.gn::blink_modules_sources("webgl")`：加入 4 个新文件。
- `011 → third_party/blink/renderer/modules/webgl/webgl_rendering_context_base.cc::WebGLRenderingContextBase::getParameter()`：`GL_RENDERER` / `GL_VENDOR` 分支移除 `kSpoofWebGLInfo`/`ContextGL()->GetString(...)` 逻辑，改为直接调用上述两个 Fingerprint 函数。

平台差异（由 `gpu_info.cc` 组装）：
- Windows：`ANGLE (NVIDIA, NVIDIA <model> (<device_id>) Direct3D11 vs_5_0 ps_5_0, D3D11)`
- Linux：`ANGLE (NVIDIA Corporation, NVIDIA <model>/PCIe/SSE2, OpenGL 4.5.0)`（无 device_id，无空格接 `/PCIe`）
- macOS：`ANGLE (Apple, ANGLE Metal Renderer: Apple <M-series>, Unspecified Version)`

---

## Canvas / WebGL 指纹

### 噪声因子初始化（Document 级）

**Patch:** `patches/extra/fingerprint/008-fix-client-rects-and-canvas-fingerprint.patch`（初版，乘法因子）、`patches/extra/fingerprint/014-client-rects.patch`（随 008 同步接入 seed）、`patches/extra/fingerprint/017-client-rects-2.patch`（改为加法偏移，并把 hash 后缀由 `canvas_noise_x/y` 改为 `offset_x/y`）

- `008/014/017 → third_party/blink/renderer/core/dom/document.cc::Document::Document()`
  - 以 `--fingerprint` seed 生成 `noise_factor_x_` / `noise_factor_y_`
  - 008/014 阶段：`noise_factor = 1.0 + norm * 0.000003`，即乘法 `1.0 ± 0.0000015`，hash 后缀 `canvas_noise_x/y`
  - 017 阶段：改为 `noise_factor = norm * 0.002`，即加法偏移 `±0.001 px`，hash 后缀 `offset_x/y`

### 像素扰动核心逻辑（ShuffleSubchannelColorData）

**Patch:** `patches/extra/fingerprint/008-fix-client-rects-and-canvas-fingerprint.patch`（将随机数替换为 seed 哈希，像素数上限 `(w+h)/128`）、`patches/extra/fingerprint/012-canvas-get-image-data.patch`（全部改为 LSB 翻转、避开纯黑/纯白、像素数上限改为 `(w*h)/128` 并夹到 2~10）

- `008/012 → third_party/blink/renderer/platform/graphics/static_bitmap_image.cc::StaticBitmapImage::ShuffleSubchannelColorData()`
  - 使用 `hash(seed + "_<i>_x")` / `hash(seed + "_<i>_y")` 生成像素坐标（距离边缘至少 1 个像素：`% (w-2) + 1`）
  - 使用 `hash(seed + "_x<x>_y<y>_r/g/b") & 1` 决定 R/G/B 通道最低位翻转
  - 若像素为纯黑或纯白则 `break` 跳过，避免图像逻辑被破坏
  - 像素数量 = `clamp((w * h) / 128, 2, 10)`
  - 支持格式：`Alpha_8`、`Gray_8`、`RGB_565`、`ARGB_4444`、`RGBA_8888`、`BGRA_8888`；其余格式直接 `return`

### 2D Canvas `getImageData()`

**Patch:** `patches/extra/fingerprint/012-canvas-get-image-data.patch`
`012 → third_party/blink/renderer/modules/canvas/canvas2d/base_rendering_context_2d.cc::BaseRenderingContext2D::getImageDataInternal()`
- `read_pixels_successful && HasSwitch(kFingerprint)` 时对返回的 pixmap 执行 `ShuffleSubchannelColorData`。
- 同一 patch 还把 `measureText()` 的触发条件由 `FingerprintingCanvasMeasureTextNoiseEnabled()` 改为 `--fingerprint` 开关（后续 015 进一步细化噪声系数）。

### 2D Canvas `toDataURL()`

**Patch:** `patches/extra/fingerprint/013-canvas-toDataURL.patch`
`013 → third_party/blink/renderer/core/html/canvas/html_canvas_element.cc::HTMLCanvasElement::ToDataURLInternal()`
- 当 `readback_type == kWebExposed` 且有 `--fingerprint` 时：
  - 分配独立 RGBA8888 缓冲 → `PaintImage::readPixels` → `ShuffleSubchannelColorData` → `SkImages::RasterFromData` 构建新 `SkImage` → `PaintImageBuilder` 包装 → `StaticBitmapImage::Create` 替换原 `image_bitmap`
- 保证**不污染原始 canvas 数据**，只影响导出字节串。

### 2D Canvas `measureText()`

**Patch:** `patches/extra/fingerprint/008`（首次接入 seed）→ `patches/extra/fingerprint/015-canvas-measure-text.patch`（细化噪声系数 `0.00001`，并增加 `text.length() > 0` 守卫）
`015 → third_party/blink/renderer/modules/canvas/canvas2d/base_rendering_context_2d.cc::BaseRenderingContext2D::measureText()`
- `hash(seed + "clientrects_noise_x")` 归一到 `[-0.5, 0.5]`，再乘 `0.00001` 作为 `noise_x`；仅当 `text.length() > 0` 调用 `text_metrics->Shuffle(noise_x)`。

### WebGL `readPixels()`

**Patch:** `patches/extra/fingerprint/016-webgl-readPixels.patch`
`016 → third_party/blink/renderer/modules/webgl/webgl_rendering_context_base.cc::WebGLRenderingContextBase::ReadPixelsHelper()`
- 在 `ContextGL()->ReadPixels(...)` 之后根据 WebGL `format`/`type` 映射到 `SkColorType`（`kUnpremul_SkAlphaType`）：
  - `GL_RGBA + GL_UNSIGNED_BYTE` → `kRGBA_8888_SkColorType`
  - `GL_RGB + GL_UNSIGNED_BYTE` → `kRGB_888x_SkColorType`
  - `GL_ALPHA + GL_UNSIGNED_BYTE` → `kAlpha_8_SkColorType`
  - 其它 → fallback 为 `kRGBA_8888_SkColorType`
- 然后对已读出的像素数据调用 `StaticBitmapImage::ShuffleSubchannelColorData()`。

### Image 编码路径（toBlob / toDataURL 通用路径）

**Patch:** `patches/extra/fingerprint/008-fix-client-rects-and-canvas-fingerprint.patch`
`008 → third_party/blink/renderer/platform/graphics/image_data_buffer.cc::ImageDataBuffer::EncodeImage()`
- 将触发条件由 `FingerprintingCanvasImageDataNoiseEnabled()` 改为 `--fingerprint` 开关。

---

## DOM 几何指纹

### `getClientRects()` / `getBoundingClientRect()`

**Patch:** `patches/extra/fingerprint/014-client-rects.patch`（由 `FingerprintingClientRectsNoiseEnabled()` 替换为 `--fingerprint` 开关，仍是 `Scale`）
`patches/extra/fingerprint/017-client-rects-2.patch`（`Scale` → `Offset`，新增 `ShouldSkipClientRectsOffset`）

- `014/017 → third_party/blink/renderer/core/dom/element.cc::Element::getClientRects()` / `::GetBoundingClientRectNoLifecycleUpdate()`
  - 017 将 `quad.Scale(...)` / `rect.Scale(...)` 改为 `quad.Offset(...)` / `rect.Offset(...)`，并在调用前增加 `if (!ShouldSkipClientRectsOffset())` 守卫
- `017 → third_party/blink/renderer/core/dom/element.cc::Element::ShouldSkipClientRectsOffset()`（新增）：当 `style.GetPosition() == EPosition::kAbsolute` 且 `top.IsZero()||top.IsFixed()` 且 `left.IsZero()||left.IsFixed()` 时返回 `true`（跳过偏移以避免破坏 UI 布局）。
- `017 → third_party/blink/renderer/core/dom/element.h`：声明 `ShouldSkipClientRectsOffset()`。
- `014/017 → third_party/blink/renderer/core/dom/range.cc::Range::getClientRects()` / `::Range::getBoundingClientRect()`：同样由 `Scale` 改为 `Offset`（无 `ShouldSkipClientRectsOffset` 守卫）。
- `017 → ui/gfx/geometry/quad_f.cc`/`.h`：新增 `QuadF::Offset(float, float)`（以及 `Offset(float)` inline 重载）；当 AABB 宽或高 `WithinEpsilon(..., 0.0f)` 时短路。

加法偏移的实际幅度由 Document 构造期间计算的 `noise_factor_{x,y}` 决定（`norm * 0.002`，约 ±0.001 px）。

---

## 音频指纹

**Patch:** `patches/extra/fingerprint/003-audio-fingerprint.patch`
`003 → third_party/blink/renderer/modules/webaudio/offline_audio_context.cc`

- 新增文件内匿名函数 `getNoiseData(number_of_frames)`：
  - `combined = fingerprint_str + std::to_string(number_of_frames) + "audio"`
  - `one_percent = max(1, number_of_frames / 100)`，`noise_limit = min(one_percent, 1000)`
  - 返回 `(hash(combined) % noise_limit) + 1`（未设 `--fingerprint` 或 seed 为空 → `0`）
- `OfflineAudioContext::OfflineAudioContext(...)` 构造时：`total_render_frames_ = number_of_frames + getNoiseData(number_of_frames)`，并以 `total_render_frames_` 创建 `OfflineAudioDestinationNode`。

---

## 字体指纹

**Patch:** `patches/extra/fingerprint/006-font-fingerprint.patch`
`006 → third_party/blink/renderer/platform/fonts/font_cache.cc::FontCache::GetFontPlatformData()`

- 仅对 `creation_params.CreationType() == kCreateFontByFamily` 且 family **不属于** `basic_fonts` 的字体生效。
- `basic_fonts` 白名单（硬编码）：`Arial`、`Times New Roman`、`Courier New`、`Georgia`、`Verdana`、`Microsoft YaHei`、`SimSun`（最后两个用于避免中文乱码）。
- `hash(fingerprint + family_name)` 归一到 `[0, 1)`（除以 `numeric_limits<uint32_t>::max()`）；若 `< 0.05` 则 `return nullptr`，等效于该字体对页面不可用，从而扰动字体枚举指纹。

---

## 时区指纹

**Patch:** `patches/extra/fingerprint/018-timezone.patch`
**Flag:** `--timezone=<IANA zone>`（switch 常量名 `kFingerprintTimezone`，但命令行名故意定义为 `"timezone"`）

- `018 → components/ungoogled/ungoogled_switches.cc`/`.h`：新增 `kFingerprintTimezone = "timezone"`。
- `018 → content/browser/renderer_host/render_process_host_impl.cc::PropagateBrowserCommandLineToRenderer()`：在开关继承数组追加。
- `018 → third_party/blink/renderer/core/timezone/timezone_controller.cc::GetCurrentTimezoneId()`：命中 switch 时直接返回用户指定值，跳过 ICU 默认查询。
- `018 → third_party/blink/renderer/core/timezone/timezone_controller.cc::SetIcuTimeZoneAndNotifyV8()`：命中 switch 时用用户指定值覆盖传入的 `timezone_id`，再创建 `icu::TimeZone`（保持 `Date` / `Intl` 行为一致）。

---

## 自动化 / Headless 检测规避

### navigator.webdriver

**Patch:** `patches/extra/fingerprint/009-webdriver.patch`
`009 → third_party/blink/renderer/core/frame/navigator.cc::Navigator::webdriver()`

- 删除 `if (RuntimeEnabledFeatures::AutomationControlledEnabled()) return true;`，仅保留 `probe::ApplyAutomationOverride` 路径；即只有 DevTools 显式注入时才会返回 true。

### Headless 产品名

**Patch:** `patches/extra/fingerprint/010-headless.patch`
`010 → headless/lib/browser/headless_browser_impl.cc` 匿名命名空间的 `kHeadlessProductName`

- `"HeadlessChrome"` → `"Chrome"`，让 headless 模式下默认 UA 不再暴露 `HeadlessChrome` 关键字。与之配合，`002-user-agent-fingerprint.patch` 删除了 `GetUserAgentInternal()` 中对 `--headless` 的 `Headless` 前缀插入。

### DevTools Runtime 绑定禁用

**Patch:** `patches/extra/fingerprint/001-disable-runtime.enable.patch`
`001 → v8/src/inspector/v8-runtime-agent-impl.cc` / `.h`

- `V8RuntimeAgentImpl::addBindings()`：函数体第一行改为 `return;`（整段原有实现被注释保留），阻止远程调试绑定被附加。
- `V8RuntimeAgentImpl::messageAdded()`：`if (m_enabled) reportMessage(message, true);` 被整行注释掉。
- `v8-runtime-agent-impl.h::V8RuntimeAgentImpl::enabled()`：`return m_enabled;` 改为 `return false;`。

---

## Shadow DOM 扩展

**Patch:** `patches/extra/fingerprint/007-shadow-root.patch`
文件与函数落点：
- `007 → third_party/blink/renderer/core/dom/element.cc`：新增 `Element::FakeShadowRoot()`（直接返回 `GetShadowRoot()`，无论 open/closed）。
- `007 → third_party/blink/renderer/core/dom/element.h`：声明 `FakeShadowRoot()`。
- `007 → third_party/blink/renderer/core/dom/element.idl`：暴露 `[PerWorldBindings, ImplementedAs=FakeShadowRoot] readonly attribute ShadowRoot? fakeShadowRoot;`

用途：提供能绕过 `OpenShadowRoot()` mode 检查的后门访问接口（便于自动化脚本访问 closed shadow root，同时不改变 `shadowRoot` 的原有可见行为）。

---

## TLS / 网络层指纹

### TLS GREASE 开关

**Patch:** `patches/extra/ungoogled-chromium/add-flag-to-disable-tls-grease.patch`
**Flag:** `--disable-grease-tls`（纯 switch，不是 feature）
落点：`net/socket/ssl_client_socket_impl.cc::SSLClientSocketImpl::SSLContext::SSLContext()`，约第 202 行

```cpp
int grease_mode = !base::CommandLine::ForCurrentProcess()->HasSwitch("disable-grease-tls");
SSL_CTX_set_grease_enabled(ssl_ctx_.get(), grease_mode);
```

- 未指定时：`grease_mode = 1`（保持 Chromium 默认 GREASE 行为）。
- 指定时：`grease_mode = 0` → ClientHello 完全确定，产生稳定 JA3/JA4。

### 未覆盖的 TLS 维度

项目未直接修改 `third_party/boringssl/`，因此 GREASE 之外的 cipher suite 顺序、TLS extension 顺序、named groups、signature algorithms、会话票据等握手指纹与官方 Chromium 136 保持一致。

---

## 总结

- 所有基于 `--fingerprint` seed 的随机化均通过 `std::hash<std::string>(seed + "<维度标识>")` 生成确定性但看似随机的值（例外：hardwareConcurrency 与 GPU 索引直接把 seed 当成整数 `stoull` / `StringToUint` 使用）。
- 开关常量与 feature flag 的声明位置：`components/ungoogled/ungoogled_switches.*`（指纹通用 seed / 品牌 / 平台 / 并发 / 时区），`third_party/blink/common/features.*`（`kReducedSystemInfo` / `kRemoveClientHints`），`services/network/public/cpp/features.*`（`kMinimalReferrers` / `kNoCrossOriginReferrers`）。
- 保护面覆盖：UA 字符串及 Client Hints、Canvas（getImageData/toDataURL/measureText）、WebGL（readPixels + VENDOR/RENDERER）、DOM 几何（getClientRects/getBoundingClientRect）、OfflineAudioContext、字体枚举、时区、hardwareConcurrency / deviceMemory、自动化检测（navigator.webdriver + headless 产品名 + DevTools runtime 绑定）、Shadow DOM 访问、TLS GREASE、HTTP Accept / Referrer / Client Hints。
