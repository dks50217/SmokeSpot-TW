# Phase 1 Data Model: 菸槍熱點地圖

對應 spec Key Entities。Domain 層實體不含持久化關注點；下列「儲存欄位」供 Infrastructure (EF Core) 對應。

## SmokeReport（菸味回報）

一筆網友提交的觀察。

| 欄位 | 型別 | 規則 |
|------|------|------|
| Id | Guid | 主鍵 |
| Latitude | double | 必填，21.0–26.5（台灣含離島）緯度範圍 |
| Longitude | double | 必填，118.0–122.5 經度範圍 |
| CreatedAt | DateTimeOffset (UTC) | 必填，伺服器產生 |
| SourceHash | string | 非個資的來源識別（會員 Id 或匿名 session 的雜湊），用於去重/節流/每日上限 |
| MemberId | Guid? | 會員回報時填；匿名為 null |
| ProximityVerified | bool | 回報者定位與所選地點是否相近（FR-014）；無定位或過遠為 false，計權時權重較低 |

**驗證規則 (FluentValidation, Application 層)**:
- 座標落在台灣 bounding box 外 → 拒絕（FR-004）。
- grid cell 以約 **100 公尺**（街區級）量化為「同一地點」單位（CellKey）— Clarify Q1。
- 同一 SourceHash + 同一 grid cell + 10 分鐘窗內已存在 → 視為重複，不新增（FR-008）。
- 同一 SourceHash 當日回報數已達每日上限（預設 ~20）→ 拒絕（FR-015）。
- 匿名回報須附通過的 CAPTCHA token；會員（帶 JWT）免（FR-016）。
- 若附回報者定位且與所選地點距離 ≤ 鄰近門檻 → ProximityVerified=true，否則 false（FR-014）。

**衍生（非儲存）**:
- 衰退權重 = `exp(-ageDays / HalfLifeDays)`，讀取時計算（FR-009）。
- 計權時：未驗證鄰近（ProximityVerified=false）的回報套用較低權重係數（FR-014）；單一 SourceHash 對單一 CellKey 的加權貢獻設上限 `MaxSourceWeightPerCell`（FR-008/SC-006）。

## Hotspot（地點熱點）— 計算聚合，非獨立資料表

由可視範圍內 SmokeReport 依 grid cell 聚合而成。

| 欄位 | 型別 | 說明 |
|------|------|------|
| CellKey | string | grid cell 識別（約 100m 量化後的經緯度，Clarify Q1） |
| CenterLat / CenterLng | double | 聚合代表座標 |
| ReportCount | int | 該 cell 回報總數（FR-005/006） |
| DensityWeight | double | 衰退加權後熱度（FR-009） |
| LastReportedAt | DateTimeOffset | 最近回報時間（FR-006） |

> ponytail: Hotspot 為查詢時即時聚合，不落表。資料量大且查詢變慢時，再加預聚合物化表。

## Feedback（反饋）

對既有標記的修正性回應（FR-010）。

| 欄位 | 型別 | 規則 |
|------|------|------|
| Id | Guid | 主鍵 |
| CellKey | string | 對應的聚合地點 |
| Type | enum {NoLongerSmoky, Inaccurate} | 反饋類型 |
| SourceHash | string | 來源識別（同上，去重用） |
| CreatedAt | DateTimeOffset | 必填 |

**效果**: 當同一 CellKey 收到 **3 個不同 SourceHash** 的有效反饋時，降低該 cell 的 DensityWeight（Clarify Q3；輕量否決，無人工審核 — 見 spec Assumptions）。同一 SourceHash 的重複反饋僅計一次。

## Member（會員）

由 ASP.NET Core Identity 管理（IdentityUser），不自訂額外資料表欄位於本階段。

| 欄位 | 說明 |
|------|------|
| Id | Identity 主鍵 |
| Email / UserName | 登入帳號 |
| PasswordHash | 由 Identity 處理 |

## 關係

- Member 1 — N SmokeReport（MemberId，可空；匿名回報無關聯）。會員可查詢與刪除自己 MemberId 的回報（FR-013，Clarify Q2）；匿名回報無法追蹤或刪除。
- SmokeReport 與 Feedback 透過 CellKey 對應同一聚合地點（非外鍵關聯到單筆 report）。
