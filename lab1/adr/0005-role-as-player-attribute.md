# ADR-0005: Роль як атрибут Player

**Status**: Proposed

**Date**: 2026-10-05

---

## Context

Вхідні дані визначають ролі Guest, Player і Admin. Guest — неавторизований відвідувач, тож зберігати його немає чого. Player і Admin входять за email + password (AC-08), тобто мають однаковий вигляд облікового запису. У вхідних даних не сказано, як різняться записи Player і Admin, чи Admin має `trainer_id` і чи може бути водночас Player (шукає й виставляє). [NEEDS CLARIFICATION: чи може Admin бути одночасно звичайним гравцем і чи потрібен Admin `trainer_id`]

## Decision Drivers

- Admin входить так само, як Player (AC-08)
- AC-02: модель без сполучних таблиць (приклад із лаби: `user_roles`)
- Мінімум нових сутностей
- Узгодженість із `spec.md`, розділ 4

## Considered Options

1. Атрибут `Player.role` ∈ {`player`, `admin`}
2. Окрема сутність `Admin`
3. Сутність `Role` і N:M між Player і Role

## Decision Outcome

Chosen option: **Option 1**, тому що Player і Admin мають однакові поля входу, ролей лише дві й вони взаємовиключні в межах вхідних даних, а Option 3 створила б N:M, який AC-02 забороняє зображати сполучною таблицею. Статус залишається `Proposed`, доки автор не підтвердить, що Admin — це обліковий запис із тими самими полями.

## Consequences

### Positive

- Одна сутність для облікових записів; вхід однаковий для обох ролей.
- Немає сполучних таблиць.

### Negative and risks

- Admin отримує поля Player (`trainer_id`, `trainer_name`), які йому можуть бути не потрібні.
- Якщо з'являться ролі з різними наборами полів, рішення треба переглянути.

### Follow-up actions

- [ ] Отримати підтвердження автора щодо Admin як облікового запису з `trainer_id`
- [ ] Після підтвердження змінити статус на `Accepted`

## Pros and Cons of the Options

### Option 1: `Player.role`

- Pros: просто; без нових сутностей; AC-08 виконується без змін.
- Cons: Admin успадковує поля Player.

### Option 2: сутність Admin

- Pros: чисте розділення полів.
- Cons: друга сутність для входу; вхідні дані не описують окремих полів Admin.

### Option 3: Role + N:M

- Pros: розширюється на багато ролей.
- Cons: потрібна лише одна роль на обліковий запис; N:M без атрибутів тут штучний.

## More Information

- `spec.md`: розділи 2, 4, 6; AC-08.
- `spec-addendum/spec-add-03-model-gaps.md`.