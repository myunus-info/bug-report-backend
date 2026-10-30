# CourseMaster Backend — Bug Report & Debugging Documentation

**Project:** CourseMaster Backend REST API (`courseMaster_Backend`)  
**Technology Stack:** Node.js, Express 5, MongoDB, Mongoose, JWT, bcrypt, Zod  
**Repository:** [https://github.com/Pullock4981/courseMaster_Backend](https://github.com/Pullock4981/courseMaster_Backend)  
**Total Identified Bugs:** 10  
**Status:** All 10 Bugs Fixed & Verified (100% Resolved)  

---

## 1. Backend Bug Summary Table

| Bug ID | Title | Severity | Priority | Status |
| :--- | :--- | :--- | :--- | :--- |
| **BUG-BE-001** | Plaintext Password Comparison in Login Service (`===` instead of `bcrypt.compare`) | Critical | P1 | Fixed & Verified |
| **BUG-BE-002** | Custom CORS Header Restricts Methods to `GET, POST`, Blocking `PUT`, `DELETE`, and `PATCH` | Critical | P1 | Fixed & Verified |
| **BUG-BE-003** | Inverted Enrollment Logic Throws Error When User Is Not Enrolled | Critical | P1 | Fixed & Verified |
| **BUG-BE-004** | JWT Verification Fail-Open Middleware Calls `next()` on Invalid or Expired Token | Critical | P1 | Fixed & Verified |
| **BUG-BE-005** | Unrestricted Role Assignment During Registration (Bypasses Admin Key Verification) | High | P1 | Fixed & Verified |
| **BUG-BE-006** | Assignment Filter Queries Schema by `courseId` Instead of Schema Field `course` | High | P2 | Fixed & Verified |
| **BUG-BE-007** | Quiz Score Multiplied by 10 Instead of 100 (100% Calculated as 10%) | High | P2 | Fixed & Verified |
| **BUG-BE-008** | Missing `await` on Asynchronous Mongoose `enrollment.save()` in Lesson Completion | Medium | P2 | Fixed & Verified |
| **BUG-BE-009** | Backend `.env.example` Overwritten with Frontend Environment Variables | High | P1 | Fixed & Verified |
| **BUG-BE-010** | Banned User Account Status Ignored During Login Authentication | High | P2 | Fixed & Verified |

---

## 2. Severity & Priority Guidelines

### Severity Guidelines
- **Critical:** A major feature is unusable, the application crashes, data corruption occurs, or there is a severe security vulnerability breaking authentication.
- **High:** An important feature fails or produces significantly incorrect behavior.
- **Medium:** A feature works incorrectly, but a workaround exists.
- **Low:** A minor issue or low-impact functional inconsistency.

### Priority Guidelines
- **P1:** Fix immediately (Blocker / Core Auth / Security).
- **P2:** Fix as soon as possible (Core Business Logic / Data Integrity).
- **P3:** Fix when higher-priority issues are resolved.
- **P4:** Low urgency; fix when appropriate.

---

## 3. Detailed Backend Bug Reports

### Bug ID & Title: BUG-BE-001 — Plaintext password comparison prevents all user logins

**Location & Steps to Reproduce:** `courseMaster_Backend/src/services/auth.service.js` (line 39) → Send `POST /api/auth/login` with valid email (`admin@coursemaster.com`) and password (`admin123`).

**Expected vs. Actual Result:** The user should authenticate successfully and receive a signed JWT token. Instead, login always fails with HTTP 400 "Invalid email or password" because the code compares plain text password to the bcrypt hash using strict equality (`password === user.passwordHash`).

**Severity & Evidence:** Critical (Priority: P1) — Error message: `{"message": "Invalid email or password"}`. Source line: `const match = (password === user.passwordHash);`.

**Fix & Verification Status:** Replaced strict equality with `await bcrypt.compare(password, user.passwordHash)`. Verified successful login and token generation for both admin and student accounts. Status: Fixed & Verified.

---

### Bug ID & Title: BUG-BE-002 — Restrictive CORS header blocks PUT, DELETE, and PATCH preflight requests

**Location & Steps to Reproduce:** `courseMaster_Backend/app.js` (lines 31-35) → Send any `PATCH /api/admin/courses/:id`, `DELETE /api/users/:id`, or `PATCH /api/users/:id/ban` from the browser.

**Expected vs. Actual Result:** The API should accept authorized cross-origin `PATCH`, `DELETE`, and `PUT` requests with appropriate CORS headers. Instead, a custom middleware explicitly sets `Access-Control-Allow-Methods: GET, POST`, causing browser CORS preflights (`OPTIONS`) to fail and blocking all admin update and delete operations.

**Severity & Evidence:** Critical (Priority: P1) — Console error: `Access to XMLHttpRequest has been blocked by CORS policy: Method PATCH is not allowed by Access-Control-Allow-Methods in preflight response`.

**Fix & Verification Status:** Removed custom restrictive header middleware and configured Express `cors()` with comprehensive allowed methods: `["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"]`. Verified all PATCH, PUT, and DELETE routes work smoothly without CORS errors. Status: Fixed & Verified.

---

### Bug ID & Title: BUG-BE-003 — Inverted enrollment logic rejects un-enrolled students

**Location & Steps to Reproduce:** `courseMaster_Backend/src/services/enrollment.service.js` (lines 14-17) → Log in as student → Click "Enroll Now" (`POST /api/enrollments/:courseId`) on an un-enrolled course.

**Expected vs. Actual Result:** The system should enroll the student and create a new Enrollment record. Instead, the check is inverted (`if (!enrollment) throw new Error("Already enrolled in this course");`), so un-enrolled students are rejected with HTTP 400 "Already enrolled in this course", making course enrollment impossible across the entire platform.

**Severity & Evidence:** Critical (Priority: P1) — Response: `{"message": "Already enrolled in this course"}` when attempting first-time enrollment.

**Fix & Verification Status:** Inverted the condition to `if (enrollment) { throw new Error("Already enrolled in this course"); }`. Verified un-enrolled students can enroll successfully while preventing duplicate enrollments. Status: Fixed & Verified.

---

### Bug ID & Title: BUG-BE-004 — Auth middleware fails open on invalid or expired JWT token

**Location & Steps to Reproduce:** `courseMaster_Backend/src/middlewares/authMiddleware.js` (lines 17-20) → Send request to any protected route (`/api/auth/me`, `/api/enrollments/my`) with an invalid or expired token.

**Expected vs. Actual Result:** The server should reject invalid tokens immediately with HTTP 401 Unauthorized (`{"message": "Invalid or expired token"}`). Instead, the `catch` block calls `next()`, allowing unauthenticated requests to proceed with `req.user = undefined`, resulting in uncaught `TypeError: Cannot read properties of undefined (reading 'id')` and HTTP 500 crashes.

**Severity & Evidence:** Critical (Priority: P1) — Error log: `TypeError: Cannot read properties of undefined (reading 'id') at me (courseMaster_Backend/src/controllers/auth.controller.js:25:47)`.

**Fix & Verification Status:** Updated `authMiddleware` `catch` block to immediately return `res.status(401).json({ message: "Invalid or expired token" })` and verified token presence before verification. Status: Fixed & Verified.

---

### Bug ID & Title: BUG-BE-005 — Unrestricted role assignment allows privilege escalation to admin

**Location & Steps to Reproduce:** `courseMaster_Backend/src/services/auth.service.js` (lines 14-20) → Send `POST /api/auth/register` with `{ name, email, password, role: "admin" }` without providing `adminKey`.

**Expected vs. Actual Result:** Newly registered users should strictly default to `"student"`. The `"admin"` role must require validating a secret `adminKey` against `process.env.ADMIN_REGISTRATION_KEY`. Instead, the service blindly accepted `role: role || "student"`, allowing any user to register directly as an Administrator.

**Severity & Evidence:** High (Priority: P1) — Privilege escalation: New user created with `role: "admin"` in MongoDB without admin key validation.

**Fix & Verification Status:** Restored strict `ADMIN_REGISTRATION_KEY` validation in `register`. If `adminKey` or `role === "admin"` is submitted, it validates against `process.env.ADMIN_REGISTRATION_KEY`, otherwise assigning `"student"`. Status: Fixed & Verified.

---

### Bug ID & Title: BUG-BE-006 — Assignment filter queries Mongoose model by courseId instead of course

**Location & Steps to Reproduce:** `courseMaster_Backend/src/services/admin.service.js` (line 42) → Admin Dashboard → Assignments → Filter submissions by course (`GET /api/admin/assignments?courseId=<ID>`).

**Expected vs. Actual Result:** The endpoint should return all student assignment submissions belonging to the selected course. Instead, the query constructs `{ courseId: query.courseId }`. Because the `AssignmentSubmission` model defines the field as `course`, the query always returns an empty list `[]`.

**Severity & Evidence:** High (Priority: P2) — Response: `[]` (empty submissions array) even when submissions exist for the specified course. Source line: `if (query.courseId) filter.courseId = query.courseId;`.

**Fix & Verification Status:** Changed query filter to `if (query.courseId) filter.course = query.courseId`. Verified assignment submissions correctly populate when filtered by course ID. Status: Fixed & Verified.

---

### Bug ID & Title: BUG-BE-007 — Quiz score percentage formula multiplied by 10 instead of 100

**Location & Steps to Reproduce:** `courseMaster_Backend/src/services/quiz.service.js` (line 34) → Submit a quiz with 2/2 correct answers (`POST /api/quizzes/submit`).

**Expected vs. Actual Result:** A score of 2/2 should be calculated as 100%: `Math.round((score / total) * 100)`. Instead, it was multiplied by 10 (`Math.round((score / total) * 10)`), so a perfect score is recorded as `10%` in the database and shown as failing to the student.

**Severity & Evidence:** High (Priority: P2) — Response payload: `{"score": 2, "total": 2, "percent": 10}` instead of `percent: 100`.

**Fix & Verification Status:** Corrected formula to `const percent = total === 0 ? 0 : Math.round((score / total) * 100)`. Verified perfect score produces `100%`. Status: Fixed & Verified.

---

### Bug ID & Title: BUG-BE-008 — Missing await on asynchronous enrollment.save() in lesson completion

**Location & Steps to Reproduce:** `courseMaster_Backend/src/services/enrollment.service.js` (line 61) → Send `PATCH /api/enrollments/:enrollId/complete-lesson`.

**Expected vs. Actual Result:** The function should await Mongoose persistence before returning the updated enrollment record. Instead, `enrollment.save()` lacks `await`, causing unhandled promise rejections and race conditions in serverless or high-concurrency environments where the response is returned before progress is saved.

**Severity & Evidence:** Medium (Priority: P2) — Code audit evidence: `enrollment.save(); return enrollment;` without `await`.

**Fix & Verification Status:** Added `await enrollment.save()`. Verified lesson completion progress persists reliably in MongoDB. Status: Fixed & Verified.

---

### Bug ID & Title: BUG-BE-009 — Backend .env.example overwritten with frontend variables

**Location & Steps to Reproduce:** `courseMaster_Backend/.env.example` → Inspect repository files after initial setup.

**Expected vs. Actual Result:** `.env.example` should contain server port, MongoDB URI, JWT secret, and admin registration key. Instead, the file contained frontend environment variables (`VITE_API_BASE_URL`), causing developers setting up the backend to lack database and JWT configuration.

**Severity & Evidence:** High (Priority: P1) — File content: `VITE_API_BASE_URL=https://course-master-backend-ochre.vercel.app/api`. Missing `PORT`, `MONGODB_URI`, `JWT_SECRET`.

**Fix & Verification Status:** Restored proper backend `.env.example` documentation with all required variables and created a local working `.env`. Status: Fixed & Verified.

---

### Bug ID & Title: BUG-BE-010 — Banned user account status ignored during login authentication

**Location & Steps to Reproduce:** `courseMaster_Backend/src/services/auth.service.js` → Ban a user (`isBanned: true`) via user management → Attempt to log in with that user's credentials (`POST /api/auth/login`).

**Expected vs. Actual Result:** The server should reject the login attempt with HTTP 400 and an informative error message ("Your account has been suspended"). Instead, `auth.service.js` only checked the password hash and ignored `user.isBanned`, allowing suspended accounts to generate valid JWTs.

**Severity & Evidence:** High (Priority: P2) — Banned user successfully logged in and received valid token despite `isBanned === true`.

**Fix & Verification Status:** Added `if (user.isBanned) throw new Error("Your account has been suspended. Please contact support.");` before generating JWT tokens. Status: Fixed & Verified.

---

## 4. Instructions for Google Docs Submission

1. Open a new document at [Google Docs](https://docs.new).
2. Title the document: **CourseMaster Backend — Bug Report & Debugging Documentation**.
3. Copy the entire contents of this document and paste directly into Google Docs.
4. Set access permissions to **"Anyone with the link can view"**.
5. Copy the sharing link as your final Backend Bug Report deliverable.
