# Applications Module — Functional Specification

> **Version**: 2.0  
> **Date**: 2026-03-12  
> **Audience**: Product, Design, Engineering  
> **Language**: English

---

## 1. Module Overview

### What It Is

The **Applications** module handles rental requests on the Maydon platform. It connects two user roles — **Tenant** (applicant) and **Owner** (landlord) — through a step-by-step request process.

### What Problem It Solves

- Lets tenants send rental requests and track what happens with them.
- Gives property owners one place to receive, review, and manage all incoming applications.
- Keeps the process clear by using a defined set of stages with specific transition rules.

### How Each Role Uses It

| Role | Primary Use |
|------|-------------|
| **Tenant** | Browse listings → Send application → Track status → Cancel if needed |
| **Owner** | Receive applications → Review → Accept or Reject → Manage pipeline via CRM |

---

## 2. Core Terminology

| Term | Definition |
|------|-----------| 
| **Application (Заявка)** | A rental request sent by a Tenant to an Owner for a specific Listing. Contains a message and goes through a series of stages. |
| **Owner (Владелец)** | A user who owns or manages property listings. The Owner receives and processes incoming applications. Also referred to as "Landlord". |
| **Tenant (Арендатор)** | A user seeking to rent a property. The Tenant sends applications and tracks their statuses. |
| **Listing (Объект / Объявление)** | A published property advertisement created by an Owner. Each Application is tied to exactly one Listing. |
| **Stage (Стадия)** | The current position of an application within the unified 6-stage pipeline (e.g., New, Initial Contact, Viewings, Negotiation, Contract Closing, Rejected). |
| **Protected Stage** | An early stage that cannot receive applications from later stages. In this module: **New Applications** and **Initial Contact** are protected. |
| **Final Stage** | A late stage from which moving an application requires the Owner to confirm the action. In this module: **Contract Closing** and **Rejected** are final. |
| **List View (Список)** | Owner-side view: a vertical status board grouping applications by stage. Used for quick review and stage changes. |
| **CRM View (CRM)** | Owner-side view: a horizontal Kanban board grouping applications by the same stages. Used for detailed management with notes, viewings, and history. |
| **Verification** | A mandatory one-time identity check (via OneID or E-imzo) that a Tenant must complete before their first application is delivered to an Owner. |

---

## 3. Stage Model

### 3.1 Unified Stage System

> [!IMPORTANT]  
> The module operates a **single unified stage system** shared between **List View** and **CRM View**:
> - Both views read/write the same `stage` field on each application.
> - The 6-stage pipeline is: **Новые заявки → Первичный контакт → Просмотры → Переговоры → Заключение контракта → Отказ**.
> - Changing a stage in one view is immediately reflected in the other view.

### 3.2 Tenant-Visible Statuses

Tenants see a **5-status model** showing the application progress from their side:

| Status | Key | Meaning | Triggered By | Final? |
|--------|-----|---------|-------------|--------|
| **Sent** | `sent` | Application submitted; waiting for system delivery to the Owner | System (on successful submit) | No |
| **Unread** | `unread` | Delivered to the Owner but not yet opened | System (automatic) | No |
| **Read** | `read` | Owner has viewed the application | Owner (opens application) | No |
| **Accepted** | `invitation` | Owner approved the application | Owner (clicks "Accept") | Yes |
| **Rejected** | `rejected` | Owner declined the application | Owner (clicks "Reject") | Yes |

> [!NOTE]  
> The Tenant **cannot change** the status of their application. Statuses are driven by Owner actions or system events. The only Tenant action affecting the lifecycle is **Cancel**.

#### Tenant-Side Status Flow

```mermaid
flowchart TD
    START((" ")) --> SENT["Sent"]
    SENT --> UNREAD["Unread"]
    SENT --> READ["Read"]
    UNREAD --> READ
    READ --> ACCEPTED["Accepted"]
    READ --> REJECTED["Rejected"]

    style START fill:#333,stroke:#333,color:#fff
```

### 3.3 Owner-Side Unified Stage Pipeline (List View & CRM View)

The Owner sees a **6-stage pipeline** used identically in both List View and CRM View:

| Stage | Key | Label (RU) | Meaning | Final? |
|-------|-----|------------|---------|--------|
| **New Applications** | `new` | Новые заявки | Incoming, not yet processed | No |
| **Initial Contact** | `contact` | Первичный контакт | Owner has reached out to the tenant | No |
| **Viewings** | `viewing` | Просмотры | Property viewing scheduled or completed | No |
| **Negotiation** | `negotiation` | Переговоры | Active discussion of terms and conditions | No |
| **Contract Closing** | `contract` | Заключение контракта | Negotiation / signing of tenant agreement | Yes |
| **Rejected** | `rejected` | Отказ | Deal fell through at any stage | Yes |

#### 3.3.1 Unified Transition Engine

> [!IMPORTANT]  
> Both **List View** and **CRM View** follow the same transition rules. The same rules apply whether the Owner uses drag-and-drop, the stage dropdown, or the Accept/Reject buttons. Every stage change also creates a timestamped **history entry** on the application record.

Every move attempt has one of three results:

| Outcome | Behavior |
|---------|----------|
| ✅ **Allowed** | Move happens immediately, no prompt |
| ⚠️ **Confirmation** | A dialog asks: *"Вы уверены, что хотите переместить заявку [Name] из стадии «[From]» в «[To]»?"* — Owner must click "Подтвердить" or "Отмена" |
| 🚫 **Blocked** | Nothing happens (drop is ignored, dropdown resets) |

#### 3.3.2 Transition Rules

**Protected stages** — "Новые заявки" and "Первичный контакт" cannot receive applications from later stages:

| Target Stage | Rule |
|---|---|
| **Новые заявки** | 🚫 Blocked from all stages — no application can move here |
| **Первичный контакт** | ✅ Allowed from "Новые заявки" only; 🚫 Blocked from all other stages |

**General transition rules:**

| Transition Type | Outcome | Example |
|---|---|---|
| Forward (natural pipeline order) | ✅ Allowed | Просмотры → Переговоры |
| To "Отказ" from any active stage | ✅ Allowed | Переговоры → Отказ |
| Backward (to an earlier active stage) | ⚠️ Confirmation | Переговоры → Просмотры |
| From final stage (Контракт / Отказ) | ⚠️ Confirmation | Контракт → Переговоры |
| To protected stage | 🚫 Blocked | Просмотры → Новые заявки |
| Same stage | 🚫 Blocked | Просмотры → Просмотры |

#### 3.3.3 Full Transition Matrix (Both Views)

| From ↓ \ To → | New | Initial Contact | Viewings | Negotiation | Contract Closing | Rejected |
|----------------|:---:|:---:|:---:|:---:|:---:|:---:|
| **New** | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Initial Contact** | 🚫 | — | ✅ | ✅ | ✅ | ✅ |
| **Viewings** | 🚫 | 🚫 | — | ✅ | ✅ | ✅ |
| **Negotiation** | 🚫 | 🚫 | ⚠️ | — | ✅ | ✅ |
| **Contract Closing** | 🚫 | 🚫 | ⚠️ | ⚠️ | — | ⚠️ |
| **Rejected** | 🚫 | 🚫 | ⚠️ | ⚠️ | ⚠️ | — |

✅ = Allowed (no prompt) · ⚠️ = Confirmation required · 🚫 = Blocked

---

## 4. Key Actions

### 4.1 Tenant Actions

#### 4.1.1 Send Application

| Property | Details |
|----------|---------|
| **Who** | Tenant (authenticated) |
| **Where** | "Send Application" button on the Listing detail page |
| **Preconditions** | Listing must be active and published |
| **Process** | 1. Tenant clicks "Send Application" → Modal opens<br>2. Tenant writes a message (max 500 characters)<br>3. Tenant clicks "Submit"<br>4. **System checks verification status**:<br>&nbsp;&nbsp;• **Verified** → Application created with status `sent` → Toast notification → Redirect to "My Applications"<br>&nbsp;&nbsp;• **Not verified** → Verification modal opens (see §4.1.4) |
| **Result** | New application created; Owner sees it in their Applications module |
| **Status Change** | Application enters status `sent` |

#### 4.1.2 View Application

| Property | Details |
|----------|---------|
| **Who** | Tenant |
| **Where** | Click any application card in "My Applications" list |
| **Process** | Preview modal opens showing: Owner info, Listing details (photo, name, address, price, specs), Tenant's original message |
| **Status Change** | None |

#### 4.1.3 Cancel Application

| Property | Details |
|----------|---------|
| **Who** | Tenant |
| **Where** | "Cancel Application" button in the preview modal |
| **Preconditions** | Application status is **not** `rejected` |
| **Result** | Application is withdrawn/cancelled |
| **Status Change** | Application is removed or marked as cancelled |
| **Restriction** | The "Cancel" button is **hidden** when the status is `rejected` |

#### 4.1.4 Identity Verification (One-Time)

| Property | Details |
|----------|---------|
| **Who** | Tenant (not yet verified) |
| **Trigger** | Automatic — system launches verification after first application submission |
| **Methods** | **OneID** (redirect to id.egov.uz) or **E-imzo** (in-page digital signature plugin) |
| **On Success** | Verified status saved to tenant profile permanently; application automatically sent to Owner |
| **On Failure/Cancel** | Application is **not** created in the system; tenant returns to the listing page |
| **Business Rule** | Verification is **one-time** — once verified, all future applications skip this step |

### 4.2 Owner Actions

#### 4.2.1 View Application (List View)

| Property | Details |
|----------|---------|
| **Who** | Owner |
| **Where** | Click any application card on the status board |
| **Process** | Modal opens with: Tenant avatar & name, entity type (Individual/Legal), listing + thumbnail + submission time, last tenant note/message, action buttons |
| **Status Change** | If the application was in `new` stage, viewing it implicitly marks it as seen |

#### 4.2.2 Accept Application

| Property | Details |
|----------|---------|
| **Who** | Owner |
| **Where** | "Принять" (Accept) button in the application modal (List View) |
| **Preconditions** | Application stage must not already be `contract` or `rejected` |
| **Result** | Application moves to "Заключение контракта" (Contract Closing) stage; action buttons are hidden on next open |
| **Stage Change** | → `contract` (Final) |
| **Tenant sees** | Status changes to "Accepted" (`invitation`) |

#### 4.2.3 Reject Application

| Property | Details |
|----------|---------|
| **Who** | Owner |
| **Where** | "Отклонить" (Reject) button in the application modal (List View) |
| **Preconditions** | Application stage must not already be `contract` or `rejected` |
| **Result** | Application moves to "Отказ" (Rejected) stage; action buttons are hidden on next open |
| **Stage Change** | → `rejected` (Final) |
| **Tenant sees** | Status changes to "Rejected" (`rejected`) |

#### 4.2.4 Move Between Stages (Drag-and-Drop — List View)

| Property | Details |
|----------|---------|
| **Who** | Owner |
| **Where** | List View — drag application card between stage groups |
| **Rules** | All cards are draggable. The transition rules (§3.3.2) decide whether the drop goes through, asks for confirmation, or is blocked. |
| **History** | Every successful stage change creates a timestamped history entry on the application record |

#### 4.2.5 Move Between Stages (Drag-and-Drop — CRM View)

| Property | Details |
|----------|---------|
| **Who** | Owner |
| **Where** | CRM View — drag card between pipeline columns **or** use stage dropdown in detail panel |
| **Rules** | Same transition rules as List View (§3.3.2). Forward moves go through freely, backward moves ask for confirmation, protected stages are blocked. |
| **History** | Every successful stage change creates a timestamped history entry on the application record |
| **Dropdown cancel** | If the Owner selects a stage that requires confirmation and then cancels, the dropdown resets to the current stage |

#### 4.2.6 CRM Detail Panel Actions

Available in the side panel when clicking a CRM card:

| Action | Tab | Description |
|--------|-----|-------------|
| **Change Stage** | Header | Dropdown to pick a new stage. Depending on the move, it may go through instantly, ask for confirmation, or be blocked (§3.3.2). |
| **View Overview** | Overview | Listing info, application details (entity type, current stage, viewings count, notes count), and full history timeline |
| **Add/Edit/Delete Notes** | Notes | Free-text notes with stage-aware suggested templates; rejection stage dropdown when deal is in "Rejected" stage |
| **Schedule Viewings** | Viewings | Add property viewings (date, time, address) with status tracking (Предстоит / Проведен) |

> [!NOTE]  
> The **Messages** tab has been removed from the CRM Detail Panel tabs. Messages are accessible via the main "Сообщения" module. The detail panel now has **3 tabs**: Overview, Notes, Viewings.

---

## 5. View Modes

### 5.1 List View (Список)

| Property | Details |
|----------|---------|
| **Purpose** | Quick decision-making on incoming applications |
| **Layout** | Vertical status board with collapsible groups: Новые заявки → Первичный контакт → Просмотры → Переговоры → Заключение контракта → Отказ |
| **Data Source** | Application **stage** field (unified with CRM View) |
| **Key interactions** | Click card → Modal with Accept/Reject buttons; Drag-and-drop between stage groups (see transition rules in §3.3.2) |
| **Available to** | Owner only |
| **Filtering** | By stage (with sub-filters for viewing status: Предстоящие просмотры / Проведенные просмотры), by listing |

### 5.2 CRM View (CRM)

| Property | Details |
|----------|---------|
| **Purpose** | Detailed deal management: tracking each application from first contact through contract closing |
| **Layout** | Horizontal Kanban board with 6 columns: Новые заявки → Первичный контакт → Просмотры → Переговоры → Заключение контракта → Отказ |
| **Data Source** | Application **stage** field (unified with List View) |
| **Key interactions** | Click card → Side panel with 3 tabs (Overview, Notes, Viewings); Drag-and-drop between stages and stage dropdown — same rules as List View (§3.3.2) |
| **Available to** | Owner only |
| **Filtering** | By stage (with sub-filters for viewing status: Предстоящие просмотры / Проведенные просмотры), by listing |

### 5.3 How List View and CRM View Relate

```mermaid
graph LR
    subgraph A["Same Dataset (Applications)"]
        APP["Application Record"]
    end
    
    APP -->|"stage field"| LV["List View (6 stages)"]
    APP -->|"stage field"| CRM["CRM View (6 stages)"]
```

> [!IMPORTANT]  
> **List View** and **CRM View** work with the **same data** and the **same stage field**:
> - Both views read and write the same `stage` field.
> - Changing a stage in List View is immediately shown in CRM View, and vice versa.
> - The difference is in **layout** (vertical board vs. horizontal Kanban) and **detail level** (quick modal vs. full side panel with notes, viewings, and history).
>
> **Both views follow the same transition rules** (§3.3.2):
> - Forward moves go through without prompts.
> - Backward moves and moves from final stages ask for confirmation.
> - Protected stages (New, Initial Contact) cannot receive applications from later stages.

### 5.4 Tenant View

| Property | Details |
|----------|---------|
| **Purpose** | View and manage sent applications |
| **Layout** | Flat list of application cards with status badges |
| **Interactions** | Click card → Preview modal; Filter by status or listing; Cancel application from modal |
| **No sub-views** | Tenants see a single flat list only (no CRM, no Analytics) |

---

## 6. Restrictions & Edge Cases

### 6.1 Duplicate Applications

| Rule | Description |
|------|-------------|
| **One active application per listing** | A Tenant can only have **one active application** per Listing at any time. If an application already exists and is not in a final status (`contract` or `rejected`), the "Send Application" button is disabled or hidden. |
| **Re-application after rejection** | If the Owner **rejects** an application, the Tenant is allowed to send a **new application** to the same Listing. |
| **Re-application after cancellation** | If the Tenant **cancels** their own application, they are allowed to send a **new application** to the same Listing. |
| **No re-application after contract** | If the Owner moves an application to **Contract Closing**, the Tenant **cannot** send another application to the same Listing. |

### 6.2 Cancellation Rules

| Scenario | Behavior |
|----------|----------|
| Tenant cancels application (status ≠ `rejected`) | "Cancel Application" button is available; application is withdrawn |
| Tenant tries to cancel a rejected application | "Cancel" button is **hidden**; no action possible |
| Owner cancels / withdraws an application | **Not supported** — Owners can only Accept or Reject |

### 6.3 Final Stages

| Stage | Reversible? | Why |
|-------|------------|-----|
| **Contract Closing** (`contract`) | ⚠️ With confirmation | Represents a commitment; reversal is uncommon but sometimes needed (e.g., contract fell through) |
| **Rejected** (`rejected`) | ⚠️ With confirmation | Represents a deliberate decision; reopening requires the Owner to explicitly confirm |

**Consequences:**
- Cards in final stages **are draggable** in both views, but moving them always triggers a **confirmation dialog**.
- Accept/Reject buttons are **hidden** in the modal for applications already in final stages.
- Moving from a final stage to a **protected stage** (New, Initial Contact) is **blocked** regardless of confirmation.

### 6.4 Protected & Blocked Transitions (Both Views)

| Transition | Result | Mechanism |
|-----------|--------|-----------|
| Any stage → New (`new`) | 🚫 Blocked | Drop is ignored |
| Any stage (except New) → Initial Contact (`contact`) | 🚫 Blocked | Drop is ignored; dropdown resets |
| Contract Closing → Active stage (Viewings, Negotiation) | ⚠️ Confirmation | Confirmation dialog; Owner must explicitly approve |
| Rejected → Active stage (Viewings, Negotiation, Contract) | ⚠️ Confirmation | Confirmation dialog; Owner must explicitly approve |
| Contract Closing / Rejected → Protected stage (New, Contact) | 🚫 Blocked | Protected stage rule takes precedence over confirmation |

### 6.5 Listing Deactivation

| Rule | Description |
|------|-------------|
| **Scope** | Only applications in **non-final** stages (`new`, `contact`, `viewing`, `negotiation`) are affected. Applications already in `contract` or `rejected` stage **remain unchanged**. |
| **Automatic cancellation** | When a Listing is deactivated, all affected applications are **automatically cancelled**. Tenants and Owners can no longer interact with these applications. |
| **Tenant notification** | Tenants with cancelled applications receive a notification that the listing is no longer available. |
| **Reactivation restores applications** | If the Owner **reactivates** the Listing, previously cancelled applications are **restored** to the stage they held before deactivation. |

### 6.6 Verification Edge Cases

| Scenario | Behavior |
|----------|----------|
| Verification fails (OneID / E-imzo) | Application is **not created**; tenant sees error with Retry / Cancel options |
| Tenant cancels verification | Application is **not created**; no data is saved |
| Already verified tenant | Submits immediately; no verification modal shown |
| Verification status persistence | Stored permanently on the tenant profile; applies to all future applications |

### 6.7 Empty States

| Scenario | Behavior |
|----------|----------|
| No applications match active filter | Message: "No applications matching selected filters" |
| Stage group is empty (List View, **no filter** active) | Group still shown with header and count = 0. **Why:** the Owner sees the full pipeline structure at all times, so they know where cards can be dragged to. |
| Stage group is empty (List View, **filter active**) | Group is **hidden entirely**. **Why:** when filtering, the Owner is focused on a specific data subset — empty groups add visual noise and are removed to keep the view clean. |

---

## 7. Resolved Decisions

The following items were originally open questions. They have been resolved with the decisions documented below.

| # | Question | Decision |
|---|----------|----------|
| 1 | **Should the CRM stage and List View status be linked?** | **Yes — they are now unified.** Both List View and CRM View use the same `stage` field with the same 6-stage pipeline. This eliminates confusion from having two independent classification systems. The difference between views is now purely presentational (vertical board vs. horizontal Kanban) and interactional (triage modal vs. full detail panel). |
| 2 | **Can an Owner undo a Reject decision?** | **Yes — with confirmation.** Both views allow moving applications out of "Rejected", but the Owner must confirm the action via a dialog. This keeps flexibility while preventing accidental changes. The Tenant can also re-apply on their own (per §6.1). |
| 3 | **What happens after Accept?** | **Stage changes to `contract` (Contract Closing).** The Owner uses CRM tabs (Notes, Viewings) to coordinate next steps manually. Contract creation and automated follow-ups can be added as a separate module later. |
| 4 | **Is there a notification system?** | **Yes — in-app notifications at minimum.** Tenants receive in-app notifications when their application status changes (Read, Accepted, Rejected). Push and email notifications can be added later. |
| 5 | **What is the difference between Tenant "Sent" and "Unread"?** | **"Sent" is a brief transitional state.** `sent` = application submitted but not yet delivered/processed. `unread` = delivered to the Owner's dashboard but not yet opened. The `sent` → `unread` transition is near-instant (backend processing). |
| 6 | **Can Owners archive old applications?** | **Not for MVP.** Applications in final stages remain in their groups. Filtering by listing is sufficient for managing volume. Archiving can be revisited if performance becomes an issue. |
| 7 | **Is the CRM "Rejected" the same as List View "Rejected"?** | **Yes — they are now the same.** With the unified stage model, there is a single `rejected` stage used in both views. Moving an application to "Отказ" in either view is reflected everywhere. |

---

## 8. Stage-Aware Features

### 8.1 Suggested Note Templates

Each stage provides context-aware suggested notes that the Owner can click to quickly add:

| Stage | Suggested Notes |
|-------|----------------|
| **Initial Contact** | "Клиент заинтересован, просит подробности", "Клиент просит перезвонить позже", "Клиент хочет назначить просмотр", "Клиент сравнивает с другими вариантами", "Клиенту не понравилось" |
| **Viewings** | "Просмотр только предстоит", "Просмотр прошел успешно", "Просмотр прошел неуспешно", "Клиент просит повторный просмотр", "Клиенту не понравилось" |
| **Negotiation** | "Обсуждаем цену и условия", "Клиент просит скидку", "Клиент согласен с условиями", "Клиент просит изменить сроки", "Клиент думает, обещал вернуться" |
| **Contract Closing** | "Договор отправлен клиенту на рассмотрение", "Клиент подписал договор", "Клиент просит изменить условия договора", "Ожидается оплата залога", "Клиенту не понравилось" |
| **Rejected** | "Клиенту не понравились условия", "Клиенту не понравилась цена", "Клиент выбрал другой вариант", "Клиент передумал арендовать", "Клиенту не понравилось" |

### 8.2 Rejection Stage Tracking

When an application is in the "Rejected" stage, the Notes tab includes an additional **"Этап отказа" (Rejection Stage)** dropdown. This allows the Owner to record at which pipeline stage the rejection occurred, with options: Первичный контакт, Просмотры, Переговоры, Заключение контракта.

### 8.3 Viewing Status Badge

For applications in the "Просмотры" (Viewings) stage, the CRM pipeline card displays a viewing status badge:
- **Предстоит** (Upcoming) — at least one viewing has `upcoming` status
- **Проведен** (Done) — all viewings have `done` status

### 8.4 Analytics View

The **Analytics** tab shows key numbers and trends across the pipeline:

| Component | Description |
|-----------|-------------|
| **Requests by Month** | Bar chart showing application volume over the last 6 months (Sep–Feb), with total count, current month count, and month-over-month change percentage |
| **Conversion Funnel** | Vertical stepped funnel showing progression: Новые заявки → Первичный контакт → Просмотры → Переговоры → Контракт, with drop-off percentages between stages |
| **Bottleneck Detection** | Identifies the stage transition with the highest loss percentage, along with the most common rejection reason |
| **Recommendation** | System-generated suggestion based on the most common rejection reason |
| **Object Filter** | Dropdown to filter all analytics data by specific listing |
