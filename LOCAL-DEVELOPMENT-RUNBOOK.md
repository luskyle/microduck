# Microduck 本地开发运行手册

本文记录在没有真实机器人板子的 Linux 开发机上运行 Microduck 的方法，以及本次 WebRTC 联调中验证过的经验。

## 1. 本地运行目标

本地运行使用 fake 后端和测试视频，不连接真实电机、摄像头、NPU 或 BLE 硬件：

- `robotd --fake`：模拟机器人控制循环，仍提供 robot IPC
- `configd --fake-net --fake-pads`：模拟网络和手柄配置
- `updaterd`：使用本地配置运行更新服务
- `mediad`：输出测试图案，通过 HTTP 和 WebRTC 提供控制台

浏览器访问 `mediad` 后，可以看到测试视频、遥测数据，并通过 DataChannel 验证控制 RPC。

## 2. 环境要求

- Rust `1.89` 或更高版本
- Linux 开发机
- GStreamer `1.24.x`，且所有 GStreamer 库和插件必须来自同一套运行时
- Conda 环境：`microduck-media`
- 本地构建的 `webrtcsink` 和匹配版本的 `webrtcbin`

Ubuntu 22.04 自带的 GStreamer 通常是 `1.20.x`，低于项目媒体链路要求，也没有项目需要的 `webrtcsink`。不要把系统 GStreamer 1.20 的插件和 Conda GStreamer 1.24 的库混用，否则会出现插件加载失败、协商失败或进程崩溃。

## 3. 编译项目

```bash
cd /media/luskyle/DATA/project/microduck-all/microduck
source "$HOME/.cargo/env"
cargo build
```

常用测试：

```bash
cargo test -p duck-control fake_io
cargo test -p sounds
cargo test -p mediad
```

## 4. 启动 fake 后端

以下命令使用 `target` 下的本地状态和日志目录：

```bash
cd /media/luskyle/DATA/project/microduck-all/microduck
source "$HOME/.cargo/env"
mkdir -p target/logs

nohup target/debug/robotd \
  --fake --no-policy --socket /tmp/robotd.sock \
  > target/logs/robotd-local.log 2>&1 &

nohup target/debug/configd \
  --socket /tmp/configd.sock \
  --state-dir target/local-config-state \
  --fake-net --fake-pads --allow-user "$USER" \
  > target/logs/configd-local.log 2>&1 &
```

等待 `/tmp/robotd.sock` 创建后，启动 updater：

```bash
nohup target/debug/updaterd \
  --config target/local-updater.toml \
  --socket /tmp/updaterd.sock \
  --robot-socket /run/robotd.sock \
  > target/logs/updaterd-local.log 2>&1 &
```

### 运行时 socket

`mediad` 的默认 IPC 地址是 `/run/*.sock`，没有 CLI 参数可以覆盖这些上游 socket。因此需要建立链接：

```bash
sudo ln -s /tmp/robotd.sock /run/robotd.sock
sudo ln -s /tmp/configd.sock /run/configd.sock
sudo ln -s /tmp/updaterd.sock /run/updaterd.sock
```

如果链接已经存在，先确认它们指向正确的 `/tmp` socket，再决定是否删除并重新创建。创建 `/run` 下的链接需要管理员权限。

## 5. 启动 mediad

本地配置文件是 `target/mediad-local.toml`，它关闭真实摄像头和检测模型，使用测试图案、`640x360@30fps`：

```bash
cd /media/luskyle/DATA/project/microduck-all/microduck
source /home/luskyle/anaconda3/bin/activate microduck-media
export LD_LIBRARY_PATH="$CONDA_PREFIX/lib"
export GST_PLUGIN_PATH="$CONDA_PREFIX/lib/gstreamer-1.0"

nohup target/debug/mediad \
  --host 0.0.0.0 \
  --port 18443 \
  --web-port 18080 \
  --config target/mediad-local.toml \
  > target/mediad-local.log 2>&1 &
```

访问控制台：

```text
http://127.0.0.1:18080/
```

同一局域网其他设备使用开发机 IP，例如：

```text
http://10.151.0.201:18080/
```

## 6. 启动验证

检查进程和 socket：

```bash
pgrep -af 'target/debug/(mediad|robotd|configd|updaterd)'
test -S /run/robotd.sock && echo robotd-sock=ok
test -S /run/configd.sock && echo configd-sock=ok
test -S /run/updaterd.sock && echo updaterd-sock=ok
```

检查端口和 HTTP：

```bash
ss -ltnp | grep -E ':18080|:18443'
curl -fsS -o /dev/null -w 'HTTP=%{http_code}\n' http://127.0.0.1:18080/
```

正常结果应包括：

- `mediad` 监听 `0.0.0.0:18080`
- `mediad` 监听 `0.0.0.0:18443`
- HTTP 返回 `200`
- 日志出现 `signalling server listening`
- 日志出现 `capture rate fps`，通常接近 30 FPS

## 7. 浏览器端到端验证

打开控制台后点击 `connect`。成功状态应为：

- 页面显示 `connected`
- 页面显示 `control open`
- 视频统计出现约 30 FPS
- RTT 出现数值
- telemetry 显示 `mode`、`policy`、`loop`、`health` 等数据
- 控制台日志能看到 WebSocket、SDP offer/answer 和 DataChannel RPC

可以点击姿态、动作、声音控制验证 JSON-RPC。fake 模式只更新模拟状态，不会让真实机器人运动。

## 8. WebRTC/GStreamer 排障经验

### 8.1 版本必须统一

确认所有关键插件来自同一套 Conda 运行时：

```bash
source /home/luskyle/anaconda3/bin/activate microduck-media
export LD_LIBRARY_PATH="$CONDA_PREFIX/lib"
export GST_PLUGIN_PATH="$CONDA_PREFIX/lib/gstreamer-1.0"

gst-inspect-1.0 webrtcsink | grep -E 'Filename|Version'
gst-inspect-1.0 webrtcbin | grep -E 'Filename|Version'
gst-inspect-1.0 videoconvertscale | grep -E 'Filename|Version'
```

本次验证使用了 GStreamer `1.24.12` 的 `webrtcbin`、libnice `0.1.22`，以及同一运行时中的 `webrtcsink`。

### 8.2 `videoconvert` 找不到

Conda 的 GStreamer `1.24.11` 提供的是 `videoconvertscale`，而不是旧的独立 `videoconvert` 和 `videoscale` factory。旧版 `gst-plugin-webrtc` 的软件 fallback 如果固定创建这两个元素，会在 codec discovery 阶段失败：

```text
element factory 'videoconvert' not found
Codec discovery pipeline failed
```

本次处理是在本地 `gst-plugins-rs` 源码的 `webrtcsink` 软件 fallback 中使用一个 `videoconvertscale` 元素代替两个旧元素，然后重新构建安装 `gst-plugin-webrtc`。这个修改位于外部源码树，不属于 Microduck 仓库；换机器时必须重新准备对应的插件版本。

### 8.3 浏览器一直 connecting

按以下顺序判断：

1. 先确认 `mediad` 没有退出，且 HTTP 返回 `200`
2. 确认 `webrtcsink`、`webrtcbin` 的 GStreamer 版本一致
3. 确认 `/run/*.sock` 链接存在且目标 socket 可用
4. 确认服务绑定 `0.0.0.0`，不要只绑定 `127.0.0.1`
5. 浏览器日志中如果 ICE 已 connected 但 SCTP 仍 connecting，优先检查 GStreamer WebRTC 版本和插件，而不是先怀疑 robotd
6. 浏览器产生的 `.local` mDNS ICE candidate 在某些 GStreamer/libnice 环境下无法解析；本地 Web 客户端已过滤这类 candidate

### 8.4 常见非致命警告

以下警告不一定影响测试图案和控制通道：

- 缺少 `/run/mediad/identity.json`
- 缺少 CUDA 的 `libnvrtc.so`
- FLAC 插件缺少 `libFLAC.so.12`
- `rtpgccbwe` 不存在

应以服务是否持续运行、视频 FPS、DataChannel 状态和 RPC 响应为准。

## 9. 停止本地服务

```bash
pkill -f 'target/debug/mediad' || true
pkill -f 'target/debug/updaterd' || true
pkill -f 'target/debug/configd' || true
pkill -f 'target/debug/robotd' || true
```

停止后如果要清理运行时链接：

```bash
sudo rm -f /run/robotd.sock /run/configd.sock /run/updaterd.sock
```

## 10. 与真实板子的边界

本手册只适合本地协议、控制逻辑、媒体链路和界面验证。它不能替代真实板子的：

- 电机总线和舵机测试
- IMU、摄像头、ToF、NPU 测试
- 电源和热管理测试
- BLE 无线链路测试
- systemd 启动、升级、回滚和救援测试

真实板子部署应使用项目 `scripts/` 中的板级安装和推送流程，并遵守运动安全要求。
