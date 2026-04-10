# Incident Timeline Management

## Overview

The incident timeline is the chronological record of everything that happened during an incident -- from the initial trigger through detection, response, mitigation, and resolution. A well-maintained timeline is essential for understanding what happened, communicating with stakeholders, and conducting effective postmortems.

This document covers timeline management, evidence collection, and documentation standards.

---

## Timeline Purpose

The incident timeline serves multiple purposes:

1. **Real-Time Coordination**: During the incident, the timeline helps the team track what has been tried, what worked, and what to try next.

2. **Stakeholder Communication**: The timeline provides the factual basis for status updates and executive briefings.

3. **Postmortem Input**: The timeline is the primary input for the postmortem root cause analysis.

4. **Regulatory Evidence**: For regulated incidents, the timeline demonstrates the organization's response diligence.

5. **Organizational Learning**: Timelines from past incidents are training material for new engineers and ICs.

---

## Timeline Structure

### Format

```markdown
## Incident Timeline

### Pre-Incident (Background)
| Time (UTC) | Event | Source | Notes |
|------------|-------|--------|-------|
| 09:30 | Deployment of v2.4.0 to production | ArgoCD | PR #1234 |
| 09:35 | Health check passing | Monitoring | Normal |

### Incident Start
| Time (UTC) | Event | Source | Notes |
|------------|-------|--------|-------|
| 09:47 | Error rate begins increasing | Prometheus | From 0.5% to 15% over 3 minutes |

### Detection
| Time (UTC) | Event | Source | Notes |
|------------|-------|--------|-------|
| 09:50 | Alert fires: HighErrorRate | PagerDuty | Threshold: 2%, actual: 18% |
| 09:50 | On-call paged | PagerDuty | @bob (primary) |

### Response
| Time (UTC) | Event | Source | Notes |
|------------|-------|--------|-------|
| 09:52 | Alert acknowledged | PagerDuty | @bob: "Looking into it" |
| 09:55 | SEV-1 declared | Slack | @bob in #incident-2024-03-15 |
| 09:57 | IC assigned | PagerDuty | @alice (IC on rotation) |
| 10:00 | War room opened | Zoom | [Recording link] |

### Investigation
| Time (UTC) | Event | Source | Notes |
|------------|-------|--------|-------|
| 10:05 | Hypothesis: bad deployment | @bob | Last deploy at 09:30 |
| 10:07 | Checked deployment logs | ArgoCD | No errors in deploy |
| 10:10 | Hypothesis: downstream dependency | @carol | Vector DB latency elevated |
| 10:12 | Confirmed: Pinecone returning 503 | Logs | All queries failing |

### Mitigation
| Time (UTC) | Event | Source | Notes |
|------------|-------|--------|-------|
| 10:15 | Decision: disable RAG, use fallback | @alice (IC) | "Switch to keyword search" |
| 10:18 | Feature flag toggled | LaunchDarkly | rag_enabled = false |
| 10:20 | Error rate decreasing | Grafana | From 18% to 3% |

### Resolution
| Time (UTC) | Event | Source | Notes |
|------------|-------|--------|-------|
| 10:30 | Error rate at baseline (< 1%) | Grafana | Stable for 5 minutes |
| 10:32 | Status page updated: Resolved | Statuspage | |
| 10:35 | Incident resolved | @alice (IC) | "Declaring resolution" |

### Post-Resolution
| Time (UTC) | Event | Source | Notes |
|------------|-------|--------|-------|
| 10:40 | Stakeholder notification sent | Email | All internal stakeholders |
| 11:00 | Postmortem scheduled | Calendar | March 18, 14:00 UTC |
```

---

## Evidence Collection

### What to Preserve

During and immediately after an incident, preserve the following evidence:

#### System Evidence
- [ ] **Logs**: Application logs, infrastructure logs, audit logs
- [ ] **Metrics**: Screenshots or exports of relevant dashboards at key moments
- [ ] **Traces**: Distributed traces showing request flow during the incident
- [ ] **Deployments**: Deployment history with commit SHAs and configuration
- [ ] **Configurations**: Current and previous configuration versions

#### Communication Evidence
- [ ] **Slack/Chat History**: All incident channel messages
- [ ] **War Room Recording**: Zoom/recording of the war room session
- [ ] **Status Page History**: All status page updates with timestamps
- [ ] **Email Notifications**: Copies of all stakeholder and customer notifications

#### Business Evidence
- [ ] **Impact Metrics**: Number of affected users, failed requests, lost revenue
- [ ] **Customer Complaints**: Support tickets received during the incident
- [ ] **Social Media**: Relevant social media mentions and sentiment

### Evidence Preservation Process

1. **Freeze Relevant Logs**: Ensure log retention policies do not delete incident logs
   ```bash
   # Extend retention for incident-related logs
   aws logs put-retention-policy \
     --log-group-name /app/genai-assistbot \
     --retention-in-days 365
   ```

2. **Export Dashboards**: Screenshot or export key dashboard states
   ```bash
   # Grafana dashboard snapshot
   curl -X POST "https://grafana/api/snapshots" \
     -d '{"dashboard": {...}, "expires": 2592000}'
   ```

3. **Preserve War Room Recording**: Download and store the recording
   ```bash
   # Download Zoom recording
   zoom download --meeting-id=<id> --output=/evidence/incident-2024-03-15/
   ```

4. **Archive Chat History**: Export the incident Slack channel
   ```bash
   # Slack channel export
   slack-export --channel=#incident-2024-03-15 \
     --output=/evidence/incident-2024-03-15/slack/
   ```

5. **Tag Git Commits**: Tag the commit that caused the incident (if applicable)
   ```bash
   git tag -a incident-2024-03-15/cause <commit-sha> \
     -m "Root cause of SEV-1 on 2024-03-15"
   ```

### Evidence Retention

| Evidence Type | Retention Period | Storage |
|--------------|-----------------|---------|
| Logs | 1 year (minimum) | ELK archive |
| Metrics | 13 months | Prometheus long-term storage |
| War room recording | 1 year | Encrypted S3 |
| Chat history | 1 year | Slack export archive |
| Dashboard snapshots | 1 year | Grafana snapshots |
| Postmortem | Permanent | Internal wiki |
| Customer notifications | 5 years (regulatory) | Compliance archive |

---

## Timeline Construction

### During the Incident

The scribe maintains the timeline in real-time:

1. **Every action logged**: Decisions, commands executed, hypotheses tested
2. **Timestamps in UTC**: Consistent timezone for all entries
3. **Actor identified**: Who made the decision or took the action
4. **Source noted**: Where the information came from (dashboard, logs, observation)

### After the Incident

The scribe (or postmortem facilitator) enhances the timeline:

1. **Fill gaps**: Add events from logs, chat history, and recordings that were missed during the incident
2. **Add pre-incident context**: What happened in the hours/days before the incident?
3. **Correlate events**: Link events across systems (deployment -> error increase -> alert)
4. **Verify timestamps**: Cross-reference timestamps across sources
5. **Annotate**: Add context that was not apparent during the incident

### Timeline Completeness Checklist

- [ ] Pre-incident events included (deployments, config changes, traffic patterns)
- [ ] Exact incident start time identified (from logs, not from alert time)
- [ ] All alerts and acknowledgments logged
- [ ] All decisions documented with rationale
- [ ] All mitigation actions logged with results
- [ ] Resolution criteria documented
- [ ] Post-incident actions captured
- [ ] Timestamps consistent (all UTC)
- [ ] Sources cited for each event
- [ ] Gaps filled from post-incident log analysis

---

## Timeline Analysis

### Key Metrics from the Timeline

| Metric | Calculation | Example |
|--------|-------------|---------|
| **Incident Duration** | Resolution time - Incident start time | 09:47 to 10:35 = 48 minutes |
| **Time to Detect (MTTD)** | Alert time - Incident start time | 09:50 - 09:47 = 3 minutes |
| **Time to Acknowledge (MTTA)** | Acknowledgment time - Alert time | 09:52 - 09:50 = 2 minutes |
| **Time to War Room** | War room open time - Alert time | 10:00 - 09:50 = 10 minutes |
| **Time to Mitigate** | Mitigation start - Alert time | 10:15 - 09:50 = 25 minutes |
| **Time to Resolve (MTTR)** | Resolution time - Alert time | 10:35 - 09:50 = 45 minutes |

### Timeline Patterns to Identify

1. **Detection Gap**: Time between incident start and detection
   - Long gaps indicate missing or poorly tuned alerts

2. **Decision Delays**: Time between identifying a problem and deciding on action
   - Long delays indicate unclear ownership or analysis paralysis

3. **Execution Delays**: Time between deciding on action and completing it
   - Long delays indicate deployment pipeline or tooling issues

4. **False Starts**: Mitigation attempts that did not work
   - Multiple false starts indicate incomplete understanding

5. **Parallel Work**: Investigation tracks happening simultaneously
   - Good: efficient use of resources
   - Bad: uncoordinated efforts duplicating work

---

## Timeline Tools

### Recommended Tools

| Tool | Purpose |
|------|---------|
| Google Doc / Notion | Real-time collaborative timeline |
| PagerDuty | Automatic alert timeline |
| Slack Export | Chat history preservation |
| Grafana Snapshots | Dashboard state preservation |
| Custom Incident Platform | Structured timeline with evidence linking |

### Automated Timeline Generation

For high-maturity organizations, build an automated timeline:

```python
class IncidentTimeline:
    """Automatically constructs incident timeline from multiple sources."""

    def __init__(self, incident_id: str):
        self.incident_id = incident_id
        self.events = []

    def gather_events(self):
        """Collect events from all sources."""
        self.events.extend(self._get_pagerduty_events())
        self.events.extend(self._get_deployment_events())
        self.events.extend(self._get_alert_events())
        self.events.extend(self._get_slack_events())
        self.events.extend(self._get_metric_anomalies())
        self.events.sort(key=lambda e: e.timestamp)

    def _get_pagerduty_events(self):
        """Get PagerDuty incident events."""
        return pd_client.get_incident_log(self.incident_id)

    def _get_deployment_events(self):
        """Get deployment events from CI/CD."""
        return argocd.get_deployment_history(
            time_range=self._incident_time_range()
        )

    def _get_alert_events(self):
        """Get alert events from Prometheus."""
        return prometheus.query_alerts(
            time_range=self._incident_time_range()
        )

    def _get_slack_events(self):
        """Get Slack messages from incident channel."""
        return slack_client.get_channel_messages(
            channel=f"#incident-{self.incident_id}",
            time_range=self._incident_time_range()
        )

    def _get_metric_anomalies(self):
        """Get metric anomalies during incident period."""
        return anomaly_detector.get_anomalies(
            time_range=self._incident_time_range()
        )

    def render(self) -> str:
        """Render timeline as markdown table."""
        return self._render_markdown_table(self.events)
```

---

## Cross-References

- [README.md](README.md) -- Incident management philosophy
- [incident-command.md](incident-command.md) -- Incident commander role
- [war-room-management.md](war-room-management.md) -- War room procedures
- [postmortem-process.md](postmortem-process.md) -- Postmortem process
- [action-item-tracking.md](action-item-tracking.md) -- Action item management
