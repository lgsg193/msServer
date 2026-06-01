# msServer 2.0.5.1 使用说明

## 一、软件概述

**软件名称：** msServer / 2.0.5.1  

**产品定位：** 面向 IPTV、酒店电视、专网直播等场景的 **流媒体转码与分发平台**。集频道转码、直播代理、Web 运维控制台、WebTV 终端播放于一体，可在单机上完成「拉流 → 转码/代理 → 多协议输出 → 终端观看」的完整链路。

**运行平台：**

| 平台 | 支持版本 | 说明 |
|------|----------|------|
| Windows | Windows 10 / Server 2016 及以上 | 推荐 64 位；需自备 FFmpeg（或在配置中指定路径） |
| Linux | Ubuntu 22.04 LTS & CentOS 7.x | 推荐用于生产环境；支持后台守护与服务安装 |

> 管理后台与 WebTV 通过浏览器访问；转码核心依赖 **FFmpeg**。英伟达（NVENC）硬件转码在 Linux 下可在控制台使用相关辅助功能（视显卡与驱动环境而定）。

---

## 二、功能介绍

### 1. IPTV 流媒体转码

- 基于 **FFmpeg** 的频道级转码，支持 **NVIDIA（NVENC）** 与 **Intel（QSV/VAAPI 等，视模板与系统）** 硬件加速，降低 CPU 占用。
- 内置多套 **编码器参数模板**（分辨率、码率、GOP、音频等），也可在控制台自定义模板并绑定到频道。
- 支持按频道启停、批量重启、运行状态与日志查看。
- 可配置 **最大并发频道数**（`config.json` 中的 `server.max_channels`），控制单机负载。

### 2. 转码拉流 / 推流协议

**输入（拉流，可绑定网卡）：**

- RTSP、RTP、UDP  
- RTMP  
- HTTP-TS、HTTP-HLS  
- HTTPS-TS、HTTPS-HLS  

**输出（推流 / 分发）：**

- RTMP  
- HTTP-TS、HTTP-HLS（本地缓存 + HTTP 访问，便于 Nginx 对外发布）  
- UDP、RTP  
- **MSKey 私有协议**（KCP 传输，默认端口 `9200`，适合专网机顶盒或自研播放器；可在配置中设为 `0` 关闭）

### 3. 直播流代理（轻量转发）

适用于源站已是标准流、仅需统一出口的场景：

- **支持输入：** RTSP、RTP、UDP、RTMP、HTTP/HTTPS-TS、HTTP/HTTPS-HLS、SRT 等（以控制台实际选项为准）。  
- **工作模式：** 主动拉流；或在指定网卡/端口接收推流（视配置而定）。  
- **支持绑定网卡。**  
- **统一输出** 为本地 HTTP-TS / HTTP-HLS（路径如 `/live/`、`/proxy/`），便于 Nginx 反代与防盗链。

### 4. 热更新与推流高可用

- 拉流、推流地址可在控制台修改后 **热更新**，一般无需重启整机。  
- 拉流短暂中断时，系统用 **彩条** 自动补齐 TS 数据继续推流，降低下游掉线风险；恢复后自动回到真实节目。  

### 5. WebTV 终端与运营

- 浏览器或机顶盒 WebView 观看频道，支持 **设备 UUID 绑定**、会话鉴权、单设备在线（抢线）等。  
- **电视 / 大屏：** `http://服务器地址/tvapp/`（遥控器操作）。  
- **手机：** `http://服务器地址/tvapp/m/`（独立手机界面；用手机打开电视地址时会自动跳转）。  
- 支持播放域名、流密钥、Nginx 反代等 **防盗链** 配置。  
- 支持 IP 黑名单、WebTV 设备封禁、在线会话查看与踢人、观看人数统计等。  

**WebTV 声音说明：** 为符合浏览器自动播放策略，默认 **静音** 起播。电视版在 **全屏播放** 时：**第一次按回车** 开启声音，**再按回车** 打开频道列表。手机版可点 **「点击开启声音」**。

### 6. Web 控制台

- 浏览器打开管理后台（默认 **`http://服务器IP:9999/ms/`**），无需单独安装客户端。  
- 模块包括：仪表盘、监控中心、频道管理、编码器模板、流代理、网卡、黑名单、WebTV 设备与会话、系统设置、Nginx 配置参考等。  
- 支持授权许可（License）、修改管理员密码。  
- 数据默认保存在同目录 **SQLite** 文件 `msServer.db`；可选 MySQL / PostgreSQL。  
- 异常退出后，已启用的频道/代理可在下次启动时 **自动恢复**。

---

## 三、安装包目录说明

将发布包解压到同一文件夹（例如 `D:\msServer\ms\` 或 `/opt/msServer/`），**运行程序时工作目录须为该文件夹**（内含 `config.json`）。

```
ms/
├── msServer.exe          # Windows 主程序（Linux 为 msServer，无后缀）
├── config.json           # 主配置（首次运行可自动生成）
├── msServer.db           # 数据库（默认 SQLite）
├── license.key           # 授权文件（若使用许可）
├── nginx.conf            # 控制台生成的 Nginx 参考配置
├── cache/                # HLS 输出缓存
│   ├── live/             # 转码频道
│   └── proxy/            # 代理频道
└── 使用说明.md           # 本文档
```

---

## 四、启动与停止

### 4.1 命令行参数一览

在程序所在目录打开命令行执行（Linux 下 `./msServer`，Windows 下 `msServer.exe`）：

| 参数 | 作用 | Windows | Linux |
|------|------|:-------:|:-----:|
| （无参数） | **前台运行**，占用当前窗口，可看日志输出 | ✓ | ✓ |
| `-d` | **后台守护** 启动（脱离当前终端，适合 SSH 断开后继续运行） | 不支持 | ✓ |
| `-stop` | **停止** 正在运行的后台实例 | 不支持 | ✓ |
| `-install` | **安装为系统服务**（systemd，服务名 `ms`），并设置开机自启 | 不支持 | ✓ |
| `-uninstall` | **卸载系统服务**（停止并删除 `ms.service`） | 不支持 | ✓ |

> **说明：** `-install`、`-uninstall`、`-d`、`-stop` 仅在 **Linux** 下可用。Windows 请用前台运行，或通过 **任务计划程序** 实现开机启动（见下文）。

### 4.2 Windows 使用方式

1. 安装 **FFmpeg** 并加入系统 PATH，或在 `config.json` 中填写完整路径，例如：

   ```json
   "server": {
     "ffmpeg_path": "C:\\ffmpeg\\bin\\ffmpeg.exe"
   }
   ```

2. **前台运行（常用）：**

   ```powershell
   cd D:\msServer\ms
   .\msServer.exe
   ```

   也可双击 `msServer.exe`（关闭窗口即退出程序）。

3. 浏览器访问：**`http://本机IP:9999/ms/`**，按提示设置管理员密码与 License（若需要）。

4. **开机自启（推荐）：** 打开「任务计划程序」→ 创建基本任务 → 触发器选「计算机启动」或「用户登录」→ 操作「启动程序」→ 程序填 `msServer.exe` 完整路径 → **「起始于」填 `msServer.exe` 所在文件夹**（必须与 `config.json` 同目录）。

### 4.3 Linux 使用方式

1. 赋予执行权限：`chmod +x msServer`  
2. 安装 FFmpeg；GPU 转码需安装对应驱动。  
3. **前台调试：**

   ```bash
   cd /opt/msServer
   ./msServer
   ```

4. **后台运行（SSH 断开后仍运行）：**

   ```bash
   ./msServer -d
   ```

   成功后会提示后台进程 PID，当前终端可关闭。

5. **停止后台实例：**

   ```bash
   ./msServer -stop
   ```

6. **安装为系统服务（需 root）：**

   安装前请确认：程序、`config.json`、`license.key`（若需要）已在 **最终运行目录** 中就位（`-install` 会把该目录写入服务配置）。

   ```bash
   cd /opt/msServer          # 进入程序目录
   sudo ./msServer -install
   ```

   执行成功后会自动完成三件事：

   - 在系统中注册服务 **`ms`**（systemd 单元文件：`/etc/systemd/system/ms.service`）  
   - **立即启动** 一次服务  
   - **开机自启**（`systemctl enable`）

7. **安装服务后的启动与停止（日常运维用 systemctl）**

   安装完成后，请使用系统自带的 **systemctl** 管理进程，**不要**再手动执行 `./msServer` 或 `./msServer -d`，以免与系统服务重复运行。

   | 操作 | 命令 | 说明 |
   |------|------|------|
   | 查看状态 | `sudo systemctl status ms` | 是否运行中、最近日志摘要 |
   | **启动** | `sudo systemctl start ms` | 服务未运行时启动 |
   | **停止** | `sudo systemctl stop ms` | 停止服务（改配置、维护时常用） |
   | **重启** | `sudo systemctl restart ms` | 修改 `config.json` 后使配置生效（部分项需重启） |
   | 开机自启 | `sudo systemctl enable ms` | `-install` 已执行过，一般无需再设 |
   | 取消开机自启 | `sudo systemctl disable ms` | 保留服务但不随系统启动 |
   | 查看运行日志 | `sudo journalctl -u ms -f` | 实时跟踪服务输出（排障用） |

   **示例：**

   ```bash
   sudo systemctl status ms      # 先看是否在跑
   sudo systemctl stop ms        # 停止
   sudo systemctl start ms       # 再启动
   sudo systemctl restart ms     # 或直接重启
   ```

   > **与 `-stop` 的区别：** `./msServer -stop` 仅用于此前用 **`./msServer -d`** 方式拉起的后台进程（通过锁文件停止）。**已 `-install` 为系统服务后，请一律用 `systemctl stop/start/restart ms`。**

8. **卸载系统服务：**

   ```bash
   sudo ./msServer -uninstall
   ```

   会自动停止服务、取消开机自启并删除 `ms.service`。卸载后若需再运行，可改用 `./msServer` 前台或 `./msServer -d` 后台方式。

9. 浏览器访问：**`http://服务器IP:9999/ms/`**。

> **注意：** 同一台机器只能运行 **一个** msServer 实例；重复启动会提示已在运行。

---

## 五、Web 控制台简要流程

| 模块 | 作用 |
|------|------|
| 仪表盘 | CPU/内存/磁盘、运行频道数、GPU 信息 |
| 监控中心 | 各路流观众与连接情况 |
| 频道管理 | 新建转码频道、拉流地址、编码模板、启停 |
| 编码器 | 转码参数模板（含 GPU 相关项） |
| 流缓存 / 代理 | 直播代理（拉流/监听 → 本地 HLS） |
| 网卡管理 | 查看 IP、绑定拉流网卡 |
| 黑名单 | 封禁 IP |
| WebTV | 终端设备、在线会话、踢人、封禁 |
| 系统设置 | 播放域名、Nginx 参考配置、改密、重启全部频道等 |

**典型步骤：** 配置编码器模板 → 新建并启用频道 → 预览测试 → 配置对外域名或 Nginx → 将 WebTV 或播放地址发给终端用户。

---

## 六、WebTV 访问地址

| 终端 | 地址 |
|------|------|
| 电视 / 机顶盒 | `http://域名或IP:端口/tvapp/` |
| 手机 | `http://域名或IP:端口/tvapp/m/` |
| 带设备绑定 | `http://.../tvapp/?uuid=设备UUID` 或 `.../tvapp/m/?uuid=设备UUID` |

设备 UUID 在管理后台 **WebTV → 设备** 中创建。公网访问建议仅开放 **80/443**，由 Nginx 转发到本机服务端口，并配置 HTTPS 与防盗链。

---

## 七、Nginx 反向代理（生产环境建议）

- 对外只暴露 **80/443**，反代到本机 **9999**（管理后台、API、WebTV）。  
- 节目流路径使用 `location ^~ /live/`、`^~ /proxy/` 等（可参考程序目录下 `nginx.conf`）。  
- 建议设置：`proxy_set_header X-MS-Real-IP $remote_addr;`，便于统计真实观众 IP。  

控制台 **系统设置** 中可生成 Nginx 参考配置；修改后需在服务器上执行 `nginx -t` 与 `reload`。

---

## 八、常用配置（config.json）

| 配置项 | 含义 | 示例 |
|--------|------|------|
| `server.port` | Web 与 API 端口 | `"9999"` |
| `server.ffmpeg_path` | FFmpeg 程序 | `"ffmpeg"` 或完整路径 |
| `server.max_channels` | 最大转码频道数 | `32` |
| `server.stream_key` | 流鉴权密钥（空=不校验） | 自定义字符串 |
| `server.mskey_port` | MSKey 端口，`0`=关闭 | `9200` |
| `stream.play_domain` | 对外播放域名 | `https://play.example.com` |
| `auth.admin_user` | 管理员账号 | `admin` |
| `tvapp.contact_message` | WebTV 无法观看时的联系提示 | 自定义文案 |

修改配置后，Linux 下部分项支持发送 **SIGHUP** 热加载；改端口等需重启程序。改配置前建议备份 `config.json` 与 `msServer.db`。

---

## 九、端口与路径

| 项目 | 说明 |
|------|------|
| `9999`（默认） | 管理后台 `/ms/`、API、WebTV |
| `9200`（可选） | MSKey 私有协议 |
| `/live/频道名.m3u8` | 转码频道 HLS |
| `/proxy/名称.m3u8` | 代理频道 HLS |

---

## 十、常见问题

**Q：控制台能打开，频道不能播放？**  
检查 FFmpeg 是否可用、拉流地址在服务器上是否可达、防火墙是否放行；在频道管理中查看运行日志。

**Q：WebTV 提示会话无效或未登记？**  
在 WebTV 设备管理中登记 UUID，使用管理员下发的链接访问。

**Q：WebTV 有画面没声音？**  
电视版：全屏时 **按一次回车** 开声音。手机版：点 **「点击开启声音」**。

**Q：Linux 安装服务后怎么启动/停止？**  
使用 `sudo systemctl start ms` 启动、`sudo systemctl stop ms` 停止、`sudo systemctl restart ms` 重启；`sudo systemctl status ms` 查看状态。不要用 `./msServer -d` 与系统服务同时跑两套实例。

**Q：Linux 后台怎么停不掉？**  
若已 `-install`：执行 `sudo systemctl stop ms`。若只是 `./msServer -d` 拉起的：执行 `./msServer -stop`。

**Q：`-install` 提示权限不足？**  
须使用 `sudo ./msServer -install`，且建议在最终部署目录下执行，以免服务工作目录错误。

**Q：Windows 能否用 `-install`？**  
不能，请用任务计划程序实现开机启动（见 4.2 节）。

**Q：GPU 转码不可用？**  
确认驱动已装，命令行执行 `ffmpeg -encoders` 能看到 NVENC/QSV；Linux 可在控制台尝试驱动相关功能。

---

## 十一、安全建议

1. 修改默认管理员密码；管理后台尽量仅内网或 VPN 访问。  
2. 公网播放使用 HTTPS、防盗链与 IP/设备封禁。  
3. 定期备份 `msServer.db` 与 `config.json`。  
4. 勿将拉流源地址泄露到公网日志中。

---

## 十二、技术支持

部署或播放异常时，请准备：软件版本（msServer 2.0）、操作系统、脱敏后的 `config.json`、频道日志截图、浏览器报错信息（如有）。

---

**版本：** msServer 2.0  
**适用对象：** 最终用户与运维人员（安装包部署，不含二次开发说明）
