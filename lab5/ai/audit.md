# Аудит: компонентна модель Umasearch

**Дата**: 2026-11-14 · 

**Статус**: Чернетка

**Звірено з**: ER-моделлю (дані), user-stories і EARS-вимогами (вимоги), sequence UC-10 (взаємодія), state UC-10 (процес), матрицею трасування.

## 1. Обсяг і результат

Аудит охоплює кожен компонент, залежність, межу системи та інтерфейс зі `spec.md` і `component-diagram.mmd`. Під час звірки знайдено три розбіжності між специфікацією та діаграмою (розд. 5), усі виправлено в обох файлах. Ще два зауваження стосуються моделі даних і вимог, а не пари spec ↔ діаграма, тому лишаються відкритими (розд. 6).

## 2. Трасування компонентів

| Компонент (шар) | Дані (ER) | Вимоги | Взаємодія / процес | Статус |
| --- | --- | --- | --- | --- |
| AuthPages, AuthController, AuthService, AuthMiddleware (UI, API, Сервіс) | `player` | AUTH-*; AC-08 (відновлення пароля немає, тому SMS/Email відсутні) | `AuthService.validateSession` дає `AuthMiddleware` дані для `ISession` | ✅ |
| SearchPage, ResultsList, SearchController, SearchService | `shared_uma`, `shared_card`, `umamusume`, `support_card`, `factor`, `rank` | SRCH-*, RES-* (фільтри, сортування, пагінація по 20) | — | ✅ |
| RentalPage, RentalController, RentalService | `shared_uma`, `shared_card`, `shared_uma_factor`, `player` | RENT-* (ліміт «1 + 1», `created_at` / `updated_at`) | — | ✅ |
| ReportButton, ReportController, ReportService | `player`, `player_report` | REP-01…03 | Sequence UC-10: `isSelfReport`, `existsPair`, `savePair`, `incrementReportsCount`, `reportAccepted` / `reportRejected` / `reportFailed`. State UC-10: `flagged` при `reports_count ≥ 5` (`setFlagged`) | ✅ |
| AdminPanel, AdminController, RoleGuard | — | ADM-*, ADM-05 (перевірка ролі); ADR-0007 (Admin успадковує Player) | `RoleGuard` передає `AdminController` `IRoleGuard` і `ISession` | ✅ |
| ReferenceDataService | `umamusume`, `support_card`, `factor`, `rank` | ADM-* (довідники веде Admin вручну) | — | ✅ |
| ModerationService | `player` (верифікація, `flagged`), `player_report` | ADM-* (`verify`, `listReports`, `unflag`) | State UC-10 (зняття `flagged`: `setFlagged`) | ✅ |
| PlayerRepository | `player` (зокрема `reports_count`, `flagged`) | AUTH-*, RENT-*, REP-*, ADM-* | Sequence UC-10: `incrementReportsCount`. State UC-10: `setFlagged`. Верифікація: `setVerificationStatus` | ✅ |
| ReportRepository | `player_report` | REP-01…03 | Sequence UC-10: `existsPair`, `savePair`. Адмін-перегляд: `listReports` | ✅ |
| SharedUmaRepository | `shared_uma`, `shared_uma_factor` | RENT-*, SRCH-* | — | ✅ |
| SharedCardRepository | `shared_card` | RENT-*, SRCH-* | — | ✅ |
| ReferenceRepository | `umamusume`, `support_card`, `factor`, `rank` | SRCH-*, RENT-*, ADM-* | — | ✅ |
| TransactionManager | — | REP-01…03 (неподільна операція) | Sequence UC-10: `beginTransaction`, `commit`, `rollback` | ✅ |
| Database | усі сутності ER + `shared_uma_factor`, `player_report` | усі | — | ⚠️ див. відкрите зауваження №1 |

## 3. Трасування залежностей

| Залежність | Обґрунтування | Статус |
| --- | --- | --- |
| UI → AuthMiddleware (HTTP API) | Єдина точка входу в систему (spec, розд. 2) | ✅ |
| AuthMiddleware → AuthService (`IAuthService.validateSession`) | Автентифікація актора перед UC, джерело `ISession` | ✅ |
| AuthMiddleware → контролери (`ISession`) | Контролери знають користувача і роль | ✅ |
| AuthMiddleware → RoleGuard (`ISession`) | `RoleGuard` перевіряє роль з сесії | ✅ |
| RoleGuard → AdminController (`IRoleGuard`, `ISession`) | ADM-05, Admin-маршрути `/admin/*` | ✅ |
| Контролер → сервіси свого домену | Тонкі контролери, правила домену в сервісах; `AdminController` викликає два сервіси (`ReferenceDataService`, `ModerationService`), решта по одному | ✅ |
| AuthService → PlayerRepository | AUTH-*, унікальність і валідація на `player` | ✅ |
| SearchService → SharedUma / SharedCard / ReferenceRepository | SRCH-*, RES-*: фільтри по довідниках | ✅ |
| RentalService → SharedUma / SharedCard / Reference / PlayerRepository | RENT-*: ліміт «1 + 1» перевіряється по власнику (`player`) | ✅ |
| ReportService → PlayerRepository (`incrementReportsCount`, `setFlagged`) | `reports_count` і `flagged` є атрибутами `player` (ER), sequence і state UC-10 | ✅ |
| ReportService → ReportRepository (`existsPair`, `savePair`) | Пара скарг у `player_report`, REP-01…03 | ✅ |
| ReportService → TransactionManager | Неподільна операція «пара + лічильник + `flagged`» | ✅ |
| ModerationService → PlayerRepository (`setVerificationStatus`, `setFlagged`) | `verify` змінює статус `player`, `unflag` змінює `flagged` | ✅ |
| ModerationService → ReportRepository (`listReports`), TransactionManager | Перегляд скарг, транзакційність `unflag` | ✅ |
| ReferenceDataService → ReferenceRepository | Довідники веде Admin вручну | ✅ |
| Репозиторії, TransactionManager → Database (`IDatabase`) | ER-схема, обмеження, транзакції | ✅ |
| Залежності від зовнішніх систем | Відсутні за вхідними документами (AC-08, оплата безкоштовна) | ✅ |

Правило «залежності лише донизу, сервіси не викликають один одного» на діаграмі дотримано: між вузлами сервісів стрілок немає.

## 4. Межа системи та інтерфейси

**Межа системи.** Усередині: UI, API, сервіси, Data Access, БД (subgraph `SYS`). Поза межею: Guest, Player, Admin і браузер. Це узгоджено з ADR-0006 (система не актор) і ADR-0007. Зовнішніх інтеграцій немає, їхня відсутність пояснена (Payment, SMS/Email, OAuth, External API, сховище скриншотів). ✅

**Provided/required-інтерфейси.**

| Перевірка | Результат |
| --- | --- |
| Кожен required має provided-пару | ✅ `HTTP API`, `ISession`, `IRoleGuard`, `I*Service`, `I*Repository`, `ITransactionManager`, `IDatabase` |
| Кожен компонент має визначені provided і required | ✅ таблиця spec, розд. 3 (розбита за компонентами) |
| `ISession` має джерело і всіх споживачів | ✅ `AuthMiddleware` будує її через `validateSession`; контролери отримують від `AuthMiddleware`, `AdminController` від `RoleGuard` |
| Методи репозиторіїв відповідають власнику даних | ✅ `reports_count` і `flagged` у `PlayerRepository`, пари скарг у `ReportRepository` |
| Підписи інтерфейсів і методів на діаграмі збігаються зі spec | ✅ |

## 5. Топ-3 розбіжності (spec ↔ діаграма)

| №   | Де | Суть розбіжності | Вплив |
| --- | --- | --- | --- |
| 1   | `spec.md` (розд. 4) проти `component-diagram.mmd` | Spec каже: «Контролер викликає один відповідний сервіс», але `AdminController` залежить від двох сервісів (`ReferenceDataService` і `ModerationService`) | Правило і діаграма суперечать одне одному |
| 2   | `component-diagram.mmd` (`ReportService` → `ReportRepository`), `spec.md` (розд. 3) | `incrementReportsCount` підписано на зв'язку з `ReportRepository`, хоча `reports_count` є атрибутом `player` в ER. Spec перелічував методи репозиторіїв одним рядком без розподілу, а метод зміни `flagged` для `unflag` не назвав | Метод у репозиторії, який не володіє даними; `unflag` без методу в Data Access |
| 3   | `spec.md` (розд. 3), `component-diagram.mmd` (`AuthMiddleware`, `RoleGuard`) | Spec вимагав `ISession` у всіх контролерів, а на діаграмі її мали лише чотири, `AdminController` отримував тільки `IRoleGuard`. `AuthMiddleware` вимагав `IAuthService`, але `AuthService` надавав лише `register` і `authenticate`, тож `ISession` не мала джерела | Інтерфейс без провайдера, Admin-контролер без сесії |