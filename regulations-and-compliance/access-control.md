# Access Control

## What Access Control Is and Why It Matters

Access control determines who can access what resources and under what conditions. For banking GenAI systems, access control is the primary defense against unauthorized data access, model manipulation, and system compromise.

**Regulatory requirements**:
- **GDPR** Article 32: Security of processing (access control requirement)
- **PCI-DSS** Requirements 7-8: Restrict access, identify users
- **SOC 2** CC6: Logical and physical access controls
- **ISO 27001** A.5.15-A.5.18: Access control, identity management
- **SR 11-7**: Model access restrictions
- **BCBS 239**: Access controls for risk data

## Access Control Models

### RBAC (Role-Based Access Control)

Access is granted based on the user's role within the organization.

```
RBAC IMPLEMENTATION:
├── Roles defined by job function
├── Permissions assigned to roles, not individuals
├── Users assigned to roles
├── Regular role review and recertification
├── Separation of duties (SoD) enforced
└── Principle of least privilege applied

BANKING ROLES EXAMPLE:
├── customer_service_agent:
│   ├── View customer profile (masked PII)
│   ├── View account balances
│   ├── View transaction history (masked)
│   ├── Access AI chatbot for customer inquiries
│   └── Create service tickets
├── fraud_analyst:
│   ├── View customer profile (full PII with audit)
│   ├── View flagged transactions
│   ├── Access fraud detection AI tools
│   ├── Create fraud investigation cases
│   └── Freeze/unfreeze accounts
├── ai_engineer:
│   ├── Access AI model code repositories
│   ├── Access AI development environment
│   ├── View model performance metrics
│   ├── NO access to production customer data
│   └── NO access to production PII
├── model_validator:
│   ├── Access model documentation
│   ├── Access validation environment
│   ├── Access model performance data
│   ├── NO access to modify production models
│   └── Independent of model development team
├── compliance_officer:
│   ├── View all compliance reports
│   ├── Access audit logs
│   ├── Access regulatory submissions
│   └── View access control configurations
└── system_administrator:
    ├── Access system infrastructure
    ├── Manage user accounts
    ├── View system logs
    ├── NO access to customer data content
    └── All actions logged and monitored
```

### ABAC (Attribute-Based Access Control)

Access is granted based on attributes of the user, resource, action, and environment.

```
ABAC IMPLEMENTATION:
├── User attributes: role, department, clearance level, location
├── Resource attributes: classification, owner, sensitivity, jurisdiction
├── Action attributes: read, write, delete, export, share
├── Environment attributes: time, location, device, network
├── Policy rules combine attributes to make access decisions
└── More granular than RBAC, but more complex to manage

ABAC POLICY EXAMPLES:
├── "Users with role 'fraud_analyst' AND clearance 'high'
│    CAN access PII for customers in their assigned cases
│    ONLY during business hours
│    ONLY from bank-managed devices
│    WITH full audit logging"
├── "Users with role 'ai_engineer'
│    CANNOT access production customer data
│    CAN access anonymized training data
│    ONLY from development network
│    WITH code review approval"
└── "Users CANNOT export more than 1000 customer records
│    WITHOUT additional approval from data owner
│    REGARDLESS of role"
```

### PBAC (Policy-Based Access Control)

Combines RBAC and ABAC with organizational policies.

```
PBAC DECISION FLOW:
┌───────────┐    ┌───────────┐    ┌───────────┐
│   USER    │    │ RESOURCE  │    │  CONTEXT  │
│ Request   │    │ Metadata  │    │ Metadata  │
└─────┬─────┘    └─────┬─────┘    └─────┬─────┘
      │                │                │
      └────────────────┼────────────────┘
                       │
              ┌────────┴────────┐
              │  Policy Engine  │
              │  (XACML, OPA)   │
              └────────┬────────┘
                       │
              ┌────────┴────────┐
              │  Decision:      │
              │  Permit/Deny/  │
              │  Obligation     │
              └─────────────────┘
```

## Privileged Access Management (PAM)

Privileged accounts (administrators, root, service accounts) have elevated access and require additional controls.

```
PAM REQUIREMENTS:
├── Privileged accounts identified and inventoried
├── Privileged access requested through ticketing system
├── Just-in-time (JIT) access (not standing privileges)
├── Time-limited access (automatic expiration)
├── Multi-factor authentication required
├── Session recording for all privileged sessions
├── Command-level logging
├── Approval workflow for privileged access requests
├── Emergency access (break-glass) procedures
│   ├── Limited number of break-glass accounts
│   ├── Stored in secure vault with dual control
│   ├── Alarm triggered on use
│   └── Post-use review mandatory
├── Regular privileged access review
│   ├── Who has privileged access?
│   ├── Is it still needed?
│   └── Has it been used appropriately?
└── Automated deprovisioning when no longer needed
```

### Break-Glass Access

```
BREAK-GLASS ACCESS PROCESS:
├── Trigger: Emergency situation requiring immediate privileged access
├── Access request:
│   ├── User requests break-glass access
│   ├── Reason documented
│   ├── Estimated duration provided
│   └── Approval from on-call manager (if possible)
├── Access granted:
│   ├── Credentials retrieved from secure vault
│   ├── Alarm triggered to security team
│   ├── Session recorded
│   └── Time limit set (e.g., 4 hours)
├── Access used:
│   ├── All commands logged
│   ├── All actions tracked
│   └── Access auto-terminated at time limit
├── Post-use review:
│   ├── Session recording reviewed by security
│   ├── Actions documented
│   ├── Any changes tracked
│   └── Findings reported to management
└── Credentials rotated after use
```

## Access Control for AI Systems

### Model Access Controls

```
MODEL ACCESS MATRIX:
┌────────────────────┬──────────┬──────────┬──────────┬──────────┐
│ Action             │ Dev      │ Validator│ Business │ Admin    │
├────────────────────┼──────────┼──────────┼──────────┼──────────┤
│ Train model        │ Yes      │ No       │ No       │ No       │
│ Deploy model       │ No*      │ No       │ No       │ Yes      │
│ Modify model       │ No*      │ No       │ No       │ No       │
│ View model output  │ Yes      │ Yes      │ Yes      │ Yes      │
│ View model metrics │ Yes      │ Yes      │ Yes      │ Yes      │
│ View training data │ No**     │ Yes      │ No       │ No       │
│ Export model       │ No       │ No       │ No       │ Yes      │
│ Delete model       │ No       │ No       │ No       │ Yes      │
└────────────────────┴──────────┴──────────┴──────────┴──────────┘

* Through approved change management process
** Anonymized/aggregated training data only
```

### AI Service Access Controls

```
AI SERVICE ACCESS:
├── API access:
│   ├── Authentication required for all API calls
│   ├── API keys rotated regularly
│   ├── Rate limiting per user/service
│   ├── Quota management
│   └── Usage logging and monitoring
├── Model inference:
│   ├── Input validation before model processing
│   ├── Output access based on user role
│   ├── PII in inputs handled per classification
│   └── Results logged for audit
├── Training pipeline:
│   ├── Access restricted to authorized data scientists
│   ├── Training data access logged
│   ├── Model artifacts access-controlled
│   └── Training environment isolated from production
└── Vector database:
    ├── Collection-level access controls
    ├── Query access based on user role
    ├── Index modification requires elevated privilege
    └── Embedding access logged
```

## Identity and Access Management (IAM)

```
IAM LIFECYCLE:
├── Joiner:
│   ├── Account created based on HR system
│   ├── Role assigned based on job function
│   ├── Access granted per role definition
│   ├── MFA enrollment required
│   └── Security awareness training required
├── Mover:
│   ├── Role changes when job changes
│   ├── Old access revoked, new access granted
│   ├── Previous access reviewed
│   └── Manager approval for role change
├── Leaver:
│   ├── Access disabled immediately on termination
│   ├── Access removed within 24 hours
│   ├── Credentials revoked
│   ├── Physical access badges returned
│   └── Exit interview includes access confirmation
└── Reviewer:
    ├── Regular access certification
    ├── Managers review team access quarterly
    ├── Data owners review data access quarterly
    ├── Privileged access reviewed monthly
    └── Findings remediated within SLA
```

## Access Logging and Monitoring

```
ACCESS LOGGING REQUIREMENTS:
├── All authentication events (success and failure)
├── All authorization decisions (allow and deny)
├── All privileged access sessions
├── All access to PII
├── All access to production systems
├── All access to model training data
├── All model deployment actions
├── All configuration changes
└── All emergency (break-glass) access

ACCESS MONITORING RULES:
├── Failed login attempts (>5 in 5 minutes)
├── Login from unusual location
├── Login outside normal hours
├── Access to data outside user's role
├── Bulk data access
├── Access from new device
├── Privileged account used without ticket
├── Break-glass account accessed
├── Access to dormant account
└── Concurrent sessions from different locations
```

## Common Interview Questions

### Question 1: "How do you implement least privilege for an AI engineering team?"

**Good answer structure**:
AI engineers need access to model code, development environments, and testing data -- but should NOT have access to production customer data or production model deployment. Specifically: (1) Development environment with synthetic or anonymized data only. (2) Code repository access for model code, not production configuration. (3) Model performance metrics access. (4) No direct production database access. (5) If production data access is needed for debugging, use just-in-time access with approval, time limits, and full logging. (6) Production model deployment is done through CI/CD pipeline with separate approval, not by engineers directly.

### Question 2: "How do you handle access control for a shared AI service used by multiple business units?"

**Good answer structure**:
I would implement ABAC or PBAC: (1) Each business unit has its own role and access scope. (2) Data is segregated by business unit -- Unit A cannot see Unit B's data. (3) Usage is tracked and billed per business unit. (4) Common AI capabilities (base model access) are shared, but customer-specific data and fine-tuned models are isolated. (5) Access to the AI service itself requires authentication, and access to specific capabilities requires authorization. (6) All usage is logged with business unit context for chargeback and audit purposes.

### Question 3: "What access control model would you recommend for a banking GenAI platform?"

**Good answer**:
A hybrid approach: RBAC for the foundation (clear role definitions, easy to manage and audit), augmented with ABAC for fine-grained control (contextual access based on data sensitivity, time, location). RBAC gives you the structure that auditors expect -- clear roles, regular access reviews, separation of duties. ABAC gives you the flexibility needed for complex banking scenarios -- "this fraud analyst can access this customer's PII only during their assigned case, from a bank device, during business hours." I would implement RBAC as the primary model with ABAC policies layered on top for specific high-sensitivity scenarios.
