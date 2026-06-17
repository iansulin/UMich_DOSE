# 7. Additional Resources

This section collects official Firebase documentation, security references, and practical guidance for further reading. Links were current as of this documentation's release; if a link moves, search the Firebase documentation site for the topic title.

---

## 7.1 Official Firebase security documentation

### Realtime Database security rules

| Resource | URL | When to use it |
|----------|-----|----------------|
| **Security Rules overview** | [firebase.google.com/docs/database/security](https://firebase.google.com/docs/database/security) | Start here for concepts and syntax |
| **Understand Rules** | [firebase.google.com/docs/database/security/core-syntax](https://firebase.google.com/docs/database/security/core-syntax) | Learn `.read`, `.write`, `$variables`, validation |
| **Rules conditions** | [firebase.google.com/docs/database/security/rules-conditions](https://firebase.google.com/docs/database/security/rules-conditions) | Conditions using `data`, `newData`, and validation (DOSE rules do not use `auth`) |
| **Validate data** | [firebase.google.com/docs/database/security/rules-structure](https://firebase.google.com/docs/database/security/rules-structure) | `.validate` rules for field requirements |
| **Rules API reference** | [firebase.google.com/docs/reference/security/database](https://firebase.google.com/docs/reference/security/database) | Complete syntax reference |

### REST API

| Resource | URL | When to use it |
|----------|-----|----------------|
| **REST API guide** | [firebase.google.com/docs/database/rest/start](https://firebase.google.com/docs/database/rest/start) | How the DOSE app communicates with Firebase |
| **Save data via REST** | [firebase.google.com/docs/database/rest/save-data](https://firebase.google.com/docs/database/rest/save-data) | POST, PUT, PATCH, DELETE behavior |

The DOSE app does not use Firebase Authentication. The [Authenticate REST requests](https://firebase.google.com/docs/database/rest/auth) guide applies only if you extend the app in the future.

### Admin SDK and server-side access (for data export)

Server-side export uses the Admin SDK — this is for **researchers exporting data**, not for participant devices.

| Resource | URL | When to use it |
|----------|-----|----------------|
| **Admin SDK setup** | [firebase.google.com/docs/admin/setup](https://firebase.google.com/docs/admin/setup) | Server-side data export and proxy writes |
| **Admin Database API** | [firebase.google.com/docs/reference/admin/node/admin.database.Database](https://firebase.google.com/docs/reference/admin/node/admin.database.Database) | Programmatic read/write for exports |

The DOSE Watch App does not use Firebase App Check or Firebase Authentication. The references below are general Firebase documentation.

---

## 7.2 Firebase security checklist and best practices

### Official checklists and guides

| Resource | URL | Summary |
|----------|-----|---------|
| **Firebase security checklist (blog)** | [firebase.blog/posts/2020/10/security-rules-checklist](https://firebase.blog/posts/2020/10/security-rules-checklist) | Step-by-step review of common rule mistakes |
| **Security Rules testing** | [firebase.google.com/docs/rules/unit-tests](https://firebase.google.com/docs/rules/unit-tests) | Automated rule tests (for technical collaborators) |
| **Firebase Emulator Suite** | [firebase.google.com/docs/emulator-suite](https://firebase.google.com/docs/emulator-suite) | Local testing without affecting production data |

### Google Cloud security (underlying platform)

| Resource | URL | Summary |
|----------|-----|---------|
| **Cloud Audit Logs** | [cloud.google.com/logging/docs/audit](https://cloud.google.com/logging/docs/audit) | Track Console and API access |
| **IAM roles for Firebase** | [firebase.google.com/docs/projects/iam/roles](https://firebase.google.com/docs/projects/iam/roles) | Understand Viewer / Editor / Owner |
| **Google Cloud compliance** | [cloud.google.com/security/compliance](https://cloud.google.com/security/compliance) | Compliance certifications (SOC, ISO, etc.) |

### Practices mapped to DOSE deployment

| Best practice | DOSE-specific application |
|---------------|---------------------------|
| Deny by default | Section 3, Examples A, B, and C |
| Never expose secrets in client apps | Do not embed service accounts or database secrets in the watch app |
| Separate test and production | Section 3.4 — separate Firebase projects |
| Validate data shape | Section 3.3 — `.validate` on survey and notification fields |
| Assign non-guessable User IDs | Section 4.5 |
| Test rules before enrollment | Section 6 — browser and curl tests |
| Review access quarterly | Section 5.5 — Console access review log |

---

## 7.3 Where to get help

### Within your institution

| Contact | When to reach out |
|---------|-------------------|
| **Principal Investigator** | Policy decisions, ethics alignment, enrollment go/no-go |
| **IRB / ethics office** | Consent language, protocol amendments, breach reporting |
| **IT security / privacy office** | Cloud service approval, service account management, audit logs |
| **Research computing / data core** | Server hosting, export pipelines, secure storage |

### Firebase and Google support

| Channel | URL / method | Notes |
|---------|--------------|-------|
| **Firebase Documentation** | [firebase.google.com/docs](https://firebase.google.com/docs) | Primary self-service resource |
| **Firebase Support** | [firebase.google.com/support](https://firebase.google.com/support) | Plans vary; Spark (free) tier has community support |
| **Stack Overflow** | Tag: `[firebase]` + `[firebase-realtime-database]` | Community Q&A for technical questions |
| **Firebase GitHub (samples)** | [github.com/firebase/quickstart-js](https://github.com/firebase/quickstart-js) | Reference implementations |
| **Google Cloud Support** | Via GCP Console | Available on paid Blaze plans |

Firebase's free Spark plan is sufficient for many research pilots. Higher usage limits may require the Blaze (pay-as-you-go) plan; consult current Firebase pricing when planning your budget.

### DOSE Watch App project documentation

| Document | Location |
|----------|----------|
| Documentation index | [../README.md](../README.md) |
| Section 1 — Introduction | [1-introduction.md](./1-introduction.md) |
| Section 2 — Default rules | [2-understanding-firebase-default-rules.md](./2-understanding-firebase-default-rules.md) |
| Section 3 — Configuring rules | [3-configuring-security-rules-for-dose.md](./3-configuring-security-rules-for-dose.md) |
| Section 4 — Device Identification | [4-device-identification-and-write-access.md](./4-device-identification-and-write-access.md) |
| Section 5 — Privacy | [5-privacy-considerations-for-research.md](./5-privacy-considerations-for-research.md) |
| Section 6 — Verification | [6-verifying-your-configuration.md](./6-verifying-your-configuration.md) |

---

## Glossary (quick reference)

| Term | Plain-language definition |
|------|---------------------------|
| **Firebase Realtime Database** | Google's cloud database that stores data as JSON and syncs in real time |
| **Security rules** | Server-side policies controlling read/write access to database paths |
| **REST API** | Standard web HTTP requests used by the DOSE app to upload data |
| **User ID / deviceID** | Study participant code entered on the watch; used as the Firebase path key |
| **UUID (in this app)** | Used only for local notification request IDs — not sent to Firebase |
| **Admin SDK** | Server-only tools with elevated access for exports and management |
| **IRB** | Institutional Review Board — ethics committee overseeing human subjects research |
| **Permission denied** | Expected error when unauthorized access is correctly blocked |

---

## Document maintenance

Review this documentation when:

- Firebase changes default project settings or Console layout
- The DOSE app is updated to use the Firebase SDK or a different identification model
- Your study changes data types, cloud region, or retention policy
- A security incident or audit identifies documentation gaps

When updating, revise the relevant section file(s) and note the change date in your study's internal change log.

---

*End of Security and Privacy documentation.*
