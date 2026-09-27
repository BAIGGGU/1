# 1Panel 控制台（Android）

一个用 Kotlin 写的 Android 原生 App，通过 1Panel 官方开放 API 远程管理你的服务器：**实时概览、容器启停、网站列表**。
代码为完整 Android Studio 工程，导入即可编译安装，无需额外依赖。

## 一、功能

| 模块 | 能力 |
| --- | --- |
| 概览 | 主机名 / 系统 / 内核 / CPU / IP / 运行时长；CPU、内存、Swap 占用条；负载、进程数、网络上下行速率、磁盘 IO；各挂载点磁盘用量；网站 / 数据库 / 应用数量。默认 5 秒自动刷新，支持下拉刷新 |
| 容器 | 容器列表（名称、镜像、状态）；启动 / 停止 / 重启 / 删除（二次确认） |
| 网站 | 站点列表（域名、类型、状态、备注），点击用浏览器打开站点 |
| 连接 | 支持 API Key 与账号密码（JWT）两种鉴权；支持 v2 `/api/v2` 与 v1 `/api/v1`；支持安全入口、自签 HTTPS 证书信任 |

## 二、使用前：在面板开启 API

1. **启用 API 接口并创建 API Key**
   - v2.2.1 之前：面板「面板设置」→ API 接口。
   - v2.2.1 之后：左下角用户菜单 → 用户信息抽屉 →「API 接口」区域启用，点「详情」维护 API Key。
2. **配置 IP 白名单**：填入手机出口 IP（如 `1.2.3.4`），可信内网可临时填 `0.0.0.0/0`（允许全部 IPv4）。
3. **注意时间同步**：签名带 Unix 时间戳，手机与服务器时间差过大（通常 > 几分钟）会返回 401「API 接口密钥错误」。
4. 接口总览可在浏览器打开：`面板地址/1panel/swagger/index.html`。

## 三、编译安装

### 方式 A：Android Studio（推荐）
1. Android Studio（Iguana 及以上，自带 JDK 17）→ **File → Open** → 选择本目录 `1PanelController`。
2. 等待 Gradle 同步完成（首次会下载依赖，需联网）。
3. 手机开启 USB 调试，点 **Run ▶**；或 **Build → Build Bundle(s)/APK(s) → Build APK(s)** 得到 `app/build/outputs/apk/debug/app-debug.apk`。

### 方式 B：命令行（本地有 JDK 17 时）
```bash
cd 1PanelController
gradle wrapper --gradle-version 8.2 --distribution-type all   # 生成 gradlew（工程未内置 jar）
./gradlew assembleDebug
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

要求：JDK 17、`compileSdk 34`、minSdk 26（Android 8.0+）。

### 方式 C：在线编译（不用装 Android Studio）

工程已内置 GitHub Actions 配置 `.github/workflows/build-apk.yml`。

1. 在 GitHub 新建仓库（设成 Private 更稳妥，工程里可能含你的面板地址），把本目录推上去：
   ```bash
   cd 1PanelController
   git init && git add . && git commit -m "1Panel controller"
   git remote add origin git@github.com:<用户名>/<仓库名>.git
   git push -u origin main
   ```
2. 打开仓库 **Actions** 页，等 Gradle 图标变绿（首次约 3–6 分钟，主要耗在拉 SDK 与依赖）。
3. 点进这次运行，在 **Artifacts** 里下载 `app-debug.zip`，解压即得 `app-debug.apk`，传到手机安装。
   也可以手动触发：**Actions → Build APK → Run workflow**。
4. 想长期留档：`git tag v1.0.0 && git push --tags`，会自动发布到 Release 页。

> 国内访问或不想用 GitHub：腾讯云 CODING 持续集成、阿里云云效 Flow 同样支持，构建环境选 Ubuntu + JDK 17，构建命令填 `./gradlew assembleDebug`（先跑一次 `gradle wrapper --gradle-version 8.2`），产物路径 `app/build/outputs/apk/debug/app-debug.apk` 即可，配置思路与上面的 workflow 一致。

> 不推荐把源码上传到各类「在线 APK 打包」小网站：源码里会带上你的面板地址与密钥配置，存在泄露风险。

## 四、App 配置说明

| 配置项 | 说明 |
| --- | --- |
| 面板地址 | 例：`https://192.168.1.10:10086`。不写协议时自动补 `https://` |
| 安全入口 | 只填入口名（如 `myentrance`），App 会自动做 Base64 并加到 `EntranceCode` 头 |
| 认证方式 | **API Key（推荐）**：官方推荐的第三方接入方式；**账号密码（JWT）**：调用 `/core/auth/login` 取 token 后走 `PanelAuthorization` 头 |
| 签名算法 | **HMAC-SHA256（推荐）**，`hmac_sha256(API-Key, "1panel:" + timestamp)`；**MD5（旧版兼容）**，`md5("1panel" + API-Key + timestamp)` |
| 接口版本 | v2 走 `/api/v2/...`（core 负责鉴权、agent 负责执行）；v1 走 `/api/v1/...` |
| 信任自签证书 | 1Panel 默认为自签 HTTPS 证书，勾选后可直连；生产环境建议换成可信证书后关闭此项 |

请求头由 `PanelClient.authHeaders()` 统一生成：
```
1Panel-Timestamp: 1737000000
1Panel-Token:    <签名>
Content-Type:    application/json
```

## 五、已对接接口

| 用途 | 方法 | 路径（v2） |
| --- | --- | --- |
| 概览基础信息 | GET | `/api/v2/dashboard/base/all/all` |
| 概览实时数据 | GET | `/api/v2/dashboard/current/all/all` |
| 容器列表 | POST | `/api/v2/containers/search` |
| 容器操作 | POST | `/api/v2/containers/operate`（`{"names":["xxx"],"operation":"start"}`） |
| 网站列表 | POST | `/api/v2/websites/search`（失败自动回退 GET） |
| 登录（JWT 模式） | POST | `/api/v2/core/auth/login` |

`operation` 取值：`start` / `stop` / `restart` / `kill` / `pause` / `unpause` / `remove`。

### 想加新功能怎么改
1. 在 `Models.kt` 加返回数据类（字段名用 `@SerializedName` 对齐面板 JSON）。
2. 在 `PanelClient.kt` 加一个方法，例如：
   ```kotlin
   fun composeList(): List<ComposeInfo> =
       items(post("containers/compose/search", mapOf("page" to 1, "pageSize" to 100)), ComposeInfo::class.java)
   ```
3. 复制 `WebsitesFragment.kt` / `item_website.xml` 改成一个新页面，在 `bottom_nav_menu.xml` 与 `MainActivity.switchTab()` 中注册即可。

## 六、安全提示

- API Key 相当于面板管理员权限，只保存在本机私有 `SharedPreferences`，请勿在已 root 或不可信设备上使用。
- 尽量用 HTTPS + 可信证书；仅在测试环境勾选「信任自签证书」。
- 白名单按最小授权配置，避免长期开放 `0.0.0.0/0`；若面板与手机都在内网，优先用内网地址或 VPN 访问。
- 切换设备或怀疑泄露时，立即在面板重置 API Key（App 内「清除配置」只清本机，不会吊销密钥）。

## 七、常见问题

| 现象 | 原因与处理 |
| --- | --- |
| 401 /「API 接口密钥错误」 | 密钥填错、API 未启用、IP 不在白名单，或手机与服务器时间不同步 |
| 连接超时 | 端口未放行（安全组 / 防火墙）、地址或协议写错（http 还是 https） |
| 404 / 405 | 接口版本选错：v2 面板用 `/api/v2`，v1 面板用 `/api/v1` |
| 证书报错 | 勾选「信任自签证书」；或为面板配置可信证书 |
| 提示需要安全入口 | 在「安全入口」填入口名，不是完整 URL |

## 八、目录结构

```
1PanelController/
├── app/src/main/java/com/yuanbao/panel/
│   ├── Prefs.kt            配置模型（PanelConfig）与本地存储
│   ├── PanelClient.kt      HTTP 客户端：签名、鉴权头、各业务接口
│   ├── Models.kt           接口返回数据模型
│   ├── Util.kt             字节 / 速率 / 百分比格式化
│   ├── LoginActivity.kt    连接配置与连通性测试
│   ├── MainActivity.kt     底部导航容器
│   ├── OverviewFragment.kt 实时概览（5 秒轮询）
│   ├── ContainersFragment.kt 容器列表与操作
│   └── WebsitesFragment.kt   网站列表
└── app/src/main/res/        布局、菜单、图标、主题
```
