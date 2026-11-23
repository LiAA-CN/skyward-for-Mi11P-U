# 小米 11 Ultra (star) 适配 Skyward — 进度存档

## 当前状态（2026-11-23 晚）

### 已打通 ✅
- 构建/打包全链路（Rust 1.100 + NDK r29 + cargo-ndk + skyward-pack）
- recovery 模式进入：`fastboot flash boot_a` + `fastboot reboot-recovery`（misc BCB）
- 屏幕显示（msm_drm / 内置驱动，V14 内核与 18PM ikun 内核均验证）
- 触摸（修复：多触摸面板中选**触控范围最大**的——主屏 vs 副屏 cyttsp5 冲突）
- MTP / 文件管理 / persist 挂载 / 已装系统信息读取
- 分区备份（E:\Android\Mi11U\backup\，V14 状态全套 10 分区）

### 仓库
- `LiAA-CN/skyward-for-Mi11P-U` 的 `star` 分支（origin；mi10/upstream remote 保留）
- 打包镜像在 `E:\Android\Mi11U\skyward-star-18pm.img`（18PM 内核底座，可刷版）
- 环境脚本：`source $HOME/.cargo/env` + `export ANDROID_NDK_HOME=/home/LiAA-CN/android_dev/tools/android-ndk-r29`

### ⚠️ 明天第一步：未完成的构建
`wlan.rs` 的 stderr 捕获改动（supplicant stderr → /skyward/run/supplicant/stderr.log）
已写好但**还没成功编译打包**——上次构建忘了设 ANDROID_NDK_HOME。设好后重新构建、
重新打包、重新刷入验证。注意 E0283（stderr 的 `.into()` 类型推断）可能需要显式
`Stdio::from(file)`。

### WiFi 未打通 — 排查到哪了
1. ✅ persist/modem 复用已有挂载（`hung()` 支持槽位后缀 _a/_b）
2. ✅ fstab.default 后缀按 cmdline 解析
3. ✅ 固件链：regdb_xiaomi.bin / bd_k1.elf 从 modem 找到 ✓
4. ✅ 内置驱动上电：fs_ready + /dev/wlan "ON\n" → **wlan0 已出现**
5. ❌ supplicant 死在最后一步：
   - vendor 版：缺 `android.hardware.wifi.supplicant@1.0.so`（18PM ROM 删了 HIDL 栈）→ 死路
   - **apex 版（当前方案）**：`wpa_supplicant_mainline`（com.android.wifi capex 挂载 ✓ 能链接能跑）
     但 "Creating the mainline supplicant instance" 后立刻 "Failed to initialize wpa_supplicant"
     （小米 mainline supplicant，-dd 也不给细节）。它内部引用 /data/misc_ce|de 路径——
     已创建 `/data/misc_ce/0/misc_wifi` + `/data/misc_de/0/misc_wifi`，未再验证。
   - 待办：stderr.log 拿到真实报错；若指向 keystore2 之类，评估替代：
     a) 从官方 V14/OS2.0 super 提取 vendor HIDL supplicant 栈（super 是小米魔改 sparse，
        simg2img 解不了——需研究其格式或找 xiaomi.eu 的 raw 镜像）
     b) 自编 wpa_supplicant（bionic + libnl + openssl，NDK 可行）
6. ⚠️ `is_declared` 方法号问题：recovery 底座 sdk<36 时走了未验证分支（=7），A14 的
   servicemanager 拒绝该调用 → 已降级为"重启目录重读描述"（不再中止 bring_up）。
   正确编号待考（AOSP A14 的 is_declared 可能是 8）。

### 明天的顺序
1. 设 env → 重新构建 + 打包 + 刷入 → 看 stderr.log 拿 supplicant 真实死因
2. 修 supplicant（按死因选路 a/b）
3. WiFi 通了之后：HIDL Keymaster 移植（解密 data，fork 的 593ec17 做参照）
4. 手机端"修补当前 boot"固化功能
5. star.toml 细节校准（马达 swr_haptics、圆角/挖孔、密度确认）

### 已知限制 / 注意
- 一切只做只读和本地操作；刷写动作由用户自己执行
- V14 官方 super.img 是魔改 sparse（file_hdr=786460 非标准），标准工具解不了
- 工具链：tools/ 下有 erofs-utils、lpunpack.py、simg2img.py、fdt-dump.py、parse_vendor_boot.py、zstd
- **grep 管道陷阱**：`build 2>&1 | grep error` 匹配到 error 行仍返回成功——判成败要看二进制时间戳
