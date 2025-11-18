# User Story Template

## Context
This is my format of standard user story, plus a check-list ot refine the story so it can answer the questions that team might need in order to develop, test, and deliver the related value.

---

## User Story Format

### Story ID: [JIRA-XXX]

**Title:** [Action-oriented, clear title - e.g., "Enable Multi-Factor Authentication for User Login"]

---

### User Story Statement

**As a** [specific role/persona - e.g., "Corporate Banking Customer"]  
**I want** [specific capability - e.g., "to authenticate using SMS code in addition to password"]  
**So that** [clear business value - e.g., "my account is protected from unauthorized access"]

---

### Business Context

**Why This Matters:**
[1-2 sentences explaining the business need or problem being solved]

**Priority:** [High/Medium/Low]  
**Business Value:** [Revenue impact, risk mitigation, user satisfaction, operational efficiency]

---

## Acceptance Criteria

Stories must have clear, testable acceptance criteria. I use Given-When-Then format for scenarios:

### Scenario 1: [Happy Path - Primary Flow]

**Given** [initial state/context - be specific about data setup]  
**When** [user action or system trigger]  
**Then** [expected outcome - specific and measurable]

**Example:**
```
Given I am a registered user with verified phone number
When I enter correct username and password on login screen
Then the system sends a 6-digit SMS code to my registered phone number
And I am presented with MFA code entry screen
And I have 5 minutes to enter the code before it expires
```

### Scenario 2: [Alternative Flow]

**Given** [different starting condition]  
**When** [different action]  
**Then** [different expected outcome]

### Scenario 3: [Error/Exception Handling]

**Given** [error condition setup]  
**When** [action triggering error]  
**Then** [system should handle gracefully]

---

### Functional Requirements

Beyond scenarios, include specific functional criteria:

- [ ] System shall [specific requirement with measurable criteria]
- [ ] User interface shall [specific UI requirement]
- [ ] System shall validate [specific validation rule]
- [ ] Error messages shall [specific error handling requirement]

**Example:**
- [ ] SMS code must be exactly 6 numeric digits
- [ ] Code expires after 5 minutes
- [ ] Maximum 3 failed attempts before account temporary lock (15 minutes)
- [ ] User can request new code with 30-second cooldown between requests

---

### Non-Functional Requirements

Performance, security, compliance, and other quality attributes:

- [ ] **Performance:** [Response time requirement]
- [ ] **Security:** [Security/encryption requirement]
- [ ] **Compliance:** [Regulatory requirement]
- [ ] **Usability:** [User experience requirement]
- [ ] **Accessibility:** [WCAG compliance level if applicable]

**Example:**
- [ ] SMS delivery within 30 seconds (95th percentile)
- [ ] Codes must be cryptographically secure random numbers
- [ ] GDPR compliant - user can opt out of SMS MFA
- [ ] Screen reader compatible for vision-impaired users

---

## Technical Details

### API Endpoints Required

**New Endpoints:**
```
POST /api/auth/mfa/send-code
POST /api/auth/mfa/verify-code
GET /api/auth/mfa/status
```

**Modified Endpoints:**
```
POST /api/auth/login - Add MFA flow handling
```

### Data Model Changes

**New Tables/Collections:**
```sql
MFA_Codes (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES Users(id),
  code_hash VARCHAR(255),
  created_at TIMESTAMP,
  expires_at TIMESTAMP,
  attempts INT DEFAULT 0,
  verified BOOLEAN DEFAULT FALSE
)
```

**Modified Tables:**
```sql
Users - Add columns:
  - mfa_enabled BOOLEAN DEFAULT FALSE
  - phone_verified BOOLEAN DEFAULT FALSE
  - phone_number VARCHAR(20)
```

### Integration Requirements

**External Services:**
- SMS Gateway API (Twilio/AWS SNS) for code delivery
- Audit logging service for security events

**Dependencies:**
- User management service must be available
- SMS service must have fallback for delivery failures

---

## Business Rules

Document all business logic and decision rules:

1. **MFA Enrollment:**
   - Users must verify phone number before enabling MFA
   - MFA is optional for standard users, mandatory for admin users
   
2. **Code Generation:**
   - Codes are single-use only
   - New code invalidates previous unused codes
   - Codes cannot be reused even if expired

3. **Account Security:**
   - After 3 failed MFA attempts, account locked for 15 minutes
   - After 5 failed attempts in 24 hours, manual unlock required
   - Security team notified of repeated failed attempts (possible attack)

4. **Edge Cases:**
   - If user changes phone number, MFA is temporarily disabled until new number verified
   - System admin can disable MFA for user in emergency (with audit trail)

---

## UI/UX Requirements

### Wireframes/Mockups
[Link to Figma/design files or embed images]

### User Flow
```
Login Screen → 
  Enter Credentials → 
    MFA Prompt → 
      Enter Code → 
        Success/Failure → 
          Dashboard OR Error Message
```

### Copy/Content
- Button text: "Send Code"
- Helper text: "Enter the 6-digit code sent to your phone ending in [XX]"
- Error messages:
  - "Invalid code. Please try again." (for incorrect code)
  - "Code expired. Please request a new code." (for expired)
  - "Too many failed attempts. Account locked for 15 minutes." (for lockout)

---

## Testing Considerations

### Test Scenarios to Cover
- Happy path with valid code
- Expired code handling
- Invalid code handling (wrong digits)
- Maximum attempts exceeded
- Code reuse prevention
- SMS delivery failure handling
- Network timeout scenarios
- Concurrent login attempts

### Test Data Needed
- Test phone numbers that don't actually send SMS (for automation)
- Pre-seeded test codes for consistent testing
- Test users with various MFA states (enabled/disabled/locked)

---

## Definition of Ready

This story is ready for development when:

- [ ] Acceptance criteria are clear and testable
- [ ] API contract defined and agreed with backend team
- [ ] Data model changes reviewed by DBA
- [ ] UI mockups approved by design team
- [ ] Security review completed
- [ ] Dependencies identified and teams notified
- [ ] Story points estimated by development team
- [ ] No blocking questions remain

---

## Definition of Done

This story is complete when:

- [ ] Code written and peer reviewed
- [ ] Unit tests written (>80% code coverage)
- [ ] Integration tests passing
- [ ] API documentation updated (Swagger/OpenAPI)
- [ ] Security testing completed (penetration test if needed)
- [ ] Performance testing meets SLA (30s SMS delivery)
- [ ] Acceptance criteria validated by QA
- [ ] User documentation created/updated
- [ ] Product Owner approval received
- [ ] Deployed to staging environment
- [ ] Ready for production deployment

---

## Story Estimation

**Complexity:** [Story Points: 5 / T-Shirt Size: M]

**Estimation Rationale:**
- Backend work: SMS integration + code generation/validation (2 days)
- Frontend work: MFA flow screens + error handling (1.5 days)
- Testing: Unit + integration + security (1 day)
- Total: ~4.5 days = 5 story points

**Risks/Uncertainties:**
- SMS gateway API might have rate limits not yet understood
- Phone number format validation rules vary by country
- Recommend spike story if international support needed

---

## Dependencies & Blockers

**Dependencies:**
- ✅ SMS gateway vendor selected and contract signed
- ⚠️ Phone number verification feature must be completed first (Story JIRA-XXX)
- ⚠️ Waiting on security team review of cryptographic requirements

**Blocking Issues:**
- None currently

**Related Stories:**
- JIRA-XXY: Phone Number Verification
- JIRA-XXZ: Account Lockout Notifications
- JIRA-XYZ: MFA Recovery Process

---

## Notes & Open Questions

**Decisions Made:**
- ✅ Decision: Use SMS instead of authenticator app for MVP (faster user adoption)
- ✅ Decision: 6-digit codes (balance between security and usability)
- ✅ Decision: 5-minute expiry (industry standard)

**Open Questions:**
- ❓ Should we support international phone numbers in MVP? (Needs PM clarification)
- ❓ What happens if user loses phone access? (Recovery process needed - separate story)
- ❓ Do we need rate limiting on "resend code" to prevent SMS bombing? (Needs security review)

**Assumptions:**
- 🔔 ASSUMPTION: Users have reliable SMS reception
- 🔔 ASSUMPTION: SMS costs are acceptable for expected volume
- 🔔 ASSUMPTION: 5-minute expiry is sufficient for most use cases

---

## Audit & Compliance

**Logging Requirements:**
- Log all MFA code generation events (without logging actual code)
- Log all verification attempts (success and failure)
- Log account lockout events
- Retain logs for 90 days minimum (compliance requirement)

**Privacy Considerations:**
- Phone numbers are PII - must be encrypted at rest
- SMS codes must not appear in any logs
- Users must consent to SMS communication (GDPR)

**Compliance:**
- GDPR: Right to delete - must include MFA data
- PSD2 (if applicable): Strong Customer Authentication requirements
- SOC2: Audit trail of all authentication events

---

## Success Metrics

How we'll measure if this story delivers value:

**Key Metrics:**
- Successful MFA completion rate (target: >95%)
- Average time to complete MFA flow (target: <45 seconds)
- Reduction in unauthorized access attempts (baseline vs. post-MFA)
- User support tickets related to MFA (monitor for usability issues)

**Business KPIs:**
- Account takeover incidents (expect reduction)
- User churn due to security concerns (expect reduction)
- Compliance audit findings (expect improvement)

---

## Attachments

- [Link to API specification document]
- [Link to security review findings]
- [Link to UI mockups in Figma]
- [Link to related technical architecture diagram]

---

## Story History

**Created:** [Date] by [BA Name]  
**Last Updated:** [Date] by [Name]  
**Status:** [Ready for Development / In Progress / Done]  
**Sprint:** [Sprint 23]

**Change Log:**
- [Date]: Updated acceptance criteria based on security review feedback
- [Date]: Added performance requirements after infrastructure team input
- [Date]: Clarified international phone number scope (out of MVP)

---

## Template Usage Notes

**When to use this template:**
- All user-facing features requiring new functionality
- Complex backend features with clear user value
- Features requiring integration with external systems

**When to use lighter format:**
- Simple bug fixes (can use simplified version)
- Minor UI copy changes (acceptance criteria only)
- Technical debt stories (focus on technical details)

**Customization:**
- Remove sections not applicable to your story
- Add domain-specific sections as needed
- Adjust detail level based on team maturity and story complexity

---

*This template represents 10+ years of refinement across Agile teams in financial services, blockchain, and enterprise software. It ensures stories are complete, testable, and provide clear value while maintaining flexibility for different contexts.*

---

**Version:** 2.0  
**Last Updated:** January 2025  
**Author:** Oleh Syzonov, Senior Business Analyst
