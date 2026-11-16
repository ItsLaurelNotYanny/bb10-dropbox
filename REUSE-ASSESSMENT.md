# BB10 Dropbox reuse assessment and delivery plan

English | [简体中文](REUSE-ASSESSMENT.zh-CN.md)

Assessment date: 2026-11-16. Status: feasibility assessment complete; implementation has not started.

## Decision

Do not adopt Basket as the application's code base for the requested combination of manual transfers and automatic two-way folder synchronization. Build a small native Qt/Cascades client using the existing BB10 toolchain and reusable, appropriately licensed platform components. Use Basket as a behavioral reference unless reuse permissions are clarified.

Basket has useful native UI and Dropbox API v2 work. However, modern authorization, reliable transfers, a persistent sync engine, and background recovery all need substantial new or replacement work. Repairing its existing behavior adds work to the same core tasks a new client requires. The savings are not sufficiently clear to justify adopting it under the user's criterion: skip reuse when the effort is close to starting fresh.

This recommendation applies to the requested complete product. Restoring only an old-style file browser and manual transfers could favor reuse if permissions were clarified. It does not establish that all existing code is unusable.

## Scope and evidence

Primary upstream snapshots:

| Repository | Exact commit | Role |
| --- | --- | --- |
| [doctorrokter/basket](https://github.com/doctorrokter/basket/tree/3d20d69b8123397dcd5a0a1ceb4e8ebc54814333) | `3d20d69b8123397dcd5a0a1ceb4e8ebc54814333` | Native UI, file operations, metadata cache |
| [doctorrokter/qdropbox](https://github.com/doctorrokter/qdropbox/tree/6c35d03aec46838d003c07f7311ceda6fbdd67c9) | `6c35d03aec46838d003c07f7311ceda6fbdd67c9` | Qt 4.8 Dropbox API v2 wrapper |
| [doctorrokter/basket_service](https://github.com/doctorrokter/basket_service/tree/69350174fa11238cfab9fba9486c1d903612b8ae) | `69350174fa11238cfab9fba9486c1d903612b8ae` | Headless service and camera upload |

Inspected repository trees, build/package declarations, authorization UI, application/controller flows, transfer implementation, metadata cache/poller, service queue and camera indexing. Selected source snapshots are under `/tmp/bb10-dropbox-assessment/`; these temporary copies are not a complete build checkout. Original upstream source was not modified.

Shell cloning failed because github.com could not be resolved. Source was retrieved through the connected GitHub tool instead. The BB10 development skill's `python3 scripts/doctor.py --profile native --probe` returned exit 2: `Image unavailable: bb10-native-dev:local`. No target compile, BAR installation, live-account authorization, or device test was performed. Compilation and runtime compatibility remain unverified. This is an adoption/effort assessment, not a code-review approval or a security certification.

## Findings that drive the decision

| Area | Observed upstream behavior | Required work |
| --- | --- | --- |
| Build and package | Basket and its service include absent `config.pri` files. The package expects a sibling `BasketService` binary. That service's source exists separately as `basket_service`; it is not missing entirely. | Reconstruct build configuration, directory/output layout and packaging; validate ARM link and startup. |
| Authorization | Basket's auth page instructs users to copy a token from an implicit-grant URL. Tokens are stored with QSettings; the examined wrapper has no PKCE or refresh-token lifecycle. | New application registration, PKCE code flow, expiry/refresh/revocation handling, credential storage and log redaction. |
| TLS | Requests use Qt networking. Source alone does not establish compatibility with today's certificate chain or TLS negotiation. | Test both API and content hosts on Passport with certificate/hostname verification enabled, including rejection of invalid certificates. Package maintained TLS/CA components if required and feasible. |
| Download integrity | `read()` and `readZip()` append to `downloadsFolder + basename`; file-open/write results are unchecked. Completed downloads are announced from network success alone. | Separate destinations by full mapping, use temporary files, check writes and space, verify size/hash and commit atomically. Prevent collisions and corrupt retries. |
| Upload/recovery | Streaming QFile uploads and chunked upload-session code already exist. Service queues are in memory; upload/network errors call `dequeue()`. | Persistent tasks, bounded retry/backoff, session recovery, cancellation and correct success/failure reporting. Review and test chunk-boundary behavior. |
| Sync meaning | The service watches camera entries and uploads newly discovered files to `/Camera`; the poller updates a remote metadata cache. | A new folder-pair sync engine: recursive state, revision/hash comparison, conflicts, deletion history and recovery. Existing cache polling is not two-way file synchronization. |
| Remote changes | Poller explicitly ignores `hasMore` while processing list-folder continuation results. | Complete pagination, invalid-cursor recovery and durable cursor/state transactions before using this pattern for sync. |
| Reuse permissions | No repository-wide license file appeared in the three primary trees; GitHub reports no detected license. Some generated/template files have Apache notices, and bundled QJson files carry LGPL 2.1 notices. | Clarify permissions for the actual application/library code and preserve component-specific notices. Template notices do not establish a license for the entire project. |

Source anchors:

- Build: [Basket.pro](https://github.com/doctorrokter/basket/blob/3d20d69b8123397dcd5a0a1ceb4e8ebc54814333/Basket.pro#L5), [BAR descriptor](https://github.com/doctorrokter/basket/blob/3d20d69b8123397dcd5a0a1ceb4e8ebc54814333/bar-descriptor.xml#L80), [service project](https://github.com/doctorrokter/basket_service/blob/69350174fa11238cfab9fba9486c1d903612b8ae/BasketService.pro).
- Auth: [authorization page](https://github.com/doctorrokter/basket/blob/3d20d69b8123397dcd5a0a1ceb4e8ebc54814333/assets/pages/AuthFlowPage.qml#L51), [token persistence](https://github.com/doctorrokter/basket/blob/3d20d69b8123397dcd5a0a1ceb4e8ebc54814333/src/applicationui.cpp#L238).
- Transfers: [download writes](https://github.com/doctorrokter/qdropbox/blob/6c35d03aec46838d003c07f7311ceda6fbdd67c9/src/qdropbox/QDropbox.cpp#L534), [upload/session implementation](https://github.com/doctorrokter/qdropbox/blob/6c35d03aec46838d003c07f7311ceda6fbdd67c9/src/qdropbox/QDropbox.cpp#L600), [service error handling](https://github.com/doctorrokter/basket_service/blob/69350174fa11238cfab9fba9486c1d903612b8ae/src/service.cpp#L348).
- Sync: [camera indexing](https://github.com/doctorrokter/basket_service/blob/69350174fa11238cfab9fba9486c1d903612b8ae/src/service.cpp#L254), [failed-upload dequeue](https://github.com/doctorrokter/basket_service/blob/69350174fa11238cfab9fba9486c1d903612b8ae/src/service.cpp#L445), [metadata poller](https://github.com/doctorrokter/basket/blob/3d20d69b8123397dcd5a0a1ceb4e8ebc54814333/src/cache/QDropboxPoller.cpp#L73).

[Clouder](https://github.com/syedshaishad/Clouder) was screened as a secondary candidate, not assessed to the same depth. It has a root Apache license but also [Pleaseread.txt](https://github.com/syedshaishad/Clouder/blob/master/Pleaseread.txt) restricting copying/reproduction. Its repository was last pushed in 2015. It offers no demonstrated advantage that changes this decision. Four discovered Basket forks also have 2018 push dates; those metadata checks do not prove their contents identical or establish a maintained replacement.

## Current Dropbox requirements

Dropbox recommends PKCE for native clients that cannot keep a client secret, and refresh tokens for offline access. A first version may complete authorization in a modern browser on another device and return the code to the BB10 client; it must keep the PKCE verifier associated with that authorization attempt and out of logs. The exact supported flow needs validation in M1. See the [official OAuth guide](https://docs.dropboxapi.com/dropbox-api/docs/oauth).

Dropbox announced API certificate-root changes starting on or after January 1, 2026. This is a reason to test the actual chain, not proof that every BB10 connection fails. See the [official certificate announcement](https://dropbox.tech/developers/api-server-certificate-changes).

Respect `Retry-After` on rate limiting and use upload sessions where necessary; see the [official performance guide](https://docs.dropboxapi.com/dropbox-api/docs/performance).

## Effort comparison

These are low-confidence planning estimates in focused engineer-days, not measured delivery promises. They assume one account, ordinary personal Dropbox folders, a native Passport UI, manual transfers, recursive two-way folder sync and background scheduling. They exclude waiting for hardware/account access, missing-toolchain restoration, license clarification, SDK redistribution and new work caused by an unexpected TLS/platform limitation.

| Route | Rough effort | Interpretation |
| --- | --- | --- |
| Repair/adopt Basket, then add the complete requested behavior | 16–28 days | Saves UI/API plumbing, adds build recovery, legacy behavior remediation and integration work; license uncertainty remains. |
| New minimal native application using established platform components | 15–25 days | Smaller initial feature surface; the same major authorization, transfer and sync work remains necessary. |

The ranges overlap substantially. There is no demonstrated material saving from adopting Basket. Do not invest in porting its whole UI, cache and sharing feature set before the new client's core feasibility has been proven. Re-estimate after M0 and M2; the table below is the more useful planning basis.

## Milestones for the recommended route

Each milestone ends with a versioned BAR or probe artifact, English/Chinese release notes, recorded checks and explicit device evidence. When implementation begins, initialize the project gate from the chosen build and run fast checks before commits and full checks before handoffs. Device tests are recorded separately from host checks and target builds.

| Milestone | Effort | Deliverable | Acceptance |
| --- | --- | --- | --- |
| M0 — Platform and connection proof | 1–2 days | Reproducible minimal ARM BAR; local storage and HTTPS probe | Install/open on Passport 10.3.x; read/write user-selected test locations including SD card if available; verify current Dropbox API/content TLS and rejection of invalid certificates. Record exact OS/root/install environment. |
| M1 — Account authorization | 2–3 days | Native account connection, minimal required scopes, refresh and disconnect | Complete PKCE authorization without entering a password into the client; survive token expiry/restart; revoked credentials stop access; no secrets in logs. Establish protected token storage and its limits on a rooted device. |
| M2 — Reliable manual transfers | 3–5 days | Folder browser, file picker, upload/download queue, progress/cancel/retry | Transfer zero-byte, Unicode-named, nested-path and large/session-upload files; verify byte integrity; same basenames in different paths cannot collide; network interruption, disk-full and restart cannot replace a valid file with partial content. Manual-only release is useful at this point. |
| M3 — Two-way sync with the app running | 4–7 days | Folder-pair configuration, recursive local/remote state, durable tasks and automatic reconciliation | Propagate creates/edits both ways; concurrent edits keep both versions; confirmed deletion policy behaves correctly; offline/restart and cursor reset do not lose data; missing SD/storage is not mistaken for mass deletion. Manual tasks share the transfer queue and run ahead of pending automatic work. |
| M4 — Background sync on Passport | 3–5 days | Native background service, scheduling, connectivity/battery controls and status | With UI closed and screen off, changes synchronize within the configured interval under tested OS conditions; reboot/network loss/service restart recover tasks; repeated runs create no duplicate transfers; record observed battery and lifecycle behavior. |
| M5 — Release hardening | 2–3 days | Release BAR, reproducible build/checks, bilingual setup/recovery documentation | Run conflict/deletion/retry/storage failure matrix, soak test, token revocation and certificate rejection checks on Passport; verify upgrade preserves account/sync state; inspect final permissions and release artifacts. |

Dependencies: M0 → M1 → M2 → M3 → M4 → M5. M2 is a usable manual-transfer release; M3 adds automatic sync while running; M4 is required before claiming background sync support. Both manual transfers and automatic two-way synchronization remain final requirements.

## Stop conditions and decisions before implementation

M0 is a bounded feasibility step. If correctly verified direct HTTPS or required native execution cannot be made to work within the planned probe effort, stop and report the specific obstacle before further product work. A native client plus NAS gateway remains an alternative, but switching architecture requires a revised estimate and a user decision. Never disable certificate verification to make a milestone pass.

Before M3, settle deletion propagation and conflict presentation. Proposed starting policy: preserve conflicting versions; disable automatic deletion propagation until explicitly enabled; then keep deletion/recovery records. This is a proposed default, not a confirmed user decision. Use revision-conditional writes so a remote change between scan and upload cannot be overwritten silently.

Prefer an app-folder permission model if an isolated Dropbox exchange folder satisfies the use case; arbitrary existing Dropbox folders require broader content access. Confirm this before app registration rather than implementing two account modes speculatively. Whole-account mirroring, team administration, shared-link management, media streaming and restoring the original system file-manager integration are outside this estimate.

The rooted, unsupported OS limits protection against another privileged process. Application work can validate connections, limit permissions and avoid data loss; it cannot promise isolation from a compromised OS. No phone connection or private file/account access was attempted during this assessment.
