# Running Technical Meetings

> "A well-run technical meeting produces decisions. A poorly run meeting produces frustration and wasted time."

## The Meeting Problem

Engineers spend 30-50% of their time in meetings. Most of these meetings could be emails. The ones that can't be emails are often poorly run:

- No clear purpose
- No agenda
- No time management
- No decisions made
- No follow-up

**A meeting is a tool.** Like any tool, it can be used well or poorly.

## When to Hold a Meeting

Hold a meeting when:

| Situation | Why a Meeting |
|-----------|--------------|
| Complex decision with multiple viewpoints | Need real-time discussion and debate |
| Relationship building | Trust and rapport need face-to-face interaction |
| Brainstorming | Building on ideas works best synchronously |
| Sensitive conversation | Tone and intent are hard to read in text |
| Incident response | Speed is critical, decisions need to be immediate |

Don't hold a meeting when:

| Situation | Better Alternative |
|-----------|-------------------|
| Sharing information | Written update, recorded video |
| Getting a decision from one person | Direct message, email |
| Status reporting | Written status update |
| Code review | Async PR review |
| Design alignment | Design doc with review cycle |

## Meeting Design

### Before the Meeting

Every meeting should have:

1. **Clear purpose.** One sentence: "The purpose of this meeting is to [decision/outcome]."
2. **Agenda.** Specific topics with time allocations and owners.
3. **Right people.** Only people who need to participate. Others can read notes.
4. **Pre-read material.** If people need context, send it 24 hours before.

```markdown
# Meeting: GenAI Architecture Review — RAG Pipeline Redesign

**Purpose:** Decide on the retrieval architecture for the next-gen RAG pipeline.

**Date/Time:** Thursday, April 10, 2:00-3:00 PM
**Location:** Room 4B + Video link
**Attendees:** @sarah, @james, @priya, @alex (decision makers)
**Optional:** @maria, @tom (for context)

**Pre-read:** Design doc: [link] — Please read before the meeting.

**Agenda:**
| Time | Topic | Owner | Goal |
|------|-------|-------|------|
| 2:00-2:10 | Context and requirements | @sarah | Shared understanding |
| 2:10-2:30 | Option A: Hybrid search | @james | Present and discuss |
| 2:30-2:50 | Option B: Dense-only search | @priya | Present and discuss |
| 2:50-3:00 | Decision and next steps | @sarah | Agree on direction |

**Decision to make:** Which retrieval architecture to implement for Q2.
```

### During the Meeting

**Facilitator responsibilities:**

| Responsibility | How |
|---------------|-----|
| **Start on time** | Don't wait for late arrivals. They'll learn. |
| **State the purpose** | "We're here to decide X. By the end of this meeting, we will have a decision." |
| **Follow the agenda** | Keep discussions on track. "That's an important topic — can we discuss it after the decision?" |
| **Manage time** | "We have 10 minutes left for this topic. Let's focus on the key question." |
| **Draw out quiet voices** | "Priya, you worked on the current system — what's your view?" |
| **Manage dominant voices** | "Thanks, James. Let me hear from someone who hasn't spoken yet." |
| **Summarize periodically** | "So far we've agreed on X and Y. We still need to decide Z." |
| **End with decisions** | "We decided X. @james will do Y by Friday. @priya will investigate Z and report back." |

### After the Meeting

**Within 1 hour:** Send meeting notes.

```markdown
# Meeting Notes: GenAI Architecture Review — RAG Pipeline Redesign
# Date: April 10, 2026

## Decisions Made
1. Use hybrid search (BM25 + dense embeddings) for retrieval
   → Rationale: Hybrid search improves recall by 23% on our benchmark
   → Dissenting view: @priya noted operational complexity. Mitigation: Use existing Elasticsearch cluster for BM25.

## Action Items
| Action | Owner | Due |
|--------|-------|-----|
| Update design doc with hybrid search architecture | @sarah | April 12 |
| Benchmark BM25 performance on Elasticsearch | @james | April 15 |
| Investigate vector index size for 10M documents | @priya | April 15 |

## Open Questions
- Do we need to re-embed existing documents or can we keep current embeddings?
- What's the impact on query latency with dual retrieval?

## Next Meeting
- April 17, 2:00 PM — Review benchmark results and finalize design
```

## Meeting Types and Their Formats

### Technical Decision Meeting (45-60 min)

```
Purpose: Make a specific technical decision
Attendees: Decision makers + subject matter experts
Pre-work: Design doc or RFC read by all attendees

Agenda:
1. Context (5 min) — Restate the decision to be made
2. Present options (20 min) — Each option presented by its advocate
3. Discussion (20 min) — Questions, concerns, trade-offs
4. Decision (10 min) — Decision maker decides, documents rationale
5. Next steps (5 min) — Action items and owners
```

### Architecture Review (60-90 min)

```
Purpose: Review and provide feedback on an architecture design
Attendees: Author + reviewers from affected teams
Pre-work: Design doc read and commented on by all reviewers

Agenda:
1. Author presentation (15 min) — Overview of the design
2. Clarifying questions (10 min) — "What does X mean?" not "Why didn't you..."
3. Feedback by area (30 min) — Security, performance, operations, etc.
4. Author response (10 min) — Address feedback
5. Next steps (5 min) — What changes, what stays, what needs follow-up
```

### Incident Response Meeting (variable)

```
Purpose: Coordinate response to an active incident
Attendees: Incident commander + responders
Pre-work: None — this is an emergency meeting

Format:
- Incident commander runs the meeting
- Status updates every 15 minutes
- Decisions made by incident commander
- Notes taken by designated scribe
- Meeting ends when incident is resolved
- Follow-up: Postmortem within 5 business days
```

### Brainstorming Session (45-60 min)

```
Purpose: Generate ideas for a problem
Attendees: Diverse perspectives (not just the usual team)
Pre-work: Problem statement shared in advance

Agenda:
1. Problem framing (5 min) — What are we solving?
2. Individual ideation (10 min) — Silent writing of ideas
3. Share and group (15 min) — Each person shares, group similar ideas
4. Discuss and refine (15 min) — Explore promising ideas
5. Prioritize (10 min) — Vote on top 3 ideas to explore further
```

### Retrospective (45-60 min)

See `engineering-culture/incident-retrospectives.md` and `engineering-culture/team-rituals.md`.

## Facilitation Techniques

### Handling Dominant Participants

- "Thanks, [Name]. Let me hear from someone we haven't heard from yet."
- "I want to make sure everyone's perspective is captured. Let's go around the room."
- Privately: "I appreciate your energy in the discussion. Can you help me draw out quieter folks?"

### Handling Silence

- "Let's take 2 minutes to write down our thoughts individually, then share."
- "What's the biggest concern with this approach?"
- "If this fails, what do you think will cause it?"
- Wait. Silence is okay. People are thinking.

### Handling Tangents

- "That's an important topic. Can we put it in the parking lot and come back if we have time?"
- "I want to make sure we make the decision we came here for. Can we return to the main question?"
- "Let's schedule a separate discussion for that topic."

### Handling Conflict

- "I hear two different views. Let me summarize: [A] vs. [B]. Is that right?"
- "What evidence would help us decide between these?"
- "Can both of you commit to supporting the decision once it's made?"
- If conflict persists: "I'm going to decide this now. Here's the decision and the rationale."

### Handling the "HiPPO" (Highest Paid Person's Opinion)

- "Before we settle on that, I want to make sure we've heard all perspectives."
- "That's one view. What are the risks or downsides of that approach?"
- "Let's make sure we're evaluating this on its merits. What are the alternatives?"

## Meeting Anti-Patterns

| Anti-Pattern | What It Looks Like | Fix |
|--------------|-------------------|-----|
| **No purpose** | "Let's sync up about the thing" | Define the purpose or cancel the meeting |
| **Too many people** | 15 people, 3 of them talking | Invite only decision makers and essential contributors |
| **No pre-read** | Spending 30 minutes bringing everyone up to speed | Send material 24 hours in advance |
| **Decision avoidance** | Discussion without conclusion | Name the decision maker and the deadline |
| **Action item amnesia** | Meeting ends, nobody knows what to do next | End with explicit action items, owners, and deadlines |
| **Recurring without review** | Same meeting every week for 6 months | Every 4 weeks, ask: "Is this meeting still valuable?" |
| **No notes** | Decisions made but not recorded | Assign a note-taker, send notes within 1 hour |

## Remote Meeting Best Practices

| Practice | Why |
|----------|-----|
| **Camera on** | Non-verbal cues are essential for effective discussion |
| **Mute when not speaking** | Reduces background noise |
| **Use the chat for side comments** | Allows parallel conversation without interrupting |
| **Screen share with cursor highlighting** | Makes it easier to follow |
| **Record the meeting** | For attendees in other time zones |
| **Use collaborative documents** | Google Docs or Confluence for live note-taking |
| **Acknowledge remote participants first** | "Let me start with the people on video" |

## Measuring Meeting Effectiveness

After important meetings, ask:

| Question | Why |
|----------|-----|
| Did we achieve the meeting's purpose? | Effectiveness |
| Was the right group of people in the room? | Efficiency |
| Could this have been an email/document instead? | Necessity |
| Do all attendees know what they need to do next? | Follow-through |

## Cross-References

- `engineering-culture/team-rituals.md` — Team meetings and ceremonies
- `engineering-culture/async-communication.md` — When NOT to meet
- `leadership-and-collaboration/giving-status-updates.md` — Status updates (meeting alternative)
- `engineering-culture/design-docs.md` — Design doc reviews (meeting format)
- `leadership-and-collaboration/negotiation.md` — Technical negotiation in meetings
