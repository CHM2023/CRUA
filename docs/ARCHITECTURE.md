# CRUA architecture

This document describes how CRUA is put together: what each component is responsible for, what the data model looks like at the entity level, what each pipeline stage consumes and produces, how alerts travel, and how the system is deployed. It deliberately stops short of implementation detail. Feature definitions, model hyperparameters, decision thresholds and schemas are product intellectual property and are not described here.

## 1. Shape of the system

CRUA has three layers.

- **Edge.** One Raspberry Pi per student, on the desk, running two sensing pipelines and a sync agent. All inference happens here. Raw audio and video never leave this layer.
- **Server.** A stateless FastAPI service in front of a PostgreSQL database with Row-Level Security, plus a Web Push sender. It authenticates devices and users, stamps every incoming row with its lesson context, stores it, raises alerts and serves role-scoped reads.
- **Presentation.** One progressive web app with four role-gated views: teacher, parent, management and admin. Installable on phones and desktops, with push notifications through a service worker.

The trust boundary is the HTTPS link between the edge and the server. Everything that crosses it is a derived value: numeric features, scores, labels, timestamps, alert records and short text. The device authenticates with a device token; people authenticate with a signed user token.

## 2. Components and responsibilities

### 2.1 Edge

**Controller.** A supervisor process that launches the two capture pipelines as independent operating-system subprocesses, each in its own isolated Python environment so their dependencies cannot clash. It keeps them alive, polls the server for session context, and gates capture on a flag derived from the school timetable: the pipelines run during a scheduled period and are stopped during breaks and out of hours, releasing the camera and microphone. If the server is unreachable the controller fails open and keeps capturing, because a network blip must not interrupt a live lesson. It also owns the vibration motor command.

**Reading pipeline.** Two cooperating processes. A passive recorder listens continuously and, when a voice-activity detector and a loudness gate agree that the student in front of the device is reading, records one variable-length clip. A worker picks up each clip, runs it through the analysis chain described in section 4.1, writes one record, and deletes the clip.

**Attention pipeline.** A capture process reads frames from the camera module and a processing step turns them into one focus verdict per short window (section 4.2). The same process evaluates the rolling focus ratio against the current class mode, raises low-focus alerts, triggers the motor and pushes its rows to the server.

**Sync agent.** Pushes new reading records, attention windows and alert records to the server in batches with the device token, and deletes rows locally only after the server has acknowledged them. The agent tracks what has been sent; the server deduplicates on stable keys, so a retry after a timeout is always safe.

**Provisioning portal.** On boot the device either joins a known Wi-Fi network silently or raises its own access point with a captive portal. A phone joins it, enters the classroom Wi-Fi credentials and the device token, and the portal validates the token against the server before saving it. No keyboard, monitor or companion app is needed. Helper scripts let an installer reset either the Wi-Fi or the token.

**Field tools.** Touchscreen launchers for camera alignment, a microphone level check with a live transcript, and a motor test, so each piece of hardware can be verified in isolation.

### 2.2 Server

**Authentication and roles.** Two identities with two trust paths. Users carry a signed token issued at login; the API verifies the signature locally and maps the user to a role and a school. Devices carry a bearer token; the API hashes it and looks up the hash, so the plaintext is never at rest. A device can never act as a user and a leaked database row cannot be replayed as a device credential.

**Sync ingest.** Three write endpoints, one per pipeline table, plus a context read. Every write resolves the device to its student and class server-side, ignores any identity fields in the body, validates every row against a strict schema, stamps the lesson context in effect at capture time, and inserts with conflict-ignore semantics on a per-table dedup key.

**Schedule and context.** A pure resolver that maps the school's local time, through its periods and weekly timetable, to the current course, chapter, teacher and mode for a class, and to whether the device should be capturing at all. A teacher can override mode, course or chapter for the current period; overrides are appended to a session history so the current state is always the latest row.

**Alerts and push.** Alert rows are created by the sync layer (reading RED and live low-focus) and by the server's own monthly screen (attention concern). Creating a new alert fans out Web Push messages to the right audience; see section 5.

**Statistics and indicators.** Aggregates for the management dashboard in a single round trip, a live-monitor endpoint for teachers, a per-session focus summary, and a rolling 30-day attention indicator per student that management can recompute on demand.

**Administration.** Creation of schools, classes, courses, chapters, periods, timetable entries, students, parent links and class offerings; staff login provisioning in one step with a one-time temporary password; device provisioning that returns the device token exactly once; token rotation.

### 2.3 Presentation

A single React and TypeScript application. The role is read once after login and every route is guarded by it. The teacher view covers classes, the current session with its mode and chapter, a live class monitor polling every ten seconds, student detail with full metrics and an acknowledge action, and a dense log of every recording. The parent view shows softened status labels, an attention chart with per-session focus, alert history, a push opt-in per child and a weekly timetable. The management view is a school dashboard with KPIs, decision distribution, alerts per day, focus by course, students ranked by alert count, and the monthly attention check. The admin view is a multi-tab console with a guided provisioning wizard. The service worker caches the application shell for poor networks and handles push events and notification taps.

## 3. Data model

Entities, in four groups. Relationships are described in words; no columns are listed.

**Organisational.** A *school* owns *classes*, *courses* and staff *users*. A course is divided into ordered *chapters*. A *user* has exactly one of four roles (parent, teacher, management, admin) and, unless a parent, belongs to one school. A *student* belongs to exactly one class and never logs in. A *device* is bound one-to-one to a student and holds only the hash of its token plus issue and rotation timestamps and a last-seen time.

**Relationships and schedule.** A *class offering* is the triple of class, course and teacher, and answers all of "what does this class take", "what does this teacher teach" and "who teaches this course to this class". A *parent link* joins a parent user to a student with a relationship label. *Periods* define the school day; *timetable entries* place a class offering into a period on a weekday; a *session history* appends one row every time a teacher sets or overrides the mode, course or chapter for a class, and a view returns the latest row per class as the current session.

**Pipeline.** A *reading record* is one analysed recording: provenance, the numeric feature record, the transcript, and the four model outputs (risk, anomaly, trend, decision with reason), plus the student and lesson context stamped by the server. An *attention window* is one short-window focus verdict with its probabilities and the same context. An *alert* is a warning event with a source, the context in effect, source-specific fields promoted to columns for filtering, a free-form diagnostic payload, and a lifecycle of created, sent and acknowledged. Each table has a unique dedup key that the sync layer relies on.

**Operational.** A *push subscription* is one browser endpoint for one user watching one student. A *baseline backup* is a versioned copy of a student's personal anomaly model so a reflashed device can restore its calibration. An *attention indicator* is the stored result of the monthly 30-day screen for one student.

**Deletion policy.** Rows that are meaningless without their parent cascade (a recording without its student, a device without its student). Rows that keep value when context changes keep the row and null the reference (a recording still happened if the class is later reorganised; an alert still fired if the acknowledging user leaves). Deleting a class with students, or a school, is restricted so it must be a deliberate cleanup.

**Isolation.** Every table has Row-Level Security enabled. Definer-rights helper functions return the caller's role, school and visible student set without recursing into the policies. Read policies give parents their linked children, teachers the students in classes they deliver, and management its own school; admin sees everything. Only a handful of narrowly scoped write policies exist for end users: updating one's own profile without changing role or school, a teacher setting a session for a class they deliver, a teacher acknowledging an alert for one of their students, and a user managing their own push subscriptions. All other writes, including everything the device sends, go through the API's privileged role after the API has done its own authorisation.

## 4. Pipeline stages

### 4.1 Reading pipeline

| Stage | Consumes | Produces |
|---|---|---|
| Capture gate | Live microphone audio | One variable-length clip when sustained, close-range speech is detected |
| Speech-to-text | The clip | A transcript with per-word timing and confidence, computed offline |
| Reading-error detection | Transcript, part-of-speech tags | Counts of skipped, repeated and filler words, located in the transcript, with no reference passage required |
| Feature extraction | Transcript timing and error output | One fixed-width record of 27 fields covering pace, errors, hesitation and recognition confidence plus provenance and the transcript; 13 numeric fields are model inputs |
| Risk scoring | The numeric features | A risk score from 0 to 1000 from a gradient-boosted tree regressor pre-trained on synthetic personas, active from the first recording |
| Personal baseline | The numeric features and the student's own history | An anomaly score from 0 to 1 and an outlier flag from an Isolation Forest trained per student, which activates after an initial history and retrains at later milestones |
| Trend analysis | The student's recent risk scores | A slope, a direction (improving, stable, worsening, insufficient data) and a confidence from a smoothed linear regression with both a significance and a magnitude filter |
| Decision engine | Rolling aggregates of risk, anomaly rate and trend over recent recordings | GREEN, YELLOW, RED or INSUFFICIENT DATA, plus a reason label, from 8 ordered rules; first match wins, severe states first |
| Record and clean up | All of the above | One appended record; the clip is deleted; a RED decision also queues an alert |

Every model stage is graceful-degradable: if a model is missing or a stage fails, its output fields are left empty and the record is still written.

### 4.2 Attention pipeline

| Stage | Consumes | Produces |
|---|---|---|
| Frame capture | The camera module | Frames in memory, at up to 30 per second; frames are rotated to upright for the detector |
| Face landmarks | Each frame | Facial landmark coordinates; processing continues only when a face is present |
| Feature aggregation | Landmarks across one short window | 6 features derived from eye, gaze and head-pose signals, averaged over the window |
| Classification | The window features | A focused probability and a distracted probability from a Random Forest classifier, and a verdict for the window |
| Mode logic | Recent window verdicts and the current class mode | A rolling focus ratio over a mode-dependent window; a low-focus alert and a motor pulse when the ratio crosses the mode's threshold; nothing in Free mode |
| Sync and retention | Window rows | Rows pushed to the server; only the rows the current mode still needs are kept locally, and only after the server has acknowledged them |

### 4.3 Long-loop attention screen

On the server, a monthly job summarises each student's last 30 days of attention windows into an average focus, a variability measure and a lapse rate, and assigns a graded label. The concern flag requires focus that is both low and inconsistent, which is the attention-difficulty pattern as opposed to a calm but slow reader. It needs a minimum number of windows in the period and returns "insufficient data" otherwise. The label is shown only to staff; a new concern raises an alert whose parent-facing wording is a softened suggestion to check in.

## 5. Alerting flow

**Sources.** Three alert sources exist: reading RED from the decision engine, live low focus from the attention mode logic, and attention concern from the monthly screen.

**Creation.** Device-side alerts arrive through the sync endpoint with the device token and are deduplicated like any other row. The monthly concern alert is created by the server itself. In both cases the alert row is committed before any delivery is attempted, so a push failure can never lose the event.

**Routing.** The audience is chosen by source. Reading RED and attention concern go to the student's linked parents and to the teachers who deliver the student's classes. Live low focus goes to teachers only; parents can see it in history but are not interrupted during a lesson.

**Delivery.** For each recipient the server looks up their push subscriptions for that student, builds a small payload with a title, body and identifiers, encrypts it to the subscription's keys, signs the request with the server's VAPID key and posts it to the endpoint the browser registered. The browser vendor's push service relays it to the phone, where the app's service worker shows a notification even if the app is closed. Repeats for the same alert coalesce. Endpoints that report themselves gone are deleted. A sent timestamp is stamped on success.

**Acknowledgement.** A teacher acknowledges an alert from the student page; the acknowledging user and time are recorded. A duplicate alert row never re-notifies.

**Local feedback.** Independently of all of the above, the device pulses its motor on the same low-focus condition that raises the teacher alert, decided from the local window, with a debounce so repeated lapses do not produce a continuous buzz.

## 6. Session context

The device never knows which student it serves; the server resolves that from the token. What the device does fetch, on a regular poll, is the lesson context: the current course, chapter, teacher and class mode, and whether a period is in session. The source of truth is the timetable, resolved in the school's time zone, with the teacher's session history layered on top as an override for the current period. Every row the device sends is stamped on the server with the context in effect at the moment it was captured, which is what lets the portals group attention by session and recordings by course and chapter later, and lets management see how many rows were captured outside any scheduled period.

## 7. Sync protocol properties

- **Identity.** One token, one student. The token travels as a bearer header; the server stores and compares a hash. Rotation invalidates a lost device instantly.
- **Idempotency.** Each pipeline table has a stable dedup key; inserts ignore conflicts and the response reports what was accepted. Retries are free.
- **Durability.** Push before delete. The device trims its local buffer only after acknowledgement, so an outage delays data rather than losing it, and a reboot resumes where it left off.
- **Bounded local state.** The reading buffer is capped at a size that still covers every rolling window the models need; the attention buffer keeps only what the current mode needs.
- **Standby.** Capture stops outside scheduled periods; the sync agent keeps draining already-captured rows during standby.
- **Fail open.** If the server cannot be reached, capture continues and the previous mode stays in effect.
- **Strict validation.** Rows that do not match the schema are rejected; the device sanitises known edge cases (a silent clip, a malformed probability) before sending so one bad row cannot block a batch.

## 8. Security model

Three independent layers, so no single failure exposes data.

1. **At the door.** Users present a signed token verified locally against the identity provider's key, with no round trip per request. Devices present a token that is hashed and looked up.
2. **In the API.** A role guard on every endpoint, plus explicit "can this caller see this student or class" checks that mirror the database policies.
3. **In the database.** Row-Level Security on every table, with definer-rights helpers and a privileged role used only by the API after its own checks.

Transport is HTTPS end to end. Secrets exist only as environment variables on the hosting platform and are never committed. Device tokens are shown once at provisioning and never stored in plaintext. Parent-facing endpoints soften numeric scores into labels at the API layer, not in the schema, so staff and parents read the same rows through different lenses.

## 9. Deployment topology

- **Edge.** Raspberry Pi 4 per student running the OS vendor's Linux, with the two pipelines in separate Python virtual environments and the controller, motor and provisioning service under the system interpreter. A systemd service brings up the provisioning portal at boot. A 3.5-inch touchscreen hosts the field tools. The device sits in a 3D-printed desk enclosure with the camera on an arm facing the student.
- **API.** The FastAPI service runs as a container on a platform-as-a-service, built from the repository and redeployed on every push to the main branch, with health and readiness endpoints. It is pinned to the same region as the database to keep per-query latency low. The free tier spins down when idle; the device's retry logic absorbs the resulting cold start.
- **Database and identity.** A managed PostgreSQL instance with an integrated authentication service that issues the user tokens. Schema is applied through plain SQL migrations.
- **Web app.** Built to static files and served from an edge network, redeployed on every push. HTTPS on both the API and the app is what makes installation and Web Push possible.
- **Push.** No infrastructure of its own: the server speaks the open Web Push protocol directly to whichever push service each browser registered with.

All hosting runs on free tiers; the prototype targeted one school, a small number of devices and a few dozen students.

## 10. Operational characteristics

- The live class monitor is a polling view; the budget from a lapse on the desk to its appearance on the teacher's screen is tens of seconds, dominated by the window length, the sync interval and the refresh interval.
- RED alerts take the push path, which fires as soon as the alert row is committed and does not depend on client polling.
- The motor pulse has no server dependency and arrives within the moment of the lapse.
- A device can be offline for hours without data loss as long as its bounded buffer is not exceeded, and indefinitely if no capture is scheduled.
- A reflashed device recovers its student binding from a new token and its baseline calibration from the latest backup.

## 11. Known limitations

- The risk model is trained on synthetic data and must be re-evaluated and retrained on real labelled recordings before any claim about real-world accuracy.
- Decision rules were calibrated against the synthetic persona ranges and are expected to move once outcome data exists.
- Consent records, an access audit log, soft-delete and per-user notification preferences are not implemented.
- Capture follows the timetable; unplanned sessions and substitute teachers are not modelled.
- Wi-Fi connectivity is checked at boot; a mid-session drop is not automatically recovered until the next boot.
- The system is a screening aid. Every output is a signal for a human to act on, never a diagnosis.
