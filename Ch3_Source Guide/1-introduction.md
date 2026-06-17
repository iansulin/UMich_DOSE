# 1. Introduction

## 1.1 Purpose of this documentation

The DOSE Watch App stores research data in **Firebase Realtime Database**, a cloud service provided by Google. When you first enable Realtime Database in a Firebase project, the Firebase Console asks you to choose **Locked mode** or **Test mode**. Test mode allows open read and write access (typically for 30 days). Locked mode denies all access until you publish custom rules. For research with participant data, neither default is sufficient on its own—you must publish restrictive rules before enrollment (Section 3).

This documentation explains:

- What data the app sends to Firebase
- Why open default rules are a serious privacy and security risk
- How to configure Firebase security rules before enrolling participants
- How device identification works (User ID) and why Firebase Authentication is not used
- How to verify that your configuration protects participant data

This guide addresses Firebase security and privacy. It supplements — but does not replace — your institution's IRB or ethics review, data protection policies, and legal obligations.

---

## 1.2 Who should read this

| Role | What you need from this guide |
|------|-------------------------------|
| **Principal investigator / study lead** | Understand risks, approve a secure deployment plan, assign Firebase Console access carefully |
| **Research coordinator** | Follow setup steps, run verification tests, maintain the pre-deployment checklist |
| **Study programmer / IT support** | Implement security rules and data export pipelines |
| **Ethics / compliance reviewer** | Confirm that Firebase configuration aligns with consent and data-handling plans |

You do not need to be a software developer to understand Sections 1, 2, 5, and 6. Sections 3 and 4 include technical steps; if you are uncomfortable with those, ask your institution's IT team or a technical collaborator to complete them while you verify outcomes using Section 6.

---

## 1.3 What data the DOSE Watch App sends to Firebase

The app uploads data using your Firebase Realtime Database URL, configured in `FirebaseManager.swift`:

```swift
static var databaseURL: String = "Replace with your Firebase Realtime Database base URL (no trailing slash)"
```

All data is stored under a path keyed by a **User ID** (called `deviceID` in the app code). Each participant enters this ID once on their Apple Watch when the study begins. It cannot be changed later without reinstalling or clearing app storage.

The app does not auto-generate a UUID for Firebase. The participant enters a study-assigned User ID, which is stored in the device Keychain. `UUID` appears in the app only for local notification scheduling and is not used in Firebase paths. See Section 4 for further detail.

### Database structure (simplified)

```
your-database/
└── devices/
    └── {User ID}/                    ← e.g., "10042" or "TEST-10042" in simulator
        ├── notifications/            ← when notifications were delivered and opened
        ├── healthkitdata/            ← step count and heart rate samples
        └── survey/                   ← survey responses
```

### Data types and example records

**Notifications** — written when a participant opens a study notification:

| Field | Example | Description |
|-------|---------|-------------|
| `delivered` | `"2025-06-15 08:00:00AM"` | When the notification was scheduled/delivered |
| `opened` | `"2025-06-15 08:03:22AM"` | When the participant opened it |

**HealthKit data** — step count and heart rate uploaded periodically:

| Field | Example | Description |
|-------|---------|-------------|
| `type` | `"Steps"` or `"HeartRate"` | Type of health measurement |
| `quantity` | `"1247 count"` | Measured value |
| `startDate` | `"2025-06-15 07:00:00AM"` | Sample start time |
| `endDate` | `"2025-06-15 08:00:00AM"` | Sample end time |

**Survey responses** — written when a participant completes a survey prompt:

| Field | Example | Description |
|-------|---------|-------------|
| `surveyNo` | `"3"` | Survey identifier |
| `response` | `"7"` | Participant's answer |
| `delivered` | `"2025-06-15 08:00:00AM"` | Related notification delivery time |
| `opened` | `"2025-06-15 08:03:22AM"` | Related notification open time |
| `submitted` | `"2025-06-15 08:05:00AM"` | Submission timestamp |

Even when User IDs are numeric study codes rather than names, the combination of health metrics, timestamps, and behavioral survey responses may still identify individuals or reveal private health information. All Firebase data should be treated as research-sensitive.

When the app runs in the Xcode simulator, the User ID is prefixed with `TEST-` (e.g., `TEST-10042`). Use separate Firebase rules or a dedicated test Firebase project to prevent test data from mixing with production participant data.

---

## 1.4 Why securing Firebase is essential before enrolling participants

### The default problem

A newly created Firebase Realtime Database typically uses rules equivalent to:

```json
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```

This means **any person or automated script** that knows (or guesses) your database URL can:

- **Read** all participant notifications, health data, and survey responses
- **Write** fake or malicious data into your database
- **Delete** existing records

Your database URL follows a predictable pattern:

```
https://YOUR-PROJECT-ID-default-rtdb.firebaseio.com
```

The project ID appears in your Firebase Console and in the app's `databaseURL` setting. It is not a secret.

### Real-world consequences for research studies

| Risk | What could happen |
|------|-------------------|
| **Privacy breach** | Unauthorized parties access participant health and survey data — a reportable incident under many ethics and privacy regulations |
| **Re-identification** | Combined timestamps and health patterns may allow inference about individual participants |
| **Study invalidation** | Compromised data may require excluding participants or repeating data collection |

### Minimum actions before first enrollment

1. **Publish restrictive security rules** (Section 3), replacing test mode or locked-mode defaults as needed
2. **Block public read access** to participant data paths
3. **Assign non-guessable User IDs** to participants (Section 4)
4. **Run verification tests** — confirm reads are blocked and app uploads still work (Section 6)
5. **Document your configuration** for ethics and audit records (Section 5)

Setting the Firebase URL in the app does not, by itself, secure the database. Access control is configured in the Firebase Console (or via the Firebase CLI), not in the Apple Watch app code.

---

## What to read next

Continue to [Section 2 — Understanding Firebase Default Rules](./2-understanding-firebase-default-rules.md) to learn how security rules work and why the defaults are unsafe for research data.
