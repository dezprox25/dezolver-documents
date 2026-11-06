# Module 4 — Course & Enrollment

## 1. Module Executive Summary

Module 4 encompasses the educational structure, content organization, syllabus tailoring, and learning journey tracking within the Dezolver platform. In the legacy architecture, educational delivery is partitioned across three interconnected domains:
1. **Course Catalog (`content` schema):** Published collections of interactive learning `rooms` arranged in ordered sequences via `course_rooms`.
2. **Curriculum & Differential Syllabi (`curriculum` schema):** Multi-level hierarchical syllabus trees (`syllabi`, `syllabus_nodes`) customized per academic cohort via an operation-based differential overlay engine (`syllabus_overlays`, `overlay_operations`) and cached in Redis.
3. **Learning Paths & Enrollment Progress (`paths` schema):** Directed learner journeys (`paths`, `path_steps`) where enrollment is modeled via `path_progress`. Student advancement is automatically computed as learners complete rooms or solve coding challenges.

A primary forensic discovery is that **traditional course-level enrollments (`CourseEnrollment`) DO NOT EXIST** in the legacy codebase. Instead, courses serve as public content groupings, while learner enrollment and completion tracking are handled exclusively at the **Learning Path** level (`PathProgress`) and **Cohort Syllabus** level.

---

## 2. Directory Layout & Document Navigation

```
docs/functional-spec/module-04-courses/
├── README.md               # Executive summary, forensic findings, and navigational guide (this file)
└── SPECIFICATION.md        # Complete 17-Section (A–Q) legacy functional reconstruction
```

- **Detailed Functional Specification:** [`SPECIFICATION.md`](./SPECIFICATION.md)
- **Authoring Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md)
- **Master Index & Registry:** [`INDEX.md`](../INDEX.md)

---

## 3. Key Forensic Discoveries & Architecture Summary

| Aspect | Legacy Source Code Implementation Reality |
|---|---|
| **Course Model** | Implemented in `content` schema ([`prisma/content/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/content/schema.prisma)) as a simple wrapper around ordered `Room` entities via `CourseRoom`. |
| **Course Enrollment** | **NON-EXISTENT.** No `CourseEnrollment` table exists. Courses cannot be directly enrolled into by students; they are browsed publicly. |
| **Syllabus & Tree Nodes** | Implemented in `curriculum` schema. Supports hierarchical nodes (`topic`, `subtopic`, `room`, `problem`, `assessment`). |
| **Cohort Overlays** | Differential JSON operation engine applying typed mutations (`hide_node`, `reorder_children`, `rename_node`, `remap_content`, `add_child_node`) on top of a tenant's base syllabus. |
| **Effective Syllabus Engine** | [`EffectiveSyllabusResolver`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/curriculum/services/effective-syllabus.resolver.ts) resolves overlays, filters hidden nodes, and caches in Redis (`effective_syllabus:{syllabusId}:{cohortId}`) for 30 minutes. |
| **Learning Path Enrollment** | Modeled in `paths` schema via `PathProgress`. Enrolled paths are queried via `findEnrolledPaths(userId)`. |
| **Progress Aggregation** | Asynchronous outbox worker ([`ProgressAggregatorConsumer`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/paths/consumers/progress-aggregator.consumer.ts)) recalculates completion percentages across all enrolled paths upon room events. |
| **Auto-Complete Integration** | [`SubmissionAcceptedConsumer`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/paths/consumers/submission-accepted.consumer.ts) marks rooms completed when code submissions pass Judge0 evaluation. |
| **Critical Bugs Uncovered** | - Querying non-existent `tenantId` column on `content.rooms` completely breaks room progress tracking (`assertRoomAccess` always throws 404).<br>- `userSyllabusAssignment` and `domain` models missing from Prisma; `GET /me/syllabus` always throws 404.<br>- `careerMap` missing from Prisma; `GET /career-maps` always returns empty array. |

---

## 4. Primary Source Code References

### Backend
- **Prisma Schemas:**
  - [`prisma/content/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/content/schema.prisma) (`Course`, `CourseRoom`, `Room`)
  - [`prisma/curriculum/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/curriculum/schema.prisma) (`Syllabus`, `SyllabusNode`, `SyllabusOverlay`, `OverlayOperation`)
  - [`prisma/paths/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/paths/schema.prisma) (`Path`, `PathStep`, `PathProgress`)
- **Content Module:** [`src/content/content.module.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/content.module.ts)
  - [`src/content/controllers/courses.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/courses.controller.ts)
  - [`src/content/controllers/rooms.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/rooms.controller.ts)
  - [`src/content/repositories/courses.repository.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/repositories/courses.repository.ts)
  - [`src/content/services/content.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/services/content.service.ts)
- **Curriculum Module:** [`src/curriculum/curriculum.module.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/curriculum/curriculum.module.ts)
  - [`src/curriculum/controllers/curriculum.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/curriculum/controllers/curriculum.controller.ts)
  - [`src/curriculum/controllers/overlays.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/curriculum/controllers/overlays.controller.ts)
  - [`src/curriculum/controllers/me-syllabus.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/curriculum/controllers/me-syllabus.controller.ts)
  - [`src/curriculum/controllers/domains.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/curriculum/controllers/domains.controller.ts)
  - [`src/curriculum/services/effective-syllabus.resolver.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/curriculum/services/effective-syllabus.resolver.ts)
  - [`src/curriculum/repositories/syllabus.repository.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/curriculum/repositories/syllabus.repository.ts)
- **Paths Module:** [`src/paths/paths.module.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/paths/paths.module.ts)
  - [`src/paths/controllers/paths.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/paths/controllers/paths.controller.ts)
  - [`src/paths/controllers/me-paths.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/paths/controllers/me-paths.controller.ts)
  - [`src/paths/controllers/career-maps.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/paths/controllers/career-maps.controller.ts)
  - [`src/paths/repositories/paths.repository.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/paths/repositories/paths.repository.ts)
  - [`src/paths/consumers/progress-aggregator.consumer.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/paths/consumers/progress-aggregator.consumer.ts)
  - [`src/paths/consumers/submission-accepted.consumer.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/paths/consumers/submission-accepted.consumer.ts)

### Frontend
- **Courses:** [`frontend/src/pages/content/courses/`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/content/courses/), [`frontend/src/hooks/useCourses.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/hooks/useCourses.ts)
- **Curriculum:** [`frontend/src/pages/curriculum/`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/curriculum/), [`frontend/src/services/api/curriculum.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/services/api/curriculum.service.ts)
- **Paths & Progress:** [`frontend/src/pages/paths/`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/paths/), [`frontend/src/services/api/path.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/services/api/path.service.ts), [`frontend/src/hooks/usePaths.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/hooks/usePaths.ts)

