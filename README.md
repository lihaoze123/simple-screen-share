# 局域网屏幕共享

运行服务的电脑安装 Docker + Docker Compose（Docker Desktop 使用 Linux 容器）；发送端和接收端使用 Chrome/Chromium/Edge 浏览器。Caddy 提供本机 HTTP 发布页和发布接口代理，MediaMTX 自己处理 MoQ 的 HTTPS 与视频传输。

## 启动

先找到运行服务的电脑上、发送端和接收端均能访问的局域网 IPv4 地址。Windows 可运行 `ipconfig`，Linux 可运行 `hostname -I` 或 `ip -4 addr`，macOS 可在系统网络设置中查看。假设是 `192.168.1.10`，在项目目录中：

```bash
cp .env.example .env
```

编辑 `.env`，将 `LAN_HOST` 改成实际地址（只填 IP，不带协议、端口或路径）。这个值用于发布页显示接收地址，以及 MediaMTX 的 WebRTC 连接地址：

```dotenv
LAN_HOST=192.168.1.10
```

启动服务；`--remove-orphans` 会移除旧配置中的 nginx 发布页容器：

```bash
docker compose up -d --remove-orphans
```

## 使用

### 网页发送端

在运行服务的本机浏览器访问 **http://localhost:8090**，点击 **开始共享屏幕**，选择整个屏幕。`localhost` 的 HTTP 页面可以采集屏幕；发布页端口只映射到宿主机的 `127.0.0.1:8090`，不要使用局域网 IP 打开发布页。

发布页通过同源 `/screen/whip` 请求 Caddy，再由 Caddy 转发到 MediaMTX 的 WebRTC 接口，无需配置发布页证书。

### 使用 OBS 推流（替代网页发送端）

使用支持 **WHIP** 的 OBS Studio，无需修改当前 Compose 配置。OBS 通过 `8889` 的 WHIP 接口发布到 `/screen`，接收端仍通过 `8892` 的 MoQ 页面观看。

1. 先停止网页中的屏幕共享，避免网页和 OBS 同时向 `/screen` 发布。
2. 在 OBS「来源」中添加屏幕采集源；Linux Wayland 使用「屏幕采集（PipeWire）」，选择要共享的屏幕。
3. 打开「设置 → 推流」，将「服务」选择为 **WHIP**，按 OBS 所在电脑填写「服务器」：

   | OBS 所在电脑 | 服务器地址 |
   | --- | --- |
   | 运行 Docker Compose 的本机 | `http://localhost:8889/screen/whip` |
   | 局域网中的其他电脑 | `http://192.168.1.10:8889/screen/whip` |

   「Bearer Token」留空，当前服务未启用身份验证。远程推流时将示例 IP 替换成服务电脑的实际局域网 IP，并确保该电脑能访问服务电脑的 `8889/TCP` 和 `8189/UDP`。
4. 建议视频使用 **H.264**、1080p、30 或 60fps，音频使用 **Opus**；若编码器可设置 B 帧，为兼容 WebRTC 接收将其设为 `0`。
5. 保存设置，点击 OBS 的「开始推流」。接收端浏览器打开 **https://192.168.1.10:8892/screen**，按下文处理首次访问的证书提示即可观看；WebRTC 接收地址仍为 **http://192.168.1.10:8889/screen**。

OBS 的服务器地址必须包含 `/screen/whip`。`https://…:8892/screen` 是浏览器的 MoQ 接收地址，不能直接填入 OBS 的 WHIP 服务器栏。当前 Compose 关闭了 RTMP，因此这里使用 WHIP 推流。

若「服务」列表中没有 WHIP，请安装带 WHIP 支持的 OBS 构建，参见 [OBS WHIP 指南](https://obsproject.com/kb/whip-streaming-guide)和 [MediaMTX OBS 指南](https://mediamtx.org/docs/publish/obs-studio)。

### 接收端

推荐使用 **MoQ（Media over QUIC）** 接收：

**https://192.168.1.10:8892/screen**

`8892` 是 MediaMTX 的 MoQ 服务端口。实际测试中，这种接收方式更快；Compose 已启用 MoQ。`8892/TCP` 提供 HTTPS 接收页，`8892/UDP` 通过 WebTransport（基于 HTTP/3 / QUIC）传输视频，两者均由 MediaMTX 直接提供。

也可以使用 WebRTC 接收：**http://192.168.1.10:8889/screen**。

按 F11 浏览器全屏。上述地址均需将示例 IP 替换成 `.env` 中的 `LAN_HOST`。

## HTTPS 证书

MoQ 必须使用 HTTPS。MediaMTX 的 `auto.key` / `auto.crt` 设置会自动生成和管理自签名网页证书，无需 Caddy CA 或在两台电脑上安装 Caddy 根证书。

首次打开 MoQ 接收页时，浏览器会提示证书不受信任；确认访问的是服务电脑的局域网 IP 后，在证书提示页选择继续访问。MediaMTX 的播放器会通过 HTTPS 获取 QUIC 证书指纹，用于校验 WebTransport 视频连接。请使用支持证书指纹校验的 Chrome/Chromium/Edge；若浏览器策略禁止访问自签名网页，可使用上述 HTTP WebRTC 接收地址。

容器重建后网页证书可能变化，再次访问时可能需要重新确认。Caddy 不签发证书，也不代理 MoQ 的 HTTPS 或 QUIC。

## 端口

- `127.0.0.1:8090/TCP`：Caddy 本机 HTTP 发布页及同源 WHIP/WHEP 代理。
- `8889/TCP`：MediaMTX WebRTC 接收页及 WHIP/WHEP 接口，使用 WebRTC 接收时须让接收端能访问。
- `8892/TCP`：MediaMTX MoQ HTTPS 接收页及证书指纹接口。
- `8892/UDP`：MediaMTX MoQ 的 HTTP/3 / QUIC WebTransport 视频连接。
- `8189/UDP`：MediaMTX WebRTC 媒体数据；发送端发布时也需要使用。

MediaMTX 原生 MoQ-over-QUIC 接口 `8893/UDP` 仅在容器内部的回环地址监听，不映射到宿主机，浏览器接收不使用。

Caddy 与 MediaMTX 使用 Compose 默认桥接网络，通过上述 TCP/UDP 端口映射提供服务，无需启用 host 网络。Caddy 通过服务名 `mediamtx:8889` 转发发布请求；MediaMTX 使用 `LAN_HOST` 和 `127.0.0.1` 作为 WebRTC 候选地址，不宣告容器内部网卡地址。更换服务电脑的局域网 IP 后，需更新 `.env` 并重新运行 `docker compose up -d`。

若已启用宿主机防火墙，请允许上述局域网服务端口的入站 TCP/UDP 流量；Windows 可在 Windows Defender 防火墙中配置。

Ubuntu/UFW 若已启用防火墙，可执行：

```bash
sudo ufw allow 8889/tcp
sudo ufw allow 8892/tcp
sudo ufw allow 8892/udp
sudo ufw allow 8189/udp
```

建议限制在受信任的局域网中，默认没有身份验证，不可直接暴露在公网。

## 常见问题

- **页面提示无法采集屏幕**：请从运行服务的本机通过 `http://localhost:8090` 打开 Chrome/Chromium。Linux Wayland 环境要确认 PipeWire 和 xdg-desktop-portal 正常运行。
- **MoQ 证书提示**：网页证书由 MediaMTX 自动生成，首次访问需确认自签名证书；不要将 `https://` 改成 `http://`。
- **MoQ 页面能打开但视频不出**：检查 `8892/udp` 防火墙，使用支持 WebTransport、WebCodecs 和证书指纹校验的 Chrome/Chromium/Edge，并确认发送端正在共享。
- **WebRTC 发布失败或接收黑屏**：检查服务电脑防火墙 `8189/udp` 及 `.env` 中的 `LAN_HOST`，确认两台电脑同网段、未启用 Wi-Fi 客户端隔离。
- **502 错误**：运行 `docker compose logs mediamtx caddy` 检查后端是否启动。MediaMTX 固定为 `1.21.2`，其 MoQ 配置项与此 Compose 一致。
- **100–300ms 延迟**：属于目标而非保证。建议有线网络、1080p、60fps；性能不足先降到 30fps（网页 `frameRate`），再调整码率。
- **音频**：网页发送端只传输屏幕视频，不传输桌面声音；OBS 可通过 WHIP 传输已启用的桌面音频或麦克风，接收端按需取消静音。
- **停止**：网页发送端点击「停止共享」，OBS 点击「停止推流」；也可运行 `docker compose down` 关闭服务。

## 官方文档

- [Caddy 反向代理](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy)
- [OBS WHIP 推流指南](https://obsproject.com/kb/whip-streaming-guide)
- [MediaMTX OBS 推流](https://mediamtx.org/docs/publish/obs-studio)
- [MediaMTX 浏览器发布](https://mediamtx.org/docs/publish/web-browsers)
- [MediaMTX 浏览器接收](https://mediamtx.org/docs/read/web-browsers)
- [MediaMTX WebRTC](https://mediamtx.org/docs/features/webrtc-specific-features)
- [MediaMTX MoQ 接收](https://mediamtx.org/docs/read/moq)
- [MediaMTX MoQ 协议与要求](https://mediamtx.org/docs/publish/moq-clients)
