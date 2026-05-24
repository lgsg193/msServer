# msServer 2.0 使用说明

## 一、软件概述

**软件名称：** msServer / 2.0  

**产品定位：** 面向 IPTV、酒店电视、专网直播等场景的 **流媒体转码与分发平台**。集频道转码、直播代理、Web 运维控制台、WebTV 终端播放于一体，可在单机上完成「拉流 → 转码/代理 → 多协议输出 → 终端观看」的完整链路。

**运行平台：**

| 平台 | 支持版本 | 说明 |
|------|----------|------|
| Windows | Windows 10 / Server 2016 及以上 | 推荐 64 位；需自备 FFmpeg（或配置路径） |
| Linux | Ubuntu 22.04 LTS | 推荐用于生产环境 |
| Linux | CentOS 7.x | 需自行安装较新 FFmpeg 与显卡驱动（若使用 GPU 转码） |

> 管理后台与 WebTV 为浏览器访问；转码核心依赖 **FFmpeg**。英伟达（NVENC）转码在 Linux 下可使用控制台内的驱动补丁辅助功能（视环境而定）。

---

## 二、功能介绍

### 1. IPTV 流媒体转码

- 基于 **FFmpeg** 的频道级转码任务，支持 **NVIDIA（NVENC）** 与 **Intel（QSV/VAAPI 等，视模板与系统）** 硬件加速，显著降低 CPU 占用。
- 内置多套 **编码器参数模板**（分辨率、码率、GOP、音频编码等），开箱即用，也可在控制台自定义模板后绑定到频道。
- 支持按频道启停、批量重启、运行状态与日志实时查看（WebSocket 监控）。
- 可配置 **最大并发频道数**（`config.json` → `server.max_channels`），便于控制单机负载。

### 2. 转码拉流 / 推流协议

**输入（拉流，可绑定网卡）：**

- RTSP、RTP、UDP  
- RTMP  
- HTTP-TS、HTTP-HLS  
- HTTPS-TS、HTTPS-HLS  

**输出（推流 / 分发）：**

- RTMP  
- HTTP-TS、HTTP-HLS（本地缓存目录 + HTTP 服务，便于 Nginx 对外发布）  
- UDP、RTP  
- **MSKey 私有协议**（KCP 传输，可配置独立端口，默认 `9200`，适合专网机顶盒或自研播放器）

### 3. 直播流代理（不转码 / 轻量转发）

适用于「源站已是标准流、仅需统一出口」的场景：

- **支持输入：** RTSP、RTP、UDP、RTMP、HTTP/HTTPS-TS、HTTP/HTTPS-HLS、SRT 等（以控制台实际选项为准）。  
- **工作模式：**  
  - **主动拉流：** 由 msServer 按地址周期性或持续拉取；  
  - **端口监听：** 在指定网卡与端口接收推流（视配置而定）。  
- **支持绑定网卡：** 多网卡环境下可指定拉流/监听使用的 NIC。  
- **统一输出：** 转换为本地 **HTTP-TS / HTTP-HLS**，与转码频道共用播放路径规范（如 `/live/`、`/proxy/`），便于 Nginx 反代与防盗链。

### 4. 热更新与推流高可用

- **拉流地址、推流地址支持热更新：** 在控制台修改并保存后，无需整机关重启即可生效（具体以频道/代理任务实现为准）。  
- **推流侧「永不掉线」设计：** 拉流短暂中断时，由内部 **彩条（Colorbar）** 数据自动补齐 TS 包并继续向推流端输出，避免下游 CDN/播放器因断流直接掉线。  
- 网络恢复后自动切回真实节目流。  
- **代理模式** 下彩条策略与转码模式可能不同（代理以稳定转发为主，转码侧由 FFmpeg 统一处理时间戳）。

### 5. WebTV 终端与运营能力

- **WebTV 模式：** 通过浏览器或机顶盒 WebView 观看频道列表，支持 **设备 UUID 绑定**、会话鉴权、单设备在线（抢线踢人）等。  
- **双端界面：**  
  - 电视 / 大屏：`/tvapp/`（遥控器焦点操作）；  
  - 手机：`/tvapp/m/`（独立手机版直播界面，竖屏操作）。  
- **防盗链：** 可配置播放域名（`play_domain`）、流密钥（`stream_key`）、签名参数等，配合 **Nginx 反向代理** 隐藏真实端口与路径。  
- **运维与风控：** IP 黑名单、WebTV 设备封禁、在线会话查看与踢人、观看上报（WS 心跳）等，便于统计 **当前观看人数** 与排障。  
- 管理端可生成带 `?uuid=` 的链接，方便测试与发放给用户。

### 6. Web 控制台与系统管理

- 全功能 **Web 管理后台**（默认路径 `/ms/`），无需安装客户端。  
- 主要模块：仪表盘、监控中心、频道管理、编码器模板、流缓存/代理、网卡管理、黑名单、WebTV 设备与会话、系统设置、Nginx 配置下发等。  
- 支持 **授权许可（License）** 校验、管理员改密、中英文界面切换（视前端配置）。  
- 数据库默认 **SQLite**（单文件 `msServer.db`），可配置 MySQL / PostgreSQL 用于多机或大规模部署。  
- 可选 **Redis** 缓存（未配置时使用内存缓存）。  
- 进程异常退出后，可根据数据库中频道/代理的「启用」状态 **自动恢复** 业务流。  
- 支持 **开机自启动**（Windows 服务 / Linux systemd，需用户自行编写单元或计划任务，将工作目录设为程序所在目录）。

---

## 三、目录与文件说明

典型部署目录（以 `ms` 文件夹为例）：

```
ms/
├── msServer.exe          # Windows 主程序（Linux 下为 msServer 无后缀）
├── config.json           # 主配置文件（首次运行可自动生成）
├── msServer.db           # SQLite 数据库（默认）
├── license.key           # 授权文件（若启用许可）
├── nginx.conf            # 控制台生成的 Nginx 参考配置
├── cache/                # HLS/TS 缓存输出目录
│   ├── live/             # 转码频道
│   └── proxy/            # 代理频道
└── 使用说明.txt          # 简要说明（可与本文档一并分发）
```

---

## 四、快速开始

### 4.1 Windows

1. 将发布包解压到例如 `D:\msServer\ms\`。  
2. 安装 **FFmpeg** 并加入 PATH，或在 `config.json` 中设置：

   ```json
   "server": {
     "ffmpeg_path": "C:\\ffmpeg\\bin\\ffmpeg.exe"
   }
   ```

3. 双击或命令行运行：

   ```powershell
   cd D:\msServer\ms
   .\msServer.exe
   ```

4. 浏览器访问管理后台（默认端口 **9999**）：

   ```
   http://服务器IP:9999/ms/
   ```

5. 首次登录按页面提示完成 **管理员密码** 与 **License 激活**（若启用）。

### 4.2 Linux（Ubuntu 22.04 / CentOS 7）

1. 上传 `msServer` 二进制与配置文件，赋予执行权限：

   ```bash
   chmod +x msServer
   ```

2. 安装 FFmpeg、（可选）NVIDIA 驱动与 NVENC 支持。  
3. 前台调试：

   ```bash
   ./msServer
   ```

4. 生产环境建议使用 **systemd** 或 **screen/tmux** 守护，工作目录必须为包含 `config.json` 的目录。

5. 访问：`http://服务器IP:9999/ms/`。

### 4.3 构建发布包（开发者）

在仓库根目录执行：

```powershell
.\build.ps1
```

将生成 `ms\msServer.exe` 及 Linux 版本，并完成前端与 WebTV 静态资源嵌入。

---

## 五、Web 控制台使用要点

| 模块 | 作用 |
|------|------|
| **仪表盘** | 总览 CPU/内存/磁盘、运行频道数、GPU 信息（如有） |
| **监控中心** | 各路流在线观众、带宽与连接详情 |
| **频道管理** | 创建转码频道：选择编码模板、拉流地址、推流方式、网卡 |
| **编码器** | 维护转码参数模板（含 GPU 相关选项） |
| **流缓存 / 代理** | 配置直播代理任务（拉流/监听、输出 HLS） |
| **网卡管理** | 查看网卡 IP，绑定拉流/服务网卡 |
| **黑名单** | 封禁恶意或异常 IP |
| **WebTV** | 登记终端设备、查看在线会话、踢人、封禁设备 |
| **系统设置** | 播放域名、WebTV 文案、Nginx 部署、改密、重启全部频道等 |

**常用操作：**

1. 在「编码器」中确认或新建模板。  
2. 在「频道管理」新建频道，填写拉流 URL，保存并 **启用**。  
3. 在频道行点击 **预览** 测试播放。  
4. 将 `stream.play_domain` 或 Nginx 对外地址配好后，将播放链接交给终端或 WebTV。

---

## 六、WebTV 访问说明

| 终端 | 地址 | 说明 |
|------|------|------|
| 电视 / 机顶盒浏览器 | `http://域名或IP:端口/tvapp/` | 遥控器操作；手机 UA 会自动跳转到手机版 |
| 手机浏览器 | `http://域名或IP:端口/tvapp/m/` | 独立手机界面：全屏播放、底部选台、搜索频道 |
| 设备绑定链接 | `http://.../tvapp/m/?uuid=设备UUID` | 由管理员在 WebTV 模块创建并分发 |

**说明：**

- 终端需先通过 `/api/v1/tvapp/session` 完成鉴权（页面会自动处理 Cookie）。  
- 播放地址由服务端签发，请勿在公网裸奔 `9999` 端口，建议走 **Nginx 443 + 防盗链**。  
- 修改 WebTV 静态页后需重新执行 `frontend/tvapp` 的 `npm run build` 及 `copy-tvapp-dist.mjs`，再编译 Go 主程序。

---

## 七、Nginx 反向代理（推荐）

生产环境建议：

- 对外只暴露 **80/443**，由 Nginx 转发到本机 `9999`（API + 管理后台 + WebTV）。  
- HLS/TS 走 `location ^~ /live/`、`^~ /proxy/` 等（参考部署目录下 `nginx.conf`）。  
- 使用 `proxy_set_header X-MS-Real-IP $remote_addr;` 以便 msServer 识别真实观众 IP（对应 `proxy.real_ip_header`）。

控制台 **系统设置 → Nginx** 可生成/下发参考配置；Linux 下需自行 `nginx -t` 与 `reload`。

> **注意：** 请勿让通用正则 `location ~ .*\.(js|css)$` 抢在 `/tvapp/` 之前，否则 WebTV 脚本会 404。应使用 `location ^~ /tvapp/` 优先匹配。

---

## 八、主要配置项（config.json）

| 配置项 | 含义 | 示例 |
|--------|------|------|
| `server.port` | HTTP API 与控制台端口 | `"9999"` |
| `server.ffmpeg_path` | FFmpeg 可执行文件 | `"ffmpeg"` |
| `server.max_channels` | 最大转码频道数 | `32` |
| `server.stream_key` | 拉流/播放鉴权基础密钥 | 空=不校验 |
| `server.mskey_port` | MSKey 私有协议端口，`0`=禁用 | `9200` |
| `stream.play_domain` | 对外播放域名（生成完整 URL） | `https://play.example.com` |
| `auth.admin_user` | 管理员账号 | `admin` |
| `database.driver` | 数据库类型 | `sqlite` / `mysql` / `postgres` |
| `redis.addr` | Redis 地址，空=内存缓存 | `127.0.0.1:6379` |
| `tvapp.contact_message` | WebTV 弹窗联系提示 | 自定义文案 |
| `proxy.real_ip_header` | 反代真实 IP 头 | `X-MS-Real-IP` |
| `license_key` | 授权密钥内容 | 由许可文件提供 |

修改 `config.json` 后，部分项支持 **SIGHUP** 热加载（Linux）；端口等关键项需重启进程。

---

## 九、端口与 URL 一览

| 端口/路径 | 用途 |
|-----------|------|
| `9999`（默认） | 管理后台 `/ms/`、REST API `/api/v1/`、WebTV `/tvapp/` |
| `9200`（可选） | MSKey 私有协议分发 |
| `/live/{name}.m3u8` | 转码频道 HLS（常由 Nginx 转发） |
| `/proxy/{name}.m3u8` | 代理频道 HLS |
| `/tv.txt`、`/tv.m3u8` | 对外频道列表（若启用） |

---

## 十、常见问题

**Q：控制台能打开，但频道无法播放？**  
检查 FFmpeg 是否可用、拉流地址是否在本机网卡可达、防火墙是否放行；查看频道日志（WebSocket 日志或后台输出）。

**Q：WebTV 提示会话无效？**  
确认设备 UUID 已在「WebTV → 设备」中登记；检查 Cookie 与 HTTPS 混合内容问题。

**Q：手机打开 `/tvapp/` 黑屏？**  
应使用 `/tvapp/m/`；若从电视页跳转失败，检查是否已部署最新静态资源。

**Q：启动报错 Gin 路由冲突（`/tvapp/*filepath`）？**  
勿单独注册与 `StaticFS("/tvapp")` 冲突的子路由；手机入口统一为 `/tvapp/m/index.html`。

**Q：GPU 转码不可用？**  
确认驱动与 `ffmpeg -encoders` 中 NVENC/QSV 可用；Linux NVIDIA 可尝试控制台「驱动补丁」功能（视显卡型号）。

**Q：如何开机自启？**  
- Windows：任务计划程序，触发器「登录/启动」，操作运行 `msServer.exe`，起始于程序目录。  
- Linux：编写 `systemd` unit，`WorkingDirectory=` 指向 `config.json` 所在目录。

---

## 十一、安全建议

1. 修改默认管理员密码，限制管理后台仅内网或 VPN 访问。  
2. 公网播放务必使用 HTTPS + 防盗链 + IP/设备封禁策略。  
3. 定期备份 `msServer.db` 与 `config.json`。  
4. 生产环境关闭不必要的 `proxy.verbose_log`，避免泄露拉流 URL。

---

## 十二、技术支持

- 控制台内 **帮助** 模块可查看操作提示（若已打包进前端）。  
- 部署问题请附带：`config.json`（脱敏）、频道日志、Nginx 配置片段与浏览器控制台报错。

---

**版本：** msServer 2.0  
**文档更新：** 与 WebTV 手机独立版（`/tvapp/m/`）及当前代码结构同步。
