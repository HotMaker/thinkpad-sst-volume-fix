# 联想 ThinkPad (Intel SST + Realtek) 内置扬声器音量无法调节 —— 诊断与根治

> 机型：ThinkPad T14s Gen 4（21F6/21F7），Intel Alder Lake + Intel SST + Realtek ALC257，Win11 内核 24H2~26H2。
> 本文记录：症状、根因、诊断证据，以及**两种可落地的解决方案**与**水印去除**。

---

## 一、症状

- **内置扬声器**：系统音量条 / Fn 音量键**完全无效**，只有 0%（静音）和 100% 两种效果（1~100 都一样响）。
- **耳机**：音量调节**完全正常**。
- **音量合成器里逐 App 调**：有效（走软件增益）。
- 更诡异：**每次重启，整体响度还不一样**（时高时低）。

## 二、根因

Windows 调音量分两层：

1. **软件音量**：音频引擎对采样乘增益（"音量合成器"逐 App 那条）。
2. **硬件音量**：若驱动在 KS 拓扑里暴露 `KSNODETYPE_VOLUME` 节点，**系统主音量就走这个硬件节点**。

这颗平台（Intel SST + Realtek UAD）的缺陷在于：**内置扬声器这一路的"硬件音量节点"存在但增益不生效**（耳机那一路正常）。因此：

- 系统主音量（走硬件节点）→ 无效；
- 逐 App 音量（走软件）→ 有效；
- 由于实际响度由"硬件节点每次上电落到的增益"决定，所以**每次重启整体响度会漂移**。

用 Core Audio 实测：`IAudioEndpointVolume::QueryHardwareSupport` 返回 **VOLUME=True**（驱动自称支持硬件音量），但改变它不影响实际响度 —— 典型的"假硬件音量节点"。

## 三、诊断证据（可复现）

- `QueryHardwareSupport = 0x3`（VOLUME+MUTE），音量范围 -65.25~0 dB，程序设定 50%/20%/5% 都被"接受"，但扬声器响度不变。
- 关"音频增强"、调 Realtek Audio Console、单独调 App —— 只有 App（软件）有效。
- 换 Realtek 驱动版本（6.0.9847.1 / 6.0.9954.1）都一样。
- 同类案例：微软问答 *"Laptop Speaker (Realtek(R) Audio) doesn't change volume when I change the master volume?"* —— 症状完全一致。

## 四、解决方案 A（推荐，根治）：WinSoftVol

[WinSoftVol](https://github.com/dechamps/WinSoftVol)（Equalizer APO 作者 dechamps 出品）是一个**内核过滤驱动**，作用是**隐藏坏掉的 `KSNODETYPE_VOLUME` 节点**，让 Windows 自动改用**软件音量** → 系统音量条原生恢复正常，且不再随重启漂移。

**它没微软签名，需要开启"测试签名模式"**（或用 SSDE 绕过，较复杂）。

一键安装：见 `install-winsoftvol.ps1`（管理员运行；开机自动加载；可卸载）。

## 五、解决方案 B（无需驱动）：VolSync

见仓库 [HotMaker/VolSync](https://github.com/HotMaker/VolSync)：用户态小程序，把系统主音量实时镜像到各 App 的会话音量。**不用驱动、不用签名**，但会覆盖单 App 微调。适合不想动签名/测试模式的人。

## 六、去掉"测试模式"水印：UWD

开了测试签名后，桌面右下角会有"测试模式"水印。用 **Universal Watermark Disabler (UWD)** 可隐藏它（COM 劫持 + DLL 代理，不改系统文件，可卸载）：

- 维护版（支持 Win11 26H2、带持久化）：`RYZCfate/universal-watermark-disabler` → `uwd_persistent_installer.exe`，管理员运行点 **Install**。
- 原理：把 `HKCR\CLSID\{ab0b37ec-56f6-4a0e-a8fd-7a8bf7c2da96}\InProcServer32` 劫持到代理 DLL `painter_x64.dll`，拦截资源管理器绘制水印文字。

## 七、重装系统后怎么办

1. 先装联想音频整包（麦克风恢复；系统音量对扬声器仍无效 —— 这是原始状态）；
2. 运行 `install-winsoftvol.ps1`（一键：下载驱动 → 装证书 → 建服务 → 挂过滤 → 开测试签名）；
3. 重启；
4. 需要的话再跑一次 UWD 去水印。

## 八、撤销

- 关测试签名：`bcdedit /set testsigning off`
- 移除过滤：删掉 MEDIA 类 `UpperFilters` 里的 `WinSoftVol`
- 删服务：`sc delete WinSoftVol`，删除 `C:\Windows\System32\drivers\WinSoftVol.sys`
- 恢复水印：再运行 UWD 点 Uninstall
- 或直接运行 `uninstall-winsoftvol.ps1`

## 九、关键踩坑（写脚本时必看）

- `sc create` 时内核驱动 **ImagePath 必须用 `\SystemRoot\System32\drivers\x.sys`**，用 `C:\...` 或相对路径会报 123/3。
- 下载来的 `.sys` 带"网络来源"标记时可能加载被拒，先 `Unblock-File`。
- 证书要装进 **本地计算机\受信任的根** 与 **受信任的发布者**。

## 参考

- WinSoftVol: https://github.com/dechamps/WinSoftVol
- Universal Watermark Disabler: https://github.com/pr701/universal-watermark-disabler
- 微软文档 软件音量: https://learn.microsoft.com/windows-hardware/drivers/audio/software-volume-control-support

## 许可

MIT
