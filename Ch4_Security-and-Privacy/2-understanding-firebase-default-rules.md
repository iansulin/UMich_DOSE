# 2. Understanding Firebase Default Rules

## 2.1 What Firebase Realtime Database security rules are

Firebase Realtime Database **security rules** are server-side policies that control who can read and write data, and what shape that data must have. Rules are enforced by Google's servers — not by the Apple Watch app.

Rules are evaluated on Google's servers before any read or write is permitted. The same policies apply whether the request comes from the DOSE app, a web browser, or any other client.

### Basic rule structure

Rules are written in JSON and use a tree structure that mirrors your database:

```json
{
  "rules": {
    "devices": {
      "$deviceId": {
        "survey": {
          ".read": false,
          ".write": "/* condition here */"
        }
      }
    }
  }
}
```

Common rule keywords:

| Keyword | Meaning |
|---------|---------|
| `.read` | Controls whether data at this path can be read |
| `.write` | Controls whether data at this path can be written (includes create, update, delete) |
| `true` | Allow the operation |
| `false` | Deny the operation |
| `$variable` | Wildcard — matches any single path segment (e.g., any User ID) |
| `auth` | Firebase login identity in rules (not used by the DOSE app — see Section 4) |
| `data` | Existing data at the path being written |
| `newData` | Data being written |

Rules are edited in the [Firebase Console](https://console.firebase.google.com/) under **Build → Realtime Database → Rules**. Changes may take a few seconds to propagate. Select **Publish** after editing; unpublished changes do not take effect.

---

## 2.2 Default rules when Realtime Database is created

When you enable Realtime Database in the Firebase Console, you are prompted to choose **Locked mode** or **Test mode**. These are the starting rule sets—not something applied automatically when you create a Firebase project alone (the database must be created separately).

### Locked mode (deny all access)

Firebase describes this as the secure starting point. Default rules:

```json
{
  "rules": {
    ".read": false,
    ".write": false
  }
}
```

| Setting | Effect |
|---------|--------|
| `".read": false` | No one can read data via the REST API or client SDKs |
| `".write": false` | No one can write or delete data |

The DOSE Watch App cannot upload data until you publish rules that allow writes to the paths it uses (Section 3, Examples B or C).

### Test mode (open access for development)

If you select **Test mode** during setup, Firebase configures rules that allow anyone to read and write the entire database for a limited period (commonly 30 days). Rules resemble:

```json
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```

Or, in some Firebase Console versions, test mode includes an expiration timestamp so access is automatically denied after the trial period.

| Setting | Effect |
|---------|--------|
| `".read": true` | Anyone can download all data from your database |
| `".write": true` | Anyone can add, change, or delete data |

Firebase may display a warning that test mode rules expire after 30 days. Do not rely on expiration alone: publish proper rules before enrolling participants, regardless of which mode you selected initially.

### Summary: which default applies?

| What you chose at database creation | Initial read access | Initial write access | Safe for participant data? |
|---------------------------------------|--------------------|--------------------|----------------------------|
| **Locked mode** | Denied | Denied | No—not until custom rules allow app uploads |
| **Test mode** | Open | Open | **No—never use with real participants** |
| **Custom rules published (Section 3)** | Denied (recommended) | Restricted to app paths | Yes—when verified (Section 6) |

### Example: what an attacker can do with open rules

If your database uses **test mode** or other open rules (`.read: true`, `.write: true`) and your database URL is `https://my-dose-study-default-rtdb.firebaseio.com`, anyone can run:

```bash
# Example only — do not run against a production database
curl "https://my-dose-study-default-rtdb.firebaseio.com/devices.json"
```

Or open this URL in a web browser:

```
https://my-dose-study-default-rtdb.firebaseio.com/devices.json
```

If data exists, the browser displays the full JSON contents — every User ID, health record, and survey response.

The Firebase Console **Data** tab displays database contents directly. Limit Console access to authorized study personnel (see Section 5.5).

---

## 2.3 What can go wrong if rules are left open

### Unauthorized read access

| Scenario | Impact |
|----------|--------|
| Curious third party finds your URL | Full dataset exposure |
| Search engine or monitoring tool indexes an exposed endpoint | Wider public disclosure |
| Competitor or bad actor scrapes data | Privacy violation, potential legal liability |

**Example:** A research assistant shares a screenshot of Firebase Console in a presentation. The project ID is visible. An attendee uses it to download participant health data that same day.

### Unauthorized write access

| Scenario | Impact |
|----------|--------|
| Random internet user posts junk data | Corrupted or unusable research dataset |
| Malicious actor deletes `/devices` | Complete data loss with no app-side backup |
| Attacker overwrites survey responses | Invalid study conclusions |

**Example:** An open database receives thousands of fake `{ "response": "999" }` survey entries overnight. Your analysis pipeline cannot distinguish real from fake responses without manual cleanup — if cleanup is even possible.

### Unauthorized delete access

With `".write": true`, deletes are allowed. A single request can remove an entire participant's history:

```bash
# Illustrative — deletes one participant's data tree
curl -X DELETE "https://my-dose-study-default-rtdb.firebaseio.com/devices/10042.json"
```

Firebase Realtime Database is not a backup system. Deleted data may be unrecoverable without separate exports or backups. Plan regular secure exports for research archives (Section 5.3).

---

## 2.4 How the DOSE app connects to Firebase

Understanding how the app communicates with Firebase helps you write appropriate rules.

### REST API, not the Firebase SDK

The DOSE Watch App sends data using **standard HTTP requests** (REST). There is no Firebase Authentication token attached to these requests in the current open-source release.

Relevant code pattern from `FirebaseManager.swift`:

```swift
let urlString = "\(databaseURL)/devices/\(deviceID)/survey.json"
var request = URLRequest(url: url)
request.httpMethod = "POST"
request.setValue("application/json", forHTTPHeaderField: "Content-Type")
// No Authorization header is set
```

### Write operations used by the app

| Data type | HTTP method | Path pattern |
|-----------|-------------|--------------|
| Notifications | `POST` | `/devices/{User ID}/notifications.json` |
| HealthKit data | `POST` | `/devices/{User ID}/healthkitdata.json` |
| Survey responses | `POST` | `/devices/{User ID}/survey.json` |

Using `POST` creates a **new child record** with an auto-generated key under each path (Firebase push ID).

**Example resulting database structure:**

```json
{
  "devices": {
    "10042": {
      "notifications": {
        "-OXabc123def": {
          "delivered": "2025-06-15 08:00:00AM",
          "opened": "2025-06-15 08:03:22AM"
        }
      },
      "healthkitdata": {
        "-OXxyz789ghi": [
          {
            "type": "Steps",
            "quantity": "847 count",
            "startDate": "2025-06-15 07:00:00AM",
            "endDate": "2025-06-15 08:00:00AM"
          }
        ]
      },
      "survey": {
        "-OXsur456jkl": {
          "surveyNo": "3",
          "response": "7",
          "submitted": "2025-06-15 08:05:00AM",
          "delivered": "2025-06-15 08:00:00AM",
          "opened": "2025-06-15 08:03:22AM"
        }
      }
    }
  }
}
```

### Implications for security rules

| Fact | Security implication |
|------|---------------------|
| No Firebase Auth in app | Normal for this app — security uses rules and User IDs, not login tokens (Section 4) |
| User ID is entered by participant | Assign non-guessable codes; restrict writes to expected paths in rules (Section 3) |
| Database URL is in app source | The URL is embedded in distributed app builds — treat it as discoverable |
| App only writes; it does not read from Firebase | You can safely deny all reads for participant paths; researchers export via Firebase Console or Admin SDK |

Security rules control access to data stored in Firebase; they do not encrypt data. Consult your institutional requirements for encryption at rest and in transit. Data in transit is protected by HTTPS for Firebase REST requests.

The database URL alone does not provide security. Protection depends on deny-by-default rules and restricted write paths, not on concealing the URL.

---

## Summary

| Default state | Safe for production research? |
|---------------|-------------------------------|
| Locked mode (`.read: false`, `.write: false`) | No for enrollment—app cannot upload until custom rules are published |
| Test mode (`.read: true`, `.write: true`) | **No—never use with real participants** |
| Reads blocked, writes open to all paths | **No—still allows data injection and deletion** |
| Reads blocked, writes restricted to app paths with validation | **Yes—target configuration (Section 3, Examples B and C)** |

---

## What to read next

Continue to [Section 3 — Configuring Security Rules for DOSE](./3-configuring-security-rules-for-dose.md) for step-by-step instructions and copy-paste rule examples tailored to this app.
