# Business Flow Diagrams - v2

This document provides Business Flow Diagrams for the DEZOLVER platform, illustrating the end-to-end workflow followed by each stakeholder. It explains how the Super Admin, Finance Team, HOD, Manager, and Student interact with the platform and how their activities flow from one stage to another. The document gives new developers a clear understanding of the overall business workflow, stakeholder responsibilities, and interactions before they begin working on the system.

## Original page 1 - Overall System Flow Diagram

```mermaid
flowchart TD
    DZ[DEZOLVER] --> SA[SUPER ADMIN]
    SA --> C[COLLEGE]
    C --> FT[FINANCE TEAM]
    C --> D[DEPARTMENT]
    C --> S[SUBSCRIPTION]
    D --> H[HOD]
    S --> P[PLANS]
    H --> SEC[SECTIONS]
    H --> CO[COURSES]
    H --> G[GROUPS]
    SEC --> M[MANAGER]
    CO --> PR[PROBLEMS]
    G --> L[LABS]
    G --> CT[CONTESTS]
    M --> ST[STUDENTS]
    CT --> CP[CONTEST PROBLEMS]
    ST --> COURSE[COURSE]
    PR --> PROBLEM[PROBLEM]
    COURSE --> CS[CODE SUBMISSION]
    PROBLEM --> CS
    CP --> PART[PARTICIPANTS]
    PART --> CS
    CS --> BE[DEZOLVER BACKEND]
    BE --> J[JUDGE0]
    J --> E[EVALUATION]
    E --> R[RESULT]
    R --> SCORE[SCORE]
```

## Original page 2 - Super Admin and Finance Team

### SUPER ADMIN

The Super Admin manages colleges at the platform level and starts the college onboarding process.

```mermaid
flowchart TD
    SA[SUPER ADMIN] --> ON[COLLEGE ONBOARDING]
    ON --> CC[COLLEGE CREATED]
    CC --> SETUP[SETUP]
    SETUP --> FT[FINANCE TEAM]
    FT --> SUB[SUBSCRIPTION]
```

### FINANCE TEAM

The Finance Team handles the college's subscription and plan-related administration.

```mermaid
flowchart TD
    FT[FINANCE TEAM] --> VP[VIEW PLAN]
    VP --> S[SUBSCRIPTION]
    S --> P[PLAN]
    P --> D[DEFAULT]
    P --> C[CUSTOM]
    S --> AP[ACTIVE PERIOD]
    S --> R[RENEWAL]
```

## Original page 3 - HOD and Manager

### HOD

The HOD manages the academic structure of the department and oversees the managers, sections and academic activities under the department.

```mermaid
flowchart TD
    H[HOD] --> D[DEPARTMENT]
    D --> S[SECTIONS]
    D --> M[MANAGERS]
    D --> ST[STUDENTS]
    D --> C[COURSES]
    D --> P[PROBLEMS]
    D --> G[GROUPS]
```

### MANAGER

A Manager manages only the students and activities within their assigned section.

```mermaid
flowchart TD
    M[MANAGER] --> S[SECTION]
    S --> ST[STUDENTS]
    ST --> C[COURSES]
    ST --> G[GROUPS]
    ST --> L[LABS]
    C --> P1[PROBLEMS]
    G --> MEM[MEMBERS]
    G --> A[ACTIVITIES]
    L --> P2[PROBLEMS]
```

## Original page 4 - Groups + Labs + Contests

Groups act as the mediator for collaborative activities such as labs and contests.

```mermaid
flowchart TD
    G[GROUP] --> L[LAB]
    G --> C[CONTEST]

    L --> LP[PROBLEM]
    LP --> LS[STUDENT]
    LS --> LCS[CODE SUBMISSION]
    LCS --> LJ[JUDGE0]
    LJ --> LE[EVALUATION]
    LE --> LSCORE[LAB SCORE]

    C --> CP[CONTEST PROBLEM]
    CP --> CS[STUDENT]
    CS --> PART[PARTICIPATION]
    PART --> CCS[CODE SUBMISSION]
    CCS --> CJ[JUDGE0]
    CJ --> SC[CONTEST SCORING]
    SC --> SR[SCORE/RANK]
```

## Original page 5 - Student

```mermaid
flowchart TD
    ST[STUDENT] --> COURSE[COURSE]
    COURSE --> P[PROBLEM]
    P --> W[WRITE CODE]
    W --> S1[SUBMIT]
    S1 --> CS1[CODE SUBMISSION]
    CS1 --> J1[JUDGE0]
    J1 --> E1[EVALUATION]
    E1 --> R[RESULT]

    ST --> G[GROUP]
    G --> L[LAB]
    L --> LP[PROBLEM]
    LP --> S2[SUBMIT]
    S2 --> CS2[CODE SUBMISSION]
    CS2 --> J2[JUDGE0]
    J2 --> E2[EVALUATION]
    E2 --> LS[LAB SCORE]

    ST --> C[CONTEST]
    C --> CP[CONTEST PROBLEM]
    CP --> S3[SUBMIT]
    S3 --> CS3[CODE SUBMISSION]
    CS3 --> J3[JUDGE0]
    J3 --> CTS[CONTEST SCORE]
    CTS --> SCORE[SCORE]
    SCORE --> RANK[RANK]
```
