# SmokeSpot TW · 菸槍熱點地圖

群眾眾包的台灣菸味熱點地圖。怕煙味的人可以看地圖避開菸味最重的地方；網友（匿名或登入會員）可以自行標記地點，回報會隨時間自然衰退以反映現況。

## 功能概要

- 🗺️ 互動地圖（Leaflet + OpenStreetMap），以聚合熱點呈現台灣各地菸味密度
- 📍 匿名或會員回報：地圖選點 / 裝置定位
- ⏳ 時間衰退：較新的回報權重較高，舊回報自然淡出
- 🛡️ 防亂回報分層：人機驗證 → 每日上限 → 同地點去重 → 鄰近驗證權重 → 社群反饋壓低

## 技術棧

| 層 | 技術 |
|----|------|
| 前端 | Vue 3 + TypeScript + Vite、Leaflet + markercluster |
| 後端 | .NET 10 ASP.NET Core Web API（Clean Architecture：Domain / Application / Infrastructure / Api） |
| 資料庫 | SQLite（EF Core，介面化可替換） |
| 認證 | 匿名回報 + 會員 JWT（ASP.NET Core Identity）；全程 HTTPS/TLS |
| 人機驗證 | Cloudflare Turnstile（免費） |

## 專案結構

```
backend/
  src/
    SmokeSpot.Domain/          # 實體、領域規則（無外部相依）
    SmokeSpot.Application/     # 使用案例、DTO、介面、驗證
    SmokeSpot.Infrastructure/  # EF Core(SQLite)、Repository、Identity、JWT、CAPTCHA
    SmokeSpot.Api/             # Controllers、DI、HTTPS 設定
  tests/
frontend/
  src/                         # components / pages / stores / services
specs/                         # Spec Kit 規格、計畫、任務（spec.md / plan.md / tasks.md…）
```

## 快速開始

需求：.NET 10 SDK、Node.js 20+

```bash
# 啟用 HTTPS 開發憑證（傳輸加密）
dotnet dev-certs https --trust

# 後端
cd backend
dotnet restore
dotnet ef database update --project src/SmokeSpot.Infrastructure --startup-project src/SmokeSpot.Api
dotnet run --project src/SmokeSpot.Api        # https://localhost:5001

# 前端（另開終端）
cd frontend
npm install
npm run dev                                    # http://localhost:5173
```

詳細驗證情境見 [`specs/001-smoker-density-map/quickstart.md`](specs/001-smoker-density-map/quickstart.md)。

## 開發狀態

規格與計畫以 [Spec Kit](https://github.com/github/spec-kit) 管理，文件位於 `specs/001-smoker-density-map/`：
`spec.md`（需求）、`plan.md`（架構）、`tasks.md`（任務清單）、`research.md`、`data-model.md`、`contracts/api.yaml`。

實作尚未開始；目前為規格定義階段。

## 授權

待定。
