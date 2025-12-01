# Public API, Function, and Component Reference

This document summarizes every externally exposed tool, API wrapper, and reusable component present in this repository. It is intended for engineers wiring the Chord DSO voice flow into NexHealth as well as anyone maintaining the supporting Node-RED and Custom Tool scripts.

---

## 1. Shared Architecture & Authentication

| System | Purpose | Base URL(s) | Auth Pattern |
| --- | --- | --- | --- |
| NexHealth V2 API | Patient, appointment, and availability management | `https://nexhealth.info` | Two-step flow: (1) call `POST /authenticates` with static API key in `Authorization` header to obtain a bearer token, (2) call target endpoint with `Authorization: Bearer <token>` plus `Accept: application/vnd.Nexhealth+json;version=2`. |
| Chord Node Fabric Workflow | Internal orchestration used by the phone agent to fetch slots and create/cancel appointments without exposing NexHealth directly | `https://c1-aicoe-nodered-lb.prod.c1conversations.io/FabricWorkflow/api/exposed/chord/*` | Direct JSON requests with no prior auth in the code samples. Validation is performed server-side, so strict input hygiene is critical. |
| Voice Components | Audio playback, telephony disconnect, session summaries | Local functions such as `AudioTool` and payload builders (`FinalPayload.json`). | No network auth; inputs validated in the function. |

**Best practices**
- Cache NexHealth bearer tokens only within the lifetime of a single tool invocation; every wrapper currently authenticates per call for simplicity.
- When chaining tools (e.g., fetch slots → create appointment), always persist the selected slot object to a variable (e.g., `$selected_appointment`) so that you can forward the exact `time` and `operatory_id` fields without mutation.
- All date-time strings must remain in ISO 8601 format with timezone offsets.

---

## 2. NexHealth Appointment Availability APIs

### 2.1 `get_ApptSlots` (Availability endpoint)
- **Wrappers:** `Old/get_ApptSlots-CustomTool.json`, `Old/ChordTest_Get_ApptSlots.json`, `Old/JL-chordEnh_getApptSlots.json` (enhanced slice control).
- **Endpoint:** `GET /availability` (sample query: `https://nexhealth.info/availability?subdomain=cdm&location_id=333724&provider_id=421458314&start_date=2026-04-13&end_date=2026-04-20`).
- **Primary inputs:**
  - `providerId` *(optional)* – filter to a specific dentist/hygienist.
  - `startDate`, `endDate` *(optional but recommended)* – restrict the window (`YYYY-MM-DD`).
  - `appointmentTypeId` *(optional)* – align slot types with intent.
- **Response shape:** array of slot objects containing provider metadata, start/end timestamps, `operatory_id`, and availability flags (`Old/NexHealth_API_Response_Structures.json`).
- **Usage guidance:**
  1. Call immediately after validating appointment type, location, and (if applicable) provider preference.
  2. Normalize colloquial ranges ("next week", "tomorrow") into concrete `startDate`/`endDate` values before invocation.
  3. Deduplicate slots per `operatory_id` if presenting condensed menus to callers.
- **Example output** (`chordGen_getApptSlots - Tool Output.json`):

```json
[
  {"time":"2026-04-13T08:00:00.000-04:00","end_time":"2026-04-13T08:15:00.000-04:00","operatory_id":24844},
  {"time":"2026-04-13T08:30:00.000-04:00","end_time":"2026-04-13T08:45:00.000-04:00","operatory_id":24844},
  {"time":"2026-04-13T09:00:00.000-04:00","end_time":"2026-04-13T09:15:00.000-04:00","operatory_id":24841}
]
```

### 2.2 `ChordTest_get_ApptSlots` (Appointment Slots endpoint)
- **Wrappers:** `Old/ChordTest_Get_ApptSlots.json`, `Old/JL-chord_getApptSlots-CustomTool.json`.
- **Endpoint:** `GET /appointment_slots` with query params `operatory_ids[]`, `pids[]`, `lids[]`, `days`, etc.
- **Inputs:**
  - `operatoryIds` *(required)* – comma-separated list; converted into repeated `operatory_ids[]` params.
  - `days` *(default 5)* – number of days ahead to search.
  - `providerId`, `appointmentTypeId` *(optional).* 
- **Response:** Nested structure: `data[0].slots` contains slot array; wrappers flatten into plain arrays.
- **Usage:**
  - When operators want a strict subset of operatories (e.g., hygiene chairs only) or when you need raw slot arrays without provider bundling.
  - Enhanced wrapper `JL-chordEnh_getApptSlots.json` supports `maxSlots` trimming for UX-limited menus and includes defensive checks for empty payloads.

---

## 3. NexHealth Appointment Lifecycle APIs

### 3.1 `create_Appt` / `schedule_Appt` / `chord_createAppointment`
- **Wrappers:** `Old/create_Appt-CustomTool.json`, `Old/ChordTest_Schedule_Appt.json`, `Old/chord_createAppointment-CustomTool.json`, `Old/JLTEST_Create_Appt-Rev.json`.
- **Endpoint:** `POST /appointments?subdomain=cdm&location_id=333724&notify_patient=false`.
- **Inputs:**
  - `appointmentTime` (`start_time`), `patientId`, `providerId`, `operatoryId` (all required).
  - `note` optional, defaults to `"Schedule by AI"` to preserve auditability.
- **Flow:** Authenticate → create headers with bearer token → send request body containing `appt` object and `appointments_per_timeslot` (always 1 in samples).
- **Successful response example** (`chordGen_createAppt_v2 - Tool Output.json`): returns appointment metadata plus `appointmentId` convenience field.
- **Usage instructions:**
  1. Always parse incoming IDs to integers before embedding into JSON body (`parseInt` is used consistently).
  2. Validate that `operatoryId` originated from the selected slot; do not invent values.
  3. Capture `id`, `start_time`, and `operatory_id` from the response for downstream confirmation/cancellation.

### 3.2 `chordNode_createAppt` (Fabric Workflow proxy)
- **Wrappers:** `Old/chordNode_createAppt.md`, `Old/JL-chordNode_createAppt.json`, `Old/JL2-chordNode_createAppt_v2.json`.
- **Endpoint:** `POST /exposed/chord/createAppt` (or `_v2`).
- **Inputs:**
  - v1: `apptTime`, optionally `operatoryId`.
  - v2: **requires** both `appointmentTime` and `operatoryId`. Docs explicitly warn against placeholders; failing to forward literal values yields validation errors.
- **Usage:**
  - Use when the voice agent should rely on the Node-RED workflow instead of hitting NexHealth directly (e.g., when additional orchestration or audit logging is needed by operations).
  - Ensure `$selected_appointment` is populated immediately after slot selection; feed `[$selected_appointment.time]` and `[$selected_appointment.operatory_id]` into this tool.

### 3.3 Confirmation and Cancellation
- **Wrappers:** `Old/ChordTest_Confirm_Appt.json`, `Old/JLTEST_Confirm_Appt-Rev.json`, `Old/ChordTest_Cancel_Appt.json`, `Old/JLTEST_Cancel_Appt-Rev.json`, `Old/JL-chordNode_cancelAppointment.json`.
- **Endpoints:**
  - NexHealth: `PATCH /appointments/{appointmentId}?subdomain=cdm&id={appointmentId}` with body `{ "appt": { "status": "confirmed" | "cancelled" } }`.
  - Chord Node proxy: `POST /exposed/chord/cancelAppt` with `{ apptId }`.
- **Usage tips:**
  - Store appointment IDs as soon as appointments are created; they are the only identifiers accepted by these reducers.
  - Capture and relay cancellation reasons verbally if required; the current wrappers only set status, so business logic should log the rationale elsewhere.
  - Example failure (`chordGen_CancelAppt - Tool Output.json`): forgetting to pass `$appointmentId` results in `"$appointmentId is not defined"`. Always validate input presence before invocation.

---

## 4. NexHealth Patient Management APIs

### 4.1 `Patient_Lookup`
- **Wrappers:** `Old/Patient_Lookup-CustomTool.json`, `Old/ChordTest_Patient_Lookup.json`, `Old/JLTEST_Patient_Lookup-Rev.json`.
- **Endpoint:** `GET /patients?subdomain=cdm&location_id=333725&date_of_birth=YYYY-MM-DD`.
- **Input:** `dateOfBirth` (required).
- **Response:** `data` array containing patient objects + `count`. See `Old/NexHealth_API_Response_Structures.json` for canonical field list (id, names, contact info, address, audit timestamps).
- **Usage:**
  1. When callers mention their DOB, run lookup before deciding between "existing" and "new" patient flows.
  2. If `count === 0`, treat as new patient and capture intake info.

### 4.2 `create_Patient`
- **Wrappers:** `Old/ChordTest_Create_Patient.json`, `Old/JLTEST_Create_Patient-Rev.json`.
- **Endpoint:** `POST /patients?subdomain=cdm&location_id=333725`.
- **Inputs:** `firstName`, `lastName`, `dateOfBirth`, `phoneNumber` (required). `email`, `address` optional.
- **Successful response example** (`patient_data.json`): returns `id`, demographics, `bio`, balance, and location IDs.
- **Error example** (`output.json`): missing provider and patient subfields leads to HTTP 400 with detailed validation errors. Always ensure nested fields (e.g., `patient[bio][phone_number]`) are provided when required by the API environment.
- **Usage tips:**
  - Normalize phone numbers to digits only; NexHealth rejects blank strings.
  - When capturing minors, ensure guardian info is handled elsewhere—current payload only captures the child.

---

## 5. Chord Node Slot APIs

| Tool | File | Endpoint | Inputs | Notes |
| --- | --- | --- | --- | --- |
| `chordNode_getApptSlots` | `Old/chordNode_getApptSlots.md` | `GET /exposed/chord/getApptSlots` | none | Returns raw slot payload with `time`, `end_time`, `operatory_id`. Used when bots rely entirely on the Fabric workflow. |
| `chordNode_getApptSlots` (parametrized v2) | `Old/JL2-chordNode_getApptSlots.json` | same | `startDate`, `endDate` (required); `providerId`, `appointmentTypeId` optional | Allows date windows derived from natural language ("this week", "next Tuesday"), enforcing slot selection hygiene before `createAppt_v2`. |
| `chordNode_createAppt` / `_v2` | `Old/chordNode_createAppt.md`, `Old/JL2-chordNode_createAppt_v2.json` | `POST /exposed/chord/createAppt[_v2]` | `apptTime` (+ `operatoryId` in v2) | v2 adds strict validation and clearer runtime errors. |
| `chordNode_cancelAppointment` | `Old/JL-chordNode_cancelAppointment.json` | `POST /exposed/chord/cancelAppt` | `apptId` | Returns workflow-defined status; always log both request and response for auditing. |

**Usage ordering:** `chordNode_getApptSlots` → store selection → `chordNode_createAppt_v2`. Never attempt to call `_createAppt_v2` without an operatory ID sourced from the previous step; the guard clauses explicitly reject placeholder strings (see `Old/chordNode_createAppt.md`).

---

## 6. Conversation Components

### 6.1 `AudioTool`
- **File:** `Old/AudioTool-CustomTool.json`.
- **Purpose:** Centralized playback helper for canned prompts (welcome, disclaimers, confirmations, transfers) with English/Spanish localization and optional custom messages.
- **Inputs:**
  - `audioType` *(required unless `customMessage` provided)* – one of `welcomeBack`, `newPatientDisclaimer`, `appointmentConfirmed`, `appointmentCancelled`, `seeYouSoon`, `transferMessage`.
  - `language` (`en` default, `es` supported).
  - `customMessage` overrides canned text.
- **Behavior:** Returns JSON describing the playback event (`operation`, `message`, timestamp). Downstream systems can look for `success: true` before queuing telephony actions.
- **Usage tip:** Validate `language` upstream to avoid repeated `Invalid language` failures; the tool already returns a structured error, so surface that to the caller when needed.

### 6.2 Telephony & Session Payloads
- **File:** `FinalPayload.json`.
- **Components:**
  - `telephonyDisconnectCall`: contains `uuiPayload`, `phoneNumber`, `uuiTreatment` – use when forcibly ending calls with metadata consumed by downstream telephony.
  - `setConfig`: DTMF gating (`maxDTMFLength`, `DTMFTimeout`, `minDTMFLength`).
  - `Call_Summary`: structured notes summarizing caller identity, intent, disposition, appointment details, and placeholder IDs.
  - `TC`: integer-like string denoting transfer count / touch count (value `"13"` in sample payload).
- **Usage:** Use as a template when persisting handoff payloads back into CRM / analytics. Update `providerId`, `operatoryId`, `Appointment ID`, etc., once actual IDs are known.

---

## 7. Sample Payloads & Testing Artifacts

| File | Description | How to Use |
| --- | --- | --- |
| `chordGen_getApptSlots - Tool Output.json` | Real slot response for April 13, 2026 | Replay to test slot-to-dialog rendering logic. |
| `chordGen_createAppt_v2 - Tool Output.json` | Successful appointment creation payload | Use as mock when building downstream confirmation messaging. |
| `chordGen_CancelAppt - Tool Output.json` | Failure scenario (`$appointmentId` missing) | Include in negative test cases for cancellation validation. |
| `patient_data.json` | Successful `create_Patient` output | Seed local databases or demos without hitting production. |
| `output.json` | Error response for incomplete patient creation | Highlight required fields during QA. |
| `Old/NexHealth_API_Response_Structures.json` | Normalized schema reference | Serves as canonical contract documentation for data contracts. |

---

## 8. Implementation Recipes & Usage Instructions

1. **Existing patient scheduling**
   - `Patient_Lookup` (confirm profile) → `get_ApptSlots` (collect preferences) → capture `$selected_appointment` → `create_Appt` (or `chordNode_createAppt_v2`) → optional `confirm_Appt` if caller explicitly confirms.
2. **New patient intake**
   - Capture demographics → `create_Patient` → store returned `patient_id` → follow existing scheduling flow with the new ID.
3. **Reschedule / cancel**
   - Lookup appointment record (internal CRM) → `cancel_Appt` or `chordNode_cancelAppointment` → optionally call `get_ApptSlots` to immediately offer alternatives.
4. **Confirmation reminders**
   - When contact center receives a confirm intent, validate `appointmentId` and run `confirm_Appt` before playing `AudioTool` with `appointmentConfirmed` prompt.
5. **Telephony wrap-up**
   - After any terminal action, populate `Call_Summary` (patient name, DOB, disposition, `Appointment_Details`), adjust `telephonyDisconnectCall` fields, and drop the payload to the orchestration layer before ending the call.

Following the sequencing and validation rules above ensures every public API, function, and component in this repository is exercised safely with predictable inputs, minimizes NexHealth rejections, and keeps the Chord DSO experience consistent across agents.
