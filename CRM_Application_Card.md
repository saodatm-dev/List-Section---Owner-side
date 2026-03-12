# CRM Application Card — Functional Specification

> **Version**: 2.0  
> **Date**: 2026-03-12  
> **Audience**: Product, Design, Engineering  
> **Language**: English

---

## 1. Overview

The **CRM Application Card** is the workspace for managing rental applications within the CRM pipeline. It has two elements:

- **Pipeline Card** — a compact card in Kanban columns showing tenant, listing, and status at a glance.
- **Detail Panel** — a slide-out panel (opened by clicking a card) with tabs: Overview, Notes, and Viewings.

**User**: Only the property **Owner** interacts with the CRM view. Tenant data is displayed but Tenants have no direct access.

> [!IMPORTANT]
> The CRM `stage` and List View `status` are **independent fields** on the same application record. Changing one does **not** affect the other.
> - **Status** (List View): triage — Unread → Read → Accepted / Rejected
> - **Stage** (CRM View): pipeline — New → Contact → Viewings → Contract → Rejected

---

## 2. CRM Stage Model

| Stage | Key | Meaning |
|-------|-----|---------|
| **Новая заявка** (New Application) | `new` | Just arrived, initial triage |
| **Первичный контакт** (Initial Contact) | `contact` | Owner has reached out to the tenant |
| **Просмотры** (Viewings) | `viewing` | Property viewing scheduled or completed |
| **Заключение контракта** (Contract Closing) | `contract` | Negotiation / signing of tenant agreement |
| **Отказ** (Rejected) | `rejected` | Deal fell through at any stage |

**Stage changes** can be made via:
- **Drag-and-drop** on the Kanban board
- **Stage dropdown** in the Detail Panel header

There are **no transition restrictions** — any stage can move to any other stage. Every stage change:
1. Updates the `stage` field
2. Creates a history entry: `"Стадия: {new}"`
3. Re-renders the board and Detail Panel

---

## 3. Pipeline Board (Kanban)

A **horizontal Kanban board** with 5 columns (one per stage), scrollable when columns exceed the viewport.

**Column header**: stage dot + title + card count badge.

### Drag-and-Drop

| Event | Behavior |
|-------|----------|
| Drag over column | Column body highlights |
| Drag leave | Highlight removed |
| Drop | Stage updated; history logged; board re-renders |

### Filtering

| Filter | Label | Options |
|--------|-------|---------|
| **Stage** | "Стадии" | Все · Новая заявка · Первичный контакт · **Предстоящие просмотры** · **Проведенные просмотры** · Заключение контракта · Отказ |
| **Object** | "Объекты" | Все · _dynamically populated from deal listings_ |

> [!NOTE]
> The "Просмотры" stage splits into two sub-filters: **Предстоящие просмотры** (at least one `upcoming` viewing) and **Проведенные просмотры** (all viewings `done`). Empty columns remain visible when filters are applied.

---

## 4. Pipeline Card

Each card shows:

| Element | Data |
|---------|------|
| **Avatar** | Tenant initials (2 chars), circular |
| **Name** | Full name + verification badge (✓ if `verified: true` — OneID / E-imzo) |
| **Entity Type** | "Физ. лицо" / "Юр. лицо" |
| **Listing** | Thumbnail + property name |
| **Viewing Badge** | "Предстоит" or "Проведен" — only when `stage === 'viewing'` and viewings exist |
| **Timestamp** | Relative time (e.g., "2 часа назад") |
| **Message Indicator** | Chat icon + count (when `msg: true`) |

**Interactions**: Click → opens Detail Panel. Drag → moves to another stage column. Hover → subtle lift effect. Dragging → semi-transparent.

---

## 5. Detail Panel

A **fixed-position side panel** that slides in from the right with a backdrop overlay. Closes via `×` button or backdrop click. Always opens to the **Обзор (Overview)** tab.

### Header

| Element | Description |
|---------|-------------|
| **Avatar** | Circle with tenant initials |
| **Name** | Full tenant name, bold |
| **Entity Type** | "Физическое лицо" / "Юридическое лицо" (full form) |
| **Stage Dropdown** | "Стадия:" + dropdown with all 5 stages; current pre-selected. Changing triggers stage update + history + re-render. |

### Tab Bar

| Обзор (Overview) | ~~Сообщения~~ | Заметки (Notes) | Просмотры (Viewings) |
|---|---|---|---|

> [!NOTE]
> The **Сообщения (Messages)** tab is present but **excluded from this spec** — documented separately.

---

## 6. Overview Tab

Read-only summary with three sections:

### ОБЪЕКТ (Listing)

| Element | Data |
|---------|------|
| **Thumbnail** | Property image |
| **Listing Name** | Property name, bold |
| **Submission Time** | "Заявка: {relative time}" |

### ИНФОРМАЦИЯ (Metadata)

A **2×2 grid**:

| Тип | Стадия |
|-----|--------|
| Entity type | Current CRM stage |
| **Просмотры** | **Заметки** |
| Viewing count | Note count |

### ИСТОРИЯ (History Timeline)

A vertical timeline showing events in **reverse chronological order** (newest first). Each entry has a dot, timestamp, and event text.

**Events logged:**
- `"Новая заявка создана"` — on creation
- `"Стадия: {new}"` — on stage change
- `"Добавлена заметка"` — on note add
- `"Просмотр назначен на {DD.MM} в {HH:MM}"` — on viewing scheduled

> [!NOTE]
> History is **append-only** — entries are never edited or removed. Timestamp format: `"Сегодня, HH:MM"`.

---

## 7. Notes Tab

### Layout (top to bottom)

1. **Existing notes** (newest first)
2. **Rejection Stage Dropdown** (only when `stage === 'rejected'`)
3. **Textarea** — placeholder: "Добавить заметку..." (or "Причина отказа..." in rejected stage)
4. **Suggested Note Chips** (stage-aware)
5. **"Добавить" button**

### Note Actions

| Action | Behavior |
|--------|----------|
| **Add** | Type text → Click "Добавить" → saved with timestamp; history entry created. Empty text is ignored. |
| **Add via chip** | Click chip → instantly saved as note with timestamp + history entry |
| **Edit** | Click pencil → note removed → text populates textarea → user re-saves as new note (original timestamp lost) |
| **Delete** | Click trash → immediate removal, no confirmation |

### Suggested Note Chips (Рекомендации)

Stage-aware templates. Clicking a chip instantly saves it as a note.

| Stage | Templates |
|-------|-----------|
| `new` | _None_ |
| `contact` | "Клиент заинтересован, просит подробности" · "Клиент просит перезвонить позже" · "Клиент хочет назначить просмотр" · "Клиент сравнивает с другими вариантами" · "Клиенту не понравилось" |
| `viewing` | "Просмотр только предстоит" · "Просмотр прошел успешно — клиенту всё понравилось" · "Просмотр прошел неуспешно — клиенту не понравилось" · "Клиент просит повторный просмотр" · "Клиенту не понравилось" |
| `contract` | "Договор отправлен клиенту на рассмотрение" · "Клиент подписал договор" · "Клиент просит изменить условия договора" · "Ожидается оплата залога" · "Клиенту не понравилось" |
| `rejected` | "Клиенту не понравились условия" · "Клиенту не понравилась цена" · "Клиент выбрал другой вариант" · "Клиент передумал арендовать" · "Клиенту не понравилось" |

### Rejection Stage Dropdown (Этап отказа)

Visible only when `stage === 'rejected'`. Records which pipeline stage the rejection occurred at (for analytics).

- **Options**: Первичный контакт · Просмотры · Заключение контракта
- **Default**: "Выберите этап"

---

## 8. Viewings Tab

### Schedule Viewing Form

| Element | Description |
|---------|-------------|
| **Title** | "Назначить просмотр" |
| **Inputs** | Date picker + Time picker (side-by-side) + Address (pre-populated with listing name) |
| **Submit** | "Назначить" — all fields required; empty fields silently block submission |

On submit: viewing created with `status: "upcoming"`, history entry logged, board + panel re-render.

### Viewing Card

Each viewing displays: **date** (day + abbreviated month) · **time** · **address** · **status badge** ("Предстоит" or "Проведен").

**Month abbreviations:** Янв, Фев, Мар, Апр, Май, Июн, Июл, Авг, Сен, Окт, Ноя, Дек

### Viewing Statuses

| Status | Key | Meaning |
|--------|-----|---------|
| **Предстоит** (Upcoming) | `upcoming` | Scheduled, not yet occurred |
| **Проведен** (Done) | `done` | Completed |

> [!NOTE]
> All viewings are created as `upcoming`. Auto-transition to `done` is not implemented — requires backend logic.

**Empty state:** "Нет просмотров" (centered).

---

## 9. Edge Cases

| Area | Rule |
|------|------|
| **Stage independence** | CRM "Rejected" ≠ List View "Rejected" — they may coincide but are not forced to sync |
| **Panel + drag** | If a card's panel is open while it's dragged to another column, the panel re-renders with the new stage |
| **Panel switch** | Opening a panel for a different card fully replaces the previous state; tab resets to Overview |
| **Empty states** | "Нет заметок" / "Нет просмотров" shown centered when no data exists |

---

## 10. Resolved Decisions

| # | Question | Decision |
|---|----------|----------|
| 1 | Should CRM stages have transition restrictions? | **No** — unrestricted for maximum flexibility |
| 2 | Should viewing status auto-transition to "Done"? | **Not for MVP** — requires backend logic |
| 3 | Should the Rejection Stage dropdown be mandatory? | **Yes** — ensures accurate analytics on bottlenecks |
| 4 | Are suggested note chips customizable? | **Not for MVP** — hardcoded per stage |
