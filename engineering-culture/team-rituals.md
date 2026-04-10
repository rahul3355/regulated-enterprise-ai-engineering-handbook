# Team Rituals

> "Rituals aren't just ceremonies — they're the rhythm that keeps a team synchronized, aligned, and human."

## Why Team Rituals Matter

In a distributed, high-pressure environment like banking GenAI engineering, rituals provide:

- **Cadence:** Regular touchpoints that create predictability
- **Alignment:** Shared understanding of priorities and progress
- **Connection:** Human moments that build trust beyond code
- **Learning:** Structured opportunities to share knowledge
- **Closure:** Clear endings that allow teams to move forward

Without rituals, teams drift. Meetings happen randomly, decisions get lost, and people work in silos.

## Essential Team Rituals

### Daily Standup (15 minutes)

**Purpose:** Surface blockers, share progress, align on priorities.

**Format:**

```
Time: 9:30 AM (or team's preferred time)
Duration: 15 minutes MAX
Format: In-person or video on

Each person shares:
1. What I did yesterday (toward our sprint goals)
2. What I'm doing today
3. What's blocking me

Rules:
- No problem-solving in standup (take it offline)
- Blockers are escalated immediately after
- Keep it brief (1-2 minutes per person)
```

**What standup is NOT:**
- A status report to the manager
- A technical discussion
- A planning session
- A blame session ("why didn't you finish?")

**Banking GenAI context:** In a GenAI platform team, standup often includes:
- Model performance metrics review ("response quality dropped 3% yesterday")
- Compliance updates ("new regulatory guidance came in")
- Cross-team dependencies ("the security team needs our API spec by EOD")

### Weekly Team Sync (45 minutes)

**Purpose:** Deeper discussion than standup, team-wide alignment.

**Format:**

```
Time: Monday 2:00 PM
Duration: 45 minutes
Agenda:
1. Sprint progress (10 min)
   - Burndown review
   - At-risk items
2. Technical discussion (15 min)
   - Design doc review
   - Architecture decision
   - Code walkthrough
3. Cross-team updates (10 min)
   - What other teams are doing that affects us
4. Open discussion (10 min)
   - Anyone can raise any topic
```

### Sprint Planning (60-90 minutes, bi-weekly)

**Purpose:** Agree on what the team will deliver in the next sprint.

**Format:**

```
Pre-work (before meeting):
- Product owner prioritizes backlog
- Engineers review tickets and estimate effort
- Dependencies identified

Meeting:
1. Review previous sprint outcomes (10 min)
   - What was delivered
   - What wasn't and why
2. Review upcoming sprint goals (15 min)
   - Product owner presents priorities
   - Engineering validates feasibility
3. Commit to sprint scope (20 min)
   - Team agrees on what can be delivered
   - Identify risks and dependencies
4. Assign work (10 min)
   - Who works on what
   - Pairing arrangements
```

**Banking GenAI context:** Sprint planning for GenAI platforms includes:
- Model upgrade planning (new model versions, evaluation)
- Compliance milestone tracking (regulatory deadlines)
- Security review scheduling (new features need review before launch)
- Capacity planning (on-call reduces available engineering time)

### Sprint Demo (30 minutes, end of each sprint)

**Purpose:** Show what was built, get feedback, celebrate wins.

**Format:**

```
Time: Friday 3:00 PM (last day of sprint)
Duration: 30 minutes
Audience: Team + stakeholders (PM, engineering manager, adjacent teams)

Agenda:
1. Demo of new features (15 min)
   - Live demo, not slides
   - Show it working in staging or production
2. Metrics review (5 min)
   - Performance, quality, user adoption
3. Shout-outs (5 min)
   - Recognize individual contributions
4. Preview next sprint (5 min)
   - What's coming
```

**Rules:**
- Demos are live, not screenshots (if it doesn't demo, it doesn't work)
- Everyone demos their work (not just the tech lead)
- Stakeholders can ask questions but not criticize during the demo
- Failed demos are okay — they're learning opportunities

### Sprint Retrospective (45 minutes, end of each sprint)

**Purpose:** Reflect on how the team worked together and identify improvements.

**Format:**

```
Time: Friday 4:00 PM (after demo, last day of sprint)
Duration: 45 minutes
Attendees: Team only (no stakeholders)

Format options:
A) Start / Stop / Continue
   - What should we START doing?
   - What should we STOP doing?
   - What should we CONTINUE doing?

B) Glad / Sad / Mad
   - What made us glad?
   - What made us sad?
   - What made us mad?

C) 4Ls
   - What did we LEARN?
   - What did we LOVE?
   - What did we LACK?
   - What did we LONG FOR?

Process:
1. Individual reflection (5 min) — silent writing
2. Share and group themes (15 min)
3. Discuss and prioritize (15 min)
4. Commit to 1-2 actions (10 min)
```

**Golden rule of retrospectives:** At least one action item from each retro must be completed in the next sprint. Otherwise the retro feels like complaining with no change.

See also: `incident-retrospectives.md` for incident-specific retrospectives.

### Tech Talks / Brown Bags (see `knowledge-sharing.md`)

### On-Call Handoff (15 minutes, weekly)

**Purpose:** Transfer knowledge from outgoing to incoming on-call engineer.

**Format:**

```
Time: Monday 9:00 AM (on-call rotation change)
Duration: 15 minutes
Attendees: Outgoing + incoming on-call

Outgoing shares:
- Open incidents (status, next steps)
- Known issues (things that might page you)
- Recent changes (deployments, config changes)
- Tips and gotchas (anything non-obvious)

Incoming confirms:
- Access to all systems
- Understanding of current situation
- Contact information for escalation
```

See `templates/oncall-handoff-template.md` for the template.

### Monthly Engineering Review (60 minutes)

**Purpose:** Strategic view of team health and trajectory.

**Format:**

```
Time: Last Friday of each month
Duration: 60 minutes
Attendees: Team + engineering manager

Agenda:
1. Metrics review (15 min)
   - Sprint velocity trends
   - Incident frequency and severity
   - Code quality metrics
   - On-call load
2. Retrospective action item review (10 min)
   - What actions are complete?
   - What actions are blocked?
   - What actions need to be dropped?
3. Process improvements (15 min)
   - What's working well?
   - What needs to change?
4. Career development (10 min)
   - Growth plan progress
   - Mentorship check-ins
5. Open discussion (10 min)
```

### Quarterly Team Health Check (90 minutes)

**Purpose:** Deep reflection on team culture, processes, and well-being.

**Format:**

```
Time: Quarterly off-site or extended session
Duration: 90 minutes
Attendees: Team only

Activities:
1. Anonymous survey (before meeting)
   - How satisfied are you with the team?
   - How supported do you feel?
   - How clear are our goals?
   - How sustainable is our workload?
2. Survey results discussion (30 min)
   - Share aggregate results (no individual responses)
   - Discuss trends and patterns
3. Working agreements review (20 min)
   - Are our agreements still working?
   - What needs to change?
4. Culture discussion (20 min)
   - What do we value as a team?
   - Are we living those values?
5. Action planning (20 min)
   - What will we change next quarter?
```

## Ritual Design Principles

### Every Ritual Should Have

1. **Clear purpose.** If you can't state the purpose in one sentence, you don't need the ritual.
2. **Consistent cadence.** Weekly on Mondays, monthly on the last Friday. Predictability matters.
3. **Time-boxed duration.** Respect people's time. If it regularly runs over, something is wrong.
4. **Facilitator.** Someone owns the ritual and ensures it happens effectively.
5. **Outcome.** Every ritual produces something: decisions, actions, shared understanding.

### When to Add a Ritual

- The team has a recurring need (alignment, learning, reflection)
- An existing gap is causing problems
- The team agrees it would be valuable

### When to Remove a Ritual

- The purpose is no longer relevant
- The ritual consistently produces no value
- The team feels it's a waste of time
- The same outcome could be achieved asynchronously

## Ritual Anti-Patterns

| Anti-Pattern | What It Looks Like | Fix |
|--------------|-------------------|-----|
| **Meeting bloat** | 15 hours of meetings per week | Audit every meeting, eliminate the unnecessary ones |
| **Ritual without purpose** | "We do standup because we've always done standup" | Revisit the purpose, adapt or eliminate |
| **Same format always** | Every retro feels identical | Rotate formats, keep it fresh |
| **No follow-through** | Retro actions are never completed | Assign owners, track in sprint |
| **Ritual held hostage** | One person's schedule always prevents the ritual | Rotate times, or make it async |
| **Mandatory fun** | Forced social activities that feel like work | Make social activities genuinely optional |

## Remote-First Rituals

For distributed teams:

| Ritual | Remote Adaptation |
|--------|------------------|
| Standup | Async in Slack (written updates by 10 AM) + optional video sync |
| Team sync | Video on, agenda in advance, recorded for timezone coverage |
| Demo | Screen share + recording, interactive chat in video |
| Retro | Miro board or similar collaborative tool, video on |
| On-call handoff | Written handoff doc + 15-min video call |
| Social | Virtual coffee chats, optional game sessions, async photo sharing |

## Cross-References

- `engineering-culture/working-agreements.md` — Working agreements that rituals support
- `engineering-culture/knowledge-sharing.md` — Knowledge sharing rituals
- `engineering-culture/mentorship.md` — Mentorship rituals
- `engineering-culture/incident-retrospectives.md` — Incident-specific retrospectives
- `templates/retrospective-template.md` — Retrospective template
- `templates/sprint-planning-template.md` — Sprint planning template
