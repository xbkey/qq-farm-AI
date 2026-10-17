# QQ 扫码登录功能合并说明

本次从 `zhi758258/qq-farm-bot`（fork 链末端，上游同为 QQ Farm Bot 系项目）合并了 QQ 扫码登录相关的增量文件。以下是完整的对比结论、合并内容与验证结果。

## 一、对比结论：QQ 扫码功能现状

对本仓库与 `zhi758258/qq-farm-bot` 的全部 QQ 扫码相关文件做了逐字节对比，结论如下。

### 已完全一致的核心文件（无需合并）

以下 QQ 扫码登录核心文件与本仓库已有实现逐字节相同，本仓库已具备完整的 QQ 扫码能力：

- `core/src/services/qrlogin.js` —— QQ 扫码登录服务（二维码获取、轮询、会话建立）
- `core/src/services/napcat-login.js` —— NapCat（QQ 协议端）登录服务
- `core/src/controllers/admin-qr-login-routes.js` —— 扫码登录管理路由
- `core/src/controllers/admin-napcat-login-routes.js` —— NapCat 登录管理路由
- `core/src/services/wx-login/native-protocol.js` —— 底层协议实现
- `core/src/utils/qrutils.js` —— 二维码工具
- `napcat/Dockerfile`、`napcat/docker-entrypoint.sh`、`napcat/plugins/qq-miniapp-openauth/` —— NapCat 容器与 QQ 小程序授权插件
- `web/src/components/WxLoginModal.vue` —— 扫码登录弹窗
- `web/src/components/login/UpdateLogModal.vue` —— 更新记录弹窗

### 本仓库版本更新的文件（保留本地，不覆盖）

以下文件双方存在差异，经逐行 diff 分析，本仓库版本功能更完整，直接覆盖会丢失本地能力：

- `core/src/services/wx-login-adapter.js` —— 本地版本包含凭证过期管理（`tokenExpiry`、`getAccessTokenExpiresAt`、`shouldRefreshWxCredential`）、续期重试控制与滚动凭证持久化；对方版本仅保留基础的 ManualAuth rejected 续期路径
- `core/src/services/wx-login/service.js` —— 本地版本包含 token 过期时间解析（`expires_in`/`expires_at`）与 `refresh_token_observed_at` 追踪
- `core/src/runtime/relogin-reminder.js` —— 本地版本采用离线通知多渠道分发架构（`offline-notification` + `notification-channels`）；对方为内联 SMTP/Pushoo 提醒实现，属两套不同通知体系
- `web/src/stores/wx-login.ts` —— 本地版本通过统一 `@/api` 客户端发起请求；对方使用原生 `fetch` 拼接头部

## 二、本次合并的增量内容

### 1. 后端：登录页链接管理路由（`core/src/controllers/admin-system-routes.js`）

本仓库前端 `useAdminSystemConfig.ts` 早已包含完整的登录页链接管理逻辑（`loadLoginLinks`、`handleSaveLoginLinks`、`handleResetLoginLinks`、`handleUploadLoginLogo`），但后端缺少对应管理路由，形成"前端有调用、后端无路由"的断层。本次补齐 4 个路由：

- `GET /api/admin/login-links` —— 读取登录页链接配置
- `POST /api/admin/login-links` —— 保存标题、副标题、加QQ群链接等配置，校验链接仅允许 http(s)、mqqapi 或站内路径（拒绝 `javascript:` 等注入）
- `POST /api/admin/login-links/reset` —— 恢复默认并删除已上传的登录图标
- `POST /api/admin/login-logo` —— 上传登录图标（PNG/JPG/WebP/GIF/SVG/ICO，≤2MB，存储于 `login-assets/`）

路由复用本仓库已有的 `store.getLoginLinks()/setLoginLinks()`、`isAllowedImageLink`、`isAllowedPublicLink`、`deleteManagedLoginLogo`、`loginLogoUpload` 等基础设施，未引入新依赖。

### 2. 前端：登录页链接设置卡片（`web/src/components/admin/AdminLoginLinksCard.vue`）

新增管理后台的「登录页链接设置」卡片，并挂载到 `web/src/views/Settings.vue` 的「连接与设备」标签页。管理员可配置：

- 登录页/侧边栏标题与图标（支持上传、更换、移除）
- 登录页副标题
- 加QQ群链接（显示在登录页与侧边栏底部的「加QQ群」按钮上）
- 保存后通过 `appStore.fetchLoginPageConfig()` 即时刷新登录页配置

配套添加了「恢复默认」的危险操作确认弹窗。

### 3. 测试：`core/test/login-links-admin-route.test.js`

新增 7 个测试用例，覆盖：

- GET 返回默认登录页链接
- POST 保存加群链接与购买地址
- POST 拒绝非 http(s) 加群链接（`javascript:` 注入防护）
- POST 拒绝非 http(s) 图标地址（`ftp:` 防护）等

## 三、不合并的部分及原因

- `web/src/views/Login.vue` —— 依赖对方仓库的用户名密码登录体系（`useUserStore`、密码强度校验等）。本仓库采用「访问 Code」登录体系（`CodeLogin.vue` + `/api/login` 访问 Code 校验），两者登录模型不兼容，且本仓库登录页为演进版本（含登录页配置加载、重试倒计时等）。
- `core/test/group-verify-napcat.test.js` 及对应群验证功能 —— 该功能依赖对方仓库的 `admin-auth-routes.js`（用户名密码登录 + 卡密/订阅计费模型，含"账号已过期，请续费"逻辑）与 `store.getGroupVerifyConfig()` 数据模型，与本仓库的访问 Code 登录体系不兼容，未引入。
- 4 个差异文件（见"本仓库版本更新的文件"）—— 保留本地更新版本。

## 四、验证结果

| 验证项 | 结果 |
| --- | --- |
| `node --check` 后端语法检查 | 通过 |
| `core/test/login-links-admin-route.test.js`（新增路由测试） | 7/7 通过 |
| `core/test/napcat-login.test.js`（QQ 扫码登录服务） | 3/3 通过 |
| `vue-tsc -b` 前端类型检查 | 通过 |
| `eslint`（admin-system-routes.js、新增测试） | 通过（顺带修复了 4 处存量 lint 告警：正则提升至模块作用域、未使用参数重命名） |
| core 全量测试套件 | 766 个测试中 765 通过；唯一失败 `local-service-scripts.test.js`（子进程后台启动）为合并前即存在的环境相关失败，与本次改动无关 |

已知存量问题（合并前即存在，非本次引入）：

- `web/test/wx-login.test.cjs` 3 个用例失败 —— 该测试通过 mock 全局 `fetch` 驱动 store，而本仓库 `wx-login.ts` 已演进为统一 `@/api` 客户端调用，测试的 mock 方式不再适配。属测试与实现的版本错位，不影响 QQ 扫码登录实际功能。
