# Module 13 — Notifications

## 1. Module Executive Summary

Module 13 governs the notification and messaging infrastructure of the Dezolver platform, encompassing email delivery, in-app notification center alerts, and real-time WebSocket push notifications.

A rigorous forensic audit of the legacy codebase reveals an asymmetric, hybrid notification architecture characterized by strong email processing pipelines paired with an entirely ephemeral in-app notification subsystem:

1. **Complete Absence of Database Persistence for In-App Notifications:** Across all 11 Prisma database schemas in `prisma/`, there is **no `Notification` or `UserNotification` model or table**. The backend provides **zero endpoints to list, query, or mark notifications as read**.
2. **Purely Ephemeral Client-Side Notification Center:** In-app notifications exist exclusively in browser RAM via a client-side Zustand store (`frontend/src/store/notificationStore.ts`). Notifications are populated in real-time as WebSocket frames arrive in `WebSocketProvider.tsx`.
3. **Session Loss on Refresh:** Because in-app notifications are not persisted in PostgreSQL or Redis, refreshing the browser or signing in from another device immediately resets the notification center to an empty state (`[]`).
4. **Asynchronous SES Email Worker Pipeline:** Transactional emails are processed asynchronously via BullMQ queue `email` and executed by `EmailConsumer` (`src/worker/consumers/email.consumer.ts`) using AWS SES. The consumer supports both AWS SES managed templates and hardcoded inline text fallback templates.
5. **Critical Outbox Dead-End (Password Reset Omission):** While `AuthService` emits an Outbox event `PasswordResetRequested` upon password reset initiation, **zero consumers listen for this event**. Consequently, password reset emails are never added to the queue or dispatched to users.
6. **Unused Fanout Queue:** Queue `notification-fanout` is registered in `QueueModule` and injected into `RegistrationReminderService`, but has **zero worker processors and zero jobs enqueued to it**.
7. **Dunning Email Address Defect:** In `DunningService`, overdue subscription warning jobs pass the tenant UUID (`sub.tenantId`) as the destination email address, causing SES job failures.

---

## 2. Directory Layout & Document Navigation

```
docs/functional-spec/module-13-notifications/
├── README.md               # Executive summary, findings, and navigational guide (this file)
└── SPECIFICATION.md        # Comprehensive 17-Section (A–Q) legacy specification & traceability matrix
```

- **Detailed Functional Specification:** [`SPECIFICATION.md`](./SPECIFICATION.md)
- **Authoring Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md)
- **Master Index & Registry:** [`INDEX.md`](../INDEX.md)

---

## 3. Subsystem Architecture & Notification Routing

```
   ┌────────────────────────────────────────────────────────┐
   │                    TRIGGER EVENTS                      │
   │  - UserCreated / UserInvitationAccepted                │
   │  - EventRegistrationReminder (24h, 1h crons)           │
   │  - CertificateIssued / SubmissionGraded                │
   │  - DunningService (overdue subscription cron)          │
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

---

## 4. Key Forensic Discoveries Table

| Subsystem / Area | Conceptual Requirement | Legacy Reality in Source Code | Forensic Status |
|---|---|---|:---:|
| **In-App Notification DB** | Persistent inbox stored in PostgreSQL | 0 models, 0 tables in Prisma; purely ephemeral Zustand RAM store. | `NOT IMPLEMENTED` |
| **Notification API** | `GET /notifications`, `PATCH :id/read` | 0 backend notification endpoints exist. | `NOT IMPLEMENTED` |
| **Email Ingress** | BullMQ queue `email` | Fully functional SES worker in `EmailConsumer`. | `CONFIRMED LEGACY IMPLEMENTATION` |
| **Password Reset Email** | Reset token emailed to user | Event `PasswordResetRequested` has no consumer; no email queued. | `DEFECT / DEAD-END OUTBOX` |
| **Notification Fanout** | Queue `notification-fanout` | Registered in queue module; 0 processors and 0 jobs enqueued. | `UNIMPLEMENTED STUB QUEUE` |
| **Real-Time Push Alerts** | WebSocket push to user channels | Emits `certificate:issued`, `submission:graded`, `event:started`.| `CONFIRMED LEGACY IMPLEMENTATION` |
| **Dunning Email Target** | Email sent to billing contact | Passes `sub.tenantId` (UUID) as email address; crashes SES. | `DEFECT / ADDRESS MISMATCH` |
| **Notification History** | User reviews past alerts | Vanishes completely on page refresh or browser close. | `CONFIRMED LEGACY BEHAVIOR` |

---

## 5. Source File Traceability Index

| Subsystem / Layer | Source File Path | Primary Functionality |
|---|---|---|
| **Email Worker** | `src/worker/consumers/email.consumer.ts` | BullMQ consumer sending SES templated and inline text emails |
| **Identity Triggers** | `src/identity/consumers/identity.consumers.ts` | Emits `welcome` email on user signup and invitation acceptance |
| **Event Reminder Cron**| `src/events/services/registration-reminder.service.ts`| 15-minute cron queueing 24h and 1h event reminder emails |
| **Dunning Service** | `src/billing/services/dunning.service.ts` | Daily cron queueing `payment_recovery` emails (with UUID defect) |
| **Real-Time Gateway** | `src/websocket/gateways/submissions.gateway.ts` | Socket.io server maintaining user channels for push notifications |
| **Cert Real-Time WS** | `src/websocket/consumers/realtime-certificates.consumer.ts` | Dispatches `certificate:issued` frames to `user:${userId}` |
| **Events Real-Time WS**| `src/websocket/consumers/realtime-events.consumer.ts` | Dispatches `event:state` frames to event rooms |
| **Frontend WS Client** | `frontend/src/providers/WebSocketProvider.tsx` | Receives WS frames, pops toasts, adds to Zustand store |
| **Zustand Store** | `frontend/src/store/notificationStore.ts` | In-memory store holding up to 50 notifications and unread count |
| **UI Notification Bell**| `frontend/src/components/dashboard/notifications/NotificationCenter.tsx` | Popover component displaying recent notification items |
| **UI Notification Item**| `frontend/src/components/dashboard/notifications/NotificationItem.tsx` | Item renderer with severity icons and relative timestamps |

