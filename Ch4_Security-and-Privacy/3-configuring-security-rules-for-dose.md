# 3. Configuring Security Rules for DOSE

This section provides step-by-step instructions for publishing security rules appropriate for the DOSE Watch App—whether you started in Locked mode or Test mode. Examples are included at each stage, together with guidance on common configuration errors.

---

## 3.1 Where to find and edit rules in the Firebase Console

### Step-by-step

1. Go to [Firebase Console](https://console.firebase.google.com/) and sign in with the Google account that owns your study project.
2. Select your **DOSE study project** from the project list.
3. In the left sidebar, click **Build → Realtime Database**.
4. Click the **Rules** tab at the top of the page.
5. You will see a JSON editor containing your current rules.
6. Replace the contents with one of the rule sets from Section 3.3 (choose based on your deployment stage).
7. Click **Publish**.
8. Confirm the publish dialog.

### What the Rules editor looks like

You should see something similar to:

```json
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```

After publishing secure rules, the editor should show your updated configuration — not the open defaults above.

If your project contains multiple Firebase databases (uncommon for DOSE deployments), confirm you are editing rules for the database whose URL matches `FirebaseManager.swift`.

Rules take effect only after you select **Publish**. Closing the editor without publishing leaves the previous rules in place.

### Alternative: Firebase CLI (for technical collaborators)

If your team uses the [Firebase CLI](https://firebase.google.com/docs/cli), rules can be stored in a `database.rules.json` file and deployed with:

```bash
firebase deploy --only database
```

This approach is recommended for teams that want version-controlled, reviewable rule changes. For most research deployments, the Console method above is sufficient.

---

## 3.2 Recommended rule strategy for research deployment

### Core principles

| Principle | What it means for DOSE |
|-----------|------------------------|
| **Deny by default** | Start with everything blocked; open only what the app needs |
| **Block all public reads** | Participants write data; researchers export via Firebase Console or Admin SDK — not via open URLs |
| **Restrict writes to expected paths** | Only `/devices/{User ID}/notifications`, `/healthkitdata`, and `/survey` |
| **Validate data shape** | Reject writes that do not match expected fields (where feasible) |
| **Separate test and production** | Use different Firebase projects or strict path prefixes for simulator data (`TEST-`) |
| **Assign non-guessable User IDs** | See Section 4 — reduces risk of unauthorized writes to known paths |

### Deployment stages

| Stage | Goal | Minimum rule posture |
|-------|------|----------------------|
| **Local / simulator testing** | Developers verify app behavior | Separate test project, or isolated rules with `TEST-` prefix |
| **Pilot study (small N)** | Real participants, limited scope | Reads denied; writes restricted (Example B or C) |
| **Production study** | Full enrollment | Reads denied; writes restricted with validation; non-guessable User IDs |

The DOSE app does not use Firebase Authentication. Production security relies on restrictive security rules and consistent User ID assignment (Section 4), not on login tokens.

---

## 3.3 Example rules aligned with DOSE data paths

The examples below progress from **emergency lockdown** (Example A) to **recommended production rules** (Examples B and C) for this app.

---

### Example A — Deny all reads, deny all writes (safest starting point)

Use this immediately if you have not yet configured rules and need to **stop public exposure** while you plan your full configuration.

```json
{
  "rules": {
    ".read": false,
    ".write": false
  }
}
```

| Effect | Detail |
|--------|--------|
| Stops data leaks | No one can read participant data via the REST API |
| Stops tampering | No one can write or delete data |
| App cannot upload | The DOSE app cannot save data until you apply write rules (Example B or C) |

**When to use:** Emergency lockdown, or a maintenance window while updating rules.

---

### Example B — Recommended production rules (deny reads, restrict writes)

This is the **recommended rule set** for DOSE research deployments. It matches how the app uploads data (REST `POST` without Firebase Auth) while blocking public read access.

```json
{
  "rules": {
    ".read": false,
    ".write": false,

    "devices": {
      "$deviceId": {
        ".read": false,

        "notifications": {
          ".write": "newData.hasChildren(['delivered', 'opened'])",
          "$notificationId": {
            ".validate": "newData.hasChildren(['delivered', 'opened'])"
          }
        },

        "healthkitdata": {
          ".write": true,
          "$recordId": {
            ".validate": "newData.isArray() || newData.hasChildren(['type', 'quantity', 'startDate', 'endDate'])"
          }
        },

        "survey": {
          ".write": "newData.hasChildren(['surveyNo', 'response', 'submitted'])",
          "$surveyId": {
            ".validate": "newData.hasChildren(['surveyNo', 'response', 'submitted'])"
          }
        }
      }
    }
  }
}
```

#### What this example does

| Path | Read | Write | Validation |
|------|------|-------|------------|
| Entire database | Denied | Denied | — |
| `/devices/{User ID}/notifications` | Denied | Allowed (with required fields) | Must include `delivered`, `opened` |
| `/devices/{User ID}/healthkitdata` | Denied | Allowed | Array or object with health fields |
| `/devices/{User ID}/survey` | Denied | Allowed (with required fields) | Must include `surveyNo`, `response`, `submitted` |
| Any other path | Denied | Denied | — |

#### Testing that reads are blocked

Open in a browser (replace with your project URL):

```
https://YOUR-PROJECT-ID-default-rtdb.firebaseio.com/devices.json
```

**Expected result:** Permission denied error (not participant data).

```json
{
  "error": "Permission denied"
}
```

#### Testing that writes still work

After a participant (or simulator) completes a survey, check the Firebase Console **Data** tab (Console access bypasses rules — this is expected). You should see new records under the appropriate User ID.

If your study assigns sequential IDs (`10001`, `10002`, …), valid paths may be easier to infer. Use non-sequential User IDs in production (Section 4.5).

This example does not explicitly block `DELETE` requests on all child paths. Verify delete behavior using the procedures in Section 6.

---

### Example C — Recommended production rules with User ID format validation

Builds on Example B by rejecting writes to paths that do not match your study's User ID format. Use this when your study assigns numeric codes (as the app UI suggests).

```json
{
  "rules": {
    ".read": false,
    ".write": false,

    "devices": {
      "$deviceId": {
        ".read": false,

        ".write": "$deviceId.matches(/^(TEST-)?[0-9]+$/)",

        "notifications": {
          "$notificationId": {
            ".validate": "newData.hasChildren(['delivered', 'opened'])"
          }
        },

        "healthkitdata": {
          "$recordId": {
            ".write": true
          }
        },

        "survey": {
          "$surveyId": {
            ".validate": "newData.hasChildren(['surveyNo', 'response', 'submitted'])"
          }
        }
      }
    }
  }
}
```

| Pattern | Matches | Does not match |
|---------|---------|----------------|
| `^(TEST-)?[0-9]+$` | `10042`, `TEST-10042` | `admin`, `10042/extra`, empty string |

Adjust the regular expression if your study uses alphanumeric codes (e.g., `P-ABC-0042`). Example:

```json
".write": "$deviceId.matches(/^(TEST-)?P-[A-Z]{3}-[0-9]{4}$/)"
```

Format validation limits accidental or casual misuse but does not replace unpredictable User ID assignment (Section 4.5).

---

### Example D — Separate test and production data in one database (use with caution)

If you must use a single Firebase project for both simulator testing and production (not ideal), allow writes only under `TEST-` prefixed paths for simulator data, and use your production User ID format for real participants:

```json
{
  "rules": {
    ".read": false,
    ".write": false,

    "devices": {
      "$deviceId": {
        ".read": false,

        ".write": "$deviceId.matches(/^TEST-[0-9]+$/) || $deviceId.matches(/^[0-9]{8}$/)",

        "notifications": {
          "$id": {
            ".validate": "newData.hasChildren(['delivered', 'opened'])"
          }
        },
        "healthkitdata": {
          "$id": {
            ".write": true
          }
        },
        "survey": {
          "$id": {
            ".validate": "newData.hasChildren(['surveyNo', 'response', 'submitted'])"
          }
        }
      }
    }
  }
}
```

| User ID | Write access in this example |
|---------|------------------------------|
| `TEST-10042` | Allowed (simulator) |
| `84729103` | Allowed (8-digit production format) |
| `admin` | Denied |

Combining test and production data in one database increases the risk of data mixing and complicates ethics review. Separate Firebase projects for test and production are strongly recommended.

---

## 3.4 Rules for development/testing vs. production

### Recommended project separation

| Environment | Firebase project | Example project ID |
|-------------|------------------|--------------------|
| Development / simulator | `dose-study-dev` | `dose-study-dev-default-rtdb.firebaseio.com` |
| Production | `dose-study-prod` | `dose-study-prod-default-rtdb.firebaseio.com` |

Update `FirebaseManager.databaseURL` in your build to point to the correct project for each release.

### Comparison table

| Setting | Development | Production |
|---------|-------------|------------|
| Default open rules | Never | Never |
| Public read access | Denied | Denied |
| Write rules (Examples B or C) | Yes (dev project with test data) | Required |
| Non-guessable User IDs | Optional (use `TEST-` codes) | Required |
| Real participant data | No | Yes |
| Firebase Console access | Developers | PI + designated staff only |

### Example workflow for a research team

1. **Week 1–2:** Create `dose-study-dev`. Apply Example B rules. Test app in simulator with `TEST-` User IDs.
2. **Before pilot:** Create `dose-study-prod`. Apply Example C rules with your User ID format. Assign non-sequential codes to participants.
3. **Before full enrollment:** Confirm rules are published, reads are blocked, app uploads succeed, Section 6 checklist complete.

Record which rule version was active during each study phase. Ethics boards and auditors may request this information.

---

## 3.5 Common configuration errors

### 1. Leaving test rules in production

**Indicator:** The full dataset is accessible via a browser URL.

**Incorrect configuration:**

```json
{ "rules": { ".read": true, ".write": true } }
```

**Resolution:** Publish Example A immediately, then deploy appropriate rules before resuming enrollment.

---

### 2. Blocking writes entirely

**Indicator:** The app runs but Firebase Console shows no new records; HTTP 401 or 403 responses appear in logs.

**Cause:** Example A (deny all) remains active, or write rules are too restrictive (for example, User ID format in Example C does not match assigned codes).

**Resolution:** Align rules with your User ID format. Verify one write path manually (Section 6).

---

### 3. Allowing reads "only for researchers" via open rules

**Incorrect configuration:**

```json
{
  "rules": {
    "devices": {
      ".read": true,
      ".write": false
    }
  }
}
```

**Why this fails:** `.read: true` on `/devices` exposes all participant data to any requester, not only to the research team.

**Correct approach:** Keep reads denied in rules. Export data using:
- Firebase Admin SDK with a service account (server-side only)
- Scheduled Cloud Functions
- Firebase Console (limited to authorized staff)

Never embed service account credentials in the Apple Watch app.

---

### 4. Overly permissive wildcards

**Incorrect configuration:**

```json
{
  "rules": {
    "devices": {
      "$deviceId": {
        ".read": true,
        ".write": true
      }
    }
  }
}
```

This configuration exposes all participant data and permits modification for every User ID.

---

### 5. Forgetting to publish

**Indicator:** Rules were edited but open access persists.

**Resolution:** Open the Rules tab, confirm the intended configuration is present, select **Publish**, and re-run verification tests.

---

### 6. Validating the wrong data shape for HealthKit uploads

The app sends `healthkitdata` as a **JSON array** in a single POST:

```json
[
  {
    "type": "Steps",
    "quantity": "847 count",
    "startDate": "2025-06-15 07:00:00AM",
    "endDate": "2025-06-15 08:00:00AM"
  }
]
```

Rules that only allow objects with `.hasChildren(['type', ...])` at the top level may **reject** array payloads. Test HealthKit uploads explicitly after publishing rules.

After publishing, trigger a HealthKit sync in the app and confirm a new record appears under `/devices/{User ID}/healthkitdata/` in the Console.

---

### 7. Using deprecated database secrets in the app

Older Firebase documentation describes a **database secret** appended to URLs for admin access. Database secrets are **deprecated** and must **never** be embedded in the Apple Watch app (anyone can extract them from the app binary). This is unrelated to Firebase Authentication — it is a legacy admin credential.

**Incorrect pattern:**

```
https://YOUR-PROJECT.firebaseio.com/devices.json?auth=DATABASE_SECRET
```

Researchers should access data through the Firebase Console or server-side Admin SDK only.

---

## Rule selection quick reference

| Example | Reads blocked | Writes restricted | Matches DOSE app? | Recommended use |
|---------|---------------|-------------------|-------------------|-----------------|
| A — Lockdown | Yes | Yes (all denied) | N/A | Emergency only |
| B — Production baseline | Yes | Yes (path + validation) | Yes | Production |
| C — + ID format | Yes | Yes (path + format + validation) | Yes | Production (preferred) |
| D — Test/prod mix | Yes | Mixed | Yes | Only if separate projects are not feasible |

---

## What to read next

Continue to [Section 4 — Device Identification and Write Access](./4-device-identification-and-write-access.md) to understand how User IDs work and how they relate to Firebase paths.
