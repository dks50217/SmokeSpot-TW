---
description: "Task list for 菸槍熱點地圖 implementation"
---

# Tasks: 菸槍熱點地圖 (Smoker Density Map)

**Input**: Design documents from `specs/001-smoker-density-map/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/api.yaml, quickstart.md

**Tests**: 規格未要求 TDD。僅對非顯而易見的邏輯（座標範圍驗證、去重/節流、時間衰退）保留針對性檢查，端到端以 quickstart.md 驗證。

**Organization**: 依 user story 分組，每個 story 可獨立實作與驗證。

## Format: `[ID] [P?] [Story] Description`

- **[P]**: 可平行（不同檔案、無未完成相依）
- **[Story]**: US1 / US2 / US3
- 路徑依 plan.md：`backend/src/SmokeSpot.{Domain,Application,Infrastructure,Api}/`、`frontend/src/`

---

## Phase 1: Setup (Shared Infrastructure)

- [ ] T001 建立後端方案與四層專案：`backend/SmokeSpot.sln`，含 `backend/src/SmokeSpot.Domain`、`SmokeSpot.Application`、`SmokeSpot.Infrastructure`、`SmokeSpot.Api`（.NET 10），設定相依方向 Api→Infrastructure→Application→Domain
- [ ] T002 [P] 初始化前端：`frontend/` 以 Vite + Vue 3 + TypeScript，安裝 vue-router、pinia、leaflet、leaflet.markercluster
- [ ] T003 [P] 設定後端套件：Infrastructure 加 `Microsoft.EntityFrameworkCore.Sqlite`、`Microsoft.AspNetCore.Identity.EntityFrameworkCore`、`Microsoft.AspNetCore.Authentication.JwtBearer`；Application 加 `FluentValidation`
- [ ] T004 [P] 設定 lint/format：後端 `.editorconfig`，前端 ESLint + Prettier 於 `frontend/`

---

## Phase 2: Foundational (Blocking Prerequisites)

**⚠️ CRITICAL**: 所有 user story 開始前必須完成

- [ ] T005 [P] 建立 `SmokeReport` 領域實體於 `backend/src/SmokeSpot.Domain/Entities/SmokeReport.cs`（Id, Latitude, Longitude, CreatedAt, SourceHash, MemberId?, ProximityVerified）（ProximityVerified 對應 FR-014/Clarify Q4）
- [ ] T006 [P] 定義台灣座標範圍值物件/常數於 `backend/src/SmokeSpot.Domain/TaiwanBounds.cs`（lat 21.0–26.5, lng 118.0–122.5）
- [ ] T007 [P] 定義 grid cell 量化規則（CellKey 產生，約 **100m** 街區級，Clarify Q1）於 `backend/src/SmokeSpot.Domain/GridCell.cs`
- [ ] T008 定義 repository 介面於 `backend/src/SmokeSpot.Application/Abstractions/ISmokeReportRepository.cs`（依範圍查詢、新增、依 SourceHash+CellKey+時間窗檢查重複、查單一 SourceHash 當日回報數、查/刪會員自己的回報）
- [ ] T009 建立 EF Core `AppDbContext`（含 Identity）於 `backend/src/SmokeSpot.Infrastructure/Persistence/AppDbContext.cs`，並實作 `SmokeReportRepository`
- [ ] T010 建立初始 EF Core migration 並產生 SQLite 資料庫於 `backend/src/SmokeSpot.Infrastructure/Persistence/Migrations/`
- [ ] T011 設定 Api 進入點 `backend/src/SmokeSpot.Api/Program.cs`：DI 註冊、Kestrel HTTPS + HSTS（傳輸加密）、CORS、全域例外處理、JWT Bearer 驗證 middleware
- [ ] T012 設定環境組態於 `backend/src/SmokeSpot.Api/appsettings.json`（連線字串、JWT 金鑰/發行者、半衰期 HalfLifeDays、去重時間窗、MaxSourceWeightPerCell、GridCellMeters≈100、DailyReportLimit≈20、ProximityThresholdMeters≈200、UnverifiedWeightFactor、CAPTCHA(Turnstile) 金鑰）
- [ ] T013 [P] 前端基礎骨架：`frontend/src/main.ts`、router、`frontend/src/services/api.ts`（fetch 至 https endpoint、附帶 JWT）、Pinia store 殼

**Checkpoint**: 後端可啟動、SQLite 就緒、前端可呼叫 API — user story 可開始

---

## Phase 3: User Story 1 - 在地圖上查看菸味熱點 (Priority: P1) 🎯 MVP

**Goal**: 訪客開啟地圖即看到台灣各地聚合後的菸味熱點，可縮放平移、點擊看明細。

**Independent Test**: 種入數筆回報後開啟前端，地圖 3 秒內呈現聚合標記，點擊顯示回報數與最近時間。

- [ ] T014 [US1] 實作熱點聚合使用案例 `GetHotspotsQuery` 於 `backend/src/SmokeSpot.Application/Reports/GetHotspots.cs`：依可視範圍 bounding box 取回報，按 CellKey 聚合為 ReportCount/CenterLat/CenterLng/LastReportedAt（US1 階段 DensityWeight = ReportCount）
- [ ] T015 [US1] 實作 `GET /api/reports` controller 於 `backend/src/SmokeSpot.Api/Controllers/ReportsController.cs`，回傳 Hotspot DTO（對應 contracts/api.yaml）
- [ ] T016 [P] [US1] 前端 `MapView.vue` 於 `frontend/src/components/MapView.vue`：Leaflet + OSM 圖磚，預設台灣中心
- [ ] T017 [US1] 前端依地圖可視範圍呼叫 `GET /api/reports` 並以 markercluster 呈現熱點於 `MapView.vue`（FR-007 聚合）
- [ ] T018 [US1] 點擊標記顯示回報數與最近時間 popup；無資料時顯示空狀態提示（FR-011）於 `frontend/src/components/HotspotPopup.vue`
- [ ] T019 [US1] reports Pinia store 於 `frontend/src/stores/reports.ts`（依範圍載入、快取）

**Checkpoint**: US1 可獨立運作 — 看地圖避煙的 MVP 完成

---

## Phase 4: User Story 2 - 網友新增菸味地點標記 (Priority: P2)

**Goal**: 匿名或登入會員可在地圖選點送出回報，含台灣範圍驗證與去重防灌水。

**Independent Test**: 未登入選點送出（過 CAPTCHA）→ 201 且地圖密度上升；台灣外座標 → 400；同來源短時重複 → 409；同來源當日超量 → 429。

- [ ] T020 [P] [US2] FluentValidation 驗證器於 `backend/src/SmokeSpot.Application/Reports/CreateReportValidator.cs`：座標必填且落在 TaiwanBounds（FR-004）
- [ ] T021 [US2] 實作 `CreateReportCommand` 於 `backend/src/SmokeSpot.Application/Reports/CreateReport.cs`：計算 CellKey；依 SourceHash+CellKey+時間窗去重（FR-008，重複回 409）；檢查 SourceHash 當日回報數達 DailyReportLimit 則拒（FR-015，回 429）；依回報者定位與地點距離設 ProximityVerified（FR-014）
- [ ] T022 [US2] 匿名來源識別：於 `backend/src/SmokeSpot.Api/` 產生/讀取工作階段 token 並雜湊為 SourceHash（非個資）；會員則用會員 Id
- [ ] T023 [US2] 實作 `POST /api/reports` controller（允許匿名與 JWT），回 201/400/409/429 於 `ReportsController.cs`
- [ ] T024 [P] [US2] 針對性測試 `backend/tests/SmokeSpot.Application.Tests/CreateReportTests.cs`：台灣外拒絕、時間窗內同來源同格去重、當日超量回 429、過遠定位 ProximityVerified=false
- [ ] T025 [P] [US2] Identity 會員註冊/登入：`AuthController.cs` 於 `backend/src/SmokeSpot.Api/Controllers/`，`POST /api/auth/register`、`/login` 發 JWT（對應 contracts）
- [ ] T026 [P] [US2] 前端 `ReportForm.vue` 於 `frontend/src/components/ReportForm.vue`：地圖選點或裝置定位（FR-012）後送出回報，附帶 reporterLat/Lng（FR-014）
- [ ] T027 [P] [US2] 前端登入/註冊 `AuthForm.vue` + auth Pinia store 於 `frontend/src/stores/auth.ts`（存 JWT、附於 api 請求）
- [ ] T028 [US2] 送出後即時刷新地圖密度（SC-003）於 `frontend/src/stores/reports.ts`
- [ ] T038 [US2] 後端 CAPTCHA 驗證（Cloudflare Turnstile）於 `backend/src/SmokeSpot.Infrastructure/Captcha/` + Application 介面 `ICaptchaVerifier`：匿名回報須附通過的 captchaToken，會員免（FR-016/Clarify Q6），失敗回 400
- [ ] T039 [P] [US2] 前端 Turnstile 小工具整合於 `ReportForm.vue`（僅匿名顯示），取得 token 隨回報送出（FR-016）

**Checkpoint**: US1 + US2 皆可獨立運作 — 完整回報循環（含防濫用）完成

### 會員管理自己的回報（FR-013 / Clarify Q2）

- [ ] T040 [US2] 後端 `GET /api/members/me/reports` 與 `DELETE /api/reports/{id}`（僅擁有者，回 204/403/404）於 `backend/src/SmokeSpot.Api/Controllers/`，使用案例於 `SmokeSpot.Application/Reports/`
- [ ] T041 [P] [US2] 前端「我的回報」頁 `frontend/src/pages/MyReports.vue` + store 動作（檢視/刪除，僅登入可見）
- [ ] T042 [P] [US2] 送出回報前知情同意確認彈窗於 `frontend/src/components/ReportForm.vue`：確認後才呼叫 API，取消則不送出（FR-018）

---

## Phase 5: User Story 3 - 標記時效與自然衰退 (Priority: P3)

**Goal**: 較舊回報權重隨時間下降，並提供反饋修正過時/不實標記。

**Independent Test**: 種入 30 天前與當天回報，當天該格 DensityWeight 明顯較高；對某格送 NoLongerSmoky 反饋達門檻後權重下降。

- [ ] T029 [P] [US3] 建立 `Feedback` 領域實體於 `backend/src/SmokeSpot.Domain/Entities/Feedback.cs`（Id, CellKey, Type{NoLongerSmoky,Inaccurate}, SourceHash, CreatedAt）
- [ ] T030 [US3] 於 T014 的聚合改用衰退權重 `exp(-ageDays/HalfLifeDays)`、低於門檻不計入（FR-009）；未驗證鄰近（ProximityVerified=false）套 UnverifiedWeightFactor 較低權重（FR-014）；扣除達門檻反饋的影響；對「單一 SourceHash 對單一 CellKey 的加權貢獻」套用上限 `MaxSourceWeightPerCell`（FR-008/SC-006），於 `GetHotspots.cs`
- [ ] T031 [P] [US3] 針對性測試 `backend/tests/SmokeSpot.Application.Tests/DecayTests.cs`：舊回報權重 < 新回報；3 個不同來源反饋後權重下降；單一來源同格加權不超過 MaxSourceWeightPerCell（SC-006）；未驗證鄰近權重 < 已驗證
- [ ] T032 [US3] 實作 `POST /api/reports/feedback` 於 `ReportsController.cs`（允許匿名與 JWT，依 SourceHash+CellKey 去重），壓低熱度門檻＝同格 **3 個不同 SourceHash**（FR-010/Clarify Q3），新增 Feedback migration
- [ ] T033 [P] [US3] 前端反饋按鈕於 `HotspotPopup.vue`（「已無菸味/不實」送出 feedback）

**Checkpoint**: 三個 user story 皆可獨立運作

---

## Phase 6: Polish & Cross-Cutting Concerns

- [ ] T034 [P] 後端整合測試 `backend/tests/SmokeSpot.Api.IntegrationTests/ReportsApiTests.cs`（WebApplicationFactory：GET/POST/feedback、會員 me/reports 與 DELETE、429 每日上限、匿名缺 CAPTCHA 回 400；CAPTCHA verifier 以假實作注入）
- [ ] T035 [P] 響應式樣式與行動裝置驗證（SC-004 數百標記順暢）於 `frontend/src/`
- [ ] T043 [P] 全頁底部免責聲明元件於 `frontend/src/components/DisclaimerFooter.vue`，嵌入 `frontend/src/App.vue`（FR-017：主觀眾包、非官方數據、平台不負確準責任）
- [ ] T044 [P] 靜態隱私政策頁於 `frontend/src/pages/PrivacyPolicy.vue`，Vue Router 路由 `/privacy`；揭露 Turnstile 第三方資料處理者與目的、申訴信箱（FR-019）
- [ ] T036 [P] README 與 quickstart 對齊於 `specs/001-smoker-density-map/quickstart.md`
- [ ] T037 執行 quickstart.md 全部 10 個驗證情境，確認 SC-001/003/004/006 達標及法律合規（FR-017/018/019）

---

## Dependencies & Execution Order

- **Setup (Phase 1)**: 無相依，先做
- **Foundational (Phase 2)**: 依賴 Setup，阻擋所有 user story
- **US1 (Phase 3)**: 依賴 Foundational；MVP
- **US2 (Phase 4)**: 依賴 Foundational；可與 US1 平行，獨立可測
- **US3 (Phase 5)**: 依賴 Foundational；T030 修改 US1 的 `GetHotspots.cs`（與 US1 同檔，需在 US1 完成後）
- **Polish (Phase 6)**: 依賴所需 user story 完成

### Within Each User Story

- Domain/Models → Application(use case/validator) → Api(controller) → Frontend
- 測試任務（T024/T031）可與同 story 實作平行撰寫

### Parallel Opportunities

- Setup：T002/T003/T004 平行
- Foundational：T005/T006/T007 平行；T013 與後端 T008–T012 平行
- US1：T016（前端 Map）與 T014–T015（後端）平行
- US2：T024/T025/T026/T027/T039/T041/T042 平行（不同檔案）；T038/T040 為後端使用案例
- 跨 story：Foundational 完成後 US1 與 US2 可由不同人平行（US3 的 T030 需待 US1 T014）

---

## Parallel Example: User Story 2

```bash
Task: "CreateReportValidator 於 SmokeSpot.Application/Reports/CreateReportValidator.cs"   # T020
Task: "AuthController 註冊/登入 於 SmokeSpot.Api/Controllers/AuthController.cs"           # T025
Task: "ReportForm.vue 於 frontend/src/components/ReportForm.vue"                          # T026
Task: "AuthForm + auth store 於 frontend/src/stores/auth.ts"                              # T027
```

---

## Implementation Strategy

### MVP First (US1)

1. Phase 1 Setup → 2. Phase 2 Foundational → 3. Phase 3 US1 → **STOP & VALIDATE**（看地圖避煙）→ Demo

### Incremental Delivery

Foundation → US1（MVP，看地圖）→ US2（網友標記）→ US3（衰退與反饋），每段獨立交付不破壞前者。

---

## Notes

- [P] = 不同檔案、無相依
- T030 刻意與 US1 的 `GetHotspots.cs` 同檔（衰退是 US1 聚合的增強），故排在 US1 之後
- 防亂回報分層（spec Clarify 2026-06-19）：CAPTCHA(T038/T039) → 每日上限(T021) → 每格去重(T021) → 鄰近權重(T021+T030) → 反饋壓低 3 來源(T032)
- T038–T042 為澄清後新增（CAPTCHA、會員管理自己的回報、知情同意彈窗）；屬 US2 範圍
- T043–T044 為法律合規新增（底部免責聲明元件、隱私政策頁）；屬 Polish 範圍，不阻擋任何 user story
- ponytail：聚合與衰退皆於讀取時計算、無排程；去重以時間窗+來源+grid，密度計權再對每來源每格設權重上限。CAPTCHA 用免費 Turnstile。資料量大再加預聚合表/空間索引（見 research.md）
- 每完成一個 checkpoint 即可獨立驗證該 story
