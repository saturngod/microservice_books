# အခန်း ၄၈: Healthcare System (EMR / Telemedicine)

## နိဒါန်း

Healthcare system သည် software engineering ၏ အရေးပါဆုံးနှင့် technical ရှုပ်ထွေးမှုအများဆုံး domain တစ်ခုဖြစ်သည်။ Patient data confidentiality ကာကွယ်ခြင်း၊ system availability ကို highest level ဖြင့် maintain ပြုခြင်း၊ medical decisions ၏ audit trail complete ထားရခြင်း၊ ပြင်ပ health systems များနှင့် interoperability ရှိခြင်း — ဤ requirements များက healthcare software ကို unique ဖြစ်စေသည်။

HIPAA (Health Insurance Portability and Accountability Act) compliance သည် US healthcare systems အတွက် မဖြစ်မနေ လိုအပ်ပြီး data breach ဖြစ်ပါက ကြီးမားသော legal consequences ကို ရင်ဆိုင်ရမည်။ European Union တွင် GDPR ပိုမိုတင်းကျပ်စွာ apply ဖြစ်သည်။

---

## ၄၈.၁ Requirements: HIPAA, Availability, Auditability

### HIPAA Compliance Key Requirements

**PHI (Protected Health Information) Protection:**
- At-rest encryption: AES-256
- In-transit encryption: TLS 1.3
- Role-based access control (RBAC)
- Complete audit logging of all PHI access
- Minimum necessary principle enforcement

**Breach Notification:**
- 60 days အတွင်း HHS ကို notify ပြုရမည်
- Affected individuals ကို notify ပြုရမည်

### Functional Requirements

- Patient registration & identity verification
- EHR (Electronic Health Records): medical history, diagnosis, prescriptions, lab results
- Appointment management & scheduling
- Telemedicine video consultation (WebRTC)
- Drug interaction checking
- Insurance claims & billing
- Consent management

### Non-Functional Requirements

| Metric | Target |
|--------|--------|
| Availability | 99.99% (4 minutes downtime/month) |
| PHI encryption | AES-256 at rest, TLS 1.3 in transit |
| Audit log retention | 6+ years (HIPAA documentation rule 45 CFR 164.316(b)(2)(i) သည် policies/procedures documentation ကို 6 နှစ် ထိန်းသိမ်းရန် လိုအပ်သည်။ Audit logs ကိုယ်တိုင် retention period ကို 45 CFR 164.308(a)(1)(ii)(D) အောက်တွင် organization ကိုယ်တိုင် သတ်မှတ်ရမည်) |
| RTO (Recovery Time Objective) | < 1 hour |
| RPO (Recovery Point Objective) | < 15 minutes |
| Data residency | Organization policy အရ သတ်မှတ်ရမည် (HIPAA သည် US-only storage ကို မလိုအပ်သော်လည်း BAA (Business Associate Agreement) နှင့် safeguards လိုအပ်သည်) |

---

## ၄၈.၂ Domain Decomposition: Patient, Provider, Appointment, EHR, Prescription, Billing, Audit

```
+------------------------------------------------------------------+
|                      Healthcare Platform                         |
+------------------------------------------------------------------+
|  +----------+  +---------+  +----------+  +------------------+  |
|  | Patient  |  |Provider |  |Appointment|  |   EHR Service   |  |
|  | Service  |  | Service |  |  Service  |  | (Event Sourced) |  |
|  +----------+  +---------+  +----------+  +------------------+  |
|                                                                  |
|  +----------+  +---------+  +----------+  +------------------+  |
|  |Prescription| | Billing |  |  Audit  |  | Consent         |  |
|  | Service  |  | Service |  |  Service |  | Management      |  |
|  +----------+  +---------+  +----------+  +------------------+  |
|                                                                  |
|  +------------------+  +----------------------------------+      |
|  | Telemedicine     |  |     Notification Service          |      |
|  | (WebRTC)         |  +----------------------------------+      |
|  +------------------+                                           |
+------------------------------------------------------------------+
```

**Patient Service:** Identity, demographics, insurance information management
**Provider Service:** Doctor/nurse profiles, specializations, availability schedules
**Appointment Service:** Scheduling, reminders, cancellation workflows
**EHR Service:** Medical records -- event-sourced, append-only, immutable audit trail
**Prescription Service:** Drug prescriptions, interaction checks via synchronous gRPC
**Billing Service:** Insurance claims, payments -- Saga pattern for distributed billing
**Audit Service:** Immutable audit log of all PHI accesses, 6+ years retention
**Consent Management:** Patient consent records, granular data access control

---

## ၄၈.၃ FHIR Standard & Interoperability

FHIR (Fast Healthcare Interoperability Resources) သည် healthcare data exchange ၏ modern standard ဖြစ်ပြီး RESTful API ဖြင့် standardized format (JSON/XML) ဖြင့် data exchange ပြုနိုင်စေသည်။ Hospitals, pharmacies, insurance companies, labs တို့အကြား seamless data sharing ဖြစ်စေရန် FHIR ကို implement ပြုရသည်။

### Key FHIR Resources

```
Patient:          { name, birthDate, gender, address, identifier }
Practitioner:     { name, qualification, identifier (NPI) }
Appointment:      { status, start, end, participant, serviceType }
Observation:      { code (LOINC), value, subject (Patient) }
Condition:        { code (ICD-10), severity, clinicalStatus }
Medication:       { code (RxNorm), ingredient, form }
MedicationRequest:{ medication, patient, dosageInstruction, prescriber }
DiagnosticReport: { code, subject, result (Observations) }
Claim:            { patient, provider, diagnosis, item (CPT codes) }
```

### FHIR API Implementation

```python
class FHIRService:

    def create_patient_resource(self, patient_data):
        return {
            "resourceType": "Patient",
            "id": patient_data.id,
            "identifier": [{
                "system": "http://hospital.org/patient-id",
                "value": patient_data.mrn  # Medical Record Number
            }],
            "name": [{"family": patient_data.last_name, "given": [patient_data.first_name]}],
            "birthDate": patient_data.dob.strftime("%Y-%m-%d"),
            "gender": patient_data.gender.lower(),
            "address": [{
                "line": [patient_data.address_line1],
                "city": patient_data.city,
                "state": patient_data.state,
                "postalCode": patient_data.zip_code,
                "country": "US"
            }]
        }

    def create_observation_resource(self, obs_data):
        """
        Vital signs observation (blood pressure, heart rate, etc.)
        LOINC codes ကို standardized measurement identification အတွက် အသုံးပြု
        """
        return {
            "resourceType": "Observation",
            "status": "final",
            "code": {"coding": [{
                "system": "http://loinc.org",
                "code": obs_data.loinc_code,  # e.g., "8480-6" for systolic BP
                "display": obs_data.display_name
            }]},
            "subject": {"reference": f"Patient/{obs_data.patient_id}"},
            "effectiveDateTime": obs_data.recorded_at.isoformat(),
            "valueQuantity": {
                "value": obs_data.value,
                "unit": obs_data.unit,
                "system": "http://unitsofmeasure.org"
            }
        }
```

---

## ၄၈.၄ Patient Record -- Append-Only Event Sourcing

Medical records ကို delete/update မပြုဘဲ append-only pattern ဖြင့် store ပြုသည်။ ဤ approach သည် HIPAA audit requirements နှင့် medical-legal requirements နှစ်ရပ်လုံးကို ဖြည့်ဆည်းသည်။ Doctor တစ်ဦးက ရေးခဲ့သမျှ၊ ပြောင်းလဲခဲ့သမျှ history အပြည့်အဝ ရှိနေမည်ဖြစ်သည်။

### Event Sourcing Architecture

```
Patient Record Events (Immutable):
+----------+------------+-------------------+------------+-------+
| event_id | patient_id | event_type        | event_data | actor |
+----------+------------+-------------------+------------+-------+
| evt_001  | pat_123    | PATIENT_REGISTERED | {...}     | admin |
| evt_002  | pat_123    | DIAGNOSIS_ADDED    | {ICD: J45}| dr_456|
| evt_003  | pat_123    | MED_PRESCRIBED     | {rxnorm}  | dr_456|
| evt_004  | pat_123    | LAB_RESULT_ADDED   | {HbA1c}   | lab_1 |
| evt_005  | pat_123    | DIAGNOSIS_UPDATED  | {ICD: J45}| dr_789|
+----------+------------+-------------------+------------+-------+

Current state = Replay all events
Point-in-time state = Replay events up to that timestamp
Complete audit trail = Built-in (every change is an event)
```

```python
class EHRService:

    async def append_medical_event(self, patient_id, event_type, event_data, actor_id):
        """
        Medical record event append -- never delete/update
        """
        await self.verify_access(actor_id, patient_id, event_type)

        event = {
            "event_id": str(uuid.uuid4()),
            "patient_id": patient_id,
            "event_type": event_type,
            "event_data": event_data,
            "actor_id": actor_id,
            "created_at": datetime.utcnow().isoformat(),
            "checksum": self.compute_checksum(event_data)  # Integrity verification
        }

        # Encrypt PHI fields before storing
        encrypted_event = self.encrypt_phi_fields(event)
        await self.event_store.append(encrypted_event)

        # Update read model (current state snapshot)
        await self.update_patient_snapshot(patient_id, event)

        # Audit log
        await self.audit_service.log_access(
            actor_id=actor_id, patient_id=patient_id,
            action=f"EHR_WRITE:{event_type}", timestamp=datetime.utcnow()
        )
        return event["event_id"]

    async def get_patient_record(self, patient_id, requesting_actor_id, as_of=None):
        """
        Patient record retrieve
        as_of: Point-in-time query (historical state)
        """
        await self.verify_read_access(requesting_actor_id, patient_id)

        if as_of:
            events = await self.event_store.get_events_before(patient_id, as_of)
            record = self.replay_events(events)
        else:
            record = await self.snapshot_store.get(patient_id)

        await self.audit_service.log_access(
            actor_id=requesting_actor_id, patient_id=patient_id,
            action="EHR_READ", fields_accessed=list(record.keys())
        )
        return self.decrypt_phi_fields(record, requesting_actor_id)
```

---

## ၄၈.၅ Telemedicine: WebRTC Video

WebRTC (Web Real-Time Communication) ကို telemedicine video consultation အတွက် အသုံးပြုသည်။ Peer-to-peer connection ဖြင့် low latency video streaming ပေးနိုင်ပြီး firewall ကြောင့် P2P မရပါက TURN server ဖြင့် media relay ပြုလုပ်သည်။

```
Doctor App                                        Patient App
    |                                                  |
    |---------> Signaling Server <--------------------|
    |           (WebSocket)                            |
    |                                                  |
    |<--- SDP Offer/Answer (session negotiation) ----->|
    |<--- ICE Candidates (network path discovery) ---->|
    |                                                  |
    |<============= WebRTC P2P Connection ============>|
    |              (Audio/Video Stream)                |
    |                                                  |
    If P2P fails (firewall):
    |<--------- TURN Server (media relay) ----------->|
```

```python
class TelemedicineService:

    async def create_consultation_session(self, appointment_id):
        appointment = await self.appointment_service.get(appointment_id)

        session_token = self.generate_session_token(
            appointment_id=appointment_id, expires_in_minutes=60
        )
        turn_credentials = self.turn_service.generate_credentials(
            username=f"session_{appointment_id}", ttl_seconds=3600
        )

        session = await self.session_db.create({
            "session_id": str(uuid.uuid4()),
            "appointment_id": appointment_id,
            "doctor_id": appointment.doctor_id,
            "patient_id": appointment.patient_id,
            "status": "CREATED",
            "turn_servers": turn_credentials,
            "expires_at": datetime.utcnow() + timedelta(hours=1)
        })

        await self.audit_service.log_session_created(session.session_id)

        return {
            "session_id": session.session_id,
            "session_token": session_token,
            "turn_servers": turn_credentials,
            "signaling_url": f"wss://signaling.hospital.com/session/{session.session_id}"
        }
```

---

## ၄၈.၆ Drug Interaction Check (gRPC)

Prescription ထုတ်ပေးချိန်တွင် drug-drug interactions ကို **synchronously** check ပြုရသည်။ Patient safety ၏ critical path ဖြစ်သောကြောင့် async ဖြင့် မဆောင်ရွက်ရ။

```
Doctor writes prescription
         |
         v (synchronous gRPC call)
+--------------------+
| Drug Interaction   |
| Check Service      |
+--------------------+
         |
    Database: FDA drug interaction tables
         |
         +-- No interactions -> Allow prescription
         +-- Mild -> Warning, Doctor confirm required
         +-- Severe -> Block prescription, Alert
```

```python
# gRPC service definition (proto)
"""
service DrugInteractionService {
    rpc CheckInteractions(CheckRequest) returns (CheckResponse);
}
message Interaction {
    string drug_a = 1;
    string drug_b = 2;
    string severity = 3;   // MILD, MODERATE, SEVERE, CONTRAINDICATED
    string description = 4;
    string clinical_evidence = 5;
}
"""

class PrescriptionService:

    async def create_prescription(self, doctor_id, patient_id, medications):
        current_meds = await self.ehr_service.get_active_medications(patient_id)

        # Synchronous gRPC -- patient safety critical path
        interaction_result = await self.drug_interaction_service.check_interactions(
            patient_id=patient_id,
            new_drug_codes=[m.rxnorm_code for m in medications]
        )

        if interaction_result.has_interactions:
            severe = [i for i in interaction_result.interactions
                     if i.severity in ["SEVERE", "CONTRAINDICATED"]]
            if severe:
                raise DrugInteractionError(
                    "Prescription blocked: Severe interaction", interactions=severe
                )
            else:
                await self.notify_doctor_warning(doctor_id, interaction_result.interactions)

        prescription = await self.db.create({
            "prescription_id": str(uuid.uuid4()),
            "patient_id": patient_id,
            "prescriber_id": doctor_id,
            "medications": medications,
            "status": "ACTIVE",
            "issued_at": datetime.utcnow(),
            "digital_signature": self.sign_prescription(doctor_id, medications)
        })

        await self.ehr_service.append_medical_event(
            patient_id=patient_id, event_type="MED_PRESCRIBED",
            event_data={"prescription_id": prescription.id, "medications": medications},
            actor_id=doctor_id
        )
        return prescription
```

---

## ၄၈.၇ Claims & Insurance Billing (Saga)

Insurance billing process တွင် multiple external systems (insurer APIs) ပါဝင်ပြီး adjudication response သည် hours to days ကြာနိုင်သောကြောင့် Saga pattern ကို အသုံးပြုသည်။

```
Billing Saga Steps:
Step 1: Create claim record (local DB)
Step 2: Validate claim (CPT codes, ICD codes)
Step 3: Submit to insurance (external API, X12_837P HIPAA EDI format)
Step 4: Wait for adjudication response (hours to days)
Step 5: Record payment/denial
Step 6: Generate patient statement

Compensating Transactions:
Step 3 fails -> Mark SUBMISSION_FAILED -> Retry or manual review
Step 4 timeout -> PENDING_ADJUDICATION -> Poll/webhook
Step 5 denial -> Create appeal workflow or patient billing
```

```python
class BillingSaga:

    async def process_claim(self, claim_id):
        claim = await self.claim_repo.get(claim_id)

        try:
            await self.validate_claim(claim)
            await self.claim_repo.update_status(claim_id, "VALIDATED")

            insurer_response = await self.insurer_gateway.submit_claim(
                claim=claim, format="X12_837P"  # HIPAA EDI format
            )
            await self.claim_repo.update(claim_id, {
                "status": "SUBMITTED",
                "insurer_claim_id": insurer_response.claim_id,
                "submitted_at": datetime.utcnow()
            })

            await self.scheduler.schedule_check(
                claim_id=claim_id,
                check_at=datetime.utcnow() + timedelta(days=3)
            )

        except ClaimValidationError as e:
            await self.claim_repo.update_status(claim_id, "VALIDATION_FAILED")
            await self.notify_billing_team(claim_id, e)

        except InsurerSubmissionError as e:
            await self.claim_repo.update_status(claim_id, "SUBMISSION_FAILED")
            await self.retry_queue.add(claim_id, retry_after_hours=24)
```

---

## ၄၈.၈ Consent Management

HIPAA ၏ "Notice of Privacy Practices" requirement အရ patient consent ကို immutable log ဖြင့် record ပြုရပြီး role + purpose + data type combination ပေါ်မူတည်ပြီး access control enforce ပြုရသည်။

```python
class ConsentManagementService:

    async def record_consent(self, patient_id, consent_type, granted, actor_id):
        """
        Consent types: TREATMENT, PAYMENT, OPERATIONS, RESEARCH,
                       MARKETING, DATA_SHARING, TELEMEDICINE
        """
        consent_event = {
            "consent_id": str(uuid.uuid4()),
            "patient_id": patient_id,
            "consent_type": consent_type,
            "granted": granted,
            "actor_id": actor_id,
            "ip_address": self.get_ip(),
            "timestamp": datetime.utcnow().isoformat(),
            "version": "2.0"
        }
        await self.consent_db.append(consent_event)  # Append-only log

    async def check_access_permission(self, requestor_id, patient_id, data_type, purpose):
        """
        PHI access permission check -- Minimum Necessary principle
        """
        requestor = await self.staff_service.get(requestor_id)

        if requestor.role == "TREATING_PHYSICIAN" and purpose == "TREATMENT":
            pass  # Broad access for treatment
        elif requestor.role == "RESEARCHER" and purpose == "RESEARCH":
            consent = await self.get_consent(patient_id, "RESEARCH")
            if not consent or not consent.granted:
                raise AccessDeniedError("No research consent")
        elif requestor.role == "BILLING_STAFF" and purpose == "PAYMENT":
            if data_type not in ["DEMOGRAPHICS", "INSURANCE", "PROCEDURES"]:
                raise AccessDeniedError(f"Billing cannot access {data_type}")
        else:
            raise AccessDeniedError("Insufficient permissions")

        await self.audit_service.log_access(
            requestor_id=requestor_id, patient_id=patient_id,
            data_type=data_type, purpose=purpose, granted=True
        )
```

---

## ၄၈.၉ Architecture Diagram

```
+------------------------------------------------------------------+
|                      Healthcare Platform                         |
+------------------------------------------------------------------+
                              |
                    +---------+---------+
                    |   API Gateway     |
                    | (TLS 1.3 only)    |
                    +---------+---------+
                    |         |         |
              Patient Auth  Provider  Billing
              Portal        Portal    Portal
                    |
        +-----------+----------+----------+
        |           |          |          |
        v           v          v          v
  +----------+ +---------+ +------+ +----------+
  | Patient  | |   EHR   | |Appt  | |Prescriptn|
  | Service  | | Service | |Svc   | | Service  |
  +----------+ +---------+ +------+ +----------+
        |           |                    |
        |    Event Store          Drug Interaction
        |    (Append-Only)        Service (gRPC)
        |
  +----------+
  |  Audit   |  (Immutable, 6+ years retention)
  |  Service |
  +----------+
        |
  +-----------+
  | Compliance|
  | Reports   |
  +-----------+

  Telemedicine:
  +----------+   WebSocket   +----------+   WebRTC   +----------+
  |  Doctor  | <-----------> | Signaling| <--------> | Patient  |
  |  App     |               | Server   |            | App      |
  +----------+               +----------+            +----------+
                                    |
                             TURN Server (if P2P fails)
```

---

## အဓိကသင်ခန်းစာများ

1. **HIPAA Compliance by Design:** PHI encryption (at rest + in transit), RBAC, complete audit logs ကို architecture ၏ core component ဖြစ်အောင် design ရသည်
2. **Event Sourcing for Medical Records:** Append-only event log သည် complete audit trail, point-in-time queries, legal defensibility ကို automatically ဖြစ်ပေါ်စေသည်
3. **FHIR Interoperability:** FHIR standard implement ခြင်းသည် hospitals, pharmacies, insurance systems တို့နှင့် seamless data exchange ဖြစ်စေသည်
4. **Drug Interaction = Synchronous gRPC:** Patient safety critical path တွင် async processing မသုံးရ
5. **Saga for Billing:** Insurance claims (days ကြာနိုင်) ကို Saga pattern ဖြင့် long-running distributed transaction manage လုပ်သည်
6. **Minimum Necessary Principle:** Role + purpose + data type combination ပေါ်မူတည်ပြီး only necessary data ကိုသာ access ခွင့်ပြုသည်
