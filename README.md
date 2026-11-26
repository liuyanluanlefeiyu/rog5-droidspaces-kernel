# ROG Phone 5 容器内核（DroidSpaces / LXC 向）

为 **ASUS ROG Phone 5（anakin / ASUS_I005D / ZS673KS）** 自编译的 5.4.210 GKI 内核，
在**完整保留原厂 vendor 模块**的前提下，开启运行 DroidSpaces / LXC 类容器所需的内核能力。

> 一句话概括：只换 boot 分区内核，不动 dtbo / vendor / super 分区，不替换任何 vendor 模块，
> 用 KABI 预留槽承载 SysV IPC 与 POSIX mqueue 字段，使导出符号 CRC 保持不变。

---

## 1. 最终产物

| 项 | 值 |
|---|---|
| 内核版本 | `5.4.210-qgki-perf`（与原厂 UTS_RELEASE 完全一致） |
| 镜像大小 | 46,309,888 字节（44.16 MiB，未压缩 `Image`） |
| SHA256 | `8123b3c79a2a1823b98fedecd458e88cc0826cc3eb4f9d30d09ab49c88745890` |
| 发布页 | [Releases · f2-5.4.210-qgki-perf](../../releases/tag/f2-5.4.210-qgki-perf) |

Release 内附：`Image`、`Image.gz`、`kernel-f2.config`、`config-vs-stock-f2.diff`、`SHA256SUMS.txt`。

## 2. 验证通过的基线

| 项目 | 值 |
|---|---|
| 机型 | ASUS ROG Phone 5（ZS673KS，SM8350） |
| 固件 | WW_33.0210.0210.242（Android 13） |
| 内核源码 | [`alfaonyt/android_kernel_asus_sm8350`](https://github.com/alfaonyt/android_kernel_asus_sm8350) @ `lineage-20`（LA.UM.9.14.r1-20800-LAHAINA.QSSI13.0） |
| 工具链 | **AOSP clang 11.0.2（r383902b）** + LTO_CLANG + CFI_CLANG，与原厂内核同版 |
| 配置基准 | 从原厂内核 `IKCONFIG` 提取的完整 `.config`（而非 `vendor/anakin_defconfig`） |
| Root 方案 | APatch（换内核后需用 APatch 重新修补 boot） |

## 3. 为什么不能直接改 `anakin_defconfig`

这台机器的 vendor 模块是按**原厂内核的 ABI** 编译的。任何改变关键结构体布局的配置项，都会让
`CONFIG_MODVERSIONS` 的符号 CRC 失配，模块加载失败，表现为**卡静态开机 logo 后看门狗重启**。

实测结论（每一项都是单独编译 + 真机 `fastboot boot` 验证）：

| 配置项 | 结论 | 原因 |
|---|---|---|
| `CGROUP_PIDS` / `CGROUP_DEVICE` | ❌ 卡开机 | `CGROUP_SUBSYS_COUNT` 是编译期常量，`struct cgroup` 内 `subsys[]` / `e_csets[]` 数组随之变长 |
| `CFS_BANDWIDTH` | ❌ 卡开机 | 依赖 `FAIR_GROUP_SCHED`，会改变调度器结构体布局 |
| `BRIDGE_NETFILTER` / `IP_SET` 等 | ❌ 卡开机 | 属冗余增强（原厂已有完整 IPv4 NAT / MASQUERADE / conntrack / VETH / BRIDGE） |
| `SYSVIPC` / `POSIX_MQUEUE` | ✅ 可用 | 字段被移入 `ANDROID_KABI_*` 预留槽，布局不变（见下） |
| `IPC_NS` / `USER_NS` / `PID_NS` | ✅ 可用 | 原厂已开 `NAMESPACES` / `UTS_NS` / `NET_NS`，这三个可安全追加 |
| `DEVTMPFS`、`TMPFS_XATTR`、`TMPFS_POSIX_ACL`、`MACVLAN`、`IP6_NF_NAT` | ✅ 可用 | 不涉及被 vendor 模块依赖的结构体 |

另外两个与工具链相关的差异也会导致失败，已在新版本中消除：

- **编译器版本**：Ubuntu clang 14 会自动开启 `CONFIG_STACKPROTECTOR_PER_TASK`
  （`CC_HAVE_STACKPROTECTOR_SYSREG`），改变 `task_struct` 布局；clang 11 下该选项自然不存在。
- **RELR**：我们的树里默认启用，原厂没有；关闭后镜像大小与原厂直接对齐（`image_size` 一字节不差）。

## 4. KABI 补丁做了什么

`include/linux/android_kabi.h` 在 `__GENKSYMS__`（CRC 计算）路径下，会把
`_ANDROID_KABI_REPLACE(spare, use)` 展开为**原来的占位符**。因此：

```c
/* include/linux/sched.h */
#ifdef CONFIG_SYSVIPC
    ANDROID_KABI_USE(6, struct sysv_sem sysvsem);
    ANDROID_KABI_USE(7, struct sysv_shm sysvshm);
#endif
```

- 正常编译时：这两个字段落在预留槽位置；
- 计算符号 CRC 时：仍然是原来的 `reserved[6]` / `reserved[7]`，**导出符号 CRC 不变**。

`include/linux/sched/user.h` 里的 `mq_bytes` 同理放进 `ANDROID_KABI_USE(1, ...)`。

实测判据（真机）：

```bash
su -c 'dmesg | grep -ci "disagrees about version of symbol"'   # → 0
```

同时 WiFi / 蓝牙 / 音频 / 显示等 vendor 模块驱动全部正常。

## 5. 能力矩阵

| 能力 | 状态 | 说明 |
|---|---|---|
| 命名空间（UTS / IPC / PID / NET / USER） | ✅ | DroidSpaces 的沙箱化基础 |
| `SYSVIPC` + `POSIX_MQUEUE` | ✅ | 含 `SYSVIPC_SYSCTL`、`SYSVIPC_COMPAT` |
| cgroups：`CGROUPS` / `MEMCG` / `CPUSETS` / `FREEZER` / `BPF` | ✅ | 原厂已有，可做内存限制、CPU 绑核、冻结 |
| `DEVTMPFS` / `TMPFS_XATTR` / `TMPFS_POSIX_ACL` | ✅ | 容器 `/dev` 与 tmpfs 权限 |
| `OVERLAY_FS`、`VETH`、`BRIDGE`、IPv4 NAT / MASQUERADE | ✅ | 原厂已有，容器联网够用 |
| `MACVLAN`（`--net=l` 模式） | ✅ | 容器可直接拿同网段独立 IP |
| IPv6 NAT（`IP6_NF_NAT` + `MASQUERADE`） | ✅ | 容器内 IPv6 可出网 |
| **进程数限制**（`CGROUP_PIDS`） | ❌ 永久不可用 | 破坏 ABI；替代：容器内 `ulimit -u` |
| **CPU 限速**（`CFS_BANDWIDTH`） | ❌ 永久不可用 | 破坏 ABI；替代：`cpuset` 绑核 |
| ipset / 桥接过滤增强 | ❌ 未启用 | 冗余且卡开机 |

DroidSpaces 特性页实测：可选特性中仅 `CPU limit` 与 `Process limit` 两项为叉，其余全部通过；
容器可正常创建、启动，网络与挂载正常。

## 6. 已知提示（正常现象）

开机后系统会弹一次：

> 您的设备内部出现了问题。请联系您的设备制造商了解详情。

成因已完整定位：Android 的兼容性矩阵（`/system/etc/vintf/compatibility_matrix.*.xml`）要求
`CONFIG_SYSVIPC=n`（Google 的安全策略），而容器需要 `SYSVIPC=y`，于是
`Build.isBuildConsistent()` → `VintfObject.verifyWithoutAvb()` 返回 1，弹出该提示。

- 该提示**只表示配置策略不符，不影响任何功能**；
- 原厂内核不弹，是因为原厂 `SYSVIPC` 本来就是关闭的；
- 同一矩阵还要求 `CONFIG_IP6_NF_NAT=n`，所以开启 IPv6 NAT 也会被判不符；
- 二者与“提示消失”不可兼得（除非改 system 分区或放弃容器能力）。

## 7. 编译

仓库内所有工作流都是 **GitHub Actions + AOSP clang 11.0.2** 的完整可复现构建，
手动 `workflow_dispatch` 触发即可：

| 工作流 | 用途 |
|---|---|
| `build-final.yml` | 生成 f1（无 macvlan / IPv6 NAT 的保守版） |
| `build-f2.yml` | 生成 f2（当前发布版：f1 + MACVLAN + IPv6 NAT） |
| `build-clang11.yml` | 工具链对齐验证（stock11 与原厂配置对照） |
| `build-bisect*.yml` | 历次 ABI 排查矩阵（保留作工程记录） |
| `release-f2.yml` | 把构建产物发布到 Releases |

关键构建参数：

```bash
make O=../out ARCH=arm64 LLVM=1 LLVM_IAS=1 \
     CROSS_COMPILE=aarch64-linux-gnu- DISABLE_WRAPPER=1 Image.gz
```

- `PATH` 指向 `clang-r383902b`；
- 配置阶段先 `vendor/anakin_defconfig`，再用 `stock.config` 覆盖并 `olddefconfig`；
- 强制 `-d LOCALVERSION_AUTO -d RELR`，并校验 `UTS_RELEASE = 5.4.210-qgki-perf`；
- 构建脚本内置硬校验：`CGROUP_PIDS` / `CGROUP_DEVICE` / `BRIDGE_NETFILTER` / `IP_SET` /
  `FAIR_GROUP_SCHED` / `CFS_BANDWIDTH` 只要有一个被开启，直接失败。

## 8. 刷入与回退

```bash
# 1) 备份（若还没有）
dd if=/dev/block/bootdevice/by-name/boot of=/sdcard/boot_apatch_backup.img

# 2) 替换内核并重打包（Termux + magiskboot）
magiskboot unpack /sdcard/boot_apatch_backup.img
cp /sdcard/Image-f2-macvlan-ipv6nat kernel
magiskboot repack /sdcard/boot_apatch_backup.img /sdcard/boot-f2.img

# 3) 先临时验证，不写分区
fastboot boot /sdcard/boot-f2.img

# 4) 确认可进系统后正式刷入（推荐再用 APatch 修补一次以保留 root）
fastboot flash boot /sdcard/boot-f2.img

# 回退
fastboot flash boot /sdcard/boot_apatch_backup.img
```

注意：换内核后若要用 APatch，需把 `Image` 交给 APatch 重新修补（superkey 沿用旧配置）。

## 9. 仓库结构

```
├── stock.config                     # 从原厂内核 IKCONFIG 提取的完整配置（构建基准）
├── droidspaces.config               # 容器所需配置清单与说明
├── 0001-rog5-sysvipc-kabi.patch     # task_struct：sysvsem / sysvshm 进 KABI 槽
├── 0002-rog5-posix-mqueue-kabi.patch# user_struct：mq_bytes 进 KABI 槽
└── .github/workflows/               # 全部构建 / 排查 / 发布工作流
```

## 10. 自检命令

```bash
su -c 'uname -r'                                                   # 5.4.210-qgki-perf
su -c 'dmesg | grep -ci "disagrees about version of symbol"'        # 0
su -c 'zcat /proc/config.gz | grep -E "^CONFIG_(SYSVIPC|IPC_NS|USER_NS|PID_NS|DEVTMPFS|MACVLAN|IP6_NF_NAT)="'
```

## 11. 致谢与声明

- 内核源码：[alfaonyt/android_kernel_asus_sm8350](https://github.com/alfaonyt/android_kernel_asus_sm8350)
- 容器方案：[DroidSpaces](https://github.com/Goldzxcbug/Droidspaces_Kernel_patch)
- KABI 机制：Android GKI `android_kabi.h`

**声明**：本仓库仅为个人设备的内核构建记录，非官方固件，不含任何 ASUS/Google 专有代码。
刷机有风险，请务必备份 boot 分区并能使用 `fastboot`。仓库未附加开源许可证，默认保留所有权利；
如需引用其中的补丁或配置，请注明出处。

> AI生成
