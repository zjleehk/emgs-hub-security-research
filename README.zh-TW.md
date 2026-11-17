# EMGS Hub Android App 安全研究報告

**語言：** [English](./README.md) | [简体中文](./README.zh-CN.md) | 繁體中文

> **負責任揭露研究說明。** 本文是一份從零改寫的描述性安全研究，**不包含任何
> 可運作的機密、反編譯原碼、二進位檔或他人個人資料**。下文引用的所有憑證與
> 私密值皆已替換為 `[REDACTED]`，只描述機密的「類別」與風險的「性質」。本文
> 僅供教育與防禦用途。App 中歷史上內嵌的任何真實憑證，應由 EMGS 自行輪替。

---

## 1. 樣本與目標

- **App：** EMGS Hub，版本 **1.0.10**（以 XAPK 形式分發）。
- **套件名（package）：** `my.com.edu.emgs`
- **versionCode：** 89
- **minSdk：** 26 / **targetSdk：** 35（Android 8.0+）。
- **所檢視的 ABI split：** `arm64_v8a`。
- **發布方：** **EMGS（Education Malaysia Global Services，馬來西亞全球
  教育服務）** 的官方行動 App，提供學生簽證與國際學生服務。

本文描述此 App 的架構，以及自其公開分發包檢視所得的若干安全觀察。**不複製或
附帶任何受版權保護的原碼或二進位檔。**

## 2. 應用架構

EMGS Hub 是一個 **Capacitor/Ionic 混合應用**。幾乎全部業務邏輯都位於
**前端 Web bundle** 中，即一個 **Angular 16 + Ionic 7 的單頁應用（SPA）**，
位於 `assets/public`。

原生 Java 層只是一個很薄的殼：一個 `BridgeActivity` 風格的宿主，其
`onCreate` 僅初始化 `FirebaseAnalytics`。原生層**沒有任何業務邏輯**——它
只是承載 WebView 並橋接原生能力。

## 3. 原生能力

原生功能透過 Capacitor/Cordova 外掛橋接給 Web 層，包括：

- **推播通知**，採雙通道：Firebase Cloud Messaging（FCM）與華為行動
  服務（HMS）。
- **生物辨識**驗證（本機）。
- **條碼／QR 掃描**（journeyapps 掃描器）。
- **檔案開啟**。
- **InAppBrowser**，用於 OAuth 登入流程。
- **進階 HTTP** 客戶端。

## 4. 前端路由（約 13 個頁面）

此 SPA 定義了大約十三個路由頁面：

- `login`（登入）
- `home`（首頁）
- `profile`（個人資料）
- `digital-id`（數位身分）
- `ikad-qr`
- `emgs-panel-clinic`（體檢指定診所列表）
- `post-arrival-checkin`（入境後簽到＋人臉核驗）
- `application-history`（申請歷史）
- `apply-visa`（申請簽證）
- `barcode-scanner`（條碼掃描）
- `camera-preview`（相機預覽）
- `expired-cards`（過期卡片）
- `feedback`（意見回饋）

## 5. 後端與介面

- **主 API 基底：** `mycas.educationmalaysia.gov.my/api/rest`。
- **認證方案：** **OAuth 1.0a**，使用 HMAC-SHA1 對請求簽名。
- **登入流程：** 於 InAppBrowser 中開啟 `/oauth/authorize`；App 註冊自訂
  回呼 scheme `emgsapp://call-back`。

可列舉的業務端點（**只列路徑，不列任何憑證**）：

- `/student/info`
- `/student/kyc`（人臉 KYC）
- `/student/applicationStatus`
- `/student/notification`
- `/student/checkIn` 與 `/student/checkInHistory`
- `/student/canCheckIn`
- `/student/ikad`
- `/emgs/clinics`
- `/preEntryUrl`
- `/cms/blocks`

## 6. 關鍵功能實作

- **登入：** 三段式 OAuth 1.0a。成功回呼 scheme 為 `emgsapp://call-back`；
  拒絕則經 `emgsapp://reject` 返回。
- **數位身分 iKad：** 憑證自 `/student/ikad` 取得。其 QR payload 為
  **明文 JSON**（含姓名、國籍、性別、大學、課程），**無簽名、無加密**——
  掃碼即可讀取。此屬**隱私風險**。
- **人臉核驗（KYC）：** 前端使用 **MediaPipe `face_mesh`（WASM）** 與
  **face-api.js** 模型進行比對後上傳。
- **PIN／生物辨識：** 僅為**本機 UI 門禁**，存於本地 storage，**不參與**
  伺服器端校驗。
- **推播：** 同時於 FCM 與華為 HMS 雙重註冊。

## 7. 風險與觀察

如實列出；**不複製任何實際值**。

1. **前端 bundle 內硬編碼多項機密。** Web bundle 內嵌多個憑證，此處一律
   顯示為 `[REDACTED]`：
   - **FCM legacy server key** —— `[REDACTED]`。**最嚴重：** 外洩的
     legacy server key 可讓攻擊者透過該 FCM 專案任意發送推播通知。
   - **OAuth consumer secret** —— `[REDACTED]`。
   - **Firebase `apiKey`** —— `[REDACTED]`。
   - **Google Maps key** —— `[REDACTED]`。
   - **Parse / Scalabl app id 與 JS key** —— `[REDACTED]`。
   - **Supabase anon key** —— `[REDACTED]`。
2. **OAuth consumer secret 隨包外洩。** 由於 consumer secret 隨 App 一併分發，
   攻擊者可離線偽造 HMAC-SHA1 簽名頭。（仍需有效的使用者 token 才能呼叫業務
   介面，因此僅憑此項並不等於完整帳號存取。）
3. **iKad QR 為明文且無簽名。** 如上所述，數位身分 QR 可被任意掃描器讀取並
   收集其中的個人欄位，且沒有任何機制防止對 payload 結構的竄改或偽造。
4. **偵錯樣例中殘留真實個人資料。** 一份偵錯樣例 JSON 殘留了一位 2023 年
   真實申請人的個人資料（姓名、護照號、申請編號）。**該資料已移除，不在此處
   重現**；僅為記錄此問題存在而說明。
5. **第三方分發渠道。** 所檢視的 XAPK 來自第三方分發渠道（APKPure 重打包），
   即**並非官方簽名包**。

## 8. 研究方法與工具鏈

- 使用 **jadx 1.5.1** CLI 進行全量反編譯／匯出。
- 於匯出樹上使用語義／原碼勘察工具。

**僅描述方法。** 本倉庫**不包含也不連結**任何反編譯產物、APK/XAPK、DEX 或
JS bundle。

## 9. 免責聲明

本報告**僅供教育與安全研究用途**。其中**不含任何可運作的機密，也不含他人
個人資料**。App 中歷史上內嵌的任何憑證，應由 **EMGS 自行輪替**。若您是
EMGS 代表並希望討論此處任何條目，歡迎透過負責任揭露管道聯繫。
