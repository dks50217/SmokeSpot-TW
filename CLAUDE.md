<!-- SPECKIT START -->
For additional context about technologies to be used, project structure,
shell commands, and other important information, read the current plan:
`specs/001-smoker-density-map/plan.md`

Active feature: 菸槍熱點地圖 (Smoker Density Map)
- Frontend: Vue 3 + TypeScript + Vite, Leaflet/OpenStreetMap + markercluster
- Backend: .NET 10 ASP.NET Core Web API, Clean Architecture (Domain/Application/Infrastructure/Api)
- Storage: SQLite via EF Core (repository-interfaced, swappable)
- Auth: anonymous reports + member JWT (ASP.NET Core Identity); HTTPS/TLS throughout
<!-- SPECKIT END -->
