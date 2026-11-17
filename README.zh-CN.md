# EMGS Hub 安卓应用安全研究报告

**语言：** [English](./README.md) | 简体中文 | [繁體中文](./README.zh-TW.md)

> **负责任披露研究说明。** 本文是一份从零改写的描述性安全研究，**不包含任何
> 可运作的机密、反编译源码、二进制文件或他人个人数据**。下文引用的所有凭证与
> 私密值均已替换为 `[REDACTED]`，只描述机密的「类别」与风险的「性质」。本文
> 仅用于教育与防御目的。应用中历史上内嵌的任何真实凭证，应由 EMGS 自行轮换。

---

## 1. 样本与目标

- **应用：** EMGS Hub，版本 **1.0.10**（以 XAPK 形式分发）。
- **包名：** `my.com.edu.emgs`
- **versionCode：** 89
- **minSdk：** 26 / **targetSdk：** 35（Android 8.0+）。
- **所检查的 ABI split：** `arm64_v8a`。
- **发布方：** **EMGS（Education Malaysia Global Services，马来西亚全球教育
  服务）** 的官方移动应用，提供学生签证与国际学生服务。

本文描述该应用的架构，以及从其公开分发包中得出的若干安全观察。**不复制或
附带任何受版权保护的源码或二进制文件。**

## 2. 应用架构

EMGS Hub 是一个 **Capacitor/Ionic 混合应用**。几乎全部业务逻辑都位于
**前端 Web bundle** 中，即一个 **Angular 16 + Ionic 7 的单页应用（SPA）**，
位于 `assets/public`。

原生 Java 层只是一个很薄的壳：一个 `BridgeActivity` 风格的宿主，其
`onCreate` 仅初始化 `FirebaseAnalytics`。原生层**没有任何业务逻辑**——它
只是承载 WebView 并桥接原生能力。

## 3. 原生能力

原生功能通过 Capacitor/Cordova 插件桥接给 Web 层，包括：

- **推送通知**，采用双通道：Firebase Cloud Messaging（FCM）与华为移动
  服务（HMS）。
- **生物识别**认证（本地）。
- **条码／二维码扫描**（journeyapps 扫描器）。
- **文件打开**。
- **InAppBrowser**，用于 OAuth 登录流程。
- **高级 HTTP** 客户端。

## 4. 前端路由（约 13 个页面）

该 SPA 定义了大约十三个路由页面：

- `login`（登录）
- `home`（首页）
- `profile`（个人资料）
- `digital-id`（数字身份）
- `ikad-qr`
- `emgs-panel-clinic`（体检定点诊所列表）
- `post-arrival-checkin`（入境后签到＋人脸核验）
- `application-history`（申请历史）
- `apply-visa`（申请签证）
- `barcode-scanner`（条码扫描）
- `camera-preview`（相机预览）
- `expired-cards`（过期卡片）
- `feedback`（反馈）

## 5. 后端与接口

- **主 API 基址：** `mycas.educationmalaysia.gov.my/api/rest`。
- **认证方案：** **OAuth 1.0a**，使用 HMAC-SHA1 对请求签名。
- **登录流程：** 在 InAppBrowser 中打开 `/oauth/authorize`；应用注册自定义
  回调 scheme `emgsapp://call-back`。

可枚举的业务端点（**仅列路径，不列任何凭证**）：

- `/student/info`
- `/student/kyc`（人脸 KYC）
- `/student/applicationStatus`
- `/student/notification`
- `/student/checkIn` 与 `/student/checkInHistory`
- `/student/canCheckIn`
- `/student/ikad`
- `/emgs/clinics`
- `/preEntryUrl`
- `/cms/blocks`

## 6. 关键功能实现

- **登录：** 三段式 OAuth 1.0a。成功回调 scheme 为 `emgsapp://call-back`；
  拒绝则经 `emgsapp://reject` 返回。
- **数字身份 iKad：** 凭证从 `/student/ikad` 获取。其 QR payload 为
  **明文 JSON**（包含姓名、国籍、性别、大学、课程），**无签名、无加密**——
  扫码即可读取。这是一项**隐私风险**。
- **人脸核验（KYC）：** 前端使用 **MediaPipe `face_mesh`（WASM）** 与
  **face-api.js** 模型进行比对后上传。
- **PIN／生物识别：** 仅为**本机 UI 门禁**，存在本地 storage 中，**不参与**
  服务器端校验。
- **推送：** 同时在 FCM 与华为 HMS 双重注册。

## 7. 风险与观察

如实列出；**不复制任何实际值**。

1. **前端 bundle 内硬编码多项机密。** Web bundle 内嵌多个凭证，此处一律
   显示为 `[REDACTED]`：
   - **FCM legacy server key** —— `[REDACTED]`。**最严重：** 泄露的
     legacy server key 可让攻击者通过该 FCM 项目任意发送推送通知。
   - **OAuth consumer secret** —— `[REDACTED]`。
   - **Firebase `apiKey`** —— `[REDACTED]`。
   - **Google Maps key** —— `[REDACTED]`。
   - **Parse / Scalabl app id 与 JS key** —— `[REDACTED]`。
   - **Supabase anon key** —— `[REDACTED]`。
2. **OAuth consumer secret 随包泄露。** 由于 consumer secret 随应用一起分发，
   攻击者可离线伪造 HMAC-SHA1 签名头。（仍需有效的用户 token 才能调用业务
   接口，因此仅凭此项并不等于完整账号访问。）
3. **iKad QR 为明文且无签名。** 如上所述，数字身份 QR 可被任意扫描器读取并
   收集其中的个人字段，且没有任何机制防止对 payload 结构的篡改或伪造。
4. **偵错样例中残留真实个人数据。** 一份偵错样例 JSON 残留了一位 2023 年
   真实申请人的个人数据（姓名、护照号、申请编号）。**该数据已移除，不在此处
   复现**；仅为记录此问题存在而说明。
5. **第三方分发渠道。** 所检查的 XAPK 来自第三方分发渠道（APKPure 重打包），
   即**并非官方签名包**。

## 8. 研究方法与工具链

- 使用 **jadx 1.5.1** CLI 进行全量反编译／导出。
- 在导出树上使用语义／源码勘察工具。

**仅描述方法。** 本仓库**不包含也不链接**任何反编译产物、APK/XAPK、DEX 或
JS bundle。

## 9. 免责声明

本报告**仅用于教育与安全研究目的**。其中**不含任何可运作的机密，也不含他人
个人数据**。应用中历史上内嵌的任何凭证，应由 **EMGS 自行轮换**。如果您是
EMGS 代表并希望讨论此处任何条目，欢迎通过负责任披露渠道联系。
