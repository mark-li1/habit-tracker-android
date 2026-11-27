# HabitProof

## Glossary

The terms below are used throughout this document and are capitalized when they carry this
meaning.

| Term | Meaning |
|---|---|
| **Habit** | Something to do on a schedule. Each Habit has one Proof. |
| **Occurrence** | One scheduled instance of a Habit, with its own Window, for example Monday 07:00–09:00. |
| **Window** | The time range, set per Habit by a start and an end, in which Proof must pass. |
| **Reminders** | The only two notifications of an Occurrence: one at the Window start, and one N minutes before the Window end. Nothing rings and nothing repeats. |
| **Skip** | An explicit "skip this time" action during the Window. It ends the Occurrence as SKIPPED. |
| **Proof available** | A Habit's Proof can actually run: the camera works and is permitted, and the Habit has a Label or at least one usable Reference. |
| **Proof** | What the camera must see, within the Window, to complete an Occurrence. There are two modes: Reference and Label. |
| **Reference** | Proof mode "this exact object". A physical object scanned in advance, stored as a name and its Embeddings. |
| **Library** | The user's set of References. They are reusable across Habits and synced through the backend. |
| **Label** | Proof mode "any object of this kind". Text the user types, mapped through Synonyms to ML Kit Image Labeling labels. |
| **Synonyms** | A dictionary bundled with the app that maps English and Russian words to ML Kit labels. |
| **Crop** | The object cut out of a camera frame by ML Kit Object Detection. |
| **Embedder** | The model that turns a Crop into an Embedding. MediaPipe Image Embedder (MobileNetV3) at first; it can be swapped. |
| **Embedding** | The vector the Embedder produces for one Crop. |
| **Match score** | The best cosine similarity between the live Crop's Embedding and a Reference's Embeddings. |
| **Proof frame** | The camera frame that passed Proof. It stays on the device. |
| **Outcome** | DONE (Proof passed within the Window), SKIPPED (Skip), MISSED (the Window ended without Proof) or UNVERIFIED (Proof was not available). |
| **Result** | What is published when an Occurrence ends: the Outcome and what the model saw. It never contains an image or an Embedding. |
| **Friend** | A mutual link between two users, created through a friend request by username. |
| **Feed** | Friends' Results. |
| **Outbox** | The local queue of changes waiting to be sent to the backend. |

## Idea

A habit tracker where a habit counts only when the camera sees its Proof within the Habit's
Window: a Reference from the user's Library, or anything that matches a Label. The app does not
nag: one Reminder when the Window opens, one shortly before it closes. Friends get the
Result, not the photo. They see `Anna - Gym bag 07:12 - Bag 91%, match 87%`, and they see
every SKIPPED and MISSED too.

## Decisions taken

- **Proof** runs on the device with the live camera (gallery for emulator execution). A
  positive decision must hold for several consecutive frames.
  - **Reference:** passes when the Match score reaches the Reference's own threshold,
    computed from its Embeddings.
  - **Label:** passes when Image Labeling returns one of the mapped labels with enough
    confidence.
- **A Habit's Proof** is one or more References from the Library (any of them passes) or one
  Label.
- **Adding to the Library:** scan 3–5 angles, name it, live test with a similar but different
  object. If that object passes too, the app warns that the Reference is too generic and
  suggests a Label.
- **Window instead of an alarm:** Proof at any moment inside the Window. No ringing, no snooze.
- **Friends:** mutual, added by exact username and a request. Up to 20.
- **Widget** (Glance): Friends' latest Results.
- **Backend:** Go REST API + PostgreSQL. `docker compose` locally, Kubernetes if deployed.

## Occurrence life cycle

- Proof is accepted only inside the Window.
- If the app was not running at the Window end (device off, app killed), the Occurrence
  becomes MISSED on the next app start.

## Reminders

- Three alarms per Occurrence: Window start, deadline Reminder, Window end (MISSED). Exact
  alarms only if the user granted them; never required.
- Re-armed on boot, app update, time or time zone change, app start and Habit change.
- Ordinary notifications: a swipe dismisses, a tap opens Proof.
- Permissions: notifications and camera only.

## Proof not available

- **When:** no Reference has Embeddings for the current Embedder, a Reference was deleted on
  another device, camera permission is revoked, or the camera fails to open.
- **Detected early** (Habit save, app start, sync, permission change): a warning and one
  notification, not at 07:00.
- **"Can't verify"** is offered only then and ends the Occurrence as UNVERIFIED. Friends see
  it; the streak breaks.
- **Deleting a Reference** used by Habits asks for confirmation.
- **No re-scan from the Proof screen**, otherwise any object scanned there would pass.

## What lives where

| Data | Device | Backend | Visible to Friends |
|---|---|---|---|
| Habits | + | + | title and streak of shared Habits |
| Library: name, Embeddings, Embedder version | + | + | Reference name inside a Result |
| Reference thumbnail | + | + owner only | - |
| Registration Crops, Proof frames | + | - | - |
| Results | + | + | + shared Habits |
| Open Occurrences | + | - | - |

No vector database: the backend only stores Embeddings; matching is brute force on the device
over a few dozen vectors.

## Defaults for open questions (confirm or change)

- **Sync:** a screen fetches from the server and falls back to the Room cache offline. Writes go
  to Room and the Outbox; a refresh does not overwrite unsent changes. The last received
  change wins. Reminders, Proof and Results never wait for the network. No account: local
  only.
- **Embedder upgrade:** re-embed from stored Crops; otherwise the Reference needs a re-scan.
- **Label** is saved only if Synonyms map it to at least one ML Kit label; otherwise the app
  suggests a Reference.
- **Limits of References:** plain generic objects (a white mug) cannot be told apart. The
  Embedder is swappable (for example DINOv2-small).
- **Trust:** Results come from the device and the server cannot check them. Friends, not
  adversaries.
- **False rejects:** thresholds favor accepting, the live Match score is shown, Skip is the
  last resort.
- **Pushes:** backend → FCM, payload is only an id.
- **Account:** optional, Google sign-in; the backend issues its own tokens. Username 3–20
  `[a-z0-9_.]`, unique.
- **Friend requests:** mutual requests are accepted automatically. Remove and block revoke
  access at once; block also hides you from their lookup.

## How each requirement is covered

From [requirements.md](requirements.md)

| Requirement | Coverage |
|---|---|
| 3+ screens + navigation | Screens below, Navigation Compose. |
| MVVM | ViewModel per screen with `StateFlow`; repositories over Room and the API; Occurrence rules in a domain layer shared with the reminder receiver and widget. |
| Local storage | Room (Occurrences, Outbox, cache), DataStore (settings), files (Crops, Proof frames). |
| External web API | Go backend; FCM for push only. |
| Security | Below. |
| 3+ automated tests | Below. |
| Release build | Signed, keystore outside the repo, R8 keep rules for ML Kit, MediaPipe, Room. |

**Screens:** Today, Habit editor, Library, Proof, History, Feed, Friends, Settings, widget.

**Security:**
- Backend: owner-only access to Habits and the Library; Results only for current Friends;
  input validation; rate limiting; secrets outside the repo.
- App: images never leave the device (only thumbnails, re-encoded without EXIF); tokens in
  Android Keystore; Crops, Proof frames and Embeddings excluded from backup.

**Automated tests:**
1. Occurrence rules on a fake clock: Window boundaries, MISSED after device off, when "Can't
   verify" is offered.
2. Proof logic: Reference threshold, Synonyms mapping, consecutive frames.
3. ViewModel state for Today or the Habit editor.
4. Room DAO: Outbox order, refresh keeps unsent changes.
5. Go authorization (testcontainers-go): non-Friends and removed Friends cannot read Results.
