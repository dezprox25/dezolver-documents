# Page to Database Mapping

This document provides a Page-to-Database Mapping for the DEZOLVER platform.
It covers all pages and screens defined in the Figma design and maps each page to the corresponding database tables/entities required to support its functionality.
The mapping is organized by stakeholder - Super Admin, Finance Team, HOD, Manager, and Student - making it easier for developers to understand which database entities are associated with each UI page and maintain consistency between the frontend design and database architecture.

## Original page 1 - Super Admin

### SUPER ADMIN

| # | Module | Page | Database tables / entities |
|---:|---|---|---|
| 01 | **Login** | Super Admin Login | `User`<br>`RefreshSession` |
| 02 | **Dashboard** | Super Admin Dashboard | `College`<br>`User`<br>`Department`<br>`Plan`<br>`Subscription`<br>`Invoice`<br>`PaymentTransaction`<br>`AuditLog` |
| 03 | **College Management** | College Management | `College`<br>`User`<br>`Department`<br>`Section`<br>`Subscription` |
| 04 | **College Details** | College Details | `College`<br>`Department`<br>`Section`<br>`User` |
| 05 | **Finance Team Management** | Finance Team Management | `College`<br>`User` |
| 06 | **Department Management** | Department Management | `College`<br>`Department`<br>`Section` |
| 07 | **User Management** | User Management | `User`<br>`College`<br>`Department`<br>`Section` |
| 08 | **Plan Management** | Plan Management | `Plan`<br>`Subscription` |
| 09 | **Subscription Management** | Subscription Management | `College`<br>`Plan`<br>`Subscription`<br>`Invoice`<br>`PaymentTransaction` |
| 10 | **Invoice Management** | Invoice Management | `Invoice`<br>`Subscription`<br>`College`<br>`PaymentTransaction` |
| 11 | **Payment Management** | Payment Management | `PaymentTransaction`<br>`Subscription`<br>`Invoice`<br>`College` |
| 12 | **Audit Logs** | Audit Logs | `AuditLog`<br>`User` |
| 13 | **Security / Sessions** | Security / Sessions | `RefreshSession`<br>`User` |

## Original page 2 - Finance Team and HOD

### FINANCE TEAM

| # | Module | Page | Database tables / entities |
|---:|---|---|---|
| 01 | **Dashboard** | Finance Team Dashboard | `College`<br>`Plan`<br>`Subscription`<br>`Department`<br>`Invoice`<br>`User` |
| 02 | **Billing & Subscription** | Billing & Subscription | `College`<br>`Plan`<br>`Subscription`<br>`Invoice`<br>`PaymentTransaction` |
| 03 | **Invoice History** | Invoice History | `Invoice`<br>`Subscription`<br>`PaymentTransaction`<br>`College` |
| 04 | **HOD Accounts** | HOD Accounts | `User`<br>`College`<br>`Department` |
| 05 | **Buy More Seats** | Buy More Seats | `Plan`<br>`Subscription`<br>`College` |
| 06 | **Support Tickets** | Support Tickets | `SupportTicket`<br>`TicketComment`<br>`User`<br>`College` |
| 07 | **Notifications** | Notifications | `Notification`<br>`User` |
| 08 | **Profile & Settings** | Profile & Settings | `User`<br>`RefreshSession`<br>`AuditLog` |

### HOD (ADMIN)

| # | Module | Page | Database tables / entities |
|---:|---|---|---|
| 01 | **Dashboard** | HOD Dashboard | `User`<br>`Department`<br>`Section`<br>`Course`<br>`Assessment`<br>`Contest`<br>`Group` |
| 02 | **Department Analytics** | Department Analytics | `Department`<br>`Section`<br>`User`<br>`Course`<br>`GroupLab`<br>`GroupContest` |
| 03 | **Managers / Lab Staff** | Managers / Lab Staff | `User`<br>`Department`<br>`Section`<br>`Group` |
| 04 | **Students** | Students | `User`<br>`Department`<br>`Section`<br>`CourseEnrollment`<br>`GroupMember` |
| 05 | **Courses & Labs** | Courses & Labs | `Course`<br>`CourseSection`<br>`CourseEnrollment`<br>`CourseProblem`<br>`GroupCourseProblem`<br>`LabAssignment`<br>`LabSubmission`<br>`CodeSubmission`<br>`Problem` |
| 06 | **Assessments** | Assessments | `Assessment`<br>`AssessmentProblem`<br>`Problem`<br>`CodeSubmission`<br>`User`<br>`Department`<br>`College` |
| 07 | **Problems** | Problems | `Problem`<br>`TestCase`<br>`CodeSubmission`<br>`User`<br>`Department`<br>`College` |
| 08 | **Contests** | Contests | `Contest`<br>`ContestProblem`<br>`ContestParticipant`<br>`ContestScoreRank`<br>`Group`<br>`GroupMember`<br>`GroupContest`<br>`Problem`<br>`CodeSubmission` |
| 09 | **Groups** | Groups | `Group`<br>`GroupMember`<br>`User`<br>`Department`<br>`GroupLab`<br>`GroupContest` |
| 10 | **Certificate Templates** | Certificate Templates | `CertificateTemplate`<br>`User`<br>`College` |
| 11 | **Ticket Verification** | Ticket Verification | `SupportTicket`<br>`User`<br>`College` |
| 12 | **Help & Support** | Help & Support | `SupportTicket`<br>`TicketComment`<br>`User`<br>`College` |
| 13 | **Notifications** | Notifications | `Notification`<br>`User` |
| 14 | **Profile & Settings** | Profile & Settings | `User`<br>`RefreshSession`<br>`AuditLog` |

## Original page 3 - Manager and Student

### MANAGER

| # | Module | Page | Database tables / entities |
|---:|---|---|---|
| 01 | **Dashboard** | Manager Dashboard | `User`<br>`Section`<br>`Course`<br>`Assessment`<br>`Problem`<br>`Contest`<br>`Group` |
| 02 | **Student Performance** | Student Performance | `User`<br>`Section`<br>`CourseEnrollment`<br>`LabSubmission`<br>`CodeSubmission`<br>`ContestScoreRank` |
| 03 | **Students** | Students | `User`<br>`College`<br>`Department`<br>`Section`<br>`CourseEnrollment`<br>`GroupMember` |
| 04 | **Groups / Classes** | Groups / Classes | `Group`<br>`GroupMember`<br>`User`<br>`Department`<br>`GroupLab`<br>`GroupContest` |
| 05 | **Courses & Labs** | Courses & Labs | `Course`<br>`CourseSection`<br>`CourseEnrollment`<br>`CourseProblem`<br>`GroupCourseProblem`<br>`LabAssignment`<br>`LabSubmission`<br>`CodeSubmission`<br>`Problem` |
| 06 | **Assessments** | Assessments | `Assessment`<br>`AssessmentProblem`<br>`Problem`<br>`CodeSubmission`<br>`User`<br>`Department`<br>`College` |
| 07 | **Problems** | Problems | `Problem`<br>`TestCase`<br>`CodeSubmission`<br>`User`<br>`Department`<br>`College` |
| 08 | **Contests** | Contests | `Contest`<br>`ContestProblem`<br>`ContestParticipant`<br>`ContestScoreRank`<br>`Group`<br>`GroupMember`<br>`GroupContest`<br>`Problem`<br>`CodeSubmission` |
| 09 | **Help & Support** | Help & Support | `SupportTicket`<br>`TicketComment`<br>`User`<br>`College` |
| 10 | **My Tickets** | My Tickets | `SupportTicket`<br>`TicketComment`<br>`User`<br>`College` |
| 11 | **Notifications** | Notifications | `Notification`<br>`User` |
| 12 | **Profile & Settings** | Profile & Settings | `User`<br>`RefreshSession`<br>`AuditLog` |

### STUDENT

| # | Module | Page | Database tables / entities |
|---:|---|---|---|
| 01 | **Dashboard** | Student Dashboard | `User`<br>`College`<br>`Department`<br>`Section`<br>`CourseEnrollment`<br>`LabSubmission`<br>`ContestParticipant`<br>`ContestScoreRank`<br>`Notification` |
| 02 | **My Courses** | My Courses | `Course`<br>`CourseSection`<br>`CourseEnrollment`<br>`CourseProblem`<br>`GroupCourseProblem`<br>`Problem`<br>`Section` |
| 03 | **Assignments** | Assignments | `LabAssignment`<br>`LabSubmission`<br>`CodeSubmission`<br>`Problem`<br>`Course` |
| 04 | **My Labs** | My Labs | `LabAssignment`<br>`LabSubmission`<br>`CodeSubmission`<br>`Problem`<br>`Course` |
| 05 | **Practice Problems** | Practice Problems | `Problem`<br>`TestCase`<br>`CodeSubmission`<br>`User` |
| 06 | **Contests** | Contests | `Contest`<br>`ContestProblem`<br>`ContestParticipant`<br>`ContestScoreRank`<br>`Problem`<br>`Group`<br>`GroupMember`<br>`GroupContest`<br>`CodeSubmission` |
| 07 | **My Groups** | My Groups | `Group`<br>`GroupMember`<br>`GroupLab`<br>`GroupContest`<br>`User`<br>`Department` |
| 08 | **My Certificates** | My Certificates | `Certificate`<br>`CertificateTemplate`<br>`Course`<br>`Contest`<br>`User`<br>`College` |
| 09 | **Leaderboard** | Leaderboard | `ContestScoreRank`<br>`Contest`<br>`GroupContest`<br>`User`<br>`College` |
| 10 | **Career Roadmaps** | Career Roadmaps | `Roadmap`<br>`User`<br>`College` |
| 11 | **My Roadmap Progress** | My Roadmap Progress | `Roadmap`<br>`User` |
| 12 | **Help & Support** | Help & Support | `SupportTicket`<br>`TicketComment`<br>`User`<br>`College` |
| 13 | **My Tickets** | My Tickets | `SupportTicket`<br>`TicketComment`<br>`User`<br>`College` |
| 14 | **Notifications** | Notifications | `Notification`<br>`User` |
| 15 | **Profile & Settings** | Profile & Settings | `User`<br>`RefreshSession`<br>`AuditLog` |
