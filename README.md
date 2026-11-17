# Security Research Notes on the EMGS Hub Android App

**Language:** English | [简体中文](./README.zh-CN.md) | [繁體中文](./README.zh-TW.md)

> **Responsible-disclosure research note.** This document is a from-scratch,
> descriptive security write-up. It contains **no working secrets, no
> decompiled source, no binaries, and no third-party personal data**. Every
> credential or private value referenced below has been replaced with
> `[REDACTED]`; only the *class* of secret and the *nature* of the risk are
> described. The purpose is educational and defensive. Any real credentials
> that were historically embedded in the app should be rotated by EMGS.

---

## 1. Sample and Target

- **App:** EMGS Hub, version **1.0.10** (distributed as an XAPK).
- **Package name:** `my.com.edu.emgs`
- **versionCode:** 89
- **minSdk:** 26 / **targetSdk:** 35 (Android 8.0+).
- **ABI split examined:** `arm64_v8a`.
- **Vendor:** Official mobile app of **EMGS (Education Malaysia Global
  Services)**, the Malaysian student-visa / international-student services
  body.

This note describes the app's architecture and a set of security
observations drawn from examining the publicly distributed package. No
proprietary source code or binary is reproduced or attached.

## 2. Application Architecture

EMGS Hub is a **Capacitor/Ionic hybrid application**. Essentially all of the
business logic lives in the **front-end web bundle**, an **Angular 16 + Ionic
7 single-page application** shipped under `assets/public`.

The native Java layer is a thin shell: a single `BridgeActivity`-style host
whose `onCreate` only initializes `FirebaseAnalytics`. There is no business
logic in the native layer — it exists to host the web view and bridge native
capabilities.

## 3. Native Capabilities

Native features are exposed to the web layer through Capacitor/Cordova
plugins, including:

- **Push notifications** over a dual channel: Firebase Cloud Messaging (FCM)
  and Huawei Mobile Services (HMS).
- **Biometric** authentication (local).
- **Barcode / QR scanning** (journeyapps scanner).
- **File opening**.
- **InAppBrowser**, used for the OAuth login flow.
- **Advanced HTTP** client.

## 4. Front-End Routes (~13 screens)

The SPA defines roughly thirteen routed screens:

- `login`
- `home`
- `profile`
- `digital-id`
- `ikad-qr`
- `emgs-panel-clinic` (list of medical-examination panel clinics)
- `post-arrival-checkin` (post-arrival check-in with face verification)
- `application-history`
- `apply-visa`
- `barcode-scanner`
- `camera-preview`
- `expired-cards`
- `feedback`

## 5. Backend and Interfaces

- **Primary API base:** `mycas.educationmalaysia.gov.my/api/rest`.
- **Auth scheme:** **OAuth 1.0a** with HMAC-SHA1 request signing.
- **Login flow:** opens `/oauth/authorize` inside the InAppBrowser; the app
  registers the custom callback scheme `emgsapp://call-back`.

Enumerable business endpoints (paths only — **no credentials listed**):

- `/student/info`
- `/student/kyc` (face KYC)
- `/student/applicationStatus`
- `/student/notification`
- `/student/checkIn` and `/student/checkInHistory`
- `/student/canCheckIn`
- `/student/ikad`
- `/emgs/clinics`
- `/preEntryUrl`
- `/cms/blocks`

## 6. Key Feature Implementations

- **Login:** three-legged OAuth 1.0a. The success callback scheme is
  `emgsapp://call-back`; rejection returns via `emgsapp://reject`.
- **Digital identity (iKad):** credentials are fetched from `/student/ikad`.
  The resulting QR payload is **plaintext JSON** (including name, nationality,
  gender, university, and course), with **no signature and no encryption** —
  it can be read by simply scanning the code. This is a **privacy risk**.
- **Face verification (KYC):** the front end performs comparison using
  **MediaPipe `face_mesh` (WASM)** and **face-api.js** models, then uploads
  the result.
- **PIN / biometrics:** purely a **local UI gate** backed by local storage;
  it does **not** participate in server-side verification.
- **Push:** dual registration with both FCM and Huawei HMS.

## 7. Risks and Observations

Listed factually; **no actual values are reproduced**.

1. **Hardcoded secrets in the front-end bundle.** The web bundle embeds
   several credentials, all shown here as `[REDACTED]`:
   - **FCM legacy server key** — `[REDACTED]`. **Most severe:** a leaked
     legacy server key allows an attacker to send arbitrary push
     notifications through that FCM project.
   - **OAuth consumer secret** — `[REDACTED]`.
   - **Firebase `apiKey`** — `[REDACTED]`.
   - **Google Maps key** — `[REDACTED]`.
   - **Parse / Scalabl app id and JS key** — `[REDACTED]`.
   - **Supabase anon key** — `[REDACTED]`.
2. **OAuth consumer secret shipped in the package.** Because the consumer
   secret ships with the app, an attacker can forge the HMAC-SHA1 signature
   header offline. (A valid user token is still required to call business
   endpoints, so this alone is not full account access.)
3. **iKad QR is plaintext and unsigned.** As above, the digital-ID QR can be
   read and its personal fields harvested by any scanner, and nothing
   prevents tampering or forgery of the payload structure.
4. **Real personal data left in a debug sample.** A debug sample JSON
   retained the personal data (name, passport number, and application
   number) of a real 2023 applicant. **This data has been removed and is not
   reproduced here;** it is noted only to document that the issue exists.
5. **Third-party distribution.** The examined XAPK originates from a
   third-party distribution channel (an APKPure repackage), i.e. it is **not
   an officially signed package**.

## 8. Methodology and Toolchain

- Full decompilation / export using the **jadx 1.5.1** CLI.
- Semantic / source-reconnaissance tooling over the exported tree.

Only the *method* is described. **No decompiled artifacts, APK/XAPK, DEX, or
JS bundle are included or linked** in this repository.

## 9. Disclaimer

This report is provided **for educational and security-research purposes
only**. It contains **no working secrets and no third-party personal data**.
Any credentials historically embedded in the application should be **rotated
by EMGS**. If you are an EMGS representative and wish to discuss any item
here, responsible-disclosure contact is welcome.
