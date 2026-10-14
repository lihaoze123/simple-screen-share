# 局域网屏幕共享 (Linux -> Windows)

Linux 电脑安装 Docker + Docker Compose；Linux 和 Windows 使用 Chrome/Chromium/Edge 浏览器。Caddy 提供本机 HTTP 发布页和发布接口代理，MediaMTX 自己处理 MoQ 的 HTTPS 与视频传输。

## 启动

先运行 `hostname -I` 或 `ip -4 addr` 找到 Windows 能访问的 Linux 局域网 IPv4 地址。假设是 `192.168.1.10`，在项目目录中：

```bash
cp .env.example .env
```

编辑 `.env`，将 `LAN_HOST` 改成实际地址（只填 IP，不带协议、端口或路径）。这个值用于发布页显示接收地址：

```dotenv
LAN_HOST=192.168.1.10
```

启动服务；`--remove-orphans` 会移除旧配置中的 nginx 发布页容器：

```bash
docker compose up -d --remove-orphans
```

## 使用

### Linux 发送端

本机浏览器访问 **http://localhost:8090**，点击 **开始共享屏幕**，选择整个屏幕。`localhost` 的 HTTP 页面可以采集屏幕；Caddy 只监听 `127.0.0.1:8090`，不要使用局域网 IP 打开发布页。

发布页通过同源 `/screen/whip` 请求 Caddy，再由 Caddy 转发到 MediaMTX 的 WebRTC 接口，无需配置发布页证书。

### Windows 接收端

推荐使用 **MoQ（Media over QUIC）** 接收：

**https://192.168.1.10:8892/screen**

`8892` 是 MediaMTX 的 MoQ 服务端口。实际测试中，这种接收方式更快；Compose 已启用 MoQ。`8892/TCP` 提供 HTTPS 接收页，`8892/UDP` 通过 WebTransport（基于 HTTP/3 / QUIC）传输视频，两者均由 MediaMTX 直接提供。

也可以使用 WebRTC 接收：**http://192.168.1.10:8889/screen**。

按 F11 浏览器全屏。上述地址均需将示例 IP 替换成 `.env` 中的 `LAN_HOST`。

## HTTPS 证书

MoQ 必须使用 HTTPS。MediaMTX 的 `auto.key` / `auto.crt` 设置会自动生成和管理自签名网页证书，无需 Caddy CA 或在两台电脑上安装 Caddy 根证书。

首次打开 MoQ 接收页时，浏览器会提示证书不受信任；确认访问的是自己的 Linux 局域网 IP 后，在证书提示页选择继续访问。MediaMTX 的播放器会通过 HTTPS 获取 QUIC 证书指纹，用于校验 WebTransport 视频连接。请使用支持证书指纹校验的 Chrome/Chromium/Edge；若浏览器策略禁止访问自签名网页，可使用上述 HTTP WebRTC 接收地址。

容器重建后网页证书可能变化，再次访问时可能需要重新确认。Caddy 不签发证书，也不代理 MoQ 的 HTTPS 或 QUIC。

## 端口

- `127.0.0.1:8090/TCP`：Caddy 本机 HTTP 发布页及同源 WHIP/WHEP 代理。
- `8889/TCP`：MediaMTX WebRTC 接收页及 WHIP/WHEP 接口，使用 WebRTC 接收时须让 Windows 能访问。
- `8892/TCP`：MediaMTX MoQ HTTPS 接收页及证书指纹接口。
- `8892/UDP`：MediaMTX MoQ 的 HTTP/3 / QUIC WebTransport 视频连接。
- `8189/UDP`：MediaMTX WebRTC 媒体数据；发送端发布时也需要使用。
- `127.0.0.1:8893/UDP`：MediaMTX 原生 MoQ-over-QUIC 接口，浏览器接收不使用。

Caddy 与 MediaMTX 使用 Docker 的 host 网络，让 MediaMTX 自动发现局域网网卡地址，不适用于 Docker Desktop for macOS/Windows。

Ubuntu/UFW 若已启用防火墙，可执行：

```bash
sudo ufw allow 8889/tcp
sudo ufw allow 8892/tcp
sudo ufw allow 8892/udp
sudo ufw allow 8189/udp
```

建议限制在受信任的局域网中，默认没有身份验证，不可直接暴露在公网。

## 常见问题

- **页面提示无法采集屏幕**：请从 Linux 本机通过 `http://localhost:8090` 打开 Chrome/Chromium。Wayland 环境要确认 PipeWire 和 xdg-desktop-portal 正常运行。
- **MoQ 证书提示**：网页证书由 MediaMTX 自动生成，首次访问需确认自签名证书；不要将 `https://` 改成 `http://`。
- **MoQ 页面能打开但视频不出**：检查 `8892/udp` 防火墙，使用支持 WebTransport、WebCodecs 和证书指纹校验的 Chrome/Chromium/Edge，并确认发送端正在共享。
- **WebRTC 发布失败或接收黑屏**：检查 Linux 防火墙 `8189/udp`，确认两台电脑同网段、未启用 Wi-Fi 客户端隔离。
- **502 错误**：运行 `docker compose logs mediamtx caddy` 检查后端是否启动。MediaMTX 固定为 `1.21.2`，其 MoQ 配置项与此 Compose 一致。
- **100–300ms 延迟**：属于目标而非保证。建议有线网络、1080p、60fps；性能不足先降到 30fps（网页 `frameRate`），再调整码率。
- **音频**：此版本仅传输屏幕视频，不传输桌面声音，以简化 Linux 跨桌面环境兼容性。
- **停止**：点击停止共享，或运行 `docker compose down` 关闭服务。

## 官方文档

- [Caddy 反向代理](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy)
- [MediaMTX 浏览器发布](https://mediamtx.org/docs/publish/web-browsers)
- [MediaMTX 浏览器接收](https://mediamtx.org/docs/read/web-browsers)
- [MediaMTX WebRTC](https://mediamtx.org/docs/features/webrtc-specific-features)
- [MediaMTX MoQ 接收](https://mediamtx.org/docs/read/moq)
- [MediaMTX MoQ 协议与要求](https://mediamtx.org/docs/publish/moq-clients)
