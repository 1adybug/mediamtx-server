# MediaMTX Docker 后端

这个项目直接使用 MediaMTX 自身作为后端服务。MediaMTX 已经内置 Control API，可以读取/修改全局配置、path 配置，并查询运行时状态；本项目把这些管理端点和主要媒体协议端口都通过 Docker 暴露出来。

参考：

- [Docker 镜像说明](https://mediamtx.org/docs/kickoff/install#docker-image)
- [Control API](https://mediamtx.org/docs/usage/control-api)
- [配置文件参考](https://mediamtx.org/docs/references/configuration-file)
- [浏览器播放](https://mediamtx.org/docs/read/web-browsers)

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

示例均使用 MediaMTX 默认端口。若本机通过 `.env` 覆盖了端口，以 `.env` 中的实际端口为准。

默认管理员：

```text
user: admin
pass: change-me
```

上线前请修改 [mediamtx.yml](./mediamtx.yml) 里的管理员密码；Control API 和 pprof 都是高权限入口，不建议直接暴露到公网。

## 暴露的入口

| 功能        | 本地地址                             |
| ----------- | ------------------------------------ |
| Control API | `http://127.0.0.1:9997`              |
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
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:9997/v3/config/global/get -Headers @{ Authorization = $auth }
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:9997/v3/paths/list -Headers @{ Authorization = $auth }
```

curl：

```bash
curl -u admin:change-me http://127.0.0.1:9997/v3/config/global/get
curl -u admin:change-me http://127.0.0.1:9997/v3/paths/list
```

## WebRTC

Docker bridge 网络下，WebRTC 客户端需要拿到可访问的主机地址。本地测试默认是：

```env
WEBRTC_ADDITIONAL_HOSTS=127.0.0.1
```

部署到服务器时，把它改成客户端访问这台机器使用的 LAN IP、公网 IP 或域名。

## RTSP 转 WebRTC

MediaMTX 不需要单独的“转换接口”。做法是先创建一个 path，让这个 path 的 `source` 指向 RTSP 地址；之后浏览器读取这个 path 的 WebRTC 页面或 WHEP 地址。

下面示例统一使用：

```text
Control API: http://127.0.0.1:9997
WebRTC: http://127.0.0.1:8889
path: cam1
```

### 判断 cam1 是否存在

最直接的方式是读取 path 配置：

```ts
const MTX_API_BASE = "http://127.0.0.1:9997"
const MTX_AUTH = "Basic " + btoa("admin:change-me")

async function getPathConfig(pathName: string) {
    const res = await fetch(`${MTX_API_BASE}/v3/config/paths/get/${encodeURIComponent(pathName)}`, {
        headers: {
            Authorization: MTX_AUTH,
        },
    })

    if (res.status === 404) {
        return null
    }

    if (!res.ok) {
        throw new Error(`读取 path 配置失败：${res.status} ${await res.text()}`)
    }

    const text = await res.text()
    return text === "" ? null : JSON.parse(text)
}

const cam1Config = await getPathConfig("cam1")
const cam1Exists = cam1Config !== null
```

也可以列出所有 path，再在前端判断：

```ts
async function listPathConfigs() {
    const res = await fetch(`${MTX_API_BASE}/v3/config/paths/list?page=0&itemsPerPage=100`, {
        headers: {
            Authorization: MTX_AUTH,
        },
    })

    if (!res.ok) {
        throw new Error(`读取 path 列表失败：${res.status} ${await res.text()}`)
    }

    return await res.json()
}

const pathList = await listPathConfigs()
const cam1Exists = pathList.items.some((item: { name: string }) => item.name === "cam1")
```

### RTSP path 配置

下面只展示 RTSP 拉流相关配置。并不是所有 value 都是默认值：`source`、`sourceOnDemand`、`rtspTransport` 是 RTSP 转 WebRTC 场景的推荐覆盖值；其余 value 是当前 [mediamtx.yml](./mediamtx.yml) 里的默认值。代码里的注释只用于阅读；实际请求会通过 `JSON.stringify` 发送合法 JSON。

```ts
function createRtspToWebrtcPathConfig(rtspUrl: string) {
    return {
        // 推荐覆盖值。默认是 publisher；这里改成摄像头或上游服务的 RTSP / RTSPS 地址。
        source: rtspUrl,

        // 默认值。RTSP / RTSPS 源证书指纹。普通 rtsp:// 明文地址留空；rtsps:// 且证书自签名或无效时才需要。
        sourceFingerprint: "",

        // 推荐覆盖值。默认是 false；这里设为 true，表示有观看者时才连接 RTSP 源。
        sourceOnDemand: true,

        // 默认值。按需拉流启动超时时间。超过这个时间还没有拉到 RTSP 流，请求会失败。
        sourceOnDemandStartTimeout: "10s",

        // 默认值。没有观看者后多久断开上游 RTSP 源。
        sourceOnDemandCloseAfter: "10s",

        // 默认值。是否把 RTSP 中的 MPEG-TS 拆成基础音视频轨道。普通 RTSP 摄像头通常保持 false。
        rtspDemuxMpegts: false,

        // 推荐覆盖值。默认是 automatic；这里设为 tcp，跨网段和摄像头场景通常更稳。
        rtspTransport: "tcp",

        // 默认值。是否允许 RTSP 源使用随机服务端端口。除非上游设备明确要求，否则保持 false。
        rtspAnyPort: false,

        // 默认值。RTSP Range 类型。普通实时流留空；回放型 RTSP 源可按需设置 clock、npt 或 smpte。
        rtspRangeType: "",

        // 默认值。RTSP Range 起始位置。普通实时流留空。
        rtspRangeStart: "",

        // 默认值。当 rtspTransport 使用 udp 或 automatic 且最终走 UDP 时，MediaMTX 可使用的本地 UDP 源端口范围。
        rtspUDPSourcePortRange: [10000, 65535],
    }
}
```

### 创建或更新 cam1

如果 `cam1` 不存在，调用 `add`；如果已经存在，调用 `patch`：

```ts
async function upsertRtspToWebrtcPath(pathName: string, rtspUrl: string) {
    const currentConfig = await getPathConfig(pathName)
    const action = currentConfig === null ? "add" : "patch"
    const method = currentConfig === null ? "POST" : "PATCH"

    const res = await fetch(`${MTX_API_BASE}/v3/config/paths/${action}/${encodeURIComponent(pathName)}`, {
        method,
        headers: {
            "Content-Type": "application/json",
            Authorization: MTX_AUTH,
        },
        body: JSON.stringify(createRtspToWebrtcPathConfig(rtspUrl)),
    })

    if (!res.ok) {
        throw new Error(`保存 path 配置失败：${res.status} ${await res.text()}`)
    }

    const text = await res.text()
    return text === "" ? null : JSON.parse(text)
}

await upsertRtspToWebrtcPath("cam1", "rtsp://user:pass@192.168.1.20:554/stream1")
```

创建成功后，可以读取运行时状态：

```ts
async function getRuntimePath(pathName: string) {
    const res = await fetch(`${MTX_API_BASE}/v3/paths/get/${encodeURIComponent(pathName)}`, {
        headers: {
            Authorization: MTX_AUTH,
        },
    })

    if (res.status === 404) {
        return null
    }

    if (!res.ok) {
        throw new Error(`读取 path 运行状态失败：${res.status} ${await res.text()}`)
    }

    return await res.json()
}
```

Control API 是管理权限接口，不建议把 `admin:change-me` 放进公开前端。生产环境建议由可信后端或内网管理页面调用 Control API，普通用户页面只访问 WebRTC 播放地址。

## reader.js 用法

如果只需要嵌入画面，优先使用 iframe，最简单：

```html
<iframe src="http://127.0.0.1:8889/cam1?controls=true&muted=true&autoplay=true&playsinline=true" scrolling="no" allow="autoplay; fullscreen"></iframe>
```

iframe 支持这些查询参数：

| 参数                      | 说明                 | 默认值  |
| ------------------------- | -------------------- | ------- |
| `controls`                | 是否显示播放器控制条 | `true`  |
| `muted`                   | 是否静音启动         | `true`  |
| `autoplay`                | 是否自动播放         | `true`  |
| `playsinline`             | 移动端是否内嵌播放   | `true`  |
| `disablepictureinpicture` | 是否禁用画中画       | `false` |

如果你需要直接控制 `<video>`、读取 `srcObject`、处理错误或传递播放鉴权，使用 `reader.js`。推荐把 MediaMTX 仓库里的 `reader.js` 放到前端静态资源目录，然后在页面里引入；临时测试也可以把脚本地址改成 `http://127.0.0.1:8889/cam1/reader.js`。

```html
<video id="myvideo" controls muted autoplay playsinline></video>
<div id="message"></div>

<script defer src="./reader.js"></script>
<script>
    let reader = null

    window.addEventListener("load", () => {
        const video = document.getElementById("myvideo")
        const message = document.getElementById("message")

        reader = new MediaMTXWebRTCReader({
            url: "http://127.0.0.1:8889/cam1/whep",

            user: "",
            pass: "",
            token: "",

            onError: err => {
                message.textContent = String(err)
            },

            onTrack: evt => {
                message.textContent = ""
                video.srcObject = evt.streams[0]
            },

            onDataChannel: evt => {
                evt.channel.binaryType = "arraybuffer"
                evt.channel.onmessage = messageEvent => {
                    console.log("收到数据通道消息", messageEvent.data)
                }
            },
        })
    })

    window.addEventListener("beforeunload", () => {
        if (reader !== null) {
            reader.close()
        }
    })
</script>
```

`MediaMTXWebRTCReader` 配置项：

| 配置            | 说明                                                                |
| --------------- | ------------------------------------------------------------------- |
| `url`           | WHEP 地址，格式是 `http://<host>:8889/<path>/whep`                  |
| `user`          | 播放鉴权用户名；没有播放鉴权时留空                                  |
| `pass`          | 播放鉴权密码；没有播放鉴权时留空                                    |
| `token`         | Bearer Token；使用 token 鉴权时填写                                 |
| `onError`       | 连接失败、流不存在、协商失败时触发                                  |
| `onTrack`       | 收到音视频轨道时触发，把 `evt.streams[0]` 赋给 video 的 `srcObject` |
| `onDataChannel` | 收到 WebRTC 数据通道时触发；不用数据通道可以省略                    |

`reader.close()` 用于关闭 WebRTC 连接，页面卸载、切换摄像头或停止播放时都应该调用。

注意：WebRTC 播放要求浏览器支持源流编码。常见 H264 通常可用；如果 RTSP 源是浏览器不支持的编码，需要先用 FFmpeg 重新编码后再交给 MediaMTX。
