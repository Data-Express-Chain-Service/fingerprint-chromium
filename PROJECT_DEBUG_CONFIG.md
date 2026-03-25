# 浏览器指纹随机化分类

## 核心机制

所有随机化基于统一的 fingerprint seed（通过 `--fingerprint` 命令行参数传入），使用 `hash(seed + "维度标识")` 生成确定性噪声，保证同一 seed 下行为一致。

## 覆盖的指纹维度

### 身份标识

| Patch   | 维度                  | 方式                                                                        |
|---------|-----------------------|-----------------------------------------------------------------------------|
| 002/019 | User-Agent / 品牌版本 | 可伪装成 Chrome / Edge / Opera / Vivaldi，修改 UA 字符串、品牌列表、Client Hints |
| 018     | 时区                  | `--timezone` 参数覆盖系统时区（ICU + V8 双层）                               |

### 硬件信息

| Patch | 维度          | 方式                                                              |
|-------|---------------|-------------------------------------------------------------------|
| 011   | WebGL GPU 信息 | 内置 65+ NVIDIA GPU 型号数据库，按 seed 选择，含平台特定格式字符串 |
| 005   | CPU 核心数    | `((seed % 13) + 4) * 2` 生成 8-32 范围的假核心数；内存固定返回 8GB |

### Canvas 指纹

| Patch | 维度               | 方式                                       |
|-------|--------------------|--------------------------------------------|
| 008   | Canvas 噪声初始化  | 用 seed 替换随机数，生成确定性偏移因子      |
| 012   | getImageData()     | 修改 2-10 个像素的 LSB，避开纯黑/纯白       |
| 013   | toDataURL()        | 在副本上做像素扰动再编码，不影响原始 canvas |
| 015   | measureText()      | 对文本度量结果施加 0.00001 级别的噪声       |
| 016   | WebGL readPixels() | 与 Canvas API 使用相同的像素混淆逻辑        |

### DOM 几何

| Patch   | 维度                                       | 方式                                             |
|---------|--------------------------------------------|--------------------------------------------------|
| 014/017 | getClientRects() / getBoundingClientRect() | 加法偏移（±0.001px）替代乘法缩放，跳过绝对定位元素 |

### 其他

| Patch | 维度                | 方式                                               |
|-------|---------------------|----------------------------------------------------|
| 003   | AudioContext 参数   | 对 OfflineAudioContext 的帧数施加 ±1% 噪声          |
| 006   | 系统字体            | 非基础字体以 5% 概率隐藏（基于 seed + 字体名 hash） |
| 009   | navigator.webdriver | 移除自动设置逻辑，只允许显式注入                   |
| 010   | Headless 检测       | 将 HeadlessChrome 产品名改为 Chrome                |
| 001   | DevTools 自动化绑定 | 禁用 V8 runtime 绑定，`enabled()` 恒返回 false     |
| 007   | Shadow DOM          | 添加 fakeShadowRoot 访问接口                       |

## 总结

项目的整体思路是：用一个 seed 驱动所有维度，通过哈希生成确定性但看似随机的值，使得同一 seed 下跨会话行为一致（可复现），不同 seed 之间行为差异足以规避指纹追踪。

---

# TLS 指纹

## 核心机制

项目有一个直接的 TLS 层 patch，以及数个影响网络指纹的 HTTP 层 patch。

## TLS GREASE 控制

**Patch:** `patches/extra/ungoogled-chromium/add-flag-to-disable-tls-grease.patch`
**Modified file:** `net/socket/ssl_client_socket_impl.cc`（~line 202）
**Flag:** `--disable-grease-tls`（chrome://flags 中对应 "Disable GREASE for TLS"）

```cpp
// Original:
SSL_CTX_set_grease_enabled(ssl_ctx_.get(), 1);

// Patched:
int grease_mode =
    !base::CommandLine::ForCurrentProcess()->HasSwitch("disable-grease-tls");
SSL_CTX_set_grease_enabled(ssl_ctx_.get(), grease_mode);
```

- **GREASE（RFC 8701）**：向 TLS ClientHello 插入随机保留值（cipher suites、extensions、named groups），使 JA3/JA4 哈希每次连接不同。
- **禁用效果**：ClientHello 完全确定，产生稳定的 JA3/JA4 指纹。

## HTTP 层网络指纹 Patch

| Patch 文件                            | Flag                                                                          | 效果                                                       |
|---------------------------------------|-------------------------------------------------------------------------------|------------------------------------------------------------|
| add-flag-to-change-http-accept-header | `--http-accept-header=<value>`                                                | 覆盖 HTTP `Accept:` 头                                     |
| add-flag-to-remove-client-hints       | `--remove-client-hints`                                                       | 屏蔽所有 `Sec-CH-*` 头 + 清空 `navigator.userAgentData`    |
| add-flag-to-reduce-system-info        | `--reduced-system-info`                                                       | 缩减 UA 副版本号，hardwareConcurrency 返回 2，通用平台字符串 |
| add-flags-for-referrer-customization  | `--remove-referrers` / `--remove-cross-origin-referrers` / `--minimal-referrers` | 通过 referrer_sanitizer 控制 `Referer:` 头策略             |

## TLS 层未覆盖的维度

以下内容未做 patch（需直接修改 `third_party/boringssl/`，本项目未涉及）：

- Cipher suite 顺序
- TLS extension 顺序（影响 JA3/JA4 extension 列表）
- 支持的椭圆曲线组（named groups）
- 签名算法（signature algorithms）
- TLS 版本偏好
- Session ticket 行为

因此，GREASE 值以外的 TLS 握手指纹（JA3/JA4）与 stock Chromium 136 相同。
