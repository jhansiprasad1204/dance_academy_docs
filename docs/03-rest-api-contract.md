# Classical Dance Academy Management System
## Complete REST API Specification (Contract First)
**API Base Path:** `/api/v1`  
**Standard Documentation:** OpenAPI 3.0 / Swagger UI (`http://localhost:8080/swagger-ui.html`)

---

## 1. Global API Conventions

### 1.1 URI & Naming Conventions
- **Resource URIs:** Plural nouns in `kebab-case` (e.g., `/api/v1/admin/dance-styles`, `/api/v1/student/class-sessions`).
- **Payload Attribute Naming:** Strict `camelCase` for all JSON properties (e.g., `studentNumber`, `danceStyleId`).
- **HTTP Methods:**
  - `GET`: Safe, idempotent read operations.
  - `POST`: Create resource or execute non-idempotent action.
  - `PUT`: Complete idempotent resource update or status replacement.
  - `PATCH`: Partial resource update.
  - `DELETE`: Resource removal or logical soft-deletion.

### 1.2 Authentication & Authorization Headers
- Protected endpoints require an `Authorization` header containing an active JWT Bearer token:
  ```http
  Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
  ```
- Public endpoints under `/api/v1/public/**` permit unauthenticated access (`PUBLIC_ANONYMOUS`).
- Missing or invalid tokens return `HTTP 401 Unauthorized`.
- Insufficient role privileges return `HTTP 403 Forbidden`.

### 1.3 Standard Response Wrappers

#### Success Single Resource: `ApiResponse<T>`
```json
{
  "success": true,
  "message": "Operation completed successfully",
  "data": { ... },
  "timestamp": "2026-10-27T11:15:30Z"
}
```

#### Success Paginated Resource: `PagedResponse<T>`
Query parameters: `?page=0&size=10&sort=createdAt,desc`
```json
{
  "success": true,
  "data": {
    "content": [ ... ],
    "pageNumber": 0,
    "pageSize": 10,
    "totalElements": 48,
    "totalPages": 5,
    "isFirst": true,
    "isLast": false
  },
  "timestamp": "2026-10-27T11:15:30Z"
}
```

#### RFC 7807 Error Response: `ProblemDetails`
All error responses strictly follow the RFC 7807 standard:
```json
{
  "type": "https://api.academy.local/errors/validation-failed",
  "title": "Validation Failed",
  "status": 400,
  "detail": "Request payload contains 2 invalid fields",
  "instance": "/api/v1/public/enquiries/admissions",
  "timestamp": "2026-10-27T11:15:30Z",
  "violations": [
    {
      "field": "email",
      "message": "Must be a well-formed email address"
    },
    {
      "field": "phone",
      "message": "Phone number must be between 10 and 15 digits"
    }
  ]
}
```

---

## 2. Authentication & Identity Endpoints (`/api/v1/auth`)

### 2.1 `POST /api/v1/auth/login`
- **Roles:** `PUBLIC_ANONYMOUS`
- **Description:** Authenticates credentials and issues short-lived JWT Access Token & Refresh Token.
- **Request Body:**
  ```json
  {
    "email": "student.aarav@academy.local",
    "password": "Student@123"
  }
  ```
- **Response `200 OK`:**
  ```json
  {
    "success": true,
    "data": {
      "accessToken": "eyJhbGciOiJIUzI1Ni...",
      "tokenType": "Bearer",
      "expiresInSeconds": 3600,
      "refreshToken": "7f8b9c2a-1123-4567-89ab-cdef01234567",
      "user": {
        "id": 5,
        "email": "student.aarav@academy.local",
        "fullName": "Aarav Sharma",
        "roles": ["ROLE_STUDENT"],
        "studentId": 1
      }
    }
  }
  ```
- **Error `401 Unauthorized`:** Bad credentials or locked account.

### 2.2 `POST /api/v1/auth/refresh`
- **Roles:** `PUBLIC_ANONYMOUS`
- **Request Body:** `{"refreshToken": "7f8b9c2a-1123-4567-89ab-cdef01234567"}`
- **Response `200 OK`:** Returns new `accessToken` and refreshed `expiresInSeconds`.

### 2.3 `POST /api/v1/auth/logout`
- **Roles:** Authenticated (`ROLE_STUDENT`, `ROLE_INSTRUCTOR`, `ROLE_ACADEMY_ADMIN`, `ROLE_SUPER_ADMIN`)
- **Response `200 OK`:** Invalidate refresh token and session.

### 2.4 `GET /api/v1/auth/me`
- **Roles:** Authenticated
- **Response `200 OK`:** Returns active session profile, assigned permissions, and linked profile IDs (`studentId` or `instructorId`).

### 2.5 `POST /api/v1/auth/change-password`
- **Roles:** Authenticated
- **Request Body:**
  ```json
  {
    "currentPassword": "Student@123",
    "newPassword": "NewSecurePassword#2026"
  }
  ```
- **Response `200 OK`:** Password updated.

---

## 3. Public Website APIs (`/api/v1/public`)

*All public endpoints are strictly unauthenticated and read-only (except review submission and enquiry intake).*

### 3.1 `GET /api/v1/public/home`
- **Description:** Aggregated payload powering the public landing page in a single round-trip.
- **Response `200 OK`:**
  ```json
  {
    "success": true,
    "data": {
      "hero": {
        "title": "Preserving the Sacred Traditions of Indian Classical Dance",
        "subtitle": "Immersive Guru-Shishya training in Bharatanatyam, Kathak, and Kuchipudi",
        "bannerImageKey": "hero_arangetram_2026.webp"
      },
      "stats": {
        "activeStudents": 350,
        "masterGurus": 12,
        "annualRecitals": 24,
        "yearsOfHeritage": 18
      },
      "featuredStyles": [ ... ],
      "upcomingEvents": [ ... ],
      "featuredReviews": [ ... ]
    }
  }
  ```

### 3.2 `GET /api/v1/public/about`
- **Description:** Detailed institutional narrative: history, mission, vision, Guru-Shishya philosophy, and achievements.

### 3.3 `GET /api/v1/public/content/{sectionKey}`
- **Path Params:** `sectionKey` (`HOME_HERO`, `ABOUT_HISTORY`, `LOCATION_INFO`, `CONTACT_INFO`, `FOOTER_INFO`)
- **Response `200 OK`:** Section title, subtitle, structured JSON body, and banner keys.

### 3.4 `GET /api/v1/public/dance-styles`
- **Description:** List of all active classical dance curricula.
- **Response `200 OK`:** Array of styles (`id`, `name`, `slug`, `origin`, `difficultyLevel`, `durationMonths`, `imageKey`, `featuredInstructor`).

### 3.5 `GET /api/v1/public/dance-styles/{slug}`
- **Path Params:** `slug` (e.g., `bharatanatyam`, `kathak`)
- **Response `200 OK`:** Detailed curriculum syllabus, prerequisites, and assigned instructor profiles.

### 3.6 `GET /api/v1/public/faculty`
- **Description:** Faculty list filtering only instructors with `isPublicFeatured = true`.
- **Response `200 OK`:** Full names, bio, specializations, qualifications, photo keys.

### 3.7 `GET /api/v1/public/classes`
- **Query Params:** `styleId`, `skillLevel`, `ageGroup`, `admissionStatus`
- **Response `200 OK`:** Active class batches, schedules, venues, and seat availability status.

### 3.8 `GET /api/v1/public/gallery`
- **Query Params:** `category`, `mediaType`
- **Response `200 OK`:** Published images and recital videos with captions and accessibility alt text.

### 3.9 `GET /api/v1/public/files/{fileKey}`
- **Description:** Streams public gallery media and CMS images.
- **Security Check:** **Strictly blocks certificate file keys!** Attempting to fetch a private document key via this public streaming endpoint triggers an automatic `403 Forbidden` / `404 Not Found`.

### 3.10 `GET /api/v1/public/events`
- **Description:** Chronological list of upcoming events with `publicationStatus = 'PUBLISHED'`.

### 3.11 `GET /api/v1/public/reviews`
- **Description:** Returns approved reviews (`moderationStatus = 'APPROVED'`).

### 3.12 `POST /api/v1/public/reviews`
- **Description:** Visitor submission of testimonial/feedback.
- **Request Body:**
  ```json
  {
    "displayName": "Priya Nair",
    "rating": 5,
    "reviewText": "The discipline and artistic purity taught at this academy transformed my daughter's Bharatanatyam journey."
  }
  ```
- **Response `201 Created`:** Review saved with `moderationStatus = 'PENDING'`.

### 3.13 `POST /api/v1/public/enquiries/contact`
- **Request Body:**
  ```json
  {
    "applicantName": "Sunil Varma",
    "email": "sunil.varma@example.com",
    "phone": "+91 98765 43210",
    "subject": "Weekend Adult Kathak Batches",
    "message": "Inquiring about beginner Kathak classes for working professionals on Saturdays."
  }
  ```
- **Response `201 Created`:** Saved with `status = 'NEW'`.

### 3.14 `POST /api/v1/public/enquiries/admissions`
- **Request Body:**
  ```json
  {
    "applicantName": "Ananya Sen",
    "email": "ananya.sen@example.com",
    "phone": "+91 98450 12345",
    "interestedStyleId": 1,
    "preferredBatch": "Children Beginner Morning",
    "message": "Enrolling 7-year-old daughter for foundational Bharatanatyam training."
  }
  ```
- **Response `201 Created`:** Admission lead generated with `status = 'NEW'`.

---

## 4. Student Portal APIs (`/api/v1/student`)

*All endpoints require `ROLE_STUDENT`. The backend resolves student identity strictly through the authenticated JWT context.*

### 4.1 `GET /api/v1/student/dashboard`
- **Response `200 OK`:**
  ```json
  {
    "success": true,
    "data": {
      "enrolledClassesCount": 2,
      "overallAttendancePercentage": 92.5,
      "totalSessionsAttended": 37,
      "upcomingSessionsThisWeek": 2,
      "certificatesEarnedCount": 1,
      "unreadAnnouncementsCount": 3,
      "nextSession": {
        "className": "Bharatanatyam - Intermediate Batch A",
        "sessionDate": "2026-10-31",
        "startTime": "10:00:00",
        "endTime": "11:30:00",
        "venue": "Studio Hall 1",
        "instructorName": "Guru Ananya Devi"
      }
    }
  }
  ```

### 4.2 `GET /api/v1/student/profile`
- **Response `200 OK`:** Full profile data (read-only official markers + contact info).

### 4.3 `PUT /api/v1/student/profile`
- **Description:** Updates permitted contact fields only.
- **Request Body:**
  ```json
  {
    "phone": "+91 99887 76655",
    "address": "42 Classical Lane, Malleshwaram, Bengaluru",
    "emergencyContactName": "Ramesh Sharma",
    "emergencyContactPhone": "+91 99887 76650"
  }
  ```
- **Response `200 OK`:** Returns updated profile.

### 4.4 `GET /api/v1/student/classes` & `GET /api/v1/student/timetable`
- **Response `200 OK`:** Active enrolled classes and calendar sessions list with status (`SCHEDULED`, `RESCHEDULED`, `CANCELLED`).

### 4.5 `GET /api/v1/student/attendance`
- **Query Params:** `classId`, `page`, `size`
- **Response `200 OK`:** Attendance summary by style + paginated log of sessions with status (`PRESENT`, `ABSENT`, `LATE`, `EXCUSED`) and remarks.

### 4.6 `GET /api/v1/student/certificates`
- **Response `200 OK`:** Array of issued certificates (`id`, `certificateNumber`, `title`, `danceStyle`, `issueDate`, `issuerName`).

### 4.7 `GET /api/v1/student/certificates/{id}/download`
- **Description:** Streams private binary PDF/image blob from `StorageService`.
- **Security Check:** Validates `certificate.studentId == currentAuthenticatedStudentId`. Access attempts by other users return `403 Forbidden`.

### 4.8 `GET /api/v1/student/announcements`
- **Response `200 OK`:** List of notices matching student audience or enrolled classes, with `isRead` flag.

### 4.9 `POST /api/v1/student/announcements/{id}/read`
- **Response `200 OK`:** Records read receipt in `announcement_reads`.

### 4.10 `POST /api/v1/student/events/{eventId}/register`
- **Response `201 Created`:** Enrolls student in event before registration deadline.

---

## 5. Admin & Instructor Portal APIs (`/api/v1/admin`)

*Endpoints require `ROLE_ACADEMY_ADMIN` or `ROLE_SUPER_ADMIN`, except instructor-scoped routes which permit `ROLE_INSTRUCTOR`.*

### 5.1 `GET /api/v1/admin/dashboard`
- **Roles:** `ROLE_ACADEMY_ADMIN`, `ROLE_SUPER_ADMIN`
- **Response `200 OK`:** Operational KPIs (Active students, instructors, sessions today, today's attendance %, pending enquiries, pending reviews).

### 5.2 Students & Instructors Management
- `GET /api/v1/admin/students` (Paginated, searchable by name, student number, or status)
- `POST /api/v1/admin/students` (Creates user credentials + student profile in single transaction)
- `GET /api/v1/admin/students/{id}`
- `PUT /api/v1/admin/students/{id}`
- `POST /api/v1/admin/students/{id}/enroll` (Assigns student to class batch)
- `DELETE /api/v1/admin/students/{id}/enroll/{classId}` (Removes student from batch)
- `GET /api/v1/admin/instructors`, `POST /api/v1/admin/instructors`, `PUT /api/v1/admin/instructors/{id}`

### 5.3 Dance Styles & Classes
- `GET /api/v1/admin/dance-styles`, `POST /api/v1/admin/dance-styles`, `PUT /api/v1/admin/dance-styles/{id}`
- `GET /api/v1/admin/classes`, `POST /api/v1/admin/classes`, `PUT /api/v1/admin/classes/{id}`

### 5.4 Timetable & Session Management
- `GET /api/v1/admin/sessions` (Filterable by class, date range, instructor)
- `POST /api/v1/admin/sessions/bulk-generate` (Generates recurring session instances for a batch over a date interval)
- `PUT /api/v1/admin/sessions/{id}` (Update timing, venue, or cancel/reschedule with remarks)

### 5.5 Instructor-Scoped Endpoints
- `GET /api/v1/admin/my-classes` (`ROLE_INSTRUCTOR`: Returns only classes taught by current guru)
- `GET /api/v1/admin/my-sessions` (`ROLE_INSTRUCTOR`: Returns only sessions assigned to current guru)

### 5.6 Attendance Roster & Marking
- `GET /api/v1/admin/attendance/sessions/{sessionId}` (`ROLE_INSTRUCTOR`, `ROLE_ACADEMY_ADMIN`: Returns enrolled students list with current attendance status)
- `POST /api/v1/admin/attendance/sessions/{sessionId}/bulk-mark`
  - **Request Body:**
    ```json
    {
      "records": [
        {"studentId": 1, "status": "PRESENT", "remarks": "Excellent mudra precision"},
        {"studentId": 2, "status": "ABSENT", "remarks": "Prior notice received"}
      ]
    }
    ```
  - **Response `200 OK`:** Upserts records preventing duplicate entries.

### 5.7 Certificate Issuance
- `GET /api/v1/admin/certificates`
- `POST /api/v1/admin/certificates` (Uploads file to `StorageService` and persists certificate record)
- `DELETE /api/v1/admin/certificates/{id}` (Marks status as `REVOKED`)

### 5.8 Content Management (CMS), Media & Reviews
- `GET /api/v1/admin/cms`, `PUT /api/v1/admin/cms/{sectionKey}`
- `GET /api/v1/admin/gallery`, `POST /api/v1/admin/gallery`, `DELETE /api/v1/admin/gallery/{id}`
- `GET /api/v1/admin/reviews` (Filterable by `PENDING`, `APPROVED`, `REJECTED`)
- `PUT /api/v1/admin/reviews/{id}/moderate`
  - **Request Body:** `{"status": "APPROVED", "reason": "Verified parent testimonial"}`

### 5.9 Enquiries & Leads
- `GET /api/v1/admin/enquiries`
- `PUT /api/v1/admin/enquiries/{id}/status`
  - **Request Body:** `{"status": "IN_PROGRESS", "assignedStaffId": 2, "resolutionNotes": "Contacted via phone"}`

### 5.10 User Administration & Security Audit (`ROLE_SUPER_ADMIN` Only)
- `GET /api/v1/admin/users`: Paginated system accounts
- `PUT /api/v1/admin/users/{id}/roles`: Assign/revoke roles
- `PUT /api/v1/admin/users/{id}/status`: Toggle account status (`ACTIVE`, `LOCKED`, `INACTIVE`)
- `GET /api/v1/admin/roles`: System role list
- `GET /api/v1/admin/audit-logs`: Searchable security and operational compliance trail

### 5.11 Analytics & CSV Export
- `GET /api/v1/admin/reports/attendance`: Aggregated metrics
- `GET /api/v1/admin/reports/attendance/export-csv`: Streams raw RFC 4180 CSV export file

