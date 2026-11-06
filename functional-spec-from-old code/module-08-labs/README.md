# Module 8 — Labs & Lab Scoring

## 1. Module Executive Summary

Module 8 covers the conceptual domain of Laboratory Coursework, Scheduled Lab Sessions, Group Labs, Lab Submissions, and Lab-Specific Evaluation & Scoring within academic institutions.

An exhaustive, code-level forensic investigation of the legacy application source code (`src/`), database schemas (`prisma/`), and client applications (`frontend/src/`) confirms that **LAB FUNCTIONALITY IS COMPLETELY NOT IMPLEMENTED IN THE LEGACY CODEBASE**.

- There are **0 Lab database models** across all 11 Prisma schemas (`Lab`, `LabAssignment`, `LabSubmission`, `GroupLab`, and `LabScore` do not exist).
- There are **0 Lab controllers, services, repositories, or routes** in the NestJS backend (`src/` contains 0 files or routes referencing `lab`).
- There are **0 Lab frontend pages, components, hooks, or navigation items** in the React application (`frontend/src/` contains 0 routes for "My Labs", "HOD Courses & Labs", or "Manager Courses & Labs").
- There are **0 roles for HOD (Head of Department) or Manager (Lab Manager)** in the RBAC matrix or identity system.
- There are **0 database entities for Group, Section, or Department** in the tenancy schema (only `Tenant` and `Cohort` exist).
- Practical coding coursework is represented by curriculum **Rooms** (`prisma/content/schema.prisma`), structured in **Courses** (`CourseRoom`), sequenced in **Paths** (`prisma/paths/schema.prisma`), or evaluated through individual **Assessments** (`prisma/assessment/schema.prisma`).

---

## 2. Directory Layout & Document Navigation

```
docs/functional-spec/module-08-labs/
├── README.md               # Executive summary, forensic findings, and navigational guide (this file)
└── SPECIFICATION.md        # Complete 17-Section (A–Q) legacy functional reconstruction & negative specification
```

- **Detailed Functional Specification:** [`SPECIFICATION.md`](./SPECIFICATION.md)
- **Authoring Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md)
- **Master Index & Registry:** [`INDEX.md`](../INDEX.md)

---

## 3. Forensic Investigation Results by Required Area

| Investigation Area | Legacy Codebase Reality | Status | Evidence |
|---|---|:---:|---|
| **A. Lab Architecture & Creation** | No `Lab` or `LabAssignment` entity exists. No creation API or form exists. | `NOT IMPLEMENTED` | 0 occurrences in `prisma/` and `src/`. |
| **B. Lab-Problem Association** | Problems are linked only to `assessment.assessments` via `problem_id` or embedded in `content.rooms`. | `NOT IMPLEMENTED` | No join tables or models exist. |
| **C. Group Lab / Cohort Distribution** | No `GroupLab` entity exists. No `Group`, `Section`, or `Department` tables exist. | `NOT IMPLEMENTED` | Only `tenancy.cohorts` exists. |
| **D. Student Submissions in Labs** | No `LabSubmission` entity exists. Code submissions exist only in `assessment.submissions`. | `NOT IMPLEMENTED` | 0 occurrences in `prisma/assessment/`. |
| **E. Lab Evaluation & Scoring** | No lab evaluation engine, manual grading rubrics, or lab scoring services exist. | `NOT IMPLEMENTED` | Scoring exists only in `computeVerdict` for assessments. |
| **F. Academic Roles (HOD, Manager)** | Neither `hod`, `head_of_department`, nor `lab_manager` roles exist in RBAC or database. | `NOT IMPLEMENTED` | [`src/shared/auth/permission.guard.ts:11-38`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/shared/auth/permission.guard.ts#L11-L38). |
| **G. Lab Frontend Views** | No "My Labs", "Create Lab", or "Lab Submissions" pages exist in React Router. | `NOT IMPLEMENTED` | 0 occurrences across `frontend/src/`. |

---

## 4. Existing Architecture Baseline for Practical Coursework

In the legacy Dezolver application, practical hands-on student learning and coding assignments are delivered through three alternative modules:
1. **Module 4 (Curriculum Rooms & Courses):** Interactive instructional units (`content.rooms`) containing Markdown theory and coding challenges, grouped into `content.courses` and tracked via `paths.room_progress`.
2. **Module 5 (Problem Catalog):** Platform-wide algorithmic challenges (`content.problems`) with sample and hidden test cases.
3. **Module 6 & 9 (Assessments):** Timed coding assessments (`assessment.assessments`) linked to problems via `problem_id`, evaluated via BullMQ and Judge0 CE.

