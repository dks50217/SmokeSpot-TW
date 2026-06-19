# Quickstart: 菸槍熱點地圖

驗證功能端到端可運作的指引。實作細節見 [data-model.md](./data-model.md) 與 [contracts/api.yaml](./contracts/api.yaml)，任務見 tasks.md。

## Prerequisites

- .NET 10 SDK
- Node.js 20+
- 開發憑證：`dotnet dev-certs https --trust`（啟用 HTTPS 傳輸加密）

## Setup

```bash
# Backend
cd backend
dotnet restore
dotnet ef database update --project src/SmokeSpot.Infrastructure --startup-project src/SmokeSpot.Api   # 建立 SQLite
dotnet run --project src/SmokeSpot.Api    # https://localhost:5001

# Frontend（另開終端）
cd frontend
npm install
npm run dev    # http://localhost:5173，VITE_API_BASE=https://localhost:5001
```

## Validation Scenarios

對應 spec 的 user story 與 success criteria。

1. **看地圖避煙 (US1 / SC-001)**：種入數筆回報後開啟前端，地圖以台灣為中心、3 秒內呈現聚合熱點；點擊聚合顯示回報數與最近時間。
2. **匿名回報 (US2 / FR-003)**：未登入下於地圖點選位置送出回報 → `POST /api/reports` 回 201；重新整理地圖該處密度上升（SC-003 5 秒內）。
3. **會員回報**：`POST /api/auth/register` → `login` 取得 JWT → 帶 Bearer 送 `POST /api/reports` 回 201。
4. **台灣範圍驗證 (FR-004)**：送出經緯度於台灣外（如東京）→ 回 400。
5. **去重/防灌水 (FR-008 / SC-006)**：同來源 10 分鐘內對同一格重複送 → 第二筆回 409，密度不重複累加。
6. **時間衰退 (US3 / FR-009)**：種入一筆 30 天前與一筆當天回報，確認當天那點 densityWeight 明顯較高。
7. **反饋 (FR-010)**：對某熱點送 `POST /api/reports/feedback`（NoLongerSmoky）→ 達門檻後該格 densityWeight 下降。

## Tests

```bash
cd backend && dotnet test          # xUnit：Domain/Application 單元 + Api 整合
cd frontend && npm run test         # Vitest
```

## 預期結果

上述 1–7 全數通過即代表 MVP（US1）與完整回報循環（US2/US3）端到端可運作。
