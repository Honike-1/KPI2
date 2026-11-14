# Специфікація архітектури: Umasearch

**Дата**: 2026-11-14 · 

**Статус**: Чернетка · 

**Основа**: ER-модель, user-stories, EARS, матриця трасування, sequence і state UC-10

## 1. Модулі та шари системи

Архітектура п'ятишарова, залежності йдуть лише зверху вниз.

| Шар | Компоненти | UC |
| --- | --- | --- |
| UI | AuthPages, SearchPage, ResultsList, RentalPage, ReportButton, AdminPanel | UC-01…UC-14, UC-19 |
| API / Backend | AuthController, SearchController, RentalController, ReportController, AdminController; AuthMiddleware, RoleGuard (ADM-05) | UC-01…UC-14, UC-19 |
| Сервіси | AuthService, SearchService, RentalService, ReportService, ReferenceDataService, ModerationService | UC-01…UC-14, UC-19 |
| Data Access | PlayerRepository, ReportRepository, SharedUmaRepository, SharedCardRepository, ReferenceRepository, TransactionManager | усі |
| База даних | player, umamusume, support_card, factor, rank, shared_uma, shared_card + службові `shared_uma_factor`, `player_report` | усі |
| Зовнішні інтеграції | немає | — |

- **UI** не містить бізнес-правил: збирає введення, показує відповіді, ховає недоступні дії за роллю.
- **API:** контролери тонкі. Ендпоінти: `POST /auth/register|login`, `GET /search/umas|cards`, `/rentals/uma|card` (POST, PUT, DELETE), `POST /players/{id}/reports`, `/admin/*`.
- **Сервіси** містять усі правила домену. `ReportService` реалізує sequence і state UC-10: `isSelfReport`, `existsPair`, неподільна операція «пара + `reports_count` +1 + `flagged` при ≥ 5» (REP-01…03). Решта сервісів тримають правила з EARS: валідацію й унікальність (`AuthService`), фільтри, сортування і пагінацію по 20 (`SearchService`), ліміт «1 + 1» і `created_at` / `updated_at` (`RentalService`), довідники (`ReferenceDataService`), верифікацію і `unflag` (`ModerationService`).
- **Data Access:** лічильник `reports_count` і прапорець `flagged` є атрибутами `player` (ER), тому ними керує `PlayerRepository`; `ReportRepository` працює лише з парами скарг (`player_report`).
- **База даних** відповідає ER (усі `id` типу `uuid`). N:M-зв'язки в ER без власних атрибутів (AC-02), але реляційна БД зберігає їх у двох службових таблицях: `shared_uma_factor` (PK пара) і `player_report` (PK пара `reporter_id`, `reported_id`, CHECK `reporter_id ≠ reported_id`).
- **Зовнішні інтеграції:** Payment (рентал безкоштовний), SMS/Email (відновлення пароля немає, AC-08), OAuth (вхід лише за email + password) та External API гри (довідники веде Admin вручну) у вхідних документах відсутні й не вводяться. Сховища скриншотів теж немає: ADM-02.2 лишається непокритою, тому Admin верифікує `trainer_id` без завантаження файлу.

## 2. Межа системи

- **Всередині:** UI, API, сервіси, Data Access, БД.
- **Поза межею:** актори Guest, Player, Admin (усі люди, Admin успадковує Player, ADR-0007) і браузер як середовище виконання UI. Гра Umamusume лише контекст, без технічної інтеграції.
- Система не є актором (ADR-0006): `flagged` при `reports_count ≥ 5`, перевірка ролі й валідація — поведінка сервісів у межах UC актора.
- Єдина точка входу — HTTP-ендпоінти API; прямого доступу до БД ззовні немає.

## 3. Provided та Required інтерфейси

| Компонент | Provided | Required |
| --- | --- | --- |
| UI | екрани й дії за ролями | HTTP API |
| AuthMiddleware | `ISession` (користувач і роль) | `IAuthService` (`validateSession`) |
| RoleGuard | `IRoleGuard`, `ISession` (передає далі Admin-контролеру) | `ISession` |
| AuthController, SearchController, RentalController, ReportController | HTTP-ендпоінти | свій сервіс (`IAuthService`, `ISearchService`, `IRentalService`, `IReportService`), `ISession` |
| AdminController | HTTP-ендпоінти `/admin/*` | `IReferenceDataService`, `IModerationService`, `IRoleGuard`, `ISession` |
| AuthService | `register`, `authenticate`, `validateSession` | `IPlayerRepository` |
| SearchService | `searchUmas`, `searchCards` | `ISharedUmaRepository`, `ISharedCardRepository`, `IReferenceRepository` |
| RentalService | `createUma`, `createCard`, `replace`, `remove` | `ISharedUmaRepository`, `ISharedCardRepository`, `IReferenceRepository`, `IPlayerRepository` |
| ReportService | `submitReport` → `reportAccepted` / `reportRejected` / `reportFailed` | `IPlayerRepository`, `IReportRepository`, `ITransactionManager` |
| ReferenceDataService | `create*` | `IReferenceRepository` |
| ModerationService | `verify`, `listReports`, `unflag` | `IPlayerRepository`, `IReportRepository`, `ITransactionManager` |
| PlayerRepository | CRUD і пошук `player`; `incrementReportsCount`, `setVerificationStatus`, `setFlagged` | `IDatabase` |
| ReportRepository | `existsPair`, `savePair`, `listReports` | `IDatabase` |
| SharedUmaRepository, SharedCardRepository, ReferenceRepository | CRUD і пошук | `IDatabase` |
| TransactionManager | `beginTransaction`, `commit`, `rollback` | `IDatabase` |
| База даних | `IDatabase` (SQL, обмеження, транзакції) | — |

## 4. Залежності між компонентами

`UI → AuthMiddleware/RoleGuard → Контролери → Сервіси → Репозиторії → БД`

- Контролер викликає сервіси лише свого домену: `AuthController`, `SearchController`, `RentalController`, `ReportController` по одному сервісу, `AdminController` два (`ReferenceDataService` і `ModerationService`). Admin-маршрути додатково проходять `RoleGuard`.
- `AuthMiddleware` будує `ISession` через `IAuthService.validateSession`. Контролери отримують `ISession` від `AuthMiddleware`, `AdminController` отримує її від `RoleGuard` разом з `IRoleGuard`.
- `AuthService` залежить від `PlayerRepository`. `SearchService` і `RentalService` залежать від репозиторіїв умок, карток і довідників (`RentalService` ще й від `PlayerRepository`).
- `ReportService` і `ModerationService` залежать від `PlayerRepository`, `ReportRepository` і `TransactionManager`. `ReferenceDataService` залежить від `ReferenceRepository`.
- Усі репозиторії та `TransactionManager` залежать лише від БД.

**Правила:** залежності лише донизу, без пропуску шарів; сервіси не викликають один одного, спільні перевірки (наприклад, довідники) йдуть через репозиторії; верхній шар залежить від інтерфейсу, а не від реалізації; залежностей від зовнішніх систем немає.