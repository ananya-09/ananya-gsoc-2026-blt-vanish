# Overview of Layers, Workflows, and UI Component Matrix

## I. Detailed Layered Architecture & Processing Workflows

The BLT-Vanish Web platform operates on a **zero-trust, zero-retention hybrid architecture** that cleanly splits execution responsibilities across three logical environments. This division ensures that no personally identifiable information (PII) or unencrypted credentials ever touch external infrastructure.

```txt
+-------------------------------------------------------------------------+
|                        LOCAL LAYER (Flutter App)                        |
|   - Inspects Raw PII   - Local Feature Extraction  - Risk Computation   |
+-----------------------------------+-------------------------------------+
|
Requests Public | Responds with
Metadata Only   | Clean JSON DTOs
v
+-------------------------------------------------------------------------+
|                       EDGE LAYER (Cloudflare Workers)                   |
|   - Domain Reputation  - Broker Data Endpoints     - Template Servers   |
+-----------------------------------+-------------------------------------+
|
v
+-------------------------------------------------------------------------+
|                  BACKEND THREAT INTELLIGENCE LAYER                      |
|   - Rate Limiting      - Secure Alert Routing      - Observability      |
+-------------------------------------------------------------------------+
```

### 1. The Local Layer (Flutter Application Client)

The local client executes all computations involving raw user identity context.

* **Workflow Execution:** When an asset (URL, message string, or credential pair) is triggered for evaluation, the local layer captures the telemetry input, processes it safely inside volatile memory, and maps it directly to local feature extractors.

* **Security Isolations:** This layer runs completely offline-first if necessary, encrypting local state snapshots with platform-native keystores (iOS Keychain, Android KeyStore, or Windows Credential Manager).

* **Contract Responsibility:** It enforces schema parsing gates using typed factory models (`SecurityAlert.fromJson`), throwing immediate exceptions if data structures arriving across boundaries are malformed or out of bounds.

### 2. The Edge Layer (Cloudflare Workers Runtime)

The edge network operates as a fast, stateless public metadata distributor.

* **Workflow Execution:** Workers respond to lookups sent by the client via endpoints like `/api/brokers`, `/api/health/:id`, `/api/reputation`, and `/api/templates/:templateId`. It handles data matching requests within a strict **1500ms timeout budget**.

* **Security Isolations:** Workers are structurally incapable of recording, processing, or logging user PII. 

* **Contract Responsibility:** The edge layer utilizes TypeScript type guards (`isSecurityAlertDTO`) to perform strict symmetric schema validation on requests and responses before passing data payloads down to the client device.

### 3. The Backend Threat Intelligence Layer

This production support tier acts as a control plane for handling infrastructure traffic rules and routing integrations securely.

* **Workflow Execution:** It handles incoming automated provider alert feeds, normalizes security warning payloads into unified events, and pushes alert context directly to the client synchronization pipeline.

* **Security Isolations:** Employs route-level rate limiting, zero-knowledge encryption pipelines, and privacy-first log redaction middleware to guarantee metadata cannot be reverse-engineered to identify a user.

---

## II. End-to-End Core Threat Workflows

### 1. Real-Time Phishing & Malicious-Link Processing Pipeline

This workflow inspects raw, untrusted links submitted by the user through manual pasting or OS sharing mechanisms.

```txt
[Input Pasted] ---> (UrlSignalExtractor) ---> [Call Edge API] ---> (RiskAssessmentBuilder) ---> (RiskClassifier) ---> [UI Draw]

```

* **Step A (Local Extraction):** The link is sent to the `UrlSignalExtractor`. It performs on-device analysis to calculate Shannon entropy scores and evaluate malicious lexical indicators:

    * `entropy`: Normalized string randomness evaluation.

    * `suspicious_tokens`: Checks for string sequences matching target phrases like `verify`, `secure`, or `login`.

    * `risky_tld`: Matches against high-frequency scam extensions like `.zip`, `.click`, `.top`, or `.xyz`.

* **Step B (Edge Enrichment):** The app strips all unique parameter query context from the domain to mask individual actions and submits the clean host string to the Cloudflare Worker `/api/reputation` endpoint to pull public domain health metrics.

* **Step C (Score Composition):** The client merges local heuristics and public intelligence values using the strict Phase 1 risk composition model:
    $$Risk = w_1U + w_2M + w_3R$$

    *(Where $U$ represents the URL lexical risk indicators, $M$ represents message indicators, and $R$ represents reputation intelligence signals)*.

* **Step D (Severity Mapping):** The computed score is passed through a deterministic threshold classifier to map it directly into standardized severity classes:

    $$severity(Risk)=\begin{cases} critical, & Risk \ge 0.85 \\ high, & 0.65 \le Risk < 0.85 \\ medium, & 0.40 \le Risk < 0.65 \\ low, & Risk < 0.40 \end{cases}$$

### 2. Zero-Plaintext Credential Hygiene & Tracking Lifecycle

This workflow tracks internal credential safety and orchestrates password strengthening tasks locally.

* **Step A (Ingestion & Hash Derivative Processing):** The user saves or imports credential references. The `CredentialVaultService` immediately processes the plaintext inside runtime-volatile memory, computes entropy and character variance snapshots, hashes the material for pattern matching, and securely clears raw strings from device memory.

* **Step B (Local Scoring):** The `CredentialAnalyzer` passes the mathematical properties of the credential through the baseline scoring model:
    $$CredentialRisk = 0.35S + 0.40R + 0.25E$$

    *(Where $S$ represents strength weakness, $R$ represents reuse count likelihood, and $E$ represents exposure posture context)*.

* **Step C (Remediation Checklist Injection):** If the risk score crosses designated limits, a warning is triggered. The `RemediationPlanner` creates a corresponding tracking container called a `RemediationTask` with an initial state marked as `notStarted`.

* **Step D (Orchestration & State Change Monitoring):** When the user executes a task, the `PasswordRotationOrchestrator` runs state rules to guide the change lifecycle (`requested -> applied -> verified -> closed`). Upon hitting a verified status, the task transitions to `completed`, safely updating the encrypted local timeline view.

### 3. Privacy-Safe Breach/Dark Web Exposure Pipeline

This service monitors historical data exposures using privacy-safe tracking keys.

* **Step A (Identity Anonymization):** The application applies a local cryptographic salt to user email identifiers and computes a non-reversible hash: `accountHash = sha256(emailLowercase + salt)`.

* **Step B (K-Anonymity External Check):** The system passes anonymized search fragments to the external feed (such as Have I Been Pwned). The edge infrastructure queries breach databases using prefix lookups, ensuring your cleartext query credentials never traverse external paths.

* **Step C (Normalization):** Incoming exposure alerts are parsed by the client's `ExposureAdapter` and normalized directly into uniform `ExposureEvent` models containing standardized urgency, recency, and impact indicators.

### 4. Suspicious Login Correlation & Threat Escalation

This backend-to-client intelligence loop processes account takeover risk profiles.

* **Step A (Context Vector Evaluation):** Whenever an account event metadata notification is ingested, the system reads specific environmental check properties, passing flags such as `newGeo`, `newDevice`, `oddHour`, and `priorPhishingAlert` through an anomaly engine.

* **Step B (Correlation Calculation):** The engine tracks environmental variations, adding a priority escalation value using the Phase 3 prioritization matrix:
    $$Priority = 0.45 \cdot Severity + 0.30 \cdot Confidence + 0.25 \cdot ActionUrgency$$

* **Step C (Playbook Selection & Local Escalation):** If a high priority score is confirmed, a specific playbook is selected (e.g., *Session Revoke / MFA Reset*). If the task remains ignored by the user over several days, the local state engine triggers internal escalation rules, updating the active tracking value from standard priority up to critical levels (`escalationLevel 1` -> `escalationLevel 2`).

---

## III. UI Component Matrix & Cross-Layer Interactivity

The application UI acts as a highly scannable, componentized presentation layer that directly binds interactive controls to our underlying asynchronous security state engines.

| UI Component File Name | Component Functional Goal | Linked Internal Workflows | Edge / Backend Interactions |
| :--- | :--- | :--- | :--- |
| **`security_center_screen.dart`** | Acts as the primary interactive entry view for link checks, manual scanning input boxes, and real-time scanning action buttons. | • Real-Time Phishing Scanning Workflow<br>• Malicious Link Classification Pipeline | Calls Cloudflare Worker endpoints (`/api/reputation`) using a strict 1500ms timeout threshold, returning fallback mock structures if offline. |
| **`severity_cards_widget.dart`** | Displays large, color-coded dashboard indicators matching computed risk tiers (`critical` = Red, `high` = Orange, `medium` = Yellow, `low` = Green). | • Deterministic Threat Severity Mapping<br>• Phishing Analysis Presentation Flow | None (Pure client rendering block driven by hydrated `SecurityAlert` payloads). |
| **`reason_code_chips.dart`** | Translates abstract backend technical identifiers into descriptive, scannable chips via translation wrappers (e.g., converts `SUSPICIOUS_TLD` text dynamically). | • System Risk Explainability Framework<br>• Threat Factor Weight Presentation | None (Renders data derived directly from the immutable `reasonCodes` list within the local data layer). |
| **`remediation_cta_button.dart`** | Displays clear contextual call-to-action controls based on threat types, mapping direct one-click routes to resolution workflows. | • Remediation Task Lifecycle Initiation<br>• Password Rotation Routine Trigger | Triggers local client operations; may request updated opt-out instruction payloads from `/api/templates`. |
| **`unified_triage_timeline.dart`** | Renders a unified chronological activity board merging cross-module risks sorted by global urgency parameters. | • Unified Triage Evaluation Engine<br>• Equal-Score Source Priority Tie-Breaking | Consumes compressed local tracking snapshots; displays push notifications pushed from the backend context processor. |
| **`exposure_timeline_view.dart`** | Displays dark-web and public breach exposure logs grouped cleanly by recency and threat weight constraints. | • Privacy-Safe Breach Ingestion Pipeline<br>• Account Identity Hashing Sequence | Displays last-sync indicators; offers manual sync triggers that run anonymized, prefix-hashed network discovery lookup requests. |
| **`playbook_action_card.dart`** | Renders an overlay view containing guided instructions for account security tasks, session cancellations, and lock procedures. | • Takeover Prevention Action Playbook Selection<br>• Account Hardening Progress Sync | Monitors state changes inside the `RemediationTask` engine, securely committing local snapshot state updates to the app database. |

---
