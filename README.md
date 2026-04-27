# MediaMTX Docker 后端

这个项目直接使用 MediaMTX 自身作为后端服务。MediaMTX 已经内置 Control API，可以读取/修改全局配置、path 配置，并查询运行时状态；本项目把这些管理端点和主要媒体协议端口都通过 Docker 暴露出来。

参考：

- [Docker 镜像说明](https://mediamtx.org/docs/kickoff/install#docker-image)
- [Control API](https://mediamtx.org/docs/usage/control-api)
- [配置文件参考](https://mediamtx.org/docs/references/configuration-file)

## 镜像选择

默认使用：

```bash
bluenviron/mediamtx:1-ffmpeg
```

它包含 MediaMTX 的常规能力和 FFmpeg，适合 x86_64/Windows/macOS/Linux Docker 环境。

严格意义上“功能最全”的镜像是：

```bash
bluenviron/mediamtx:1-ffmpeg-rpi
```

它额外包含 Raspberry Pi Camera 支持，但 Docker manifest 只有 Linux ARM/ARM64，不能作为普通 x86_64 主机的默认镜像。树莓派部署时使用：

```bash
docker compose -f compose.yml -f compose.rpi.yml up -d
```

## 启动

```bash
docker compose up -d
```

当前机器上已有 `mediamtx-gateway` 占用 `9997`，所以本地 `.env` 已把 API 映射到 `19997`。如果你的环境没有端口冲突，可以删除 `.env` 或把 `API_PORT` 改回 `9997`。

默认管理员：

```text
user: admin
pass: change-me
```

上线前请修改 [mediamtx.yml](./mediamtx.yml) 里的管理员密码；Control API 和 pprof 都是高权限入口，不建议直接暴露到公网。

## 暴露的入口

| 功能        | 本地地址                             |
| ----------- | ------------------------------------ |
| Control API | `http://127.0.0.1:19997`             |
| Metrics     | `http://127.0.0.1:9998/metrics`      |
| pprof       | `http://127.0.0.1:9999/debug/pprof/` |
| Playback    | `http://127.0.0.1:9996`              |
| RTSP        | `rtsp://127.0.0.1:8554/<path>`       |
| RTMP        | `rtmp://127.0.0.1:1935/<path>`       |
| HLS         | `http://127.0.0.1:8888/<path>/`      |
| WebRTC      | `http://127.0.0.1:8889/<path>/`      |
| SRT         | `srt://127.0.0.1:8890`               |

如果修改 `.env` 里的端口，以 `.env` 为准。

## API 测试

PowerShell：

```powershell
$auth = 'Basic ' + [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes('admin:change-me'))
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:19997/v3/config/global/get -Headers @{ Authorization = $auth }
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:19997/v3/paths/list -Headers @{ Authorization = $auth }
```

curl：

```bash
curl -u admin:change-me http://127.0.0.1:19997/v3/config/global/get
curl -u admin:change-me http://127.0.0.1:19997/v3/paths/list
```

## WebRTC

Docker bridge 网络下，WebRTC 客户端需要拿到可访问的主机地址。本地测试默认是：

```env
WEBRTC_ADDITIONAL_HOSTS=127.0.0.1
```

部署到服务器时，把它改成客户端访问这台机器使用的 LAN IP、公网 IP 或域名。
