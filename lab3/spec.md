# Специфікація sequence-діаграм: UC-10 «Поскаржитися на гравця» (Umasearch)

**Дата**: 2026-11-02 · 

**Статус**: Чернетка

**Основа**: `spec.md` (Лабораторна 2), `umasearch.feature`, `requirements_ears.md`, `erDiagram.mmd`, ADR-0006...0009.

Позначка ✱ — припущення, якого немає у вхідних документах.

## 1. Обраний use case

**UC-10 Поскаржитися на гравця** (US-05, AC-12). Актор — Player (Admin успадковує його UC, ADR-0007 ✱). Передумова: Player A авторизований (ADR-0009), Player B існує. Результат: пара «скаржник — відповідач» збережена, `reports_count` відповідача +1, при ≥ 5 — `flagged` у тій самій операції. Діаграми описують лише цей UC, не всю систему.

## 2. Учасники та їхня відповідність ER

| Учасник | Тип | Відповідність |
| --- | --- | --- |
| Player A / C (скаржник) | актор | сутність `Player`, `role = player` |
| Player B / P (відповідач) | об'єкт | сутність `Player`: `reports_count`, `verification_status` |
| Player–Player «скаржиться на» | об'єкт | зв'язок N:M `Player }o--o{ Player` (ER; без сполучної таблиці, AC-02) |
| ReportService | сервіс | у ER немає; логічний сервіс застосунку ✱ (ER сервісів не описує) |

## 3. Сценарії та діаграми

| Сценарій | Вимога | Кроки (за Gherkin) | Фрагменти | Діаграма |
| --- | --- | --- | --- | --- |
| SC-040 | REP-01 | A скаржиться на B (count 0): пару збережено, count = 1 | `alt` (REP-02), `alt` (поріг 5) | `sequence/sequence-040` |
| SC-041 | REP-01 | Збій збереження пари або збільшення лічильника: жодна зміна не застосована, count = 2 | `alt` (REP-02), `alt` (яка складова збійна) | `sequence/sequence-041` |
| SC-042 | REP-02 | Повторна скарга A на B відхилена, count = 1 | `alt` (пара вже є) | `sequence/sequence-042` |
| SC-043 | REP-02 | Скарга A на себе відхилена, count = 0 | `alt` (скаржник = відповідач) | `sequence/sequence-043` |
| SC-044 | REP-03 | count 4 → 5, `flagged` у тій самій операції | `alt` (REP-02), `alt` (≥ 5) | `sequence/sequence-044` |
| SC-045 | REP-03 | Те саме для `unverified` і `verified` | `loop` (приклади), `alt`, `alt` | `sequence/sequence-045` |
| SC-055 | ADM-04, REP-03 | Після повернення Admin (UC-14) count 5 → 6, знову `flagged` ✱ | `alt` (REP-02), `alt` (≥ 5) | `sequence/sequence-055` |

Для кожної діаграми є `.mmd` (джерело) і `.svg` (рендер, Mermaid CLI 11.12). SC-055 також тегований UC-14: повернення з `flagged` показано як «Дано», а не кроками діаграми.

## 4. Правила побудови

- **Activation bar.** `ReportService` активується на `submitReport(...)` і деактивується після останнього фрагмента, тож не обривається в гілках `alt`/`loop`. Решта учасників активується лише на час виклику.
- **Неподільна операція (REP-01).** `savePair` і `incrementReportsCount` (та `setVerificationStatus` при ≥ 5) виконуються між `beginTransaction()` і `commit()` / `rollback()` (виділений блок `rect`).
- **loop/par.** Лише `loop` у SC-045 (приклади Gherkin-структури). `par` не застосовано: сценарії UC-10 паралельних гілок не мають.
- **Назви дій:** `submitReport`, `isSelfReport`, `existsPair`, `savePair`, `incrementReportsCount`, `setVerificationStatus`, `reportAccepted`, `reportRejected`, `reportFailed`.