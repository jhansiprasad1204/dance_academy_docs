# Classical Dance Academy Management System
## Functional Requirements, Acceptance Criteria & RBAC Specification

---

## 1. Project Overview & Scope

The **Classical Dance Academy Management System** provides a centralized, secure, multi-tenant-ready digital platform for traditional Indian classical dance academies (specializing in Bharatanatyam, Kathak, Kuchipudi, Odissi, and Mohiniyattam). 

The platform bridges three critical operational domains:
1. **Public Presence:** Promoting heritage, faculty, curriculum, dance styles, upcoming events, and handling visitor admission enquiries.
2. **Student Academic Portal:** Providing enrolled students with transparent, self-service access to attendance logs, class timetables, secure certificate downloads, and announcements.
3. **Academy Administration & Faculty Portal:** Equipping administrators with comprehensive management over admissions, student/instructor profiles, batch scheduling, attendance sheets, certificate issuance, reviews moderation, and content management.

### 1.1 In-Scope MVP Features
- **Public Academy SPA (`:4200`):**
  - Modern, responsive landing page with deep-maroon (`#4a0e17`), warm ivory, and muted temple-gold (`#d97706`) aesthetics.
  - Section-by-section experience: 16:9 Hero, About (History, Mission, Guru-Shishya philosophy), Dance Styles catalog, Faculty directory, Classes directory, Lightbox Gallery, Upcoming Events, Moderated Reviews display & submission, Location/Map, and Admission/Contact enquiry forms.
  - Client-side validation with zero false-submission guarantees (form state depends on HTTP 200/201 response).
- **Core Security & Identity (`:8080`):**
  - Stateless JWT authentication with access/refresh tokens.
  - 4 standard system roles: `ROLE_SUPER_ADMIN`, `ROLE_ACADEMY_ADMIN`, `ROLE_INSTRUCTOR`, `ROLE_STUDENT`.
  - Passwords hashed using BCrypt (`cost factor = 12`).
  - Strict server-side horizontal privilege protection (Insecure Direct Object Reference / IDOR defense): a student can never access another student's profile, schedule, attendance, or certificate files.
- **Student Portal SPA (`:4201`):**
  - Sections A–G: Dashboard KPIs, Profile details (read-only official, editable emergency contact info), Classes & Timetable, Attendance Logs, Private Blob Certificate Downloads, Announcements (with read receipts), and Account Settings.
- **Admin & Instructor Portal SPA (`:4202`):**
  - Role-aware administrative shell with dynamic sidebar based on user privileges.
  - Modules 1–16: Dashboard KPIs, Student Management, Instructor Management, Dance Styles, Class Batches, Timetable Session Generator, Attendance Sheet Marking, Certificate Management, Events CMS, Website Content CMS, Gallery Media Manager, Reviews Moderation Gate, Enquiries Management, User/Role Administration (Super Admin only), Attendance CSV Reporting, and Audit Logs Viewer.
  - Mandatory confirmation modal (`ConfirmDialogComponent`) before any status alteration or destructive action.
- **Backend Architecture & Database:**
  - Spring Boot 3.x modular monolith with clean domain modules (`publicapi`, `student`, `admin`, `auth`, `common`).
  - Strict 20-table MySQL 8.0 schema versioned via Flyway (`V1__init_schema.sql` and `V2__seed_initial_data.sql`).
  - Centralized RFC 7807 `ProblemDetails` error handling and OpenAPI 3.0 documentation (`/swagger-ui.html`).

### 1.2 Out-of-Scope (Deferred Post-MVP)
- Online payment gateway integration (Stripe/Razorpay for automated tuition fee billing).
- Visual drag-and-drop certificate designer tool.
- SMS / WhatsApp automated messaging gateway (notifications are delivered via in-app announcements).
- Native biometric fingerprint hardware scanner integration for studio check-ins.
- Multi-branch cross-academy hierarchical federation.

---

## 2. User Personas

| Persona | Role Key | Motivations & System Interactions |
|:---|:---|:---|
| **Prospective Student / Parent** | `PUBLIC_ANONYMOUS` | Explores academy dance styles, reviews instructor qualifications, views fee/duration expectations, submits admission or contact inquiries, and reads verified student testimonials. |
| **Enrolled Dance Student** | `ROLE_STUDENT` | Authenticates securely to review upcoming batch schedules, inspect attendance percentage, view performance remarks from instructors, download verified certificates, and read academy notices. |
| **Academy Instructor / Guru** | `ROLE_INSTRUCTOR` | Views assigned classes and batches, inspects student enrollment lists for their sessions, and marks real-time attendance records with status codes (Present, Absent, Late, Excused) and pedagogical remarks. |
| **Academy Administrator / Registrar** | `ROLE_ACADEMY_ADMIN` | Coordinates day-to-day operations: schedules class sessions, admits new students, creates instructor profiles, issues certificates, publishes gallery media and events, moderates visitor reviews, and resolves incoming inquiries. |
| **Super Administrator / Academy Director** | `ROLE_SUPER_ADMIN` | Oversees total system governance: provisions admin and instructor logins, assigns or revokes security roles, inspects security audit logs, exports academy performance reports, and modifies global configurations. |

---

## 3. Non-Functional Requirements (NFR)

### 3.1 Accessibility (WCAG 2.1 Level AA)
- Semantic HTML5 structure throughout all components (`<header>`, `<nav>`, `<main>`, `<article>`, `<section>`, `<footer>`).
- Contrast ratio between text and background of at least `4.5:1` for regular body text and `3:1` for large headings.
- Explicit visual keyboard focus rings (`:focus-visible`) for all interactive buttons, links, and form fields.
- Descriptive ARIA attributes (`aria-label`, `aria-expanded`, `aria-live`, `aria-describedby`) for dialogs, drawer menus, accordions, and star ratings.
- Zero reliance on color alone to convey state (all badges pair color with text or iconography).
- Full support for `@media (prefers-reduced-motion: reduce)` disabling non-essential transitions and animations.

### 3.2 Responsive Design Viewports
All user interfaces must adapt smoothly across three primary benchmark viewports:
1. **Mobile (375px):** Single-column stacked layouts, hamburger navigation drawer, touch targets with minimum dimensions of `44x44px`, horizontal swipe/scroll tables with sticky key identifiers.
2. **Tablet (768px):** Two-column card grids, condensed sidebars or collapsible navigation bars, balanced spacing.
3. **Desktop (1440px):** Full multi-column dashboard grids, fixed or persistent sidebars, optimized data tables with sticky headers.

### 3.3 Security & Authorization
- **Stateless Authentication:** JSON Web Tokens (JWT) signed with HMAC-SHA256 (`HS256`) or RSA. Short-lived access tokens (15–60 minutes) paired with secure refresh tokens.
- **Server-Side Enforcement:** Every endpoint protected with Spring Security method-level annotations (`@PreAuthorize("hasRole('ROLE_...')")`).
- **Data Isolation:** For `ROLE_STUDENT`, the active `student_id` is extracted strictly from the JWT authentication claims or verified via database lookup against `user_id`. Request parameters attempting to supply an arbitrary `student_id` are rejected or ignored.
- **File Access Protection:** Certificates and private academic files are stored securely outside web roots (`StorageService`). Streaming requires active authentication and an explicit ownership verification check (`certificate.student_id == authenticated_student_id`). Direct file access is blocked.
- **Cross-Site Protection:** Strict CORS configurations specifying exact allowed origins, Content Security Policy (CSP) headers, and XSS sanitization on all free-text input.

### 3.4 Data Integrity & Auditing
- Composite unique database constraints prevent duplicate class enrollments and duplicate session attendance marks.
- Critical administrative operations (role modification, account status changes, certificate revocations) generate structured rows in `audit_logs` storing actor ID, action, target entity, timestamp, and JSON delta.

---

## 4. Functional Requirements & Acceptance Criteria

### 4.1 Website 1: Public Academy SPA (`dance-academy-public`)

#### Section 1: Header & App Shell
- **FR-PUB-01:** Sticky navigation header with academy branding, desktop menu links (Home, About, Styles, Faculty, Classes, Gallery, Events, Reviews, Location, Contact), and "Apply for Admission" call-to-action button.
- **AC-PUB-01.1:** Mobile viewport collapses links into a fully accessible slide-out drawer with focus trapping.
- **AC-PUB-01.2:** Smooth scrolling links navigate cleanly to target anchor sections on the home page.

#### Section 2: Home Hero
- **FR-PUB-02:** Prominent 16:9 banner displaying classical Indian dance imagery, inspiring headline, concise introductory summary, and two primary action buttons: "Explore Classes" and "Enquire About Admissions".
- **AC-PUB-02.1:** Background images use responsive picture elements with descriptive alt text.

#### Section 3: About Us
- **FR-PUB-03:** Content sections illustrating academy history, mission, vision, Guru-Shishya parampara teaching philosophy, and verified achievements.
- **AC-PUB-03.1:** Content is consumed dynamically from `/api/v1/public/about` with graceful fallback to labeled demo content if the backend is offline.

#### Section 4: Dance Styles Catalog
- **FR-PUB-04:** Presentation of classical dance forms offered (Bharatanatyam, Kathak, Kuchipudi, Odissi, Mohiniyattam).
- **AC-PUB-04.1:** Each card details style title, cultural origin, difficulty tier, curriculum duration, cover image, and lead instructor.
- **AC-PUB-04.2:** Clicking a style opens a detailed overview modal or route (`/dance-styles/:slug`).

#### Section 5: Faculty Directory
- **FR-PUB-05:** Showcase of instructors where `is_public_featured = true`.
- **AC-PUB-05.1:** Displays guru's full name, portrait photo, specialization style, certifications, and years of experience.

#### Section 6: Classes & Batches
- **FR-PUB-06:** Interactive catalog of active courses filterable by dance style, age category (Children, Teens, Adults), skill level (Beginner, Intermediate, Advanced), and admission status (Open, Waitlist, Closed).
- **AC-PUB-06.1:** Each class card includes a button ("Enquire for This Batch") that smoothly transitions to the admissions form with the batch pre-selected.

#### Section 7: Media Gallery
- **FR-PUB-07:** Visual gallery presenting studio photography and recital videos classified by category tags.
- **AC-PUB-07.1:** Filterable by photo vs. video and category.
- **AC-PUB-07.2:** Clicking an item opens an accessible lightbox viewer with keyboard navigation (`Escape`, `ArrowLeft`, `ArrowRight`).

#### Section 8: Upcoming Events & Recitals
- **FR-PUB-08:** Chronological list of academy events, arangetrams, workshops, and seasonal productions.
- **AC-PUB-08.1:** Displays date, timing, venue address, banner graphic, registration deadline, and admission ticket details.

#### Section 9: Testimonials & Reviews
- **FR-PUB-09:** Carousel or grid of approved community reviews (`moderation_status = APPROVED`).
- **AC-PUB-09.1:** Provides an interactive "Write a Review" form allowing visitors to submit display name, rating (1–5 stars), and review text.
- **AC-PUB-09.2:** Displays a distinct confirmation modal explaining that the review has been submitted for administrative moderation and will appear after review.

#### Section 10: Location, Studio Directions & Footer
- **FR-PUB-10:** Studio contact details, operating hours, interactive map link/embed, emergency numbers, legal terms, and privacy policy links.

#### Section 11: Contact & Admission Enquiries
- **FR-PUB-11:** Reactive forms for general queries (`/enquiries/contact`) and student admission applications (`/enquiries/admissions`).
- **AC-PUB-11.1:** Real-time client-side validation (email syntax, phone format, mandatory fields).
- **AC-PUB-11.2:** Submissions show loading spinners, disable double-submissions, and report explicit backend error or success feedback.

---

### 4.2 Website 2: Student Portal SPA (`dance-academy-student`)

#### Section A: Authentication & Portal Shell
- **FR-STU-01:** Dedicated JWT login interface with credentials hint for demo environments (`Student@123`).
- **AC-STU-01.1:** Rejects logins from users without `ROLE_STUDENT` with clear error messaging.
- **AC-STU-01.2:** Enforces auto-logout and redirect upon receiving HTTP 401 Unauthorized responses.

#### Section B: Student Dashboard
- **FR-STU-02:** At-a-glance KPI summary tiles:
  - Total Enrolled Classes
  - Cumulative Attendance Percentage (%)
  - Next Upcoming Session (Date, Time, Venue, Guru)
  - Issued Certificates Count
  - Unread Announcements Count

#### Section C: Student Profile Management
- **FR-STU-03:** View official student profile details and update authorized contact fields.
- **AC-STU-03.1:** Read-only fields: Student ID/Number, Full Legal Name, Date of Birth, Enrollment Date, Account Status.
- **AC-STU-03.2:** Editable fields: Phone Number, Mailing Address, Emergency Contact Name, Emergency Contact Phone.

#### Section D: Classes & Timetable
- **FR-STU-04:** Multi-tab academic schedule viewer:
  - Tab 1: Enrolled Classes list with style name, instructor, batch timing, and venue room.
  - Tab 2: Calendar or chronological timetable of scheduled sessions with status badges (`SCHEDULED`, `RESCHEDULED`, `CANCELLED`) and instructor remarks.

#### Section E: Attendance Logs
- **FR-STU-05:** Granular attendance history breakdown:
  - Aggregate attendance score per enrolled dance style.
  - Filterable session history log displaying Session Date, Batch, Attendance Status (`PRESENT`, `ABSENT`, `LATE`, `EXCUSED`), and Guru's pedagogical remarks.

#### Section F: Certificate Repository & Secure Downloads
- **FR-STU-06:** List of issued graduation and competency certificates.
- **AC-STU-06.1:** Each certificate displays Title, Dance Style, Issue Date, Certificate Unique Serial Number, and Verification Badge.
- **AC-STU-06.2:** "Download Certificate" button issues an authenticated request to `/api/v1/student/certificates/{id}/download` and streams the binary blob directly to disk.

#### Section G: Announcements & Settings
- **FR-STU-07:** Feed of academy and batch-specific announcements.
- **AC-STU-07.1:** Unread indicator dot; clicking an announcement marks it as read via `/api/v1/student/announcements/{id}/read`.
- **AC-STU-07.2:** Account settings allow password change with old password verification.

---

### 4.3 Website 3: Admin & Instructor Portal SPA (`dance-academy-admin`)

#### Module 1: Executive Dashboard (M1)
- **FR-ADM-01:** Operational KPI overview: Active Students count, Active Instructors, Today's Scheduled Classes, Today's Attendance Metric, Pending Enquiries, and Pending Reviews requiring moderation.

#### Module 2: Student Management (M2)
- **FR-ADM-02:** Full student lifecycle management:
  - Searchable, paginated student registry table.
  - Student creation drawer (provisions `users` account + `student_profiles` record).
  - Status toggle (`ACTIVE`, `INACTIVE`, `SUSPENDED`).
  - Class Enrollment Drawer: Enrolling student into a class or revoking enrollment with unique constraint enforcement.

#### Module 3: Instructor Management (M3)
- **FR-ADM-03:** Guru and instructor profiles management:
  - Create/edit instructor code, qualifications, years of experience, and biography.
  - Toggle `is_public_featured` flag to control visibility on the public website.

#### Module 4: Dance Styles Curriculum (M4)
- **FR-ADM-04:** CRUD operations for academy dance forms: Title, URL Slug, Description, Difficulty Level, Duration in Months, and Assigned Lead Guru.

#### Module 5: Class Batches Management (M5)
- **FR-ADM-05:** Configuration of recurring academy classes: Title, Dance Style FK, Instructor FK, Batch Name, Age Group, Skill Level, Room Venue, Student Capacity limit, and Admission Status (`OPEN`, `WAITLIST`, `CLOSED`).

#### Module 6: Timetable Session Generator (M6)
- **FR-ADM-06:** Batch creation and scheduling of individual class sessions:
  - Tool to generate weekly recurring session instances across a calendar date range.
  - Reschedule or cancel individual sessions with mandatory remarks.

#### Module 7: Attendance Sheet & Marking (M7)
- **FR-ADM-07:** Real-time session roster marking for instructors and administrators:
  - Displays registered students for a selected class session.
  - Bulk or individual status toggling (`PRESENT`, `ABSENT`, `LATE`, `EXCUSED`) with custom notes.
  - Upsert logic preventing duplicate attendance records for the same student in the same session.

#### Module 8: Certificate Issuance & Repository (M8)
- **FR-ADM-08:** Provisioning official certificates:
  - Assign certificate to student and dance style with auto-generated unique certificate number (`CERT-YYYY-XXXX`).
  - Securely upload certificate file PDF/image via `StorageService`.
  - Revoke issued certificates with reason logging.

#### Module 9: Events & Recitals CMS (M9)
- **FR-ADM-09:** Publishing academy workshops, guest recitals, and arangetram dates:
  - Manage dates, venue, capacity, banner graphic, registration deadlines, and lifecycle state (`DRAFT`, `PUBLISHED`, `ARCHIVED`).

#### Module 10: Website Content Management (M10)
- **FR-ADM-10:** CMS section editor for public web copy (`HOME_HERO`, `ABOUT_HISTORY`, `ABOUT_MISSION`, `ABOUT_VISION`, `ABOUT_PHILOSOPHY`, `LOCATION_INFO`, `CONTACT_INFO`, `FOOTER_INFO`).

#### Module 11: Media Gallery Manager (M11)
- **FR-ADM-11:** Upload, tag, categorize, and publish/unpublish high-resolution photos and video links.

#### Module 12: Reviews & Testimonials Moderation (M12)
- **FR-ADM-12:** Moderation queue for submitted visitor and parent reviews:
  - Filter by `PENDING`, `APPROVED`, `REJECTED`.
  - Admin action to approve or reject with internal moderation notes.

#### Module 13: Enquiries & Admissions Desk (M13)
- **FR-ADM-13:** Unified inbox for Contact Enquiries and Admission Applications:
  - Filter by status (`NEW`, `IN_PROGRESS`, `RESOLVED`, `CLOSED`).
  - Assign enquiry to staff member and log resolution follow-ups.

#### Module 14: User & Security Role Administration (M14)
- **FR-ADM-14:** Accessible strictly to `ROLE_SUPER_ADMIN`:
  - List all accounts, assign/revoke system roles, unlock or deactivate user accounts, reset passwords.

#### Module 15: Attendance Analytics & CSV Export (M15)
- **FR-ADM-15:** Generate aggregated attendance percentage reports filtered by style, batch, and date range, with one-click "Export CSV" functionality.

#### Module 16: Security Audit Trail (M16)
- **FR-ADM-16:** Read-only log viewer recording critical administrative actions, actor IDs, IP sources, affected entities, and JSON diffs.

---

## 5. Detailed Role-Based Access Control (RBAC) Matrix

Legend:
- **C**: Create
- **R**: Read / View
- **U**: Update / Edit
- **D**: Delete
- **A**: Approve / Moderate / Mark
- **-**: No Access (HTTP 401 or 403)

| Domain / Operational Module | `PUBLIC_ANONYMOUS` | `ROLE_STUDENT` | `ROLE_INSTRUCTOR` | `ROLE_ACADEMY_ADMIN` | `ROLE_SUPER_ADMIN` |
|:---|:---:|:---:|:---:|:---:|:---:|
| **Public Website Content & Hero** | R | R | R | C / R / U | C / R / U / D |
| **Dance Styles Curriculum** | R | R | R | C / R / U | C / R / U / D |
| **Faculty Public Directory** | R | R | R | C / R / U | C / R / U / D |
| **Public Classes & Batches** | R | R | R | C / R / U | C / R / U / D |
| **Gallery Media (Public)** | R | R | R | C / R / U / D | C / R / U / D |
| **Upcoming Events (Public)** | R | R | R | C / R / U | C / R / U / D |
| **Visitor Reviews (Approved)** | R | R | R | R | R |
| **Visitor Review Submission** | C *(Pending)* | C *(Pending)* | C *(Pending)* | C *(Pending)* | C *(Pending)* |
| **Review Moderation Gate** | - | - | - | A / U / D | A / U / D |
| **Contact / Admission Enquiries** | C | C | - | R / U / A | R / U / D |
| **Student Self-Profile** | - | R / U *(Contact Only)* | - | R / U | R / U / D |
| **Student Own Timetable & Classes**| - | R *(Enrolled Only)* | - | R | R |
| **Student Own Attendance Logs** | - | R *(Own Only)* | - | R | R |
| **Student Own Certificates** | - | R / Download *(Own)*| - | R | R |
| **Student Announcements** | - | R *(Audience/Enrolled)*| R | C / R / U / D | C / R / U / D |
| **Instructor Assigned Classes** | - | - | R *(Assigned Only)* | R | R / U / D |
| **Instructor Attendance Marking**| - | - | C / R / U *(Assigned)*| C / R / U / A | C / R / U / D |
| **Admin Student Management** | - | - | - | C / R / U | C / R / U / D |
| **Admin Instructor Management** | - | - | - | C / R / U | C / R / U / D |
| **Admin Timetable Generator** | - | - | - | C / R / U / D | C / R / U / D |
| **Admin Certificate Issuance** | - | - | - | C / R / U / D | C / R / U / D |
| **Admin Reports & CSV Export** | - | - | - | R / Export | R / Export |
| **User & Role Administration** | - | - | - | - | C / R / U / D |
| **Security Audit Logs** | - | - | - | - | R |

---

## 6. Placeholders & Data Replacement Guidelines

To comply with privacy mandates and institutional guidelines:
- All initial demo records must use clearly marked placeholder names (e.g., `Guru Ananya Devi [Demo Faculty]`, `Aarav Sharma [Demo Student]`).
- Studio address defaults to `108 Natya Marg, Cultural Corridor, Bengaluru, Karnataka 560001 [Demo Placeholder]`.
- Phone numbers default to standard non-routable demo prefixes (`+91 98000 00000`).
- No genuine student or client PII may be committed to version control, test scripts, or presentation slide decks.

