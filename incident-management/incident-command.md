# Incident Commander Role

## Overview

The Incident Commander (IC) is the single point of accountability for managing an incident from detection through resolution. The IC does not need to be the most technical person in the room -- their role is to coordinate the response, make decisions, and communicate status.

This document defines the IC role, responsibilities, selection criteria, rotation schedule, and best practices.

---

## When an IC Is Required

| Severity | IC Required? | Timeline |
|----------|-------------|----------|
| SEV-1 | Yes | Assigned within 15 minutes of detection |
| SEV-2 | Yes | Assigned within 30 minutes of detection |
| SEV-3 | No (may be assigned at IC discretion) | If assigned, within 1 hour |
| SEV-4 | No | N/A |

---

## IC Responsibilities

### During the Incident

1. **Own the Response**
   - Take full accountability for coordinating the incident response
   - Make final decisions on mitigation actions
   - Prioritize actions (investigation, containment, communication)

2. **Assemble the Team**
   - Identify and page required Subject Matter Experts (SMEs)
   - Assign roles: Scribe, Communications Lead, SMEs
   - Open war room (Zoom/physical) for SEV-1

3. **Drive Investigation**
   - Ensure the team is systematically investigating the issue
   - Ask clarifying questions: "What do we know?", "What don't we know?", "What are our hypotheses?"
   - Prevent rabbit holes: "We've been investigating X for 10 minutes without progress. Should we try a different angle?"

4. **Make Decisions**
   - Decide on mitigation actions (rollback, scale, disable feature)
   - Authorize emergency changes (bypass normal deployment process)
   - Decide when to escalate to executive leadership

5. **Communicate**
   - Ensure regular status updates (every 30 minutes for SEV-1)
   - Approve customer-facing communications
   - Brief executive leadership as needed
   - Ensure incident timeline is being documented

6. **Manage the War Room**
   - Keep the discussion focused and productive
   - Prevent side conversations that distract from the investigation
   - Ensure all hypotheses and findings are captured in the timeline
   - Mute participants who are not actively contributing

7. **Declare Resolution**
   - Confirm resolution criteria are met
   - Declare the incident resolved
   - Ensure stakeholder notification of resolution
   - Schedule postmortem

### After the Incident

1. **Postmortem Facilitation**
   - Assign a Postmortem Facilitator (may be the IC or a different person)
   - Ensure postmortem is scheduled within 5 business days (SEV-1/SEV-2)
   - Review and approve the postmortem document

2. **Action Item Tracking**
   - Ensure all action items are captured with owners and deadlines
   - Follow up on action items until complete
   - Escalate overdue action items to engineering leadership

3. **Lessons Shared**
   - Ensure postmortem summary is shared with the engineering organization
   - Highlight learnings in team meetings
   - Update runbooks and playbooks based on incident learnings

---

## IC Authority

The IC has the authority to:

1. **Deploy Emergency Changes**: Bypass normal CI/CD processes for mitigation
2. **Disable Features**: Turn off any feature to contain an incident
3. **Page Anyone**: Request any engineer to join the incident response
4. **Escalate to Executives**: Notify VP Engineering, CTO, or CEO as needed
5. **Authorize Customer Communication**: Approve customer-facing status updates
6. **Commit Resources**: Allocate additional compute, budget, or personnel
7. **Declare Resolution**: Officially close the incident

This authority is scoped to the incident duration. Post-incident, normal processes resume.

---

## IC Qualifications

### Required Skills

1. **Technical Breadth**: Understand the system architecture well enough to ask the right questions and evaluate proposed actions
2. **Decision-Making Under Pressure**: Comfortable making decisions with incomplete information
3. **Communication**: Clear, concise communication to technical and non-technical audiences
4. **Calm Under Stress**: Maintains composure and focus during high-pressure situations
5. **Process Discipline**: Follows the incident management process consistently

### Training Requirements

All ICs must complete:
1. **IC Training Course** (4 hours): Incident management theory, role responsibilities, decision-making frameworks
2. **Shadowing** (2 incidents): Shadow an experienced IC during real incidents
3. **Reverse Shadowing** (2 incidents): Lead an incident with an experienced IC observing and coaching
4. **Game Day Participation** (quarterly): Participate in simulated incident exercises
5. **Annual Recertification**: Complete a game day scenario demonstrating IC competency

---

## IC Rotation

### Rotation Schedule

- **Primary ICs**: 6-8 trained engineers across all squads
- **Rotation**: Weekly rotation, Monday to Monday
- **Backup IC**: Next person in rotation (paged if Primary IC does not respond within 5 minutes)
- **Notification**: Slack reminder 1 day before rotation begins

### IC On-Call Calendar

```
Week 1: Alice (Platform Squad)
Week 2: Bob (Inference Squad)
Week 3: Carol (Data Squad)
Week 4: Dave (Frontend Squad)
Week 5: Eve (Platform Squad)
Week 6: Frank (Inference Squad)
...
```

### IC Handoff

When IC rotation changes (every Monday):
1. Outgoing IC briefs incoming IC on any open incidents or action items
2. Incoming IC confirms PagerDuty escalation policy is correct
3. Incoming IC reviews current runbooks and playbooks
4. Outgoing IC documents any learnings or process improvements

---

## IC Decision Framework

When facing a decision during an incident, the IC should:

### 1. Gather Information
- "What do we know?"
- "What are our hypotheses?"
- "What evidence supports each hypothesis?"

### 2. Evaluate Options
- "What actions can we take?"
- "What are the risks of each action?"
- "What happens if we do nothing?"

### 3. Make a Decision
- "I'm deciding we [action]. The rationale is [reason]."
- "Who will execute this? What is the expected timeline?"
- "If this does not work, what is our fallback?"

### 4. Communicate
- Post decision in incident channel
- Update status page if customer-facing impact
- Brief stakeholders on decision and rationale

### 5. Monitor Outcome
- "Did the action have the expected effect?"
- "If not, why? What do we try next?"

---

## Common IC Mistakes

### 1. Doing Technical Work Instead of Commanding

**Wrong**: IC dives into debugging, leaving no one coordinating the response.
**Right**: IC delegates technical work to SMEs and focuses on coordination.

### 2. Waiting for Certainty Before Acting

**Wrong**: "We need to understand the root cause before we take action."
**Right**: "We do not know the root cause, but rolling back the last deployment is low-risk and may resolve the issue. Let's do it."

### 3. Not Communicating Regularly

**Wrong**: 90 minutes of investigation with no stakeholder updates.
**Right**: Status update every 30 minutes, even if the update is "we are still investigating."

### 4. Letting Side Conversations Derail the War Room

**Wrong**: Three parallel discussions happening simultaneously in the war room.
**Right**: "Let's focus on one hypothesis at a time. We'll discuss [X] after we resolve [Y]."

### 5. Not Declaring Resolution

**Wrong": Service has been stable for 2 hours, but the incident is still "open."
**Right**: "Service has been stable for 2 hours and the root cause is understood. I'm declaring resolution. Let's schedule the postmortem."

---

## IC Checklist

### Initial Response (First 15 Minutes)

- [ ] Acknowledge incident and assume IC role
- [ ] Confirm severity classification
- [ ] Open war room (Zoom/physical)
- [ ] Page required SMEs
- [ ] Assign Scribe and Communications Lead
- [ ] Begin incident timeline documentation
- [ ] Initial assessment: "What do we know?"

### Investigation Phase

- [ ] Ensure systematic hypothesis testing
- [ ] Prevent rabbit holes
- [ ] Request additional resources if needed
- [ ] First stakeholder update (within 30 minutes)

### Mitigation Phase

- [ ] Evaluate mitigation options
- [ ] Decide on mitigation action
- [ ] Authorize emergency deployment if needed
- [ ] Monitor mitigation outcome
- [ ] Update stakeholders on mitigation progress

### Resolution Phase

- [ ] Confirm resolution criteria met
- [ ] Declare resolution
- [ ] Notify stakeholders of resolution
- [ ] Update status page
- [ ] Schedule postmortem (within 5 business days)

### Post-Incident

- [ ] Review and approve postmortem document
- [ ] Ensure action items captured with owners and deadlines
- [ ] Escalate overdue action items
- [ ] Share lessons learned with team

---

## IC Scripts

### Opening the War Room

> "Welcome to the SEV-1 war room for [incident name]. I'm [name], the Incident Commander. Let me quickly set the context:
>
> - What we know: [summary]
> - What we don't know: [gaps]
> - Our immediate priorities are: [priorities]
>
> [SME name], I'd like you to lead the investigation. [Scribe name], please document the timeline. [Comms name], please prepare the first stakeholder update.
>
> Let's start with what we know. [SME], walk us through the situation."

### Making a Decision

> "I've heard the options. I'm deciding we [action] because [rationale]. [Engineer name], please execute. Expected timeline is [time]. If this doesn't work, our fallback is [fallback].
>
> I'll post this decision in the incident channel and update stakeholders."

### Regular Status Update

> "Status update [time]:
>
> - **Current status**: [investigating / mitigating / monitoring]
> - **Impact**: [scope and severity]
> - **What we know**: [key findings]
> - **What we're doing**: [current actions]
> - **Next update**: [time]
> - **ETA for resolution**: [estimate or 'too early to tell']"

### Declaring Resolution

> "I'm declaring this incident resolved at [time]. Resolution criteria:
>
> - [ ] Service restored to normal operation: Yes
> - [ ] Root cause understood: Yes
> - [ ] Mitigation in place: Yes
> - [ ] No further risk of recurrence: Yes
>
> The postmortem will be scheduled within 5 business days. Thank you to everyone who contributed to the response."

---

## Cross-References

- [README.md](README.md) -- Incident management philosophy
- [incident-classification.md](incident-classification.md) -- Severity levels
- [war-room-management.md](war-room-management.md) -- War room procedures
- [communication-during-incidents.md](communication-during-incidents.md) -- Stakeholder communication
- [postmortem-process.md](postmortem-process.md) -- Postmortem process
- [on-call-best-practices.md](on-call-best-practices.md) -- On-call rotation
- [genai-specific-incidents.md](genai-specific-incidents.md) -- GenAI incident patterns
