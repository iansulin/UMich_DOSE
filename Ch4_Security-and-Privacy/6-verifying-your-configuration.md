# 6. Verifying Your Configuration

Configuration errors are common, especially when working under enrollment deadlines. This section provides checklists and hands-on tests to confirm your Firebase setup protects participant data **before** the first real enrollment.

You do not need programming experience for most tests — a web browser and the Firebase Console are sufficient.

---

## 6.1 Pre-deployment security checklist

Complete every item. Do not enroll participants until all **Required** items are checked.

### Firebase project setup

- [ ] **Required:** Production Firebase project is separate from development/testing project
- [ ] **Required:** Firebase Realtime Database region is documented and matches ethics application
- [ ] **Required:** Default open rules (`.read: true`, `.write: true`) have been replaced
- [ ] **Required:** `FirebaseManager.databaseURL` in the app points to the **production** project URL
- [ ] Recommended: Project is owned by an institutional Google account, not personal email

### Security rules

- [ ] **Required:** App successfully uploads data in end-to-end tests (Section 6.3, Tests 5–7)
- [ ] **Required:** Browser read of `/devices.json` returns permission denied (Section 6.2, Test 1)
- [ ] **Required:** Writes are restricted to expected paths only (`notifications`, `healthkitdata`, `survey`)
- [ ] **Required:** Rules were **Published** (not just edited) — verify timestamp in Console
- [ ] **Required:** Rule version documented in study security records
- [ ] Recommended: User ID format validation matches your enrollment codes (Section 3, Example C)

### Device identification and User IDs

- [ ] **Required:** User ID assignment procedure documented (non-sequential codes recommended — Section 4)
- [ ] **Required:** Participants enter assigned User ID before data collection begins
- [ ] Recommended: Separate test User IDs (`TEST-` prefix) used only in development Firebase project

### Privacy and access

- [ ] **Required:** Consent form mentions cloud storage (Section 5.4)
- [ ] **Required:** Firebase Console access limited to named authorized staff (Section 5.5)
- [ ] **Required:** Enrollment key (User ID → identity) stored separately from Firebase
- [ ] Recommended: 2FA enabled on all accounts with Firebase access
- [ ] Recommended: Data export and retention procedure documented (Section 5.3)

### App configuration

- [ ] **Required:** Participants cannot use `test` or `demo` as User IDs (app enforces this)
- [ ] **Required:** Testing mode flag reviewed in `MainView.swift` (`isTestingMode`) before release build
- [ ] Recommended: Simulator testing uses `TEST-` prefixed IDs in dev project only

The app includes a testing mode flag (`isTestingMode` in `MainView.swift`) that displays debug controls when set to `1`. Set this to `0` for participant-facing release builds unless your protocol specifies otherwise.

---

## 6.2 How to test that public read access is blocked

These tests confirm that participant data cannot be downloaded via public URLs.

### Test 1 — Browser read attempt (most important)

1. Open a web browser (use a private/incognito window).
2. Navigate to (replace with your database URL):

```
https://YOUR-PROJECT-ID-default-rtdb.firebaseio.com/devices.json
```

**Expected result (secure):**

```json
{
  "error" : "Permission denied"
}
```

**Insecure result (rules not protecting data):**

```json
{
  "10042": {
    "survey": { ... },
    "healthkitdata": { ... }
  }
}
```

If you see participant data, **stop enrollment immediately**, publish lockdown rules (Section 3, Example A), and fix rules before proceeding.

---

### Test 2 — External write attempt (path and format validation)

This test checks whether someone **without Firebase Console access** can inject data from outside the app (e.g., via curl or a browser tool).

Run in Terminal (macOS/Linux) or use a REST client like [Postman](https://www.postman.com/):

```bash
curl -X POST \
  "https://YOUR-PROJECT-ID-default-rtdb.firebaseio.com/devices/99999999/survey.json" \
  -H "Content-Type: application/json" \
  -d '{"surveyNo":"99","response":"INVALID","submitted":"2025-06-15 12:00:00PM","delivered":"","opened":""}'
```

**Expected results depend on which rules you published:**

| Rules in use | User ID in test | Expected result | What it means |
|--------------|-----------------|-----------------|---------------|
| **Example B** | `99999999` | May **succeed** or fail on validation only | Writes are allowed to any `/devices/{id}/survey` path with valid fields — use non-guessable User IDs (Section 4.5) |
| **Example C** | `99999999` (non-matching format) | **Permission denied** | Format validation blocks unknown ID patterns |
| **Example C** | Valid-format ID not in your enrollment list | May **succeed** | Rules validate format, not enrollment — assign unpredictable codes |

The DOSE app writes to Firebase without a login step. Examples B and C are not designed to block every external write; their primary control is denying public reads (Test 1). Secondary controls include path restriction, field validation, and unpredictable User ID assignment.

If a test write succeeds, remove the record from Firebase Console immediately. Do not use a User ID assigned to a real participant for this test.

---

### Test 3 — Read individual participant path

Even if the top-level read fails, verify nested paths are also protected:

```
https://YOUR-PROJECT-ID-default-rtdb.firebaseio.com/devices/10042/survey.json
```

**Expected:** Permission denied (unless `10042` is a real test ID you are authorized to check via Console — browser tests should still fail).

---

### Test 4 — External delete attempt

```bash
curl -X DELETE \
  "https://YOUR-PROJECT-ID-default-rtdb.firebaseio.com/devices/99999999/survey.json"
```

**Expected:** Permission denied.

Do not run DELETE tests against real participant User IDs. Use a dedicated test ID such as `99999999`.

---

### Test results log (example)

| Test | Date | Performed by | Result | Notes |
|------|------|--------------|--------|-------|
| Browser read `/devices.json` | 2025-06-10 | J. Lee | Denied | Incognito Chrome |
| External POST (invalid User ID format) | 2025-06-10 | J. Lee | Denied | Example C active |
| External DELETE | 2025-06-10 | J. Lee | Denied | — |
| App survey upload (Test 7) | 2025-06-10 | J. Lee | Success | Test watch, User ID `99999` |

Retain this log for ethics and audit documentation.

---

## 6.3 How to test that the app can still write data successfully

Security rules must block public read access **without** blocking legitimate app uploads.

### Test 5 — End-to-end app upload (notifications)

1. Install the app on a test Apple Watch or simulator.
2. Enter a **test User ID** (e.g., `99999` in simulator → path `TEST-99999`).
3. Ensure `databaseURL` points to your **test or production** project (whichever you are validating).
4. Trigger a notification open flow that saves to Firebase (follow your study's test procedure).
5. Open Firebase Console → **Realtime Database → Data**.
6. Navigate to `/devices/TEST-99999/notifications/` (or your test ID path).

**Expected:** A new record with `delivered` and `opened` fields appears within a few seconds.

**If uploads fail:**

| Indicator | Possible cause |
|---------|----------------|
| No record appears | Rules too restrictive; wrong `databaseURL`; User ID format mismatch; no network |
| Record appears under wrong User ID | User ID entry error during setup |

---

### Test 6 — HealthKit upload

1. On a physical Apple Watch with HealthKit permissions granted, allow the app to sync health data.
2. Wait for upload status message in the app (e.g., "Uploaded N records - ST").
3. Check Firebase Console → `/devices/{User ID}/healthkitdata/`.

**Expected:** New record(s) containing arrays or objects with `type`, `quantity`, `startDate`, `endDate`.

HealthKit data may be limited or synthetic in the simulator. Verify HealthKit uploads on a physical Apple Watch where possible.

If HealthKit uploads fail after a rule change, validation rules may be rejecting array payloads (see Section 3.5, item 6). Adjust the rules and repeat the test.

---

### Test 7 — Survey upload

1. Trigger a survey prompt on the test device.
2. Complete and submit a survey response.
3. Check Firebase Console → `/devices/{User ID}/survey/`.

**Expected:** New record with `surveyNo`, `response`, `submitted`, and optionally `delivered`, `opened`.

**Example expected record:**

```json
{
  "surveyNo": "3",
  "response": "7",
  "submitted": "2025-06-15 08:05:00AM",
  "delivered": "2025-06-15 08:00:00AM",
  "opened": "2025-06-15 08:03:22AM"
}
```

---

## 6.4 Indicators of misconfigured rules

Monitor for these indicators during pilot and production phases.

### App-side indicators

| Indicator | Possible misconfiguration |
|------|---------------------------|
| Data stopped appearing in Firebase after rule change | Writes blocked entirely (too strict) or User ID format mismatch |
| App shows upload success but Console is empty | Wrong Firebase project URL in app |
| Only HealthKit fails | Array validation rules too strict |

### Console-side indicators

| Indicator | Possible misconfiguration |
|------|---------------------------|
| Unexpected User IDs in `/devices/` | Sequential IDs guessed; monitor enrollment log |
| Nonsense survey responses (`response: "999"`) | Possible unauthorized write — review rules and User ID practices |
| Missing data for active participants | Overly strict rules; User ID format mismatch |
| Data under `TEST-` prefix in production project | Simulator testing against production |

### External indicators

| Indicator | Action |
|------|--------|
| Firebase emails about insecure rules | Update rules immediately |
| Google Cloud security alert | Contact institutional IT |
| Participant reports app "not saving" | Run Tests 5–7; check rules and User ID format |

### Response when you suspect a breach

1. **Immediately** publish lockdown rules (Section 3, Example A)
2. Document date/time and who discovered the issue
3. Notify your PI and institutional security/privacy office per breach policy
4. Preserve Firebase audit logs before making further changes
5. Do not resume enrollment until root cause is identified and fixed

---

## Post-verification documentation

After all tests pass, record:

| Item | Your study's value |
|------|-------------------|
| Firebase project ID | |
| Database URL | |
| Rule version / date published | |
| User ID assignment procedure | |
| Tests performed (date, person) | |
| Known limitations (if any) | |
| Next scheduled security review | |

Store this record with your IRB documentation and data management plan.

---

## What to read next

Continue to [Section 7 — Additional Resources](./7-additional-resources.md) for links to official Firebase documentation and support channels.
