# Game Days

## Overview

Game days are planned exercises where the team simulates incidents to test detection, response, and mitigation capabilities. Regular game days ensure the team is prepared for real incidents and reveal gaps in monitoring, runbooks, and processes before they matter.

This document covers game day planning, execution, and learning.

---

## Why Game Days Matter

1. **Practice Makes Prepared**: Teams that practice incident response respond faster and more effectively to real incidents.

2. **Reveal Gaps**: Game days expose missing alerts, outdated runbooks, and process gaps that would only be discovered during a real incident.

3. **Build Confidence**: Engineers who have practiced incident response are more confident and less panicked during real incidents.

4. **Test Systems**: Game days verify that monitoring, alerting, and mitigation mechanisms actually work.

5. **Onboard New Engineers**: Game days are an effective training tool for new team members.

---

## Game Day Types

### 1. Tabletop Exercise

**Format**: Discussion-based scenario walkthrough without actually triggering anything.

**Duration**: 60-90 minutes

**Participants**: 5-15 people

**Best for:**
- New IC training
- Process review
- Complex scenarios that are hard to simulate
- Regulatory incident scenarios (compliance team participation)

**Example scenario:**
> "It's Monday morning. Your monitoring system alerts you that the RAG vector database is returning errors for 60% of queries. Walk me through your response."

### 2. Technical Game Day

**Format**: Hands-on simulation in a staging environment with real tools.

**Duration**: 2-4 hours

**Participants**: 3-8 people (IC, SMEs, observer)

**Best for:**
- Testing runbooks
- Validating alerting
- Practicing mitigation actions
- New on-call engineer training

**Example scenario:**
> In the staging environment, a test deployment introduces a bug that causes the embedding pipeline to stall. The team must detect, investigate, and resolve the issue using real monitoring and deployment tools.

### 3. Chaos Engineering Game Day

**Format**: Controlled failure injection in production (or staging) to test system resilience.

**Duration**: Half-day to full day

**Participants**: 5-10 people (engineers, SREs, observers)

**Best for:**
- Testing system resilience
- Validating failover mechanisms
- Chaos engineering experiments
- Capacity planning validation

**Example experiments:**
- Kill a random pod in production
- Simulate vector database outage
- Introduce network latency between services
- Exhaust GPU memory on a node

### 4. Full-Scale SEV-1 Simulation

**Format**: End-to-end SEV-1 incident simulation with full war room, executive participation, and stakeholder communication.

**Duration**: 3-4 hours

**Participants**: 15-25 people (IC, SMEs, executives, comms, compliance)

**Frequency**: Quarterly

**Best for:**
- Testing the full incident management process
- Executive incident awareness
- Cross-functional coordination
- Regulatory notification practice

---

## Game Day Planning

### Planning Checklist

- [ ] **Select scenario**: Based on recent incidents, new systems, or known risks
- [ ] **Define objectives**: What are we testing? (detection, response, mitigation, communication)
- [ ] **Choose environment**: Staging or production (with safeguards)
- [ ] **Prepare infrastructure**: Set up failure injection, monitoring, observation tools
- [ ] **Write scenario brief**: Detailed scenario description with injection timeline
- [ ] **Assign roles**: IC, SMEs, observer, facilitator, safety officer
- [ ] **Notify stakeholders**: Inform affected teams (but not participants, if testing detection)
- [ ] **Prepare safety measures**: Abort criteria, rollback plan, blast radius containment
- [ ] **Schedule**: Book participants' time, reserve war room

### Scenario Selection

Prioritize scenarios based on:

1. **Recent incidents**: Test the same failure mode to verify fixes
2. **New systems**: Test newly launched features or infrastructure
3. **Known risks**: Test failure modes that have not been tested
4. **GenAI-specific**: Test AI-specific failure modes (prompt injection, model degradation, etc.)
5. **Regulatory**: Test scenarios with regulatory notification requirements

### Scenario Brief Template

```markdown
# Game Day Scenario Brief

## Scenario: [Name]

## Objectives
1. [What are we testing?]
2. [What do we expect to happen?]
3. [What would surprise us?]

## Setup
- [Environment: staging/production]
- [Pre-conditions]
- [Failure injection method]

## Injection Timeline
| Time | Action | Expected Effect |
|------|--------|----------------|
| T+0 | [Inject failure] | [Alert should fire] |
| T+5 | [Secondary effect] | [Additional symptom] |

## Success Criteria
- [ ] Detection within [X] minutes
- [ ] Correct severity classification
- [ ] IC assigned within [X] minutes
- [ ] Runbook followed
- [ ] Mitigation successful within [X] minutes
- [ ] Stakeholder communication sent

## Safety Measures
- [Abort criteria]
- [Rollback plan]
- [Blast radius containment]

## Observers
- [Names of people observing and what they should watch for]
```

---

## Game Day Execution

### Pre-Game Briefing (10 minutes)

> "Welcome to today's game day exercise. I'm [facilitator name], and I'll be running this session.
>
> Scenario: [Brief description]
> Objectives: [What we are testing]
> Rules of engagement: [What participants can and cannot do]
> Safety measures: [Abort criteria, rollback plan]
>
> [Safety officer], please confirm safety measures are in place.
>
> Let's begin."

### During the Game Day

**Facilitator:**
- Injects failures per the scenario timeline
- Observes participant actions and decisions
- Does not guide participants (let them make mistakes)
- Notes gaps and issues for the debrief

**Safety Officer:**
- Monitors blast radius (especially in production)
- Can abort the exercise if real customer impact occurs
- Confirms rollback capability before each injection

**Observer(s):**
- Documents what participants do and say
- Notes runbook accuracy, alert effectiveness, process adherence
- Does not participate or provide hints

**Participants:**
- Respond to the incident as if it were real
- Use real tools (monitoring, deployment, communication)
- Follow the incident management process

### Common Game Day Injections

| Injection | Method | Expected Response |
|-----------|--------|------------------|
| Pod failure | `kubectl delete pod` | Auto-restart, monitoring alert |
| High latency | `tc qdisc add dev eth0 delay 500ms` | Latency alert, investigation |
| Database failure | Stop database process | Service outage alert, failover |
| Vector DB outage | Block network to VDB | RAG failure alert, fallback |
| GPU exhaustion | Run memory-intensive process | GPU alert, pod eviction |
| Prompt injection | Send injection payload | Detection alert, containment |
| Token cost spike | Simulate high-volume traffic | Cost alert, investigation |

---

## Debrief

### Immediately After (15-20 minutes)

The facilitator leads a structured debrief:

1. **What happened?**
   - Walk through the timeline from the observer's notes
   - Compare actual events to expected events

2. **What went well?**
   - Detection worked within target time
   - Runbook was accurate and followed
   - Communication was timely

3. **What did not go well?**
   - Alert was missed or ignored
   - Runbook was outdated
   - Mitigation action was incorrect
   - Communication was delayed

4. **What did we learn?**
   - New failure mode discovered
   - Process gap identified
   - Tooling limitation revealed

5. **What will we change?**
   - Action items with owners and deadlines

### Debrief Template

```markdown
# Game Day Debrief: [Scenario Name]

**Date:** [Date]
**Facilitator:** [Name]
**Participants:** [Names]

## Scenario
[Brief description]

## Objectives vs. Results

| Objective | Expected | Actual | Pass/Fail |
|-----------|----------|--------|-----------|
| Detect within 5 min | 5 min | 3 min | PASS |
| Classify as SEV-2 | SEV-2 | SEV-3 | FAIL |
| Follow runbook | Runbook followed | Runbook outdated | FAIL |
| Mitigate within 30 min | 30 min | 25 min | PASS |

## What Went Well
1. [Observation]
2. [Observation]

## What Did Not Go Well
1. [Observation]
2. [Observation]

## Action Items
| # | Action | Owner | Deadline |
|---|--------|-------|----------|
| 1 | [Description] | [@person] | [Date] |
| 2 | [Description] | [@person] | [Date] |

## Lessons Learned
[Key takeaways]
```

---

## GenAI-Specific Game Day Scenarios

### Scenario 1: Prompt Injection Attack

**Setup**: Upload a document containing a prompt injection payload to the staging RAG system.

**Expected Response:**
1. Prompt injection detection alert fires
2. SEV-1 classified
3. IC assigned
4. Document analysis feature disabled
5. Scope assessment initiated

**What to Watch For:**
- Does the injection detection alert fire?
- Does the team recognize it as a security incident?
- Is the containment action correct?

### Scenario 2: Model Degradation

**Setup**: Deploy a fine-tuned model with known accuracy regression (5% drop) to staging.

**Expected Response:**
1. Model quality monitoring detects drift
2. SEV-2 classified
3. Investigation identifies the bad model
4. Rollback to previous model version
5. Accuracy verified against test set

**What to Watch For:**
- Is model quality monitoring in place and alerting?
- Does the team check model version and deployment history?
- Is rollback straightforward?

### Scenario 3: Vector DB Outage

**Setup**: Block network access to the vector database in staging.

**Expected Response:**
1. Vector DB unreachable alert fires
2. SEV-1 or SEV-2 classified
3. Fallback to keyword search activated
4. Service continues in degraded mode
5. Investigation of vector DB

**What to Watch For:**
- Does the fallback mechanism work?
- Does the team communicate the degraded service state?
- Is the vector DB investigation systematic?

### Scenario 4: Token Cost Explosion

**Setup**: Deploy a configuration that increases context window size 5x.

**Expected Response:**
1. Token cost anomaly alert fires
2. SEV-3 classified
3. Investigation identifies the cost driver
4. Mitigation: reduce context, rate limit, or downgrade model
5. Cost monitoring dashboard reviewed

**What to Watch For:**
- Is cost monitoring in place and alerting?
- Does the team identify the cost driver quickly?
- Are mitigation options appropriate?

### Scenario 5: PII Leakage

**Setup**: Configure the RAG system to retrieve cross-customer data in staging.

**Expected Response:**
1. PII detection alert fires on LLM output
2. SEV-1 classified
3. Service taken offline
4. Scope of leakage assessed
5. Tenant isolation fix implemented

**What to Watch For:**
- Does PII detection work?
- Is the response appropriate for a data breach?
- Is the regulatory notification process initiated?

---

## Game Day Frequency

| Type | Frequency | Participants |
|------|-----------|-------------|
| Tabletop | Monthly | Squad members |
| Technical | Bi-monthly | On-call engineers |
| Chaos Engineering | Quarterly | SRE + engineering |
| Full-Scale SEV-1 | Quarterly | Cross-functional |

---

## Measuring Game Day Effectiveness

| Metric | Target | Description |
|--------|--------|-------------|
| **Detection Time** | < 5 minutes | Time from injection to detection |
| **Classification Accuracy** | 100% | Correct severity assigned |
| **Runbook Accuracy** | > 90% | Runbook steps were correct and complete |
| **Mitigation Time** | < 30 minutes | Time from investigation start to mitigation |
| **Action Item Completion** | > 90% | Game day action items completed |
| **Repeat Findings** | 0% | Same gap not found in subsequent game days |

---

## Cross-References

- [README.md](README.md) -- Incident management philosophy
- [on-call-best-practices.md](on-call-best-practices.md) -- On-call rotation
- [genai-specific-incidents.md](genai-specific-incidents.md) -- GenAI incident patterns
- [triage-playbooks.md](triage-playbooks.md) -- Triage patterns
- [incident-metrics.md](incident-metrics.md) -- Incident metrics
- [mitigation-strategies.md](mitigation-strategies.md) -- Mitigation strategies
