# eac-arm64-emulation-poc

**一个概念验证：在 arm64 Linux 机器上通过 FEX-Emu 运行带正版 Easy Anti-Cheat 的 VRChat —— 以及为此所付出的一切。**

> **提醒：此处有龙，请留意相关风险。** 我本人尚未观察到任何问题，但这门做法是在开发者并不支持的配置（模拟的 x86
> CPU，且常常位于虚拟机内）上运行受反作弊保护的游戏。反作弊系统或游戏的用户协议可能将其视为违规，账号处罚是有可能的；
> 请使用你能承受失去的账号。脚本会替换 Steam 的 FEX 工具中的文件（`restore` 会把 Valve 原版放回去），本文所述内容
> 未获 Valve、Epic Games、FEX-Emu 项目或任何游戏开发商背书。不提供任何担保；见[许可协议](#许可协议)。

这是一个独立仓库。它记录了一项调查过程，以及对 **FEX-Emu**（2610；在 2609.1 上开发）所做的修改——这些修改是在 arm64
主机上，把一个 x86-64 游戏连同 Epic 为 Linux/Proton 提供的 Easy Anti-Cheat 跑起来所必需的：所有修改都以补丁文件的形式
保留，你可以自行应用，并附带展示每个补丁修复了什么问题的测试，以及定位这些问题的诊断工具。

![在 Apple M2 上、于 Fedora Asahi Remix 下通过打过补丁的 FEX-Emu 运行的 VRChat，旁边是主机的 `fastfetch`](docs/img/vrchat-m2.png)

它刻意限定为**一个游戏、一颗 SoC**（Apple M2 上的 VRChat：在 [muvm](https://github.com/AsahiLinux/muvm) 微虚拟机
内的 Fedora Asahi Remix 下开发，另在 macOS 上验证过，见下文）。请把它当作起点并继续推进——换一个游戏、换一颗 SoC、
换一台主机——但发现问题时请在这里反馈，而不是只在自己的 fork 里修掉；见[Bugs、修复与上游贡献](#bugs修复与上游贡献)。

**喜欢它？来帮它一把。** 给仓库点个星、出问题时开个 issue（附上你的硬件和日志）、以及 pull request 都很欢迎。
如果你维护的 fork 里有修复或改进，也请在这里开一个 pull request：它会得到审查、记入你的贡献，并交付给所有人。

**macOS。** 同一份打过补丁的 FEX 也能在 macOS 27 下的 Apple M2 MacBook Air 上、于
[steamac](https://github.com/fxgl/steamac) 的 libkrun 客户机内运行带正版 EAC 的 VRChat（客户机是 Valve 的 ARM64
SteamOS，Steam 自带的 FEX 兼容工具被替换为打过补丁的版本）。结果、耗时与限制见：[docs/macos-port.md](docs/macos-port.md)。
安装步骤与下游修复（包括 Direct3D 11 的 DXMT 图形路径）位于单独的 fork
[StonkGuy/steamac](https://github.com/StonkGuy/steamac)（`main` 及其 release 供玩家使用，`fex-eac` 用于开发）。

**Steam Frame。** 你是从一篇关于 Steam Frame 的帖子找过来的？请先看 [docs/steam-frame.md](docs/steam-frame.md)：拿到
shell，在 Frame 上用 [`scripts/steamos/`](scripts/steamos) 构建打过补丁的 FEX，把它换进 Steam 的 FEX 工具，设置启动
选项。已由另一位用户报告可用；此处未做测试。

真正的难点不在 CPU 速度，而在**保真度**。反作弊启动器用 `ptrace` 把自己的客户端注入游戏，模拟器必须在 `ptrace`、信号
和 `/proc` 方面表现得像真正的 Linux 内核——此前没有任何东西对 FEX 提出过这样的要求；后来，VRChat 自己的网络线程又暴露出
两处 FEX 信号处理与 Linux 不一致的地方。

## 需要什么

| 你需要 | 原因 |
|---|---|
| **带补丁的 FEX-Emu 2610：[StonkGuy/FEX](https://github.com/StonkGuy/FEX/tree/eac-2610) 标签 `eac-2610.5`** | 见 [docs/patches.md](docs/patches.md)；用 `scripts/build-fex.sh` 构建 |
| **一台 arm64 Linux 主机**（已测试：Apple M2 + Fedora Asahi Remix + muvm，或 macOS 27 + steamac 的 libkrun 客户机） | 16 KB 页的主机需要 4 KB 页的客户机，因此要用 muvm 虚拟机；在 4 KB 页主机上 FEX 可直接运行（未测试） |
| **Proton Experimental 11.0** 与 *Proton EasyAntiCheat Runtime*，且 Steam 与 FEX 位于 VM 边界的同一侧 | 本文全部数据均在这些版本上测得 |
| 主机内存调优（`scripts/host-tune.sh`，Asahi，16 GB） | 16 GB 主机在约 8 GB 虚拟机加 GPU 缓冲区时就会内存不足；zswap/zram/watermark 设置可避免 OOM killer 介入 |
| `scripts/` 中的启动选项与 DXVK 配置项 | `scripts/set-launch-options.sh`、`scripts/dxvk.conf` |

补丁系列分为**修复**与**诊断**两类——补丁 0001（`ptrace`）、0002（SMC 热页）和 0007（信号掩码）是运行所必需的；
0008 与 0010 是同一领域的正确性修复，其余则是性能、内核保真度或工具。[docs/patches.md](docs/patches.md) 解释了每个补丁
及其推理：

| 补丁 | 类别 | 用途 |
|---|---|---|
| 0001 `ptrace` 模拟 | 修复 | EAC 启动器用 `ptrace` 注入其客户端；FEX 需要模拟被跟踪进程的 x86-64 视图。缺少它启动器会失败（`Unexpected error. (#1)`） |
| 0007 信号掩码 | 修复 | 客户机信号处理器以 Linux 的信号掩码运行。缺少它，几乎每隔一次会话就会出现 Photon 超时 |
| 0008 系统调用信息 | 修复 | 从系统调用进入的处理器会保留其设置的寄存器 |
| 0010 seccomp trap | 修复 | `SECCOMP_RET_TRAP` 的 `SIGSYS` 报告的是系统调用之后的那条指令，与 Linux 一致；缺少它，Wine 的系统调用捕获过滤器会无限重复执行被捕获的系统调用 |
| 0004 廉价失效 | 性能 | 在约 200 个线程下，数据页上的 `mmap`/`mprotect` 便宜约 22 倍；缺少它，加入要 60 秒以上 |
| 0002 SMC 热页 | 修复 | 反复触发自修改代码缺页的页不再被写保护。缺少它反作弊客户端永远加载不完，游戏无法启动 |
| 0003/0005/0006/0009 | **诊断** | 统计、采样器与线程快照（0009 把 futex word 加入快照）——是工具，不是修复 |
| 0011–0020 内核保真度 | 保真 | 真实内核在 `arch_prctl`（0011）、调试寄存器（0012）、`/proc/<pid>/status`（0013）、未知 regset（0014）、`restart_syscall`（0015）和信号帧（0016–0020）上给出的答案。不在 VRChat 的路径上；某个游戏或 Wine 会观察到它们 |
| 0022–0026 第二波 | 保真 | 每次 seccomp trap 泄漏的主机栈（0022）、客户机读取 FEX 的 call/ret 影子栈保护页（0023）、IRETQ 目标上的 TF 陷入（0025）、线程在该栈上退出时的信号备用栈（0026）；0024 是一个 profiler 选项。在另一款游戏（THE FINALS）上发现，其防篡改引导程序会观察到这些；仅涉及陷入、缺页与退出路径 |
| 0033 `/proc/<pid>/maps` | 修复 | maps 文件只列出被模拟进程的映射（低于 2^47），不含 FEX 及其 AArch64 主机库。缺少它，EAC 启动器中会从子进程 maps 里挑一个库注入的模块会挑到主机库，并以 `210 Unexpected error. (#1)` 失败（在 Mini Royale 经 Proton 运行时观察到；VRChat 的部署仍在使用不受影响的旧模块） |
| 0035 SMC、远程写入 | 修复 | 客户机设为可写的代码页（`mprotect` 从 RX 改为 RWX）对其他进程也保持可写，因此用 `NtWriteVirtualMemory` 写入的 hook 不再以 `STATUS_PARTIAL_COPY` 失败。基于 yungtry 的补丁 0028；使用方为 The Division 2（据其作者）；仅在 `SMCChecks=mtrack` 时生效 |
| 0036–0039、0041–0053（eac-2610.3） | 修复 / 保真 | Wine 11 使用 seccomp 与 Syscall User Dispatch 陷入路径的方式（被捕获的系统调用从不执行、SUD 被模拟、不丢失不泄漏陷入帧、EFLAGS 第 31 位）、seccomp 下的 `execve`/`execveat`、`rt_sigreturn` 之后的 `InSyscallInfo`、错误指针返回 `-EFAULT`、内核写入代码页、跨 4 GiB 的 `munmap`、对跟踪器隐藏的 SIGBUS 停止（The Division 2 的 EAC 启动器）。见 [docs/patches.md](docs/patches.md#h-fixes-since-eac-26102--patches-00360039-00410053) |
| 0054–0070（eac-2610.4） | 修复 / 保真 | THE FINALS 的防篡改代码探测的异常（带 NT 的 IRETQ、写入 FS/GS 的 MOV、超长指令、XSETBV、取指缺页地址）、编译时以 fault-safe 方式读取客户机代码（THE FINALS 对局内卡死）、该 fork 自己的磁盘缓存格式版本、上游的 `0F B8/BC/BD` 解码修复，以及仅对可能触及正在运行代码块的存储做 SMC 重启（Hell Let Loose 的 EAC `210 (#2)`）。见 [docs/patches.md](docs/patches.md#i-fixes-since-eac-26103--patches-00540070) |
| 0071（eac-2610.5） | 修复 | Proton Experimental 11.0-20261027b 在 Syscall User Dispatch 下重试 EAC 的本机系统调用；信号返回到被捕获的 `syscall` 时恢复了过期的 FEX 状态，导致 EAC 游戏几秒内就退出 |

## 哪些地方改了什么

涉及三类不同的改动。只有第一类是新增代码；Wine 和 Proton 中没有任何补丁。

| 层 | 内容 | 类型 | 位置 |
|---|---|---|---|
| **FEX-Emu（模拟器）** | 0001 `ptrace` 模拟、0007/0008/0010 信号与 seccomp 保真、0004 失效、0002 SMC 热页、0011–0020 内核保真度缺口（调试寄存器、`/proc`、`arch_prctl`、regset、`restart_syscall`、信号帧）、0022/0023/0025/0026 主机栈、缺页与单步保真、0033 `/proc/<pid>/maps`、0035 SMC 远程写入、0036–0039/0041–0053 陷入、exec 与系统调用修复、0024 profiler 选项、0003/0005/0006/0009 诊断 | **新增源代码**，在 FEX-2610 上每个补丁一个 commit | [StonkGuy/FEX](https://github.com/StonkGuy/FEX/tree/eac-2610) 分支 `eac-2610`（测试在 `eac-tests/`），由 `scripts/build-fex.sh` 构建；`patches/` 是生成的快照 |
| **Proton / Wine** | `WINE_CPU_TOPOLOGY=16:…`（上报 16 个 CPU；修复加入前的卡顿） | **原生 Proton 设置**，无补丁 | 启动选项，`scripts/set-launch-options.sh` |
| | `EAC_LAUNCHERDIR`、`PROTON_EAC_RUNTIME`（加载 EAC 运行时）、`WINEDEBUG=-all`、`PROTON_USE_XALIA=0` | 原生 Proton 设置 | 启动选项 |
| | `DXVK_CONFIG_FILE` → `dxgi.maxDeviceMemory`/`maxSharedMemory` = 3072 | 原生 DXVK 设置 | `scripts/dxvk.conf` |
| **虚拟机与主机** | 客户机网卡 MTU 限制为 1500；客户机 RAM ≥ 8 GB；muvm 的 4 KB 页客户机 | 环境 | `scripts/vm/steam-vm.sh`、`scripts/vm/guest-setup.sh` |
| | 16 GB 主机上的 zswap/zram/watermark/预读调优 | 环境 | `scripts/host-tune.sh` |
| | 可选的 SMBIOS/DMI/PCI/主机名拟真层 | 环境，可选 | `scripts/vm/realism.sh` |

## 其中经历了什么

1. **EAC 启动器**需要其被跟踪进程的 x86-64 `ptrace` 视图、客户机二进制的 `/proc/<pid>/exe`，以及客户机 `int3` 的
   信号投递停顿；整个交互过程由一个独立测试（`eac-tests/ptrace-inject`，[docs/how-it-works.md](docs/how-it-works.md)）复现。
2. **加入要花一分钟**，原因是约 200 线程的进程中代码失效的开销（补丁 0004，用补丁 0003 的统计发现）。
3. **整台虚拟机被主机 OOM 杀死**：内存调优与虚拟机规格（[docs/tuning.md](docs/tuning.md)）。
4. **Photon 超时**（大约每隔一次会话）是 Wine 等待协议中的唤醒丢失：FEX 用错误的信号掩码运行客户机信号处理器
   （补丁 0007）。定位它需要一个不对反作弊执行 `ptrace` 的线程快照设施、一个独立的 Wine 复现程序，以及查看
   wineserver 的 epoll 注册（[docs/disconnects.md](docs/disconnects.md)）。测试 0007 时又发现了同族的第二个 FEX 缺陷
   （信号处理器内寄存器丢失，*补丁 0008*）。
5. 某些启动时**约 60 秒的卡死**（Photon 区域查询超时，随后进入 `Disconnecting` 重连循环）是游戏的 IL2CPP 线程池在
   4 vCPU 虚拟机上被饿死。不是 FEX 或 Wine 的缺陷：用 Proton 的 `WINE_CPU_TOPOLOGY` 上报 16 个 CPU 即可消除。
   通过进程内线程快照和 wineserver socket 状态的被动读取器发现（[docs/disconnects.md](docs/disconnects.md)）。

## 快速开始（已验证平台：Fedora Asahi Remix）

在 Steam Frame（或其他 SteamOS arm64 设备）上，请改为参考 [docs/steam-frame.md](docs/steam-frame.md)。

```sh
sudo dnf install steam muvm fex-emu && steam          # 基础安装、登录、安装 VRChat、启动一次
scripts/build-fex.sh                                  # 克隆并构建 StonkGuy/FEX 标签 eac-2610.5 到 work/FEX
scripts/vm/install-overlay.sh
(cd work/FEX/eac-tests/ptrace-inject && ./build.sh clang)
scripts/vm/run-test-in-vm.sh work/FEX/eac-tests/ptrace-inject/ptrace_inject_test   # 必须打印 RESULT: PASS
scripts/vm/run-test-in-vm.sh <其他 eac-tests 二进制>  # 例如 signal-mask、signal-regs：RESULT: PASS
sudo scripts/host-tune.sh                             # 内存调优（可选，16 GB 主机上强烈推荐）
scripts/set-launch-options.sh                         # 需先关闭 Steam
REALISM=1 VM_MEM_MB=10240 VM_VRAM_MB=3072 scripts/vm/steam-vm.sh   # 在虚拟机中启动 Steam（已验证配置；REALISM=1 可选），然后启动 VRChat
```

完整的 A 到 Z 指南、确切的已验证版本，以及什么会弄坏可用的配置：[docs/setup-asahi.md](docs/setup-asahi.md)。工具：
[docs/tools.md](docs/tools.md)。哪些能用、哪些仍未解决：[docs/status.md](docs/status.md)。

## 范围与限制

* **这不是反作弊绕过。** 任何东西都不伪造、重放或短路 EAC 的判定结果；不包含任何 Epic 或 VRChat 的二进制、密钥或
  追踪记录。补丁只是让 FEX 在 `ptrace`、信号和 `/proc/<pid>/exe` 方面表现得像真正的 Linux 内核，而运行起来的 EAC
  客户端就是 Valve 的 Proton EAC Runtime 中未经修改的原版。
* **一处需要直说的保留。** 可选的 `scripts/vm/realism.sh` **不会**触碰 EAC，但它也不是纯粹模拟保真：它会向客户机
  呈现看似合理的 SMBIOS/DMI 数据，使其不那么像微虚拟机（并且，比 VRChat 的指南更进一步，还有 PCI 设备列表与主机名）。
  那属于*环境适配*，遵循 VRChat 自己发布的指南《Using VRChat in a Virtual Machine》，该指南说明 EAC 的虚拟机封禁来自其
  CPUID hypervisor-vendor 检查，并且**"You can get virtualization working alongside EAC in some cases, and we don't mind
  if you do this."** 请将其视为政策声明而非许可：属于"被容忍但不被支持"，且本仓库将该脚本标记为**可选**：不运行它
  VRChat 也能跑。对于反作弊会检查机器硬件身份的其他游戏，它可能有用。它不会修改游戏或反作弊。
* **不受 VRChat 支持。** VRChat 不支持虚拟机或模拟；Epic/VRChat 可随时更改任何内容，封禁与否由他们决定。风险自负，
  绝不要运行被修改的客户端。绝不要对运行中的游戏附加调试器或 `perf`：EAC 会报告。

## Bugs、修复与上游贡献

**发现了缺陷，或者发现打过补丁的 FEX 仍与 x86-64 Linux 行为不同？** 请在这里以 issue 形式反馈。如果你已在自己的
fork 中修好，请考虑同时在这里开一个 pull request（附带测试），这样所有人都能得到该修复，测试套件也会覆盖它。
FEX 代码改动请以 pull request 形式提交到 [StonkGuy/FEX](https://github.com/StonkGuy/FEX)，针对当前的 `eac-<release>`
分支，并在其 `eac-tests/` 中附上测试。

本仓库不向 FEX-Emu 或 Proton 本身发送补丁，也与它们无从属关系（steamac 的改动直接与其维护者讨论）。如果你审查了某个
补丁并认为它足以进入上游，**欢迎你自行提交**：连同其测试一起，遵循上游项目自己的流程，并注明来源。对其他被修改的组件
（libkrun、Mesa/KosmicKrisp、virglrenderer、DXMT）同样适用，除非该项目自己的规则另有规定——例如 DXMT 不接受 AI 辅助
的贡献。

## 维护与支持

我会尽量让被修改的组件跟上上游发布，每次更新都重跑回归测试，并在上游覆盖某补丁的行为后移除该补丁。对于特定游戏的问题，
我会尽量在能力范围内提供帮助，但无法保证每个游戏、每个修复都能生效。

每个 FEX 版本都以同样方式处理：把补丁变基到该版本，作为 StonkGuy/FEX 的 `eac-<release>` 分支，在客户机中以及在本机
（作为对照基准）运行 `eac-tests`，检查一次实际的 VRChat 会话，移除上游已提供其行为的每个补丁，然后打标签；本仓库与
steamac fork 随后固定到新标签。

## 其他 EAC 游戏与 FEX 主机

Proton EAC 运行时对所有随 Linux EAC 发布的游戏都是同一套启动器/客户端机制，所以 `ptrace` 相关工作并非 VRChat 专属；
但每个游戏仍有自己的 EAC 构建、完整性检查和服务器端策略。VRChat 已端到端验证；Mini Royale 的 EAC 启动器在 0033 下
也能到达 `301`（游戏本身未越过账号提示进行游玩）。结果因游戏而异：每款作品都自带 EAC 构建与检查，因此未测试的游戏可能
能用也可能不能用。除上文 Asahi 与 macOS 客户机外，其他 FEX 主机均未测试，例外是 Steam Frame（来自另一位用户
@VirtuallyNoxide 的发现；安装见 [docs/steam-frame.md](docs/steam-frame.md)）——见
[docs/other-platforms.md](docs/other-platforms.md)。

## 许可协议

MIT（见 [LICENSE](LICENSE)）；这些补丁修改的 FEX-Emu 同样为 MIT。
