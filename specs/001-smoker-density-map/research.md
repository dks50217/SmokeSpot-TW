# Phase 0 Research: 菸槍熱點地圖

所有 Technical Context 項目皆已由使用者指定或以合理預設決定，無未解決 NEEDS CLARIFICATION。

> 已納入 spec Clarifications（2026-06-19）：grid 100m（§11）、會員可管理自己回報（§7）、反饋門檻 3 來源（§8）、軟性鄰近驗證（§9）、每來源每日上限（§8）、匿名 CAPTCHA（§10）。

## 1. 開源免費地圖

- **Decision**: Leaflet + OpenStreetMap 圖磚，標記聚合用 leaflet.markercluster。
- **Rationale**: Leaflet 開源免費、輕量、行動裝置友善；OSM 圖磚免費無金鑰；markercluster 直接解決 FR-007 密集聚合與 SC-004 效能。
- **Alternatives considered**: Google Maps（需金鑰、計費）、Mapbox（免費額度有限需金鑰）。皆不符「開源免費」要求。
- **Note**: 公用 OSM 圖磚有使用政策，正式上線需評估自架或 MapTiler/自家 tile 來源；MVP 用公用即可。

## 2. 菸味密度呈現

- **Decision**: 個別回報座標群以 markercluster 聚合；點擊聚合顯示回報數、最近時間。熱度＝時間衰退加權後的回報數。
- **Rationale**: 直接滿足 FR-002/005/006/007；衰退在後端計算（見下），前端只呈現權重。
- **Alternatives considered**: heatmap 漸層層（leaflet.heat）— 視覺好但難顯示單點明細，列為後續增強。

## 3. 時間衰退 (FR-009)

- **Decision**: 後端查詢時依回報 `CreatedAt` 套用衰退權重 `weight = exp(-ageDays / HALF_LIFE_DAYS)`，預設半衰期 14 天，低於門檻（如 0.1）的回報不計入熱度。半衰期設為可調設定值。
- **Rationale**: 連續、無排程任務即可反映現況；門檻避免一次性事件永久標記（spec User Story 3）。
- **Alternatives considered**: 背景排程硬刪舊資料 — 失去歷史且增加運維；以固定 N 天硬截斷 — 較粗糙。半衰期較平滑。
- **ponytail**: 衰退在讀取時計算，O(n) 掃描。資料量大時再加空間索引／預聚合表。

## 4. Clean Architecture（.NET）

- **Decision**: 四層 — Domain（實體/規則，無相依）、Application（使用案例、介面、DTO、FluentValidation）、Infrastructure（EF Core Sqlite、Repository、Identity、JWT）、Api（Controllers、DI、middleware）。相依方向由外向內。
- **Rationale**: 使用者明確要求；介面化讓 SQLite→其他 DB 可替換而不動內層。
- **Alternatives considered**: 單層 minimal API — 較少程式碼但不符使用者明確的 Clean Architecture 要求。

## 5. 資料庫（SQLite + EF Core）

- **Decision**: EF Core Sqlite provider、Code-First migrations，Repository 介面定義於 Application、實作於 Infrastructure。
- **Rationale**: 滿足「先用 SQLite」；EF Core 換 provider 成本低。
- **座標查詢**: SQLite 無原生地理索引；MVP 以經緯度 bounding-box 範圍查詢（地圖可視範圍）即可，資料量大再導入 spatial 擴充或換 PostGIS。

## 6. 傳輸加密（前後端）

- **Decision**: 全程 HTTPS/TLS（後端 Kestrel HTTPS + HSTS，前端僅打 https endpoint），開發用 dotnet dev-certs。
- **Rationale**: TLS 即業界標準的「前後端資料傳輸加密」；不在應用層重造加密。
- **Alternatives considered**: 應用層自訂欄位加密 — 過度設計，TLS 已涵蓋傳輸層機密性。
- **ponytail**: 傳輸機密性交給 TLS，不自寫加密。需要端對端或欄位級加密時再加。

## 7. 認證：匿名 + 會員 (FR-003 / 使用者要求)

- **Decision**: 回報端點允許匿名（不帶 token）與會員（JWT Bearer）兩種。會員用 ASP.NET Core Identity 註冊登入發 JWT。匿名以伺服器產生的工作階段識別（隨機 token，存於回報的非個資 SourceHash 欄）做去重/節流（FR-008）。
- **Rationale**: 同時滿足「匿名或登入會員回報」；匿名不收個資符合 spec Assumptions。
- **Member 管理（FR-013, Clarify Q2）**: 會員可檢視/刪除自己 MemberId 的回報（`GET /api/members/me/reports`、`DELETE /api/reports/{id}`，僅擁有者）；匿名回報無 MemberId，送出後不可追蹤或刪除。此為登入相較匿名的差異化價值。
- **Alternatives considered**: 強制登入 — 降低參與；純匿名 — 無法提供會員附加價值。兩者並存最符需求。

## 8. 防亂回報：分層防線 (FR-008 / FR-015 / FR-016 / FR-010)

- **Decision**: 五層由前到後——(1) 匿名 CAPTCHA（§10）、(2) 每來源每日上限（預設 ~20 筆，FR-015）、(3) 同來源+同格+10 分鐘窗去重（FR-008）、(4) 鄰近驗證權重（§9，未驗證權重較低）、(5) 社群反饋壓低（同格 **3 個不同 SourceHash** 有效反饋，FR-010 / Clarify Q3）。密度計權時單一來源對單一地點貢獻另設上限（SC-006）。
- **Rationale**: 單一手段都可被繞過（匿名身分可重置），分層才能有效。各層皆以可調設定值控制門檻。
- **ponytail**: 去重/上限/門檻皆為簡單計數查詢，無需專用中介層。真有大規模分散式洗點再上 IP 速率限制/WAF。

## 9. 鄰近驗證 (FR-014, Clarify Q4)

- **Decision**: 軟性檢查。回報可附回報者裝置定位；與所選地點距離 ≤ 鄰近門檻（預設 ~200m）則 `ProximityVerified=true`，否則（含未提供定位）為 false。兩者皆可送出，但未驗證者於密度計權套較低權重係數。
- **Rationale**: 擋掉「在家亂標遠方」的最大破口，又不把關閉定位的使用者排除（spec 主客群是怕煙味者，需可純瀏覽/手動選點）。
- **Alternatives considered**: 硬性拒絕（必須開定位）— 體驗差、誤殺；完全不檢查 — 無法抑制遠端亂標。

## 10. 人機驗證 / CAPTCHA (FR-016, Clarify Q6)

- **Decision**: 匿名回報送出前須通過 CAPTCHA，登入會員免。採 **Cloudflare Turnstile**（免費、無限用量、隱私友善、無廣告追蹤）。後端驗證 token 後才接受回報。
- **Rationale**: 匿名 SourceHash 可藉清 cookie 重置以繞過去重/每日上限，CAPTCHA 是擋自動化腳本 CP 值最高的一層，且免費。
- **Alternatives considered**: Google reCAPTCHA / hCaptcha（皆有免費額度，但 Turnstile 隱私與整合較佳）；不加 CAPTCHA — 腳本可輕易洗點。

## 11. 法律合規策略 (FR-017 / FR-018 / FR-019, Clarify 2026-06-19)

- **Decision**: 三層輕量法遵，全為靜態 UI / 文字，無後端需求：(1) 所有頁面底部固定免責聲明（主觀眾包，非官方數據，平台不負確準責任）；(2) 匿名回報送出前確認彈窗，建立知情同意紀錄；(3) 獨立靜態隱私政策頁揭露 Cloudflare Turnstile 為第三方資料處理者、其收集目的（人機驗證）、申訴聯絡信箱（人工審查）。
- **Rationale**: 符合台灣《個人資料保護法》第 8 條告知義務（揭露 Turnstile 資料流向）及《民法》第 184 條、《刑法》第 310 條侵權風險最小化（免責聲明 + 知情同意）；成本極低，純靜態頁面 + 2 個前端元件，無新 API 端點。
- **SourceHash 個資法適用**: 採裝置指紋（UA + 靜態瀏覽器屬性，不含 IP）；國家發展委員會及主流見解下非可識別個人之資料，無需申報蒐集。
- **Alternatives considered**: 獨立服務條款需使用者勾選同意（成本高、摩擦大）；不加免責聲明（若發生商業名譽爭議時平台防禦性薄弱）。
- **Turnstile 資料傳輸**: 免費方案收集瀏覽器 TLS 指紋、HTTP 標頭、行為信號由 Cloudflare 處理；隱私政策頁須列明此第三方資料處理者及目的。

## 13. Grid cell 粒度 (Clarify Q1)

- **Decision**: 以約 **100 公尺**（街區級）量化經緯度為 CellKey，作為聚合與去重的「同一地點」單位。
- **Rationale**: 貼近「這個路口很多人抽菸」的認知，不會把捷運出口與對街合併；粒度為可調設定值。
- **Alternatives considered**: 50m（過細、叢集破碎）、250m（過粗、跨街合併）。
