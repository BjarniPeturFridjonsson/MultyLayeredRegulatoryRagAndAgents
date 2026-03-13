# AddOns - Planned Extensions

Standalone features that build on the core platform without changing its architecture.
Each is self-contained and can be implemented independently after the main build phases are complete.

---

## 1. Pilot Medical Certificate Reader

### What
Upload a scanned or photographed pilot medical certificate, extract the structured data automatically using Claude vision, and populate the existing member medical records in MSSQL. Reviewed and approved by the head of instructors before becoming active.

### Why
The EASA requires that the DTO (Declared Training Organisation) has to have validated up to date medical certificate records (The qualifications) of pilot instructors and students before first solo flight. 

### Why it fits
The C# platform already has the member data model, the medical certificate fields, and all relevant API endpoints. The only missing piece is the intake: getting data from a physical or scanned certificate into the database without manual re-keying. The HITL review pattern (already built for SIB verification) is a direct reuse.

### What is new
| Component | Work |
|---|---|
| MSSQL table - `member_medical_uploads` | Tracks raw uploads, extraction results, review status |
| Claude vision extraction | Single prompt, structured output: certificate number, class, issue date, expiry date, issuing authority, limitations |
| Upload endpoint (C# API) | Accepts image/PDF, stores to MinIO, triggers extraction job |
| Extraction worker (Python) | Calls Claude vision, writes structured fields + confidence scores to MSSQL |
| HITL review UI (SvelteKit) | Near-identical to SIB review: source image left, extracted fields right, approve/reject actions |
| Role gate | Review queue visible to head of instructors only |

### What is reused
- MinIO for raw document storage
- SIB extraction worker pattern (same pull-based job model)
- SIB HITL review UI component (same layout, different fields)
- Existing member medical data model and API endpoints in C# platform
- Cost tracking infrastructure (token usage logged per extraction)

### Format variation
EASA medical certificates vary by issuing country and class (LAPL, Class 2). The extraction prompt must handle layout differences gracefully and flag low-confidence extractions for human attention rather than guessing. Confidence score stored alongside each extracted field. Low-confidence fields highlighted in the review UI.

### GDPR note
Medical data is a special category under GDPR Article 9. Before implementing:
- Explicit member consent on upload required
- Access restricted to head of instructors and above
- Retention policy defined and enforced
- Raw images deleted from MinIO after approval (or per retention policy)

---
