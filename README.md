# 局域网屏幕共享 (Linux -> Windows)

仅需在 Linux 电脑安装 Docker + Docker Compose；Linux 和 Windows 上使用 Chrome/Chromium/Edge 浏览器。

## 启动

在解压后的 `lan-screenshare` 目录中：

```bash
docker compose up -d
```

### Linux 发送端

本机浏览器访问 **http://localhost:8090**，点击 **开始共享屏幕**，选择整个屏幕。屏幕采集必须从 localhost 访问，不能用 `http://192.168...:8090`。

### Windows 接收端

先运行 `hostname -I` 或 `ip -4 addr` 在 Linux 上找到局域网 IP。假设是 `192.168.1.10`：

**http://192.168.1.10:8889/screen**

按 F11 浏览器全屏。

## 端口

- `8090/TCP`: **仅 Linux 本机**网页发布 UI
- `8889/TCP`: MediaMTX WebRTC / WHIP / WHEP，必须让 Windows 能访问
- `8189/UDP`: WebRTC 媒体数据，必须让 Windows 能访问

Linux 使用 Docker 的 host 网络，让 MediaMTX 自动发现局域网网卡地址（不需要手动填 IP）。不适用于 Docker Desktop for macOS/Windows。

Ubuntu/UFW 若已启用防火墙，可执行：

```bash
sudo ufw allow 8889/tcp
sudo ufw allow 8189/udp
```

建议限制在受信任的局域网中，默认没有身份验证，不可直接暴露在公网。

## 常见问题

- **页面提示无法访问屏幕**：请在 Linux 本机通过 `http://localhost:8090` 打开 Chrome/Chromium；不要通过 `192.168...` 地址打开共享页面。Wayland 环境要确认 PipeWire 和 xdg-desktop-portal 正常运行。
- **Windows 有页面但视频不出**：检查 Linux 防火墙 `8189/udp`，并确认两台电脑同网段、未启用 Wi-Fi 客户端隔离。
- **100–300ms 延迟**：属于目标而非保证。建议有线网络、1080p、60fps；性能不足先降到 30fps（网页 `frameRate`），再调整码率。
- **音频**：此版本仅传输屏幕视频，不传输桌面声音，以最大限度简化 Linux 跨桌面环境兼容性。
- **停止**：点击停止共享，或运行 `docker compose down` 关闭服务。

## 官方文档

- https://mediamtx.org/docs/publish/web-browsers
- https://mediamtx.org/docs/read/web-browsers
- https://mediamtx.org/docs/features/webrtc-specific-features
