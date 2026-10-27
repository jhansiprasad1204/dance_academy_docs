# Classical Dance Academy Management System
## Database Backup & Restore Runbook and Viva Demonstration Guide

---

## 1. Local MySQL Backup & Restore Operations

### 1.1 Complete Database Backup (`mysqldump`)
Run the following command from your terminal to generate a complete, atomic snapshot of `dance_academy_db`.

```bash
# Set timestamp variable
BACKUP_TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_FILE="backup_dance_academy_db_${BACKUP_TIMESTAMP}.sql"

# Execute mysqldump with atomic transaction and complete metadata
mysqldump \
  --host=127.0.0.1 \
  --port=3306 \
  --user=root \
  --password \
  --default-character-set=utf8mb4 \
  --single-transaction \
  --quick \
  --routines \
  --triggers \
  --events \
  dance_academy_db > "${BACKUP_FILE}"
```

**Key Parameter Flags Explained:**
- `--single-transaction`: Produces a consistent snapshot of InnoDB tables without locking the database or disrupting ongoing reads/writes.
- `--quick`: Forces `mysqldump` to retrieve rows line-by-line rather than buffering entire tables into memory.
- `--routines --triggers --events`: Ensures stored procedures, database triggers, and scheduled events are fully captured.
- `--default-character-set=utf8mb4`: Preserves Indian language transliterations, Sanskrit diacritics, and emoji characters without corruption.

---

### 1.2 Disaster Recovery & Database Restoration

```bash
# 1. Connect to MySQL and recreate an empty database with correct collation
mysql -h 127.0.0.1 -P 3306 -u root -p -e "
  DROP DATABASE IF EXISTS dance_academy_db;
  CREATE DATABASE dance_academy_db 
    CHARACTER SET utf8mb4 
    COLLATE utf8mb4_unicode_ci;
"

# 2. Import the backup snapshot
mysql -h 127.0.0.1 -P 3306 -u root -p dance_academy_db < "${BACKUP_FILE}"
```

---

### 1.3 Post-Restore Verification Checklist
Execute these SQL queries to ensure complete recovery and schema integrity:

```sql
USE dance_academy_db;

-- 1. Verify exactly 20 application tables + flyway_schema_history exist
SELECT count(*) AS total_tables 
FROM information_schema.tables 
WHERE table_schema = 'dance_academy_db';
-- Expected result: 21 (20 domain tables + flyway_schema_history)

-- 2. Verify Flyway migrations status
SELECT installed_rank, version, description, type, success 
FROM flyway_schema_history;
-- Expected result: V1 (init schema) and V2 (seed initial data) marked success = 1

-- 3. Verify core data presence
SELECT 'users' AS entity, count(*) FROM users
UNION ALL SELECT 'roles', count(*) FROM roles
UNION ALL SELECT 'dance_styles', count(*) FROM dance_styles
UNION ALL SELECT 'classes', count(*) FROM classes;
```

---

## 2. Local-to-Cloud Migration Checklist

When transitioning the system from local workstations to cloud hosting:

1. **Database Provisioning:**
   - Deploy a managed MySQL 8.0 instance (e.g., Oracle Cloud Always Free MySQL HeatWave, AWS RDS MySQL, or Aiven).
   - Ensure `character_set_server = utf8mb4` and `collation_server = utf8mb4_unicode_ci`.
2. **Schema Migration Strategy:**
   - **Recommended:** Let the Spring Boot application run Flyway on cloud startup (`flyway.migrate()`) rather than importing raw local SQL dumps. This guarantees deterministic schema versioning.
3. **Environment Variable Injection:**
   - In your cloud runtime, configure the standard environment variables without committing secrets into Git:
     ```env
     DB_HOST=<cloud-mysql-endpoint>
     DB_PORT=3306
     DB_NAME=dance_academy_db
     DB_USER=<cloud-db-username>
     DB_PASS=<strong-cloud-password>
     JWT_SECRET=<min-64-character-cryptographic-random-secret>
     CORS_ALLOWED_ORIGINS=https://academy.pages.dev,https://student.academy.pages.dev,https://admin.academy.pages.dev
     STORAGE_LOCAL_ROOT=/data/academy/storage
     ```
4. **Certificate & Media Storage Abstraction:**
   - The system utilizes `StorageService`. In cloud environments, configure the cloud storage adapter (e.g., S3/OCI Object Storage) so that private certificates remain isolated from public access.

---

## 3. College Viva Demonstration Script (8–10 Minutes)

### Phase 1: Architecture & Architecture Defense (0:00 – 2:00)
- **Opening Statement:**
  > *"Good morning, esteemed panel. Today we present the Classical Dance Academy Management System—a full-stack enterprise platform engineered for classical performing arts institutions."*
- **Key Points to Highlight:**
  - **Modular Monolith Backend:** Developed in Java 25 (LTS) with Spring Boot 3.5.x, avoiding microservice network overhead while maintaining clean internal domain boundaries.
  - **Tri-Portal Architecture:** 3 dedicated Angular SPAs customized for distinct user experiences:
    1. Public Academy Website (port `:4200`)
    2. Student Self-Service Portal (port `:4201`)
    3. Academy Administration & Guru Portal (port `:4202`)
  - **Database Integrity:** 20 relational MySQL 8 tables strictly governed through versioned Flyway migrations (`V1` Schema, `V2` Seed Data).
  - Open `/swagger-ui.html` on port `:8080` to demonstrate the OpenAPI 3.0 specification covering all 40+ endpoints.

---

### Phase 2: Public Academy Experience (2:00 – 4:00)
- **URL:** `http://localhost:4200`
- **Demo Actions:**
  1. **Visual System:** Highlight the classical Indian aesthetic (temple-gold `#d97706` and deep maroon `#4a0e17` accents) and full WCAG AA contrast compliance.
  2. **16:9 Hero Section & Curriculum:** Browse through the 5 classical styles (Bharatanatyam, Kathak, Kuchipudi, Odissi, Mohiniyattam) and featured Guru bios.
  3. **Admissions Workflow:** Scroll to the Admissions Form. Submit a new application for a student (`Ananya Sen`). Show that the form reports immediate success and saves to the database with `status = 'NEW'`.
  4. **Community Feedback:** Submit a visitor review. Note that the review does **not** appear publicly right away because our system mandates administrative moderation.

---

### Phase 3: Administration, Timetable & Attendance (4:00 – 6:30)
- **URL:** `http://localhost:4202`
- **Demo Actions:**
  1. **Authentication:** Log in using Academy Admin credentials (`staff@academy.local` / `Admin@123`).
  2. **Executive Dashboard (Module 1):** Point out real-time KPI counters (Active Students, Classes Today, Attendance Metric, Pending Reviews).
  3. **Review Moderation (Module 12):** Locate the visitor review submitted in Phase 2. Approve it with an admin note. Switch back to `:4200` to show it now immediately appears in the public testimonials grid.
  4. **Admissions Desk (Module 13):** Locate the admission application for `Ananya Sen`. Update status to `IN_PROGRESS` and assign to staff.
  5. **Attendance Sheet Marking (Module 7):** Open today's Bharatanatyam class session. Toggle student attendance statuses (`PRESENT`, `ABSENT`). Point out the composite unique constraint preventing duplicate attendance records for the same session.
  6. **Certificate Issuance (Module 8):** Issue a graduation certificate for student `Aarav Sharma` with auto-generated serial number `CERT-2026-0001`.

---

### Phase 4: Student Portal & IDOR Security Proof (6:30 – 8:30)
- **URL:** `http://localhost:4201`
- **Demo Actions:**
  1. **Student Login:** Log in as enrolled student `student.aarav@academy.local` (`Student@123`).
  2. **Personalized Dashboard:** Show that Aarav sees only his personal statistics: 92.5% attendance, next class session, and unread announcements.
  3. **Timetable & Attendance:** Inspect the session logs with Guru remarks and reschedule notices.
  4. **Secure Certificate Download:** Click "Download Certificate". Show that the private binary blob streams directly to the browser.
  5. **Security / IDOR Defense Proof:** Open Developer Tools. Attempt to call `/api/v1/student/certificates/999/download` (belonging to another student). Demonstrate that the Spring Security context intercepts the request and issues an explicit `403 Forbidden` / `404 Not Found`.

---

### Phase 5: Technical Defense & Viva Q&A Cheat Sheet (8:30 – 10:00)

| Potential Viva Question | Architectural Defense Answer |
|:---|:---|
| **Why a modular monolith instead of microservices?** | Microservices introduce distributed transactions, network latency, and operational complexity unsuitable for a medium-scale academy. A modular monolith provides identical domain separation and testability within a single deployment unit while maintaining high performance. |
| **How is student privacy protected against IDOR attacks?** | The student portal never passes `student_id` in request payloads or URL parameters. The backend extracts user identity directly from the verified cryptographic JWT claims in `SecurityContextHolder`. Even if a student crafts a malicious request with another student's ID, the service layer rejects the operation with `403 Forbidden`. |
| **Why use Flyway instead of JPA's `ddl-auto=update`?** | `ddl-auto=update` is non-deterministic, hazardous in production, and does not maintain an auditable migration history. Flyway enforces reproducible, version-controlled SQL files (`V1`, `V2`) ensuring complete parity between local development and cloud deployments. |
| **How are passwords stored securely?** | Passwords are never stored in plaintext. They are salted and hashed using Spring Security's `BCryptPasswordEncoder` with a cost factor of 12, resisting offline rainbow table and dictionary attacks. |
| **How does the system ensure zero false form submissions?** | Frontends utilize reactive Angular form controls with real-time validators, and submission state transitions are tied strictly to HTTP 200/201 response receipts from the backend API. |
