# Module 13 — Notifications
## Comprehensive Functional Specification & Forensic Reconstruction

**Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md) (`DEZ-STD-FS-001`)  
**Domain:** Notification Infrastructure, Email Delivery, Real-Time Push Alerts, and In-App Ephemeral Center  
**Legacy Package:** `src/worker/consumers/email.consumer.ts`, `src/websocket/`, `frontend/src/store/notificationStore.ts`  
**Reconstruction State:** `COMPLETED & FORENSICALLY VERIFIED`  
**Investigation Mode:** Strict Forensic Codebase Extraction (Zero Hallucination, Zero Assumptions)  

---

## SECTION A: Module Overview & Scope Boundaries

### A.1 Functional Definition
Module 13 governs the communication, notification, and alerting infrastructure across the Dezolver platform. In an enterprise technical education and assessment environment, the notification subsystem is responsible for delivering transactional emails (account provisioning, password resets, event reminders, dunning warnings), dispatching real-time browser push notifications upon asynchronous milestone completions (submission grading, certificate issuance, contest state changes), and providing learners and staff with a centralized, searchable in-app notification center.

### A.2 Implementation Reality & Divergence
A forensic audit of the legacy codebase uncovers an asymmetric, hybrid notification architecture characterized by robust email worker execution alongside an entirely non-persistent in-app notification subsystem:
1. **Zero Database Persistence for In-App Notifications:** Across all 11 PostgreSQL schemas declared in `prisma/` (`identity`, `tenancy`, `curriculum`, `assessment`, `problems`, `events`, `credentials`, `billing`, `platform`, `paths`, `analytics`), there is **no `Notification` or `UserNotification` table**. The backend database maintains zero records of in-app alerts.
2. **Missing Notification Ingress API:** There are **zero backend endpoints** to list, filter, query, or update notifications. Endpoints such as `GET /notifications` or `PATCH /notifications/:id/read` do not exist.
3. **Purely Ephemeral Client-Side Zustand Architecture:** In-app notifications exist exclusively in browser RAM via a client-side Zustand store (`frontend/src/store/notificationStore.ts`). This store is dynamically populated in real-time by incoming WebSocket frames in `WebSocketProvider.tsx`.
4. **Session Erasure on Browser Refresh:** Because notifications are not persisted in PostgreSQL or Redis, refreshing the browser, closing the tab, or signing in from another device immediately resets the notification center to an empty state (`[]`).
5. **Functional Asynchronous SES Email Pipeline:** Transactional emails are processed via BullMQ queue `email` and dispatched by `EmailConsumer` (`src/worker/consumers/email.consumer.ts`) using AWS SES. The consumer supports both managed SES templates and hardcoded inline text fallback templates.
6. **Critical Password Reset Dead-End:** While `AuthService` emits an Outbox event `PasswordResetRequested`, **zero consumers listen for this event**. Consequently, password reset emails are never enqueued or sent.
7. **Unimplemented Fanout Queue:** Queue `notification-fanout` is defined in `QueueModule` and injected into `RegistrationReminderService`, but has **zero worker processors and zero jobs enqueued to it**.
8. **Dunning Email Address Defect:** In `DunningService`, overdue subscription warning jobs pass `sub.tenantId` (a raw UUID) as the recipient email address, causing SES job failures.

### A.3 Scope Boundaries

#### In-Scope (Implemented & Traced in Source Code)
- BullMQ `email` queue processing and `EmailConsumer` execution.
- Amazon Simple Email Service (SES) integration with templated and inline text modes.
- Hardcoded email templates (`welcome`, `password-reset`, `invitation`, `submission-graded`, `certificate-issued`).
- Identity email triggers (`welcome` email on signup and invitation acceptance).
- Contest reminder email cron (`event_reminder_24h` and `event_reminder_1h` via `RegistrationReminderService`).
- Subscription dunning email cron (`payment_recovery` via `DunningService`).
- Real-time WebSocket push notifications over user channels (`user:${userId}`).
- Client-side in-memory Zustand store (`notificationStore.ts`) with 50-item buffer cap.
- Frontend UI components: `NotificationCenter.tsx`, `NotificationItem.tsx`, and Sonner toast popups.

#### Out-of-Scope (Proven NOT IMPLEMENTED in Legacy Code)
- **Database Notification Storage:** No tables exist in PostgreSQL for notifications.
- **REST API Endpoints:** No `GET /notifications`, `PATCH /notifications/:id/read`, or `DELETE /notifications`.
- **Cross-Device Notification Sync:** Notifications received on Device A cannot be viewed on Device B.
- **Persistent Unread Badges:** Unread counts reset to 0 on every page reload.
- **Notification Fanout Worker:** No consumer exists for `notification-fanout`.
- **SMS / Push Notification Gateways:** No Twilio, Firebase Cloud Messaging (FCM), or Apple APNs integrations.
- **User Notification Preferences:** No notification settings, category toggles, or unsubscribe preferences.

---

## SECTION B: Architecture & Dependency Topology

### B.1 Notification Routing Topology

```
   ┌────────────────────────────────────────────────────────┐
   │                   PLATFORM EVENTS                      │
   │  - UserCreated / UserInvitationAccepted (Identity)     │
   │  - PasswordResetRequested (Identity — UNHANDLED)       │
   │  - RegistrationReminder (Events Cron)                  │
   │  - SubmissionGraded (Assessment)                       │
   │  - CertificateIssued (Credentials)                     │
   │  - DunningDayTriggered (Billing Cron)                  │
   └───────────┬────────────────────────────────┬───────────┘
               │                                │
               ▼                                ▼
   ┌────────────────────────┐       ┌────────────────────────┐
   │     EMAIL PIPELINE     │       │   REAL-TIME WS PUSH    │
   └───────────┬────────────┘       └───────────┬────────────┘
               │                                │
               ▼                                ▼
   ┌────────────────────────┐       ┌────────────────────────┐
   │  BullMQ Queue: 'email' │       │   RealtimeBroadcaster  │
   │  Job Name: 'send'      │       │   (Redis PubSub)       │
   └───────────┬────────────┘       └───────────┬────────────┘
               │                                │
               ▼                                ▼
   ┌────────────────────────┐       ┌────────────────────────┐
   │     EmailConsumer      │       │   SubmissionsGateway   │
   │ (src/worker/consumers) │       │   (Socket.io Gateway)  │
   └───────────┬────────────┘       └───────────┬────────────┘
               │                                │
               ▼                                ▼
   ┌────────────────────────┐       ┌────────────────────────┐
   │     AWS SES Client     │       │   WebSocketProvider    │
   │ (Templated or Inline)  │       │ (Browser Client Frame) │
   └────────────────────────┘       └───────────┬────────────┘
                                                │
                                                ▼
                                    ┌────────────────────────┐
                                    │  useNotificationStore  │
                                    │  (In-Memory Zustand)   │
                                    └───────────┬────────────┘
                                                │
                                                ▼
                                    ┌────────────────────────┐
                                    │   NotificationCenter   │
                                    │   (Bell Icon Popover)  │
                                    └────────────────────────┘
```

### B.2 Dependency Inventory

#### Internal Module Dependencies
- **`src/worker/consumers/email.consumer.ts`**: BullMQ queue worker for sending emails.
- **`src/shared/queue/queue.module.ts`**: Registers BullMQ queues (`email`, `notification-fanout`).
- **`src/shared/outbox/consumer-registry.ts`**: Dispatches outbox events triggering email and WS notifications.
- **`src/websocket/gateways/submissions.gateway.ts`**: Socket.io server routing push events to authenticated rooms.
- **`src/websocket/consumers/`**: Subscribes to outbox events and pushes real-time WebSocket payloads.

#### External Runtime Dependencies
- **Amazon SES (`@aws-sdk/client-ses`)**: SMTP/API delivery client for email dispatch (`ap-south-1`).
- **Redis 7.0+**: BullMQ queue storage and Redis pub/sub backbone for WebSocket broadcasts.
- **Zustand (`frontend/src/store/notificationStore.ts`)**: In-memory state manager for the frontend notification bell.
- **Sonner (`sonner`)**: Frontend toast notification library for pop-up alert banners.

---

## SECTION C: Data Models & Persistence Mechanics

### C.1 Database Reality: Zero Models in Prisma
A forensic inspection of all 11 Prisma schemas confirms that **no database models exist for notifications**:
```
prisma/
├── analytics/schema.prisma     (0 notification models)
├── assessment/schema.prisma    (0 notification models)
├── billing/schema.prisma       (0 notification models)
├── credentials/schema.prisma   (0 notification models)
├── curriculum/schema.prisma    (0 notification models)
├── events/schema.prisma        (0 notification models)
├── identity/schema.prisma      (0 notification models)
├── paths/schema.prisma         (0 notification models)
├── platform/schema.prisma      (0 notification models)
├── problems/schema.prisma      (0 notification models)
└── tenancy/schema.prisma       (0 notification models)
```

### C.2 Transient Job Data Model (`EmailJobData` - `src/worker/consumers/email.consumer.ts`)
When jobs are enqueued to Redis via BullMQ, the payload adheres to the following interface:

```typescript
export interface EmailJobData {
  to: string;                          // Destination email address
  templateId: string;                  // Template key or AWS SES template name
  variables: Record<string, unknown>;  // Dynamic interpolation variables
  tenantId?: string;                   // Optional owning tenant UUID
  replyTo?: string;                    // Optional reply-to email address
}
```

### C.3 Transient Client Data Model (`Notification` - `frontend/src/types/common.types.ts`)
When WebSocket frames arrive at the browser, they are wrapped into the following transient TypeScript interface:

```typescript
export interface Notification {
  id: string;                          // Client-generated UUID (crypto.randomUUID())
  title: string;                       // Brief title header
  message: string;                     // Detailed message body
  severity: 'info' | 'success' | 'warning' | 'error';
  read: boolean;                       // Client-side read toggle
  createdAt: string;                   // ISO timestamp of receipt
  link?: string;                       // Optional navigation target URL
}
```

---

## SECTION D: API & Endpoint Specifications

### D.1 Backend Ingress Endpoints: Proven Non-Existent
A search across all NestJS controllers in `src/` confirms that **no notification HTTP endpoints exist**:
- `GET /notifications`: **404 Not Found**
- `POST /notifications`: **404 Not Found**
- `PATCH /notifications/:id`: **404 Not Found**
- `DELETE /notifications/:id`: **404 Not Found**

### D.2 WebSocket Ingress & Egress Channels

Notifications are delivered exclusively through asynchronous WebSocket events:

| Socket Event Name | Destination Channel | Trigger Origin | Payload Structure |
|---|---|---|---|
| `notification` | `user:${userId}` | Generic backend alert | `{ title, message, severity, link? }` |
| `certificate:issued` | `user:${userId}` | `RealtimeCertificatesConsumer` | `{ certificateId, title? }` |
| `certificate:issuing`| `user:${userId}` | Credentials worker | `{ certificateId }` |
| `submission:graded` | `user:${userId}` | `CodeExecutionConsumer` | `{ verdict, assessmentTitle? }` |
| `event:started` | `event:${eventId}` | `RealtimeEventsConsumer` | `{ eventId }` |
| `event:ended` | `event:${eventId}` | `RealtimeEventsConsumer` | `{ eventId }` |
| `event:extended` | `event:${eventId}` | `RealtimeEventsConsumer` | `{ eventId, newEndsAt }` |
| `event:reminder` | `user:${userId}` | `RegistrationReminderService` | `{ eventId, minutesUntilStart }` |

---

## SECTION E: State Machines & Transition Tables

### E.1 Client-Side Notification Item State Machine
Because notifications exist purely in browser memory, their lifecycle is trivial:

```
                      ┌────────────────────────┐
                      │    RECEIVED (UNREAD)   │
                      │      (read: false)     │
                      └───────────┬────────────┘
                                  │
                                  ├──────────────────────────────┐
                                  │ Click Item                   │ Click "Mark all as read"
                                  ▼                              ▼
                      ┌────────────────────────┐     ┌────────────────────────┐
                      │      VIEWED (READ)     │     │      VIEWED (READ)     │
                      │      (read: true)      │     │      (read: true)      │
                      └───────────┬────────────┘     └───────────┬────────────┘
                                  │                              │
                                  └──────────────┬───────────────┘
                                                 │
                                                 │ Click "Clear all" OR Page Refresh
                                                 ▼
                                     ┌───────────────────────┐
                                     │  DESTROYED / CLEARED  │
                                     └───────────────────────┘
```

---

## SECTION F: Business Logic, Algorithms & Invariants

### F.1 Email Delivery Modes & Logic (`EmailConsumer`)

In `src/worker/consumers/email.consumer.ts`:
```typescript
async process(job: Job<EmailJobData>): Promise<void> {
  const { to, templateId, variables, replyTo } = job.data;

  // Dev/test mode: log and skip SES call
  if (process.env.NODE_ENV !== 'production') {
    this.logger.debug(`[EMAIL MOCK] To: ${to} | Template: ${templateId} | Vars: ${JSON.stringify(variables)}`);
    return;
  }

  try {
    if (this.useSesTemplates) {
      await this.sendViaTemplate(to, templateId, variables, replyTo);
    } else {
      await this.sendInline(to, templateId, variables, replyTo);
    }
  } catch (err) {
    this.logger.error(`Email failed: ${(err as Error).message}`);
    throw err;
  }
}
```

#### 1. Development Mock Mode
- When `NODE_ENV !== 'production'`, no network requests are dispatched to AWS SES.
- The payload is written to NestJS debug logs (`[EMAIL MOCK]`).

#### 2. SES Managed Templates Mode (`sendViaTemplate`)
- Active when environment variable `SES_USE_TEMPLATES === 'true'`.
- Calls `SendTemplatedEmailCommand` with `Template: templateId` and `TemplateData: JSON.stringify(variables)`.

#### 3. Inline Fallback Mode (`sendInline`)
- Active when `SES_USE_TEMPLATES !== 'true'`.
- Uses internal string interpolator `TEXT_TEMPLATES[templateId]`:
  - `welcome`: `Welcome to Dezolver, ${v['fullName']}!...`
  - `password-reset`: `Reset your Dezolver password:\n\nhttps://app.dezolver.com/reset-password?token=${v['resetToken']}...`
  - `invitation`: `You've been invited to join ${v['tenantName']} on Dezolver.\n\nAccept: https://app.dezolver.com/invitations/${v['token']}`
  - `submission-graded`: `Your submission for "${v['problemTitle']}" has been graded.\n\nVerdict: ${v['verdict']}`
  - `certificate-issued`: `Congratulations! Your certificate "${v['title']}" is ready.\n\nView: ${v['pdfUrl']}`
- Falls back to raw JSON dump if `templateId` is unrecognized:
  `Notification: ${templateId}\n\n${JSON.stringify(variables, null, 2)}`.

### F.2 Ephemeral In-Memory Store Logic (`useNotificationStore`)

In `frontend/src/store/notificationStore.ts`:
- **50-Item Ring Buffer Cap:**
  ```typescript
  addNotification: (notification) =>
    set((state) => ({
      notifications: [notification, ...state.notifications].slice(0, 50),
      unreadCount: state.unreadCount + (notification.read ? 0 : 1),
    }))
  ```
  The store unshifts new notifications to index 0 and truncates the list at 50 elements (`slice(0, 50)`). Older notifications beyond 50 are silently evicted from memory.
- **Unread Counter Clamping:**
  ```typescript
  markAsRead: (id) =>
    set((state) => ({
      notifications: state.notifications.map((n) =>
        n.id === id ? { ...n, read: true } : n
      ),
      unreadCount: Math.max(0, state.unreadCount - 1),
    }))
  ```
  `Math.max(0, ...)` ensures the badge counter never decrements below zero.

---

## SECTION G: Authentication, Authorization & RBAC

### G.1 WebSocket Channel Authorization
Because in-app notifications are received exclusively over WebSocket rooms:
- **Personal Channel (`user:${userId}`):** A client can join `user:${id}` only if `id === principal.userId` or the caller holds `platform_admin` (`channel-authorizers.ts`, Line 15).
- **Tenant Channel (`tenant:${tenantId}`):** Allowed if `id === principal.tenantId` or caller holds `platform_admin`.

### G.2 Identity Omission Defect Impact
As proven in Module 10, `AuthService.buildLoginResponse` omits `cohort_id` from the JWT token. Consequently, `socket.join('cohort:${user.cohortId}')` is never executed for students, preventing cohort-wide push notifications from ever reaching learners!

---

## SECTION H: Real-Time & Event-Driven Mechanics

### H.1 WebSocket Frame Dispatch & Toast Integration

In `frontend/src/providers/WebSocketProvider.tsx` (Lines 63–76):
```typescript
const handleNotification = (payload: Omit<Notification, 'id' | 'read' | 'createdAt'>) => {
  const notification: Notification = {
    id: crypto.randomUUID(),
    read: false,
    createdAt: new Date().toISOString(),
    ...payload,
  }
  addNotification(notification)

  // Surface desktop toast via Sonner
  if (payload.severity === 'success') toast.success(payload.title, { description: payload.message })
  else if (payload.severity === 'warning') toast.warning(payload.title, { description: payload.message })
  else if (payload.severity === 'error') toast.error(payload.title, { description: payload.message })
}
```

Every incoming notification frame triggers two concurrent actions:
1. Adds the record to the in-memory Zustand store.
2. Renders an interactive, floating toast banner via Sonner.

---

## SECTION I: Frontend Architecture & User Interface

### I.1 Notification Center Popover (`NotificationCenter.tsx`)

In `frontend/src/components/dashboard/notifications/NotificationCenter.tsx`:
- Rendered in the top navigation header bar as a ghost button with a Bell icon (`<Bell className="h-4 w-4" />`).
- **Badge Counter:** When `unreadCount > 0`, a red destructive badge displays over the bell (`unreadCount > 9 ? '9+' : unreadCount`).
- **Popover Dropdown (`PopoverContent` - width 320px):**
  - Header: Displays "Notifications" and unread count badge.
  - Action Controls:
    - `<Button variant="ghost" onClick={markAllAsRead}>Mark all read</Button>`
    - `<Button variant="ghost" onClick={clearAll}>Clear all</Button>`
  - Scroll Area: Vertically scrollable list of `NotificationItem` components.
  - Empty State: Renders a grey bell icon with "No notifications yet".

### I.2 Notification Item Component (`NotificationItem.tsx`)

In `frontend/src/components/dashboard/notifications/NotificationItem.tsx`:
- Severity Icon Mapping:
  - `info`: Blue `<Info />`
  - `success`: Emerald `<CheckCircle2 />`
  - `warning`: Amber `<AlertTriangle />`
  - `error`: Destructive red `<XCircle />`
- Unread Indicator: Blue accent pill dot (`bg-primary`).
- Timestamp: Formatted dynamically using `formatRelativeTime(createdAt)` (e.g. "5m ago", "2h ago").

---

## SECTION J: Cross-Module Interactions & External Integrations

```
  ┌────────────────────────────────────────────────────────┐
  │                 MODULE 13 NOTIFICATIONS                │
  └───────┬───────────────────────┬────────────────────────┘
          │                       │
          ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│ AWS SES CLIENT   │    │ REDIS PUB/SUB    │
│ Outbound SMTP /  │    │ Transmits live   │
│ API delivery for │    │ socket frames to │
│ emails.          │    │ gateways.        │
└──────────────────┘    └──────────────────┘
```

---

## SECTION K: Edge Cases, Failure Modes & Resilience

### K.1 Total Loss of Notification History on Page Reload
- **Scenario:** A student receives 10 notifications regarding competition start times, graded submissions, and certificate generation while logged in. The student hits F5 (refresh).
- **Result:** The notification bell resets to 0. The dropdown is empty ("No notifications yet"). There is no backend endpoint to retrieve previous alerts.

### K.2 Password Reset Outbox Dead-End
- **Scenario:** A user requests a password reset via `POST /auth/password-reset/request`.
- **Backend Execution:** `AuthService.resetPassword` creates a record in `identity.password_resets` and inserts an Outbox event `PasswordResetRequested`.
- **Failure:** No consumer registers for `PasswordResetRequested`. The event remains unconsumed in `platform.outbox_events`. No email is queued or delivered.

### K.3 Dunning Service UUID Failure
- **Scenario:** An institution's subscription is 7 days past due. `DunningService.runDunning` executes at 09:00 IST.
- **Backend Execution:** Lines 45–49 queue job:
  ```typescript
  await this.emailQueue.add('send', {
    to: sub.tenantId, // e.g. "c1f7b88e-6482-4f3e-8c3d-71bfae8d4a99"
    templateId: 'payment_recovery',
    variables: { ... },
  });
  ```
- **Failure:** In production, AWS SES rejects the destination address because a raw UUID is not a valid email address, throwing `InvalidParameterValueException`.

---

## SECTION L: Performance, Caching & Scaling

- **Zero Database Load:** Because in-app notifications are non-persistent, notification delivery imposes 0 database write IOPS or query load on PostgreSQL.
- **Email Throttling & Concurrency:** The BullMQ worker executes jobs with concurrency limits defined in `WorkerModule` (`concurrency: 5`), preventing SES rate limit exhaustion (SES default rate: 14 emails/second).

---

## SECTION M: Security, Threat Model & Compliance

- **Email Spoofing Prevention:** In production, SES verifies domain DKIM, SPF, and DMARC records for `SES_FROM_EMAIL` (`noreply@dezolver.com`).
- **Data Leakage Prevention via WebSocket:** `ChannelAuthorizers` strictly restricts `user:${userId}` channels so students cannot listen to other students' submission or certificate notifications.

---

## SECTION N: Observability, Metrics & Audit Logging

- **Email Delivery Logs:** `EmailConsumer` logs:
  `Sending email templateId=${templateId} to=${to}`
  `Email sent templateId=${templateId} to=${to}`
- **Email Failure Alerts:** Failed jobs are logged with stack traces:
  `Email failed templateId=${templateId} to=${to}: ${err.message}`
- **Audit Logging Omission:** Notification delivery operations are **not audited** in `platform.audit_logs`.

---

## SECTION O: Negative Specification (What is NOT Implemented)

```
┌────────────────────────────────────────────────────────────────────────┐
│                   DEZOLVER MODULE 13 NEGATIVE MATRIX                   │
├──────────────────────────────────┬─────────────────────────────────────┤
│ Architectural Requirement        │ Legacy Implementation Reality       │
├──────────────────────────────────┼─────────────────────────────────────┤
│ 1. Persistent Notification DB    │ 0 tables, 0 models in PostgreSQL    │
│ 2. Notification Ingress REST API │ 0 controllers or endpoints in NestJS│
│ 3. Password Reset Email Delivery │ Event emitted with zero consumers   │
│ 4. Notification Fanout Worker    │ Empty queue; zero jobs or workers   │
│ 5. Cross-Device Notification Sync│ Ephemeral client RAM state only     │
│ 6. Notification Preferences / Opt│ 0 preference models or toggles      │
│ 7. SMS / WhatsApp Notifications  │ 0 integrations (SES email only)     │
│ 8. In-App Notification Search    │ Not supported (purely 50-item list) │
└──────────────────────────────────┴─────────────────────────────────────┘
```

---

## SECTION P: Forensic Code Verification & Traceability Matrix

| Component | Source File Path | Line Reference | Implementation Reality Proof |
|---|---|---|---|
| **Email Worker Processor** | `src/worker/consumers/email.consumer.ts` | Lines 40–76 | `@Processor('email')`, SES client calls. |
| **Inline Email Templates** | `src/worker/consumers/email.consumer.ts` | Lines 19–38 | `TEXT_TEMPLATES` and `SUBJECT_MAP` definitions. |
| **Welcome Email Trigger** | `src/identity/consumers/identity.consumers.ts` | Lines 18–24 | Queues `welcome` email on `UserCreated`. |
| **Contest Reminder Trigger**| `src/events/services/registration-reminder.service.ts`| Lines 44–50 | Queues `event_reminder` on 15m cron. |
| **Dunning Email Defect** | `src/billing/services/dunning.service.ts` | Line 46 | Passes `to: sub.tenantId` (UUID). |
| **Unhandled Reset Event** | `src/identity/services/auth.service.ts` | Line 195 | Emits `PasswordResetRequested` with 0 consumers. |
| **Unused Fanout Queue** | `src/shared/queue/queue.module.ts` | Line 13 | Declares `'notification-fanout'` with 0 workers. |
| **Frontend Store Definition**| `frontend/src/store/notificationStore.ts` | Lines 13–38 | Zustand store with 50-item cap. |
| **Frontend WS Ingress** | `frontend/src/providers/WebSocketProvider.tsx` | Lines 63–140 | Listens to socket events, pops toasts, adds to store. |
| **Hook Documentation** | `frontend/src/hooks/useDashboard.ts` | Lines 42–46 | Comments prove no API fetch exists ("doesn't fetch from API"). |
| **UI Notification Popover**| `frontend/src/components/dashboard/notifications/NotificationCenter.tsx` | Lines 25–116 | Popover with mark read and clear all actions. |

---

## SECTION Q: Target State Recommendations & Remediations

### Q.1 Immediate Critical Remediations
1. **Implement `PasswordResetRequested` Consumer:** In `identity.consumers.ts`, register a listener for `PasswordResetRequested` that extracts `email` and enqueues a job to `emailQueue` with `templateId: 'password-reset'`.
2. **Fix Dunning Service Email Recipient:** In `DunningService.runDunning`, resolve the tenant's `primaryContactEmail` from `tenants.tenants` before queueing the email, passing `to: tenant.primaryContactEmail` instead of `sub.tenantId`.
3. **Persist In-App Notifications to Database:** Create table `platform.notifications` with fields `(id, user_id, tenant_id, title, message, severity, read, link, created_at)` to preserve history across logins and reloads.
4. **Implement REST Notification Endpoints:** Expose `GET /notifications` (paginated), `PATCH /notifications/:id/read`, and `POST /notifications/mark-all-read` in `src/platform/controllers/notifications.controller.ts`.

