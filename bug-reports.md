# 🐛 High-Severity Defect Showcase & Root-Cause Analysis

This document showcases actual high-severity defects identified, isolated, and documented during manual testing cycles. Each report demonstrates a structured approach to defect reporting: providing precise reproduction steps, isolation steps to pinpoint root causes, API payload analysis, and clear expected vs. actual outcomes.

---

## 🛑 Defect 1: [Mobile/API] Race Condition on Rapid Sub-Seat Allocation Leading to Stripe Webhook Sync Failure

* **Severity:** High (Data Integrity & Billing Mismatch)
* **Platform:** Mobile (iOS/Android) & API backend
* **Area:** Subscription & Team Seat Provisioning

### Defect Overview
When rapidly inviting a user and assigning them a paid "Pro" doctor seat in quick succession via the mobile UI, concurrent API requests are triggered (`POST /api/invitations` and `PATCH /api/team/billing`). The backend fails to lock the database row before writing, resulting in a race condition. The database saves the user as a Pro doctor, but the Stripe webhook fails to sync the updated quantity, leaving a funded seat in the database that is completely unpaid in Stripe.

### Steps to Reproduce
1. Log in to the mobile application as a **Team Owner**.
2. Navigate to **Team Dashboard** -> **Invite Member**.
3. Enter a valid email address, select the **Doctor** role, and choose the **Pro Plan** (Paid Seat).
4. Tap the **"Send Invitation"** button multiple times rapidly (or simulate double-tap throttling bypass on low network speeds).
5. Inspect the team billing status.

### Expected Result
* The UI throttles subsequent clicks after the first tap.
* Only one API request executes successfully.
* The backend serializes the requests: creating the member record first, then safely incrementing the Stripe subscription quantity.

### Actual Result
* The UI allows multiple rapid taps, launching concurrent API payloads.
* The database creates a pending team member assigned to a paid doctor seat, but Stripe registers a webhook timeout error. 
* Result: A "funded" seat is granted in-app without a corresponding active Stripe charge (unbilled premium access).

---

## 🛑 Defect 2: [Web] Role-Based Access Bypass on Private-to-Team Visibility Mutation

* **Severity:** High (Security & Privacy Violation)
* **Platform:** Web (Chrome/Safari)
* **Area:** Case Visibility & Authorization Gates

### Defect Overview
A support staff user (who is restricted from self-assigning cases and is subject to a strict active-case limit) can bypass visibility restrictions on cases. By manually altering the payload of a `PATCH /api/cases/:id` request from a stale browser session, a support user can change a case's visibility from **Private** to **Team**. This bypasses the backend authorization checks that ensure the assigned doctor remains eligible to view the case on the newly chosen team.

### Steps to Reproduce
1. Log in as a **Support Staff** user on a team that has a funded doctor.
2. Open a **Private** case currently assigned to you (within your active case cap).
3. Open Chrome DevTools -> Network Tab.
4. Attempt to change the case visibility to **Team**, but intercept the PATCH request.
5. Inject a `team_id` payload belonging to a team where the assigned doctor is **unpaid/inactive** or not a member.
6. Submit the payload.

### Expected Result
* The server must validate that the assigned doctor is an active, funded member of the target team.
* The request should be blocked with `403 Forbidden` or `400 Bad Request` ("Assignee must be eligible on the target team").

### Actual Result
* The server successfully updates the case visibility to **Team** and binds it to the injected `team_id`.
* The unauthorized team members can now view a case containing sensitive private clinical data, bypassing the authorization barrier.

---

## 🛑 Defect 3: [Desktop] Standalone macOS Build Local SQLite Database Lockout on App Relaunch

* **Severity:** High (Functional Block / Crash)
* **Platform:** Desktop (macOS Standalone Build)
* **Area:** Offline Synchronization & Local Data Persistence

### Defect Overview
On standalone macOS desktop builds, if the application is closed or loses connection while trying to sync offline-cached clinical cases back to the cloud, the local SQLite database fails to release its write-ahead log (WAL) lock. Upon relaunching the desktop application, the UI freezes on the splash screen indefinitely because the application cannot establish a connection with its locked local database.

### Steps to Reproduce
1. Install and launch the macOS standalone desktop application build.
2. Toggle offline mode and create several offline clinical drafts.
3. Turn on the network connection to initiate the auto-sync sequence.
4. Force close the application (e.g., Command+Q or Force Kill via Activity Monitor) precisely while the syncing indicator is active.
5. Relaunch the application.

### Expected Result
* The sync process should run as an isolated background transaction.
* If interrupted, the local database connection should safely close, releasing any file locks.
* On relaunch, the app should initialize normally, detect the interrupted state, and resume syncing safely.

### Actual Result
* The app hangs indefinitely on the splash screen.
* macOS Console Logs show a persistent database lock exception: `SQLite3 Error: database is locked (SQLITE_BUSY)`.
* User is entirely blocked from accessing the application unless they manually clear their local application support cache.
