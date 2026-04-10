# War Room Management

## Overview

The war room is the central coordination point for incident response. It is a dedicated space (physical room or virtual meeting) where the incident response team collaborates in real-time. This document defines war room setup, facilitation procedures, and note-taking standards.

---

## When to Open a War Room

| Severity | War Room? | Timing |
|----------|-----------|--------|
| SEV-1 | Yes | Within 15 minutes of detection |
| SEV-2 | Yes (if not resolved within 30 minutes) | Within 30 minutes |
| SEV-3 | No (ad-hoc channel sufficient) | N/A |
| SEV-4 | No | N/A |

---

## War Room Setup

### Virtual War Room (Default)

For distributed teams, the virtual war room is the primary coordination channel:

**Platform**: Zoom or Google Meet with:
- Screen sharing for dashboards
- Recording enabled (for post-incident review)
- Waiting room disabled (for rapid access)
- Host controls given to IC

**Slack Channel**: Dedicated incident channel named:
- Format: `#incident-YYYY-MM-DD-[brief-description]`
- Example: `#incident-2024-03-15-assistbot-outage`

**Channel Topic**:
```
SEV-1 | AssistBot Outage | IC: @alice | Started: 09:47 UTC | War Room: [Zoom link] | Status Page: [link]
```

### Physical War Room (SEV-1, On-Site)

For critical incidents when team members are in the office:

**Room Requirements:**
- Large display screen for dashboards
- Whiteboard for diagrams and timelines
- Conference phone for remote participants
- Reliable WiFi
- Minimal interruptions (book the room for the full day)

**Room Setup:**
```
┌─────────────────────────────────────┐
│         LARGE DISPLAY SCREEN         │
│  Dashboards: Grafana, Status Page   │
│                                    │
│  ┌─────────┐  ┌─────────┐         │
│  │ Metrics │  │  Logs   │         │
│  └─────────┘  └─────────┘         │
│                                    │
│  ┌──────────────────────────────┐  │
│  │    Incident Timeline (live)   │  │
│  └──────────────────────────────┘  │
└─────────────────────────────────────┘

        [Whiteboard wall behind screen]

     ┌──────┐     ┌──────┐     ┌──────┐
     │ IC   │     │ SME  │     │Scribe│
     └──────┘     └──────┘     └──────┘
```

---

## War Room Roles

| Role | Responsibilities | Who |
|------|-----------------|-----|
| **Incident Commander** | Runs the war room, makes decisions | Assigned IC |
| **Scribe** | Documents timeline, captures decisions and actions | Designated engineer |
| **Communications Lead** | Drafts updates, manages status page | Product manager or engineer |
| **SMEs** | Investigate and propose actions | Engineers with relevant expertise |
| **Observer** | Listens and learns, does not speak unless asked | Junior engineers, new IC trainees |

---

## War Room Facilitation

### Opening the War Room

The IC opens the war room with a brief context setting:

> "Welcome to the SEV-1 war room for [incident name]. I'm [name], the Incident Commander.
>
> Here is what we know:
> - [Brief summary of the situation]
> - [Impact assessment]
> - [Any actions already taken]
>
> Here is what we need to figure out:
> - [Open questions]
>
> [SME name], I'd like you to lead the investigation.
> [Scribe name], please document the timeline.
> [Comms name], please prepare the first stakeholder update.
>
> Let's start. [SME], walk us through what you see."

### Facilitation Rules

1. **One Conversation at a Time**
   - Only one person speaks at a time
   - Side conversations happen in text, not voice
   - The IC controls the floor

2. **Stay Focused**
   - Discuss one hypothesis at a time
   - Time-box investigations (10 minutes max before reassessing)
   - "We have been investigating X for 10 minutes without progress. Let's try a different angle."

3. **Mute When Not Speaking**
   - All participants muted unless speaking
   - Reduces background noise for remote participants

4. **Document Everything**
   - Every hypothesis, finding, and decision goes in the timeline
   - The scribe is responsible for capturing all discussion
   - If it is not documented, it did not happen

5. **Respect the Scribe**
   - Speak clearly and slowly when stating findings
   - Repeat important numbers and timestamps
   - Confirm the scribe captured it correctly

### Managing Large War Rooms

When the war room grows beyond 10 people:

1. **Designate a Technical Lead**: The IC delegates investigation management to a Tech Lead
2. **Breakout Rooms**: Split investigation into parallel tracks (e.g., "investigate database" and "investigate network")
3. **Check-In Cadence**: Breakout teams report back every 10 minutes
4. **Strict Mute Policy**: Only active investigators unmuted; others listen

### Managing the War Room Chat

The Slack channel runs parallel to the voice call:

1. **All updates posted in the channel**: Even if said verbally
2. **Screenshots of dashboards**: With context ("P95 latency at 10:15 UTC")
3. **Key decisions confirmed**: "IC decision: rolling back to v2.3.1"
4. **No side conversations**: Create a separate channel for non-essential discussion

---

## Note-Taking Standards

### Incident Timeline Format

The scribe maintains a real-time timeline in the incident ticket:

```markdown
## Incident Timeline

| Time (UTC) | Event | Actor | Notes |
|------------|-------|-------|-------|
| 09:47 | Alert: AssistBot error rate > 20% | Monitoring | PagerDuty alert fired |
| 09:48 | Alert acknowledged | @bob (on-call) | "Looking into it" |
| 09:50 | Confirmed: AssistBot returning 500 errors | @bob | All requests failing |
| 09:52 | SEV-1 declared | @bob | Paged IC |
| 09:55 | War room opened | @alice (IC) | [Zoom link] |
| 10:00 | First stakeholder update sent | @carol (comms) | Internal notification |
| 10:05 | Hypothesis: bad deployment | @bob | Last deployment at 09:30 |
| 10:08 | Confirmed: rollback to v2.3.1 restores service | @bob | Testing in staging |
| 10:10 | Decision: rollback in production | @alice (IC) | "Proceeding with rollback" |
| 10:15 | Rollback initiated | @bob | kubectl rollout undo |
| 10:18 | Rollback complete | @bob | All pods running v2.3.1 |
| 10:20 | Error rate returning to normal | @bob | < 1% and decreasing |
| 10:25 | Status page updated: Monitoring | @carol | |
| 10:30 | Incident resolved | @alice (IC) | "Declaring resolution" |
```

### Scribe Checklist

- [ ] Every alert and acknowledgment timestamped
- [ ] Every decision documented with rationale
- [ ] Every action logged with owner
- [ ] Every stakeholder update captured
- [ ] Every hypothesis and its outcome recorded
- [ ] Resolution time and criteria documented

### Post-War Room Documentation

After the incident is resolved:
1. Scribe reviews and cleans up the timeline
2. Add any missing events from logs or chat history
3. Export timeline to the postmortem document
4. Archive the war room recording (per retention policy)

---

## War Room Dashboards

The main display should show:

### Essential Dashboards

1. **Service Health Overview**
   - Request rate, error rate, latency (golden signals)
   - Per-service status (green/yellow/red)

2. **Infrastructure Metrics**
   - CPU, memory, GPU utilization
   - Node health
   - Network throughput

3. **GenAI-Specific Metrics**
   - Token consumption rate
   - Model inference latency
   - Vector DB query rate and latency
   - PII detection alerts

4. **Recent Deployments**
   - Last 5 deployments per service
   - Rollback status

5. **Active Alerts**
   - Current firing alerts
   - Alert history (last 1 hour)

### Dashboard Access

All participants should have the dashboards open on their own screens:
- Grafana: `/dashboards/incident-overview`
- PagerDuty: Active incidents view
- Status page: Internal and external views
- Deployment dashboard: ArgoCD or equivalent

---

## War Room Etiquette

### Do

- Join on time when paged
- Mute your microphone when not speaking
- State your name before speaking (in large war rooms)
- Post all relevant information in the Slack channel
- Respect the IC's decisions
- Ask for clarification if something is unclear
- Take breaks (incidents can last hours)

### Do Not

- Join the war room to give unsolicited opinions
- Have side conversations on the voice call
- Blame individuals for the incident
- Make unilateral changes without IC approval
- Leave without telling the IC
- Share incident details outside the war room

---

## War Room Closure

The IC closes the war room when:
1. The incident is resolved
2. All mitigation actions are complete
3. Stakeholders have been notified of resolution
4. Postmortem is scheduled

**Closing Script:**

> "The incident is resolved. I'm closing the war room. Thank you to everyone who contributed.
>
> Summary:
> - Duration: [X hours Y minutes]
> - Root cause: [brief description]
> - Impact: [brief description]
> - Resolution: [brief description]
>
> The postmortem will be scheduled within 5 business days. The war room recording and chat history will be preserved for reference.
>
> Great work, everyone."

---

## Cross-References

- [README.md](README.md) -- Incident management philosophy
- [incident-command.md](incident-command.md) -- Incident commander role
- [incident-timeline.md](incident-timeline.md) -- Timeline management
- [communication-during-incidents.md](communication-during-incidents.md) -- Communication procedures
- [postmortem-process.md](postmortem-process.md) -- Postmortem process
