# 4. Device Identification and Write Access

The DOSE Watch App does **not** use Firebase Authentication. It identifies each watch with a **User ID** that the participant enters once at study setup. Data is uploaded to Firebase over HTTPS using standard REST requests, with no login tokens or Firebase Auth credentials.

This section explains how that identification works, what role UUIDs play in the app (and what they do not), and how **security rules** and **User ID practices** protect participant data.

---

## 4.1 How the app identifies each watch

### User ID (device identifier for Firebase)

When a participant first uses the app, they are prompted to enter a **User ID** on their Apple Watch (`ParticipantView.swift`). This value:

1. Is entered manually by the participant (or study staff during setup)
2. Is saved to the device **Keychain** via `StorageManager.getUniqueIdentifierForWatch(userSuppliedDeviceID:)`
3. Cannot be changed later without clearing app storage or reinstalling
4. Is used as the `deviceID` in all Firebase upload paths

**Code flow:**

```swift
// Participant enters ID once
StorageManager.shared.getUniqueIdentifierForWatch(userSuppliedDeviceID: userID)

// All Firebase uploads retrieve the stored ID
let deviceID = StorageManager.shared.getUniqueIdentifierForWatch()
let urlString = "\(databaseURL)/devices/\(deviceID)/survey.json"
```

**Example Firebase paths for User ID `84729103`:**

```
/devices/84729103/notifications.json
/devices/84729103/healthkitdata.json
/devices/84729103/survey.json
```

In the app UI and code, this value is called **User ID** or **deviceID**. It is a study-assigned identifier, not an automatically generated device serial number.

### Simulator prefix

When running in the Xcode simulator, the app prepends `TEST-` to the stored User ID:

| User ID entered | Value sent to Firebase |
|-----------------|------------------------|
| `84729103` (physical watch) | `84729103` |
| `84729103` (simulator) | `TEST-84729103` |

This is determined by checking for the `SIMULATOR_UDID` environment variable in `StorageManager.swift`.

---

## 4.2 UUID in this app — what it is and what it is not

The app uses `UUID()` in one place only: **local notification scheduling**.

```swift
// NotificationManager.swift — identifier for a scheduled notification request
let request = UNNotificationRequest(identifier: UUID().uuidString, content: content, trigger: trigger)
```

| UUID usage | Purpose | Sent to Firebase? |
|------------|---------|-------------------|
| `UUID().uuidString` for notification requests | Unique ID for each scheduled notification on the watch | **No** |
| User ID in Keychain | Participant study code for Firebase data paths | **Yes** — as the `/devices/{User ID}/` path key |

Firebase data is not keyed by a device-generated UUID. It is keyed by the **User ID** the participant enters. Study materials that refer to a "device UUID" should be aligned with this design: the app uses a study-assigned User ID.

The app also rejects obviously invalid User IDs such as `test` or `demo` (`ParticipantView.swift`), but does not auto-generate IDs.

---

## 4.3 No Firebase Authentication — by design

### What the app does not include

| Feature | Present in DOSE app? |
|---------|---------------------|
| Firebase iOS SDK | No |
| Firebase Authentication | No |
| Auth token on REST requests (`?auth=`) | No |
| Login screen or password | No |

All uploads in `FirebaseManager.swift` use plain `POST` requests with JSON bodies and no authorization header:

```swift
var request = URLRequest(url: url)
request.httpMethod = "POST"
request.setValue("application/json", forHTTPHeaderField: "Content-Type")
// No Firebase Auth token is attached
```

This is intentional for the open-source release: researchers configure a Firebase database URL, assign User IDs to participants, and secure the database with **security rules** (Section 3).

Firebase Authentication, custom tokens, and login flows are not required to deploy this app. Security is established through Firebase security rules and consistent User ID management.

---

## 4.4 How security works without Firebase Authentication

Without Firebase Auth, protection relies on a combination of:

| Layer | What it does |
|-------|--------------|
| **Security rules — deny reads** | Nobody can download participant data via public URLs |
| **Security rules — restrict writes** | Writes allowed only to `/devices/{User ID}/notifications`, `healthkitdata`, and `survey` |
| **Security rules — validate data** | Incoming records must match expected fields (survey, notification, health data) |
| **User ID assignment** | Study team assigns non-guessable codes so paths are hard to discover |
| **Firebase Console access control** | Only authorized researchers can view data in the Console (Section 5.5) |
| **HTTPS** | Data encrypted in transit between watch and Firebase |

### What security rules cannot verify on their own

| Limitation | Mitigation |
|------------|------------|
| Anyone who knows a valid User ID and database URL could theoretically POST data to that path | Assign **non-sequential, non-guessable** User IDs; never publish the database URL; monitor data for anomalies |
| Rules cannot verify that a request comes from a specific physical watch | Acceptable for many EMA research designs where User ID is the participant key |
| User ID is visible in Firebase paths | Keep reads blocked; maintain a separate enrollment log mapping User ID → participant record |

Blocking public reads is the most critical rule change: test mode or other open rules expose the full dataset. The recommended rules in Section 3 (Examples B and C) deny reads and restrict writes to the paths this app uses.

---

## 4.5 User ID assignment for research studies

Because the User ID acts as the Firebase path key, how you assign IDs affects security and data organization.

### Recommended practices

| Practice | Example | Why |
|----------|---------|-----|
| Use study-specific codes, not names | `84729103` not `jsmith` | Avoids direct identification in Firebase |
| Use non-sequential codes | `84729103`, `52910477` not `0001`, `0002` | Harder to guess valid paths |
| Pre-generate a list before enrollment | Spreadsheet of 200 unique 8-digit codes | Consistent format; no duplicates |
| Store identity mapping separately | Enrollment log: `84729103 → participant record` | Firebase holds codes only |
| Document assignment procedure | "IDs handed to participant at Visit 1" | Audit trail for ethics review |

### Example enrollment workflow

1. Generate 100 unique codes: `84729103`, `52910477`, `10394827`, …
2. At enrollment, assign one code per participant and enter it on their watch
3. Record the assignment in your secure enrollment log (not in Firebase)
4. Participant completes surveys; data appears under `/devices/84729103/...`
5. Researcher exports data from Firebase Console or Admin SDK using Console credentials

### What the User ID is not

| User ID is… | User ID is not… |
|-------------|-----------------|
| A study participant code | A Firebase login credential |
| A password | Encrypted in the database path |
| Set once and stored on-device | Auto-generated by the app |
| Required before the app can upload data | A UUID from the watch hardware |

---

## 4.6 Configuring the app for your study

### Steps for researchers

1. **Create a Firebase project** and Realtime Database (see Section 3)
2. **Publish security rules** — use Section 3, Example B or C (deny reads, restrict writes)
3. **Set the database URL** in `FirebaseManager.swift`:

```swift
static var databaseURL: String = "https://YOUR-PROJECT-ID-default-rtdb.firebaseio.com"
```

4. **Build and distribute** the app to study watches
5. **At enrollment**, enter each participant's assigned User ID on their watch
6. **Verify** uploads appear under the correct path (Section 6)
7. **Export data** for analysis via Firebase Console or server-side tools — not via open read rules

If no User ID has been entered, `getUniqueIdentifierForWatch()` returns an empty string and uploads will fail or target an invalid path. The app requires a User ID before scheduling notifications.

---

## 4.7 When to involve IT support

Most DOSE deployments can be completed using Firebase Console and the steps in this guide. Involve your institution's IT or security team if:

| Situation | Reason |
|-----------|--------|
| Your IRB or institution requires a formal cloud security review | Firebase may need vendor approval |
| You need automated data export pipelines | Admin SDK setup on a server |
| You want a separate development Firebase project | IT can manage project ownership and billing |
| Health data regulations apply (HIPAA, etc.) | Additional controls beyond this guide may be required |

You do **not** need IT solely to implement Firebase Authentication — this app does not use it.

---

## Summary

| Question | Answer for DOSE Watch App |
|----------|---------------------------|
| Does the app use Firebase Authentication? | **No** |
| What identifies each participant in Firebase? | **User ID** entered at setup, stored in Keychain |
| Does the app use UUID for Firebase paths? | **No** — UUID is only for local notification IDs |
| How is data protected? | **Security rules** (deny reads, restrict writes) + User ID practices + Console access control |
| What must researchers configure? | Firebase rules, database URL, User ID assignment |

---

## What to read next

Continue to [Section 5 — Privacy Considerations for Research](./5-privacy-considerations-for-research.md) for IRB alignment, retention, and team access controls.
