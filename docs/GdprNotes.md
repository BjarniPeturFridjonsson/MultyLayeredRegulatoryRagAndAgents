# GDPR Notes

Working notes for review by club leadership. Not part of the technical platform documentation.

---

## Overall Position

The platform's privacy posture is strong by design. No member personal data is sent to external AI services. All AI processing (Claude, bge-m3) operates exclusively on regulatory documents and club operational content. Member data stays within the platform. This is a deliberate architectural decision and should be stated explicitly in the privacy policy.

---

## Cookies and Tracking

The existing cookie policy is correct and sufficient. Session cookies used solely for authentication are **strictly necessary** and explicitly exempt from consent requirements under the ePrivacy Directive. No consent banner is required.

The existing Icelandic policy text accurately describes the situation: security-only cookies, no analytics, no third-party cookies, no remarketing. An English translation alongside it would be good practice for clubs with non-Icelandic members.

---

## Flight Logs - Retention and Erasure

Flight logs must be retained by law (EASA and national CAA regulatory requirements). A member's right to erasure under Article 17 does not apply to data where retention is required by a legal obligation, Article 17(3)(b) explicitly covers this.

**Position:** Flight records are retained in full for the legally required period. No anonymisation required. Members should be informed of this in the privacy policy.

**Action needed:** Confirm the exact retention period required under Icelandic CAA rules and document it.

---

## Data Deletion Policy - Non-Flight Data

For personal data not subject to a legal retention requirement, a clear deletion policy is needed.

**Data in scope for deletion:**
- Logbook texts (personal narratives)
- Uploaded images
- Medical certificates (once the medical reader feature is built)
- License number
- Agent memory entries (`member_agent_memory` conversation summaries and extracted preferences per agent)

**Proposed policy:**
- Active members: retain while membership is active
- Resigned / lapsed members: retain for a defined period after membership ends (recommend 2 years, sufficient for any outstanding queries or disputes, short enough to be proportionate)
- After retention period: delete

**Action needed:** Club leadership to agree on the post-membership retention period and document it formally.

---

## Right of Access - Article 15

Members have the right to request a copy of all personal data held on them. Response required within one month of request (extendable by two further months for complex cases, but member must be notified within the first month).

**Proposed process:**
- Member contacts club administrator by email with a specific request
- Administrator verifies identity before releasing any data
- Administrator generates a complete member data export report (see below)
- Report delivered to member within the one-month window
- Request and response logged internally (date received, date responded, who handled it)

Admin-mediated access is legally sufficient. It is also more secure than self-service. Identity is verified by a human, and there is a clear audit trail.

**The process must be documented in the privacy policy**  members need to know how to make a request and who to contact.

---

## Member Data Export Report

To fulfil Article 15 requests efficiently, a single admin action should generate a complete export of all data held on a specific member. If data is scattered across multiple screens, compilation becomes slow and error-prone.

**Report should include:**
- Personal details (name, contact, membership dates)
- Flight records
- Logbook entries and texts
- Uploaded images (list with download links or bundled archive)
- Certifications and ratings
- Medical certificate data (once built)
- License number
- Any flags or admin notes associated with the member
- Agent memory entries (per-agent memory store: preferences, summaries, weak topics)

**This is a platform development task**  a dedicated admin report endpoint that produces a complete, downloadable member data package on demand.

---

## Medical Certificate Data - Special Category

Medical data is a special category under GDPR Article 9 and requires stricter handling than general member data.

**Before the medical certificate reader feature is built:**
- Define the legal basis for processing (likely legitimate interest for aviation safety that must be documented)
- Explicit member consent on upload
- Access restricted to head of instructors and above
- Separate retention policy (recommend: delete raw images after approval, retain structured fields for duration of medical validity plus a defined period after expiry)
- Document all of the above formally before the feature goes live

---

## AI Processing - Clean Position

No member personal data is used as input to any AI service (Claude API, embedding model, or any other external service). All AI operations process regulatory documents, operational content, and club knowledge, not personal member data.

This should be stated explicitly in the privacy policy. It is a genuine differentiator and a strong position for member trust.

---

## Summary of Actions

| Action | Owner | Priority |
|---|---|---|
| Confirm CAA flight log retention period | Club leadership / legal | High |
| Agree post-membership data retention period | Club leadership | High |
| Document Article 15 request process in privacy policy | Club leadership | High |
| Add English translation of cookie/privacy policy | Administrator | Medium |
| Build member data export report (admin tool) | Development | Medium |
| Document legal basis for medical data processing before feature goes live | Club leadership / legal | High (before medical feature) |
