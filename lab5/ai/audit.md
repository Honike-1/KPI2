# Аудит: компонентна модель Umasearch

**Дата**: 2026-11-13 · 

**Об'єкт аудиту**: `spec.md`, `component-diagram.mmd` · 

**Статус**: Чернетка

**Звірено з**: ER-моделлю (дані), user-stories і EARS-вимогами (вимоги), sequence UC-10 (взаємодія), state UC-10 (процес), матрицею трасування.

## 2. Трасування компонентів

| Компонент (шар) | Дані (ER) | Вимоги | Взаємодія / процес | Статус |
| --- | --- | --- | --- | --- |
| AuthPages, AuthController, AuthService, AuthMiddleware (UI, API, Сервіс) | `player` | AUTH-*; AC-08 (відновлення пароля немає, тому SMS/Email відсутні) | — | ✅ |
| SearchPage, ResultsList, SearchController, SearchService | `shared_uma`, `shared_card`, `umamusume`, `support_card`, `factor`, `rank` | SRCH-*, RES-* (фільтри, сортування, пагінація по 20) | — | ✅ |
| RentalPage, RentalController, RentalService | `shared_uma`, `shared_card`, `shared_uma_factor`, `player` | RENT-* (ліміт «1 + 1», `created_at` / `updated_at`) | — | ✅ |
| ReportButton, ReportController, ReportService | `player`, `player_report` | REP-01…03 | Sequence UC-10: `isSelfReport`, `existsPair`, `savePair`, `incrementReportsCount`, `reportAccepted` / `reportRejected` / `reportFailed`. State UC-10: `flagged` при `reports_count ≥ 5` | ✅ |
| AdminPanel, AdminController, RoleGuard | — | ADM-*, ADM-05 (перевірка ролі); ADR-0007 (Admin успадковує Player) | — | ✅ |
| ReferenceDataService | `umamusume`, `support_card`, `factor`, `rank` | ADM-* (довідники веде Admin вручну) | — | ✅ |
| ModerationService | `player` (верифікація, `flagged`), `player_report` | ADM-* (`verify`, `listReports`, `unflag`) | State UC-10 (зняття `flagged`) | ✅ |
| PlayerRepository | `player` | AUTH-*, RENT-*, REP-*, ADM-* | — | ✅ |
| ReportRepository | `player_report` | REP-01…03 | Sequence UC-10 | ✅ |
| SharedUmaRepository | `shared_uma`, `shared_uma_factor` | RENT-*, SRCH-* | — | ✅ |
| SharedCardRepository | `shared_card` | RENT-*, SRCH-* | — | ✅ |
| ReferenceRepository | `umamusume`, `support_card`, `factor`, `rank` | SRCH-*, RENT-*, ADM-* | — | ✅ |
| TransactionManager | — | REP-01…03 (неподільна операція) | Sequence UC-10: `beginTransaction`, `commit`, `rollback` | ✅ |
| Database | усі сутності ER + `shared_uma_factor`, `player_report` | усі | — | ⚠️ див. розбіжність №1 |

## 3. Трасування залежностей

| Залежність | Обґрунтування | Статус |
| --- | --- | --- |
| UI → AuthMiddleware (HTTP API) | Єдина точка входу в систему (spec, розд. 2) | ✅ |
| AuthMiddleware → AuthService (`IAuthService`) | Автентифікація актора перед UC | ✅ |
| RoleGuard → AdminController | ADM-05, Admin-маршрути `/admin/*` | ✅ |
| Контролер → відповідний сервіс | Тонкі контролери, правила домену в сервісах | ⚠️ див. розбіжність №3 |
| AuthService → PlayerRepository | AUTH-*, унікальність і валідація на `player` | ✅ |
| SearchService → SharedUma / SharedCard / ReferenceRepository | SRCH-*, RES-*: фільтри по довідниках | ✅ |
| RentalService → SharedUma / SharedCard / Reference / PlayerRepository | RENT-*: ліміт «1 + 1» перевіряється по власнику (`player`) | ✅ |
| ReportService → PlayerRepository, ReportRepository, TransactionManager | Sequence UC-10, REP-01…03 | ✅ |
| ModerationService → PlayerRepository, ReportRepository, TransactionManager | `verify` змінює статус `player`, `unflag` змінює `flagged` і пов'язані дані | ✅ |
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
| Кожен компонент має визначені provided і required | ✅ таблиця spec, розд. 3 |
| Підписи інтерфейсів на діаграмі збігаються зі spec | ✅ |

Зауваження: `AuthMiddleware` вимагає `IAuthService`, але `AuthService` у spec надає лише `register` і `authenticate`. Метод перевірки токена чи сесії, з якого будується `ISession`, не названий. Це не змінює архітектури, але інтерфейс слід уточнити.

## 5. Топ-3 розбіжності

| №   | Де | Суть розбіжності | Вплив |
| --- | --- | --- | --- |
| 1   | ER-модель, `spec.md` (розд. 1), `Database` на діаграмі | Службові таблиці `shared_uma_factor` і `player_report` відсутні в ER, а AC-02 прямо вказує, що сполучних таблиць немає. Spec пояснює їх як фізичну реалізацію N:M, але в моделі даних цього рішення не зафіксовано | Модель даних і архітектура описують різні схеми БД; у `PlayerRepository` / `ReportRepository` / `SharedUmaRepository` з'являється залежність від таблиць, яких немає в ER |
| 2   | Вимога ADM-02.2, `spec.md` (розд. 1) | Немає сховища скриншотів і компонента, що його обслуговує. Admin верифікує `trainer_id` без завантаження файлу, тому ADM-02.2 лишається непокритою | Вимога без компонента, порушення повного трасування вимога → компонент |
| 3   | `spec.md` (розд. 4) проти `component-diagram.mmd` | Spec каже: «Контролер викликає один відповідний сервіс», але `AdminController` залежить від двох сервісів (`ReferenceDataService` і `ModerationService`) | Внутрішня суперечність: правило і діаграма розходяться |