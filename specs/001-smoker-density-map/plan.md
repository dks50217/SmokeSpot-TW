# Implementation Plan: 菸槍熱點地圖 (Smoker Density Map)

**Branch**: `001-smoker-density-map` | **Date**: 2026-06-19 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/001-smoker-density-map/spec.md`

## Summary

群眾眾包的台灣菸味熱點地圖：訪客看地圖避開菸味重的地方、網友（匿名或登入會員）標記地點、回報隨時間衰退以反映現況。前端 Vue 3 + Leaflet/OpenStreetMap，後端 .NET (ASP.NET Core Web API) 採 Clean Architecture，資料先用 SQLite（EF Core），全程 HTTPS/TLS 傳輸加密，會員以 JWT 認證，匿名回報以工作階段層級識別做去重節流。

## Technical Context

**Language/Version**: Frontend TypeScript 5.x / Vue 3.5；Backend C# / .NET 10

**Primary Dependencies**: Frontend — Vite、Vue Router、Pinia、Leaflet（OSM 圖磚）、leaflet.markercluster；Backend — ASP.NET Core Web API、EF Core (Sqlite)、ASP.NET Core Identity + JWT Bearer、FluentValidation

**Storage**: SQLite（EF Core，Code-First migrations）。資料層介面化，後續可換 PostgreSQL/SQL Server 而不動 Domain/Application

**Testing**: Backend — xUnit + FluentAssertions（Domain/Application 單元測試、WebApplicationFactory 整合測試）；Frontend — Vitest + Vue Test Utils

**Target Platform**: 響應式 Web，行動裝置瀏覽器優先、桌面亦支援。後端跨平台（Kestrel）

**Project Type**: Web application（frontend + backend 分離）

**Performance Goals**: 地圖含既有密度於 3 秒內呈現（SC-001）；新回報 5 秒內反映（SC-003）；單區數百標記縮放平移順暢（SC-004，靠聚合）

**Constraints**: 全程 HTTPS/TLS；匿名亦可回報且不收個資；座標限台灣（含離島）地理範圍；Clean Architecture 分層依賴方向內向

**Scale/Scope**: 初期單機 SQLite，社群眾包規模（千級回報量）；3 個 user story、12 條 FR

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

專案 `constitution.md` 仍為未填寫的樣板（佔位符），尚無已批准的原則可強制。**Gate 狀態：N/A（無已批准之憲章）**，無違反項。採用的 Clean Architecture 分層、介面化資料層、測試優先屬一般良好實務，與日後可能訂定之原則方向一致。建議後續以 `/speckit-constitution` 正式批准原則。

## Project Structure

### Documentation (this feature)

```text
specs/001-smoker-density-map/
├── plan.md              # 本檔
├── research.md          # Phase 0 技術決策
├── data-model.md        # Phase 1 實體模型
├── quickstart.md        # Phase 1 啟動與驗證指引
├── contracts/           # Phase 1 API 合約 (OpenAPI)
│   └── api.yaml
└── tasks.md             # /speckit-tasks 產生（非本指令）
```

### Source Code (repository root)

```text
backend/
├── src/
│   ├── SmokeSpot.Domain/          # 實體、值物件、領域規則（無外部相依）
│   ├── SmokeSpot.Application/     # 使用案例、DTO、介面（IReportRepository…）、驗證
│   ├── SmokeSpot.Infrastructure/  # EF Core (Sqlite)、Repository 實作、Identity、JWT
│   └── SmokeSpot.Api/             # ASP.NET Core 進入點、Controllers、DI、HTTPS 設定
└── tests/
    ├── SmokeSpot.Domain.Tests/
    ├── SmokeSpot.Application.Tests/
    └── SmokeSpot.Api.IntegrationTests/

frontend/
├── src/
│   ├── components/      # MapView、ReportMarker、ReportForm、AuthForm
│   ├── pages/          # HomeMap、Login、Register
│   ├── stores/         # Pinia：reports、auth
│   ├── services/       # api client（fetch + JWT、HTTPS）
│   └── types/
└── tests/
```

**Structure Decision**: Option 2（Web application）。前後端各自獨立倉內目錄。後端依 Clean Architecture 四層，相依方向一律由外向內（Api → Infrastructure → Application → Domain；Domain 不相依任何外層）。前端以 Pinia 管理地圖與認證狀態，services 層集中處理加密傳輸與 JWT。

## Complexity Tracking

> 無 Constitution 違反項，無需記錄。

四層後端為使用者明確要求（Clean Architecture），非投機性抽象，故不視為過度設計。SQLite 經 EF Core 介面化以滿足「先用 SQLite、後可替換」需求。
