# Classical Dance Academy Management System

A production-grade, modular-monolith management platform and digital portal suite tailored for classical dance institutions (Bharatanatyam, Kathak, Kuchipudi, Odissi, Mohiniyattam). Designed as a tier-1 college group project with local-first architecture, clean separation of concerns, strict Role-Based Access Control (RBAC), and 20-table relational integrity.

---

## 1. System Architecture Overview

The system consists of **3 distinct Angular 17/18+ Single Page Applications (SPAs)** communicating with **1 shared Java 25 (LTS) + Spring Boot 3.5.x REST API** backed by **MySQL 8.0** with Flyway database migrations.

```
                           +----------------------------------------+
                           |       End Users & Client Devices       |
                           +----------------------------------------+
                                  |            |            |
                 :4200            |            |            | :4202
            +-------------+       |            |       +-------------+
            |   Public    |       | :4201      |       |    Admin    |
            | Academy SPA |       |      +-------------+ | & Instructor|
            +-------------+       |      |   Student   | |     SPA     |
                   \              |      | Portal SPA  | +-------------+
                    \             |      +-------------+       /
      (Public REST)  \            |             |             / (Admin REST)
                      \           |             |            /
                       v          v             v           v
                     +---------------------------------------+
                     |   dance-academy-api (Spring Boot 3)   |
                     |             Port: 8080                |
                     |   [Spring Security 6 + JWT + RBAC]    |
                     +---------------------------------------+
                                         |
                                         | (JPA / Hibernate / HikariCP)
                                         v
                     +---------------------------------------+
                     |       MySQL 8.0 Database Instance     |
                     |       (Port 3306: dance_academy_db)   |
                     |       Flyway Managed (V1, V2 Schema)  |
                     +---------------------------------------+
```

---

## 2. Six-Repository Map

| Repository | Tech Stack | Local Port | Scope & Responsibility |
|:---|:---|:---|:---|
| **`dance-academy-docs`** | Markdown / Mermaid | N/A | Central architecture specifications, RBAC matrix, 20-table ER design, REST API contract, QA plans, and viva presentation scripts. |
| **`dance-academy-api`** | Java 25 (LTS), Spring Boot 3.5.x, Maven, Flyway, MySQL | `:8080` | Shared backend modular monolith: Spring Security 6 stateless JWT, REST endpoints, RFC 7807 problem details, StorageService, and AuditLogService. |
| **`dance-academy-public`** | Angular 17/18+ (Standalone, SCSS) | `:4200` | Unauthenticated public website with temple-gold/deep-maroon aesthetic, 16:9 hero, styles catalog, faculty, gallery, events, moderated reviews, and admission enquiries. |
| **`dance-academy-student`** | Angular 17/18+ (Standalone, SCSS) | `:4201` | Authenticated student portal (`ROLE_STUDENT`): dashboard KPIs, profile, class schedules, attendance logs, secure blob certificate downloads, and announcements. |
| **`dance-academy-admin`** | Angular 17/18+ (Standalone, SCSS) | `:4202` | Authenticated management portal (`ROLE_SUPER_ADMIN`, `ROLE_ACADEMY_ADMIN`, `ROLE_INSTRUCTOR`): 16 operational modules, attendance marking, certificate generation, CMS, review moderation, and CSV reports. |
| **`dance-academy-platform`** | Docker Compose, Bash, YAML | N/A | Local orchestration (`mysql:8.0` + API + Nginx static previews), `.env.example`, database backup/restore scripts (`mysqldump`), CI/CD templates, and cloud deployment guides. |

---

## 3. Local Development Prerequisites

Ensure the following runtimes and CLI tools are installed on development machines before scaffolding:

| Prerequisite | Minimum Supported Version | Verification Command | Notes / Purpose |
|:---|:---|:---|:---|
| **Java Development Kit (JDK)** | OpenJDK / Eclipse Temurin 25 (LTS) | `java -version` | Core backend runtime |
| **Apache Maven** | 3.9.x+ | `mvn -version` | Java dependency management & builds |
| **Node.js & npm** | Node.js 20.x+ LTS / npm 10.x+ | `node -v` && `npm -v` | Angular toolchain runtime |
| **Angular CLI** | 17.x or 18.x | `ng version` | SPA scaffolding, serving, and testing |
| **MySQL Server** | 8.0.x+ Community Server | `mysql --version` | Primary relational database (`utf8mb4`) |
| **Git** | 2.40+ | `git --version` | Version control across all 6 repositories |
| **Docker & Compose** *(Optional)* | Docker 24+ / Compose v2 | `docker compose version` | Containerized MySQL & multi-app preview |

---

## 4. Branching & Pull Request (PR) Conventions

To maintain codebase stability in a group environment:

### Branch Naming Convention
- `feat/<repo>-<feature-description>`: New functional capability (e.g., `feat/api-jwt-auth`, `feat/public-hero-section`)
- `fix/<repo>-<bug-description>`: Bug correction or defect fix (e.g., `fix/student-timetable-timezone`)
- `docs/<description>`: Documentation and contract modifications (e.g., `docs/api-contract-v1`)
- `refactor/<description>`: Non-functional internal restructuring (e.g., `refactor/api-exception-handler`)

### Conventional Commit Messages
Commit messages must adhere strictly to conventional commit syntax:
- `feat(scope): concise summary of addition`
- `fix(scope): concise summary of patch`
- `docs(scope): documentation update`
- `style(scope): formatting or styling change without code logic change`
- `test(scope): addition or modification of automated tests`

### Git Workflow Rules
1. `main` branch is always protected; direct commits are prohibited.
2. Every feature or fix must originate from a dedicated feature branch branched off `main`.
3. Before raising a PR, run local test builds (`mvn test` for backend, `ng test` and `ng build` for frontends).
4. Code review approval from at least one peer is required before merging into `main`.

---

## 5. Team Role Assignment Matrix

| Project Role | Primary Focus Area | Assigned Student | Backup / Secondary |
|:---|:---|:---|:---|
| **Project Manager & Requirements Lead** | Deliverable tracking, API contract alignment, ER diagram sign-off, demo coordination | `[Team Member Name 1]` | `[Team Member Name 2]` |
| **Public SPA Lead Developer** | `dance-academy-public` design system, responsive UI, hero/catalog sections, inquiry forms | `[Team Member Name 2]` | `[Team Member Name 1]` |
| **Portals Lead Developer** | `dance-academy-student` & `dance-academy-admin` SPAs, route guards, state management, UI forms | `[Team Member Name 3]` | `[Team Member Name 4]` |
| **Backend & Security Engineer** | `dance-academy-api` Spring Boot modular monolith, Spring Security 6 JWT, RBAC, REST controllers | `[Team Member Name 4]` | `[Team Member Name 5]` |
| **Database & QA/DevOps Owner** | MySQL schema, Flyway migrations, Docker Compose, automated verification scripts, backup runbooks | `[Team Member Name 5]` | `[Team Member Name 3]` |

---

## 6. Development Phasing Workflow

1. **Phase 0–1 (Docs & Contracts)**: Lock requirements, complete 20-table ER design, and freeze REST API contract in `dance-academy-docs`.
2. **Phase 2–5 (Backend Engine)**: Scaffold `dance-academy-api`, run Flyway migrations (`V1`, `V2`), implement Spring Security 6 with JWT, and build Public, Student, and Admin API endpoints with tests.
3. **Phase 6–8 (Public Website)**: Scaffold and complete `dance-academy-public` on port `4200`, connecting to live public endpoints.
4. **Phase 9–10 (Student Portal)**: Scaffold and complete `dance-academy-student` on port `4201`, enforcing student ID data isolation.
5. **Phase 11–13 (Admin Portal & Platform)**: Scaffold `dance-academy-admin` on port `4202`, implement 16 operational modules, and configure `dance-academy-platform` with Docker Compose.
6. **Phase 14 (Verification & Viva)**: Execute end-to-end automated smoke tests (`verify-e2e.sh`), audit security boundaries, and rehearse viva demonstration script.
