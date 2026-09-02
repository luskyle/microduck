# Microduck 项目整体理解

## 1. 项目是什么

这个仓库是 Microduck 的“大脑”代码库。它不是一个单体程序，而是一个 Rust workspace，里面拆成了多个独立的 daemon、库和工具，每个部分各司其职，构成一套完整的机器人控制与更新系统。

项目主入口在 [README.md](../README.md)，也说明了它的设计目标：

- 这是一个约 25 cm、800 g 的双足机器人
- 依靠强化学习策略控制运动
- 每个功能模块分别运行在独立的服务里
- 升级过程需要安全、可回滚、健康检查

它不是“一个程序里塞满所有功能”，而是一套分层架构：

- 控制循环负责运动
- 更新系统负责安全升级
- 配置系统负责网络和身份
- 蓝牙系统负责远程通信
- 媒体系统负责相机和 WebRTC
- CLI 工具负责控制和排障

---

## 2. 整体架构

从代码和文档看，这个项目最核心的结构是：

- `robotd`：控制 daemon，50 Hz 控制循环，真正驱动机器人
- `configd`：网络、配置、身份、配对等
- `updater`：更新引擎 + `updaterd`，负责安装、验证、回滚
- `btd`：蓝牙入口，用于手机或本地客户端访问机器人
- `padd`：手柄输入，转成机器人意图
- `mediad`：摄像头和 WebRTC 流媒体
- `tof`：深度传感器 `tofd`
- `duck-control`：控制核心，模型、bus、IMU、策略、安全层等
- `duck-ipc-proto`：统一的 IPC 协议定义
- `robotctl` / `duckctl`：控制和调试 CLI

这几个模块之间通过 Unix socket 和 JSON-RPC 约定通信，而不是直接调用彼此内部对象。

换句话说：

- 代码中“模块分离”很明显
- 运行时“进程分离”也很明显
- 这是为了提高安全性、可恢复性和避免单体故障把整个机器人弄挂

---

## 3. 为什么是多个 daemon，而不是一个大程序

这个项目把每个职责拆成单独的进程，根本原因是机器人系统里很多事情都需要“容错”和“可恢复性”。

例如：

- `robotd` 运行控制循环，负责底层运动和安全
- `configd` 可以在 `robotd` 死掉时仍然可用
- `btd` 和 `updaterd` 要在 `robotd` 停止时仍能工作
- 更新中断时，系统要能回滚，而不是直接变砖

这就是为什么架构中把更新、网络、蓝牙等拆到独立进程中，而不是同一套代码里强耦合。

---

## 4. 关键代码入口

### 4.1 workspace 入口

[Cargo.toml](../../Cargo.toml) 定义了整个 workspace。

它说明：

- 这是多个 crate 的集合
- 各个 daemon 是兄弟 crate，而不是一个大型单项目
- 默认成员和 workspace 成员管理了哪些二进制/库会被编译

### 4.2 README 入口

[README.md](../../README.md) 解释了项目的目标和“它做什么”。

它同时告诉你：

- 这个项目牵涉 robot 软硬件控制
- 策略来自外部的 `microduck_rl`
- 运行时是 ONNX 策略 + 机器人控制 loop
- 系统包含更新、控制、安全、相机、蓝牙等功能

### 4.3 代码说明和设计文档

项目的 `docs/design/` 和 `docs/project/` 非常重要，基本就是“为什么这样设计”的来源。重点文档包括：

- `docs/design/architecture.md`
- `docs/design/robotd-design.md`
- `docs/design/updater-design.md`
- `docs/design/boot-recovery-net.md`
- `docs/robot/dev-push.md`

这些文档比代码更能解释项目的真实设计意图。

---

## 5. 运行方式：本地开发 vs 真实板子部署

这个项目最容易误解的一点是：它并不是在开发机器上直接“跑一个机器人”。

它实际上有两套思路：

### 5.1 本地开发

本地开发的目标通常是：

- 在没有真实板子的情况下验证逻辑
- 运行控制回路的 fake 模式
- 验证 socket、IPC 协议、更新逻辑
- 跑单测、模拟状态、检查错误路径

本地最重要的入口是：

- [duck-control/src/io.rs](../../duck-control/src/io.rs)
- [robotd/src/main.rs](../../robotd/src/main.rs)

其中 `RobotIo` 的 `FakeIo` 就是无硬件模拟对象，用于测试控制逻辑，而不是依赖真实舵机。

`robotd` 本身支持 `--fake` 参数，表示“没真实总线、没真实机器人，但仍然启动控制进程”，这就是本地开发最直接的运行方式。

### 5.2 真正板子部署

如果要真正跑在机器人硬件上，就要走发布和安装流程：

- 使用 `scripts/dev-push.sh` 从本地编译并安装到板子
- 或者执行完整的 `setup-board.sh` / `install.sh` 系统初始化脚本

这是一套“板级部署”思路，不是本地直接运行。

---

## 6. `scripts` 目录的作用

这是整个项目最关键的“运维和部署层”，不是随便写的一堆 shell。它负责板子准备、环境配置、更新部署和救援逻辑。

### 6.1 本地开发脚本

- `dev-push.sh`：最重要的“本地构建并部署到板子”的一键脚本。
- `board-test.sh`：验证安装后的板子行为是否正确。
- `systemd-test.sh`：验证 systemd 相关的更新和重启逻辑。
- `cross-sysroot.sh`：交叉编译支持。

这些脚本让开发者在本地可以以“正常发布流程”的方式测试代码，而不是只跑单测。

### 6.2 板子初始化脚本

- `setup-board.sh`：系统初始化，设置蓝牙、串口、内核参数、基础依赖等
- `migrate-network.sh`：扩展网络组网迁移，确保网络由 NetworkManager 接管
- `install.sh`：安装 release artifact 和 service units
- `provision-board.sh`：组织板子第一次准备流程
- `provision.sh`：更高层 orchestration

这些脚本通常在板子第一次刷好系统后执行。

### 6.3 运行时与诊断脚本

- `robot-boot-check`：开机检查，确认 release 是否成功起起来
- `robot-rescue`：恢复脚本，回退到安全版本
- `pad-link-test.sh`：测试 pad 和无线链路
- `pad-stack-report.sh`：诊断 pad 连接状态和链路问题

这类脚本对应的是“系统在出问题时怎样自救和排查”的设计。

### 6.4 环境依赖脚本

- `setup-gstreamer.sh`：安装 `mediad` 所需 GStreamer
- `setup-npu.sh`：安装 NPU 运行时
- `setup-rkaiq.sh`：图像处理与 3A 算法相关依赖
- `setup-login.sh`：登录设置/用户环境特定配置

这些脚本说明这个项目不是“只写代码”，而是要把板子的运行环境也一起准备好。

---

## 7. 这个项目最重要的“真实价值”

如果只看代码，容易觉得它“只是一个机器人控制器”，但它的重点其实在以下几个地方：

### 7.1 安全升级

它不允许简单地替换二进制并重启。更新必须：

- 验证签名
- 校验 artifact
- 检查兼容性
- 执行 pre/post install hooks
- 验证 daemon 是否健康
- 若失败则回滚

这是最重要的工程设计之一。

### 7.2 健康门禁（health gate）

`updaterd` 不是简单装个新版本，它会检查 robotd 是否健康。如果不健康，就回滚，避免带来危险状态。

这点是整个项目“工程设计”的关键，而不是普通脚本式部署。

### 7.3 模拟测试优先

项目中大量逻辑都能在 `FakeIo`/`real process`/`golden update` 路径下测试，而不用依赖真实硬件。也就是说，这个仓库很重视“在无硬件条件下验证真实行为”。

### 7.4 运行时恢复

`robot-rescue`、`robot-boot-check`、回滚机制说明它并不是把机器人当成“能一直稳定运行”的对象，而是做成“失败后能恢复”的系统。

---

## 8. 本地最直接的运行方式（不部署到板子）

最适合本地调试的方式是：

```bash
cd /media/luskyle/DATA/project/microduck-all/microduck
source $HOME/.cargo/env
cargo run -p robotd -- --fake --socket /tmp/robotd.sock
```

这条命令在本地启动 fake 模式的 `robotd`。

它的输出会显示类似：

- `serving robot IPC path=/tmp/robotd.sock`
- `--fake: no bus, no robot`
- `control loop running joints=15 hz=50.0 driving=false`

这说明：

- 进程成功启动
- IPC 套接字正常工作
- 50 Hz loop 正在运行
- 没有真实硬件依赖

随后可以用 `robotctl` 看状态：

```bash
cargo run -p robotctl -- --robot-socket /tmp/robotd.sock version
```

或者：

```bash
cargo run -p robotctl -- --robot-socket /tmp/robotd.sock health
```

这就是最直观地“看到项目运行效果”的方式。

---

## 9. 结论

这个项目的核心理解可以概括成一句话：

它不是单个“机器人控制程序”，而是一个“工程化的机器人系统”，包括控制、通信、更新、安全恢复、设备初始化和运维脚本的完整闭环。

从工程视角看，它最厉害的地方在于：

- 代码模块明确
- 进程职责明确
- 更新链路安全
- 故障恢复设计完整
- 本地 fake 测试与真实板子部署都被设计进来了

这是一套非常典型的“嵌入式/机器人系统工程”布局，而不是“随手写一个 demo”。

---

## 10. 建议阅读顺序

如果你想快速入门，建议按这个顺序：

1. [README.md](../../README.md)
2. [CONTRIBUTING.md](../../CONTRIBUTING.md)
3. [docs/design/architecture.md](../design/architecture.md)
4. [docs/design/robotd-design.md](../design/robotd-design.md)
5. [docs/robot/dev-push.md](../robot/dev-push.md)
6. [robotd/src/main.rs](../../robotd/src/main.rs)
7. [duck-control/src/io.rs](../../duck-control/src/io.rs)
8. [scripts/dev-push.sh](../../scripts/dev-push.sh)

这样可以先理解“为什么这么设计”，再看“代码怎么实现”。
