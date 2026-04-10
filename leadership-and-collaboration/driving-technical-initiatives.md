# Driving Technical Initiatives

> "Driving a technical initiative across teams is not about having the best technical argument. It's about building the coalition, managing the change, and delivering the outcome."

## What Does "Driving" Mean?

Driving a technical initiative means you are the person responsible for taking it from idea to production across multiple teams. You are the **single-threaded owner** — not necessarily doing all the work, but ensuring it gets done.

In banking GenAI, examples include:
- Migrating the GenAI platform from single-provider to multi-provider
- Implementing enterprise-wide prompt injection detection standards
- Building a shared RAG pipeline used by multiple teams
- Upgrading Kubernetes clusters across all GenAI services
- Implementing audit logging standards across all AI systems

## The Initiative Lifecycle

```
IDEATION
  │
  ├─── Identify the problem and opportunity
  ├─── Build the initial case (data, rationale)
  └─── Socialize with key stakeholders
  │
  ▼
DESIGN
  │
  ├─── Write the strategy or design document
  ├─── Gather feedback from affected teams
  ├─── Address concerns and incorporate feedback
  └─── Get formal approval from decision-makers
  │
  ▼
PLANNING
  │
  ├─── Break into phases with milestones
  ├─── Assign owners for each phase/team
  ├─── Identify dependencies and risks
  └─── Establish communication cadence
  │
  ▼
EXECUTION
  │
  ├─── Track progress against milestones
  ├─── Unblock teams as they work
  ├─── Communicate progress and risks regularly
  ├─── Adapt the plan as reality changes
  └─── Manage scope creep
  │
  ▼
DELIVERY
  │
  ├─── Coordinate rollout
  ├─── Monitor for issues
  ├─── Communicate completion
  └─── Capture lessons learned
  │
  ▼
OPERATIONALIZATION
  │
  ├─── Hand off to operations
  ├─── Document runbooks and procedures
  ├─── Train support teams
  └──— Establish ongoing maintenance
```

## Phase-by-Phase Guide

### Phase 1: Ideation

**Your job:** Build a compelling case that there's a problem worth solving.

```markdown
Ideation Checklist:
- [ ] Problem is clearly defined with data
- [ ] Impact is quantified (cost, risk, time, user experience)
- [ ] You've talked to at least 3 people affected by the problem
- [ ] You have a rough idea of the solution direction
- [ ] You've identified who would need to be involved
- [ ] You've checked that nobody else is already working on it
```

**Example: Multi-Provider Migration**

```
Problem identified: 100% dependency on OpenAI creates risk
(outages, pricing changes, no negotiation leverage).

Impact quantified: $180K/year potential cost increase from
pricing changes. 2 outages in 6 months from provider issues.

Stakeholders identified: GenAI Platform team (build), Security
(review), Compliance (review), all consuming teams (affected).

Solution direction: Multi-provider abstraction layer with
automatic routing based on cost, availability, and capability.

Already being worked on? No.
```

### Phase 2: Design

**Your job:** Create a design that all affected teams can support.

```markdown
Design Checklist:
- [ ] Design document written using the standard template
- [ ] All affected teams have reviewed the design
- [ ] Security and compliance have reviewed (if applicable)
- [ ] Open questions are resolved or have owners
- [ ] Decision is formally approved by decision-makers
- [ ] Dissenting views are documented
```

**Key principle:** Don't write the design in isolation. Involve the people who will build it.

### Phase 3: Planning

**Your job:** Create a plan that is realistic, trackable, and adaptable.

```markdown
Planning Checklist:
- [ ] Work is broken into phases of 2-4 weeks each
- [ ] Each phase has clear success criteria
- [ ] Each team has a named owner for their work
- [ ] Dependencies between teams are mapped
- [ ] Risks are identified with mitigations
- [ ] Communication cadence is established (weekly updates)
- [ ] Rollback plan is defined for each phase
```

**The Planning Template:**

```markdown
# Initiative Plan: [Name]

## Phase 1: Foundation (Weeks 1-4)
| Task | Owner | Team | Dependencies | Success Criteria |
|------|-------|------|-------------|-----------------|
| Design multi-provider API contract | @sarah | GenAI Platform | Design approved | Contract reviewed by all consuming teams |
| Build provider abstraction interface | @james | GenAI Platform | API contract defined | Interface tested with mock providers |
| Security review kickoff | @security | Security | Design document | Review started |

## Phase 2: Implementation (Weeks 5-10)
| Task | Owner | Team | Dependencies | Success Criteria |
|------|-------|------|-------------|-----------------|
| Implement OpenAI provider adapter | @priya | GenAI Platform | Abstraction interface | Passes integration tests |
| Implement Anthropic provider adapter | @alex | GenAI Platform | Abstraction interface | Passes integration tests |
| Build routing logic | @sarah | GenAI Platform | Both adapters complete | Routes correctly based on config |

## Phase 3: Rollout (Weeks 11-14)
| Task | Owner | Team | Dependencies | Success Criteria |
|------|-------|------|-------------|-----------------|
| Deploy to staging, run load tests | @james | GenAI Platform | Implementation complete | Meets performance targets |
| Deploy to production (1% traffic) | @sarah | GenAI Platform | Staging validation | Zero errors |
| Gradual rollout (1% → 10% → 50% → 100%) | @sarah | GenAI Platform | 1% successful | 100% on new routing |

## Risks
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Security review takes longer than 4 weeks | Medium | High | Started review in Phase 1; engaged security early |
| Anthropic API has different rate limits | Low | Medium | Load testing will validate; adjust routing if needed |
| Consuming teams need to update their code | Medium | Medium | API contract is backward-compatible |
```

### Phase 4: Execution

**Your job:** Keep the initiative on track, unblock teams, communicate.

```markdown
Execution Checklist:
- [ ] Weekly progress update sent to stakeholders
- [ ] Blockers identified and escalated within 48 hours
- [ ] Scope changes are evaluated against impact
- [ ] Cross-team dependencies are actively managed
- [ ] Risks are monitored and mitigated
- [ ] Team morale is maintained (celebrate milestones)
```

**The Weekly Update:**

```markdown
# Weekly Update: Multi-Provider Migration — Week 6

## Status: 🟢 On Track

## Completed This Week
✅ OpenAI adapter implemented and tested
✅ Anthropic API integration started (60% complete)
✅ Security review: 2 of 4 sections completed

## Next Week
📋 Complete Anthropic adapter
📋 Start routing logic design
📋 Security review: sections 3-4

## Blockers
None

## Risks
⚠️ Security review finding on data residency may require
   design change. Discussing with security team this week.

## Metrics
- Phase completion: 45% (on track for Week 10 completion)
- Open issues: 3 (down from 7 last week)
- Budget spent: $12K of $80K (15%)
```

### Phase 5: Delivery

**Your job:** Ensure the initiative is fully delivered, not just "code merged."

```markdown
Delivery Checklist:
- [ ] All phases complete
- [ ] Security and compliance sign-offs received
- [ ] Rollout complete (100% traffic/users on new system)
- [ ] Old system decommissioned (if applicable)
- [ ] Documentation updated (runbooks, architecture docs)
- [ ] Support teams trained
- [ ] Monitoring and alerting in place
- [ ] Completion communicated to all stakeholders
- [ ] Lessons learned captured
```

### Phase 6: Operationalization

**Your job:** Ensure the system is sustainable after the initiative team moves on.

```markdown
Operationalization Checklist:
- [ ] On-call team has runbooks for all failure modes
- [ ] Monitoring dashboards are built and reviewed
- [ ] Alert thresholds are set and tested
- [ ] Regular maintenance procedures are documented
- [ ] Knowledge transfer to operations team complete
- [ ] Post-initiative review completed
```

## Driving Across Teams: Key Challenges

### Challenge: Competing Priorities

**Problem:** Your initiative is not the top priority for every team involved.

**Solution:**
1. **Get executive sponsorship.** A VP-level sponsor can align priorities across teams.
2. **Show the shared benefit.** How does this initiative help each team's own goals?
3. **Phase the work.** Let teams contribute when they have capacity, not all at once.
4. **Escalate priority conflicts.** If a team genuinely can't contribute, escalate to their manager to resolve the priority conflict.

### Challenge: Scope Creep

**Problem:** "While we're doing this, can we also add X?"

**Solution:**
1. **Define scope explicitly.** "This initiative covers X. It does not cover Y."
2. **Evaluate additions against criteria.** "Does X help us deliver this initiative faster or with less risk? If not, it's a separate initiative."
3. **Create a "Phase 2" backlog.** "That's a great idea. Let's capture it for Phase 2."
4. **Get sponsor agreement on scope.** If the sponsor keeps adding scope, have a direct conversation about timeline and resource impact.

### Challenge: Team Resistance

**Problem:** A team is slow to engage or actively resistant.

**Solution:**
1. **Understand the resistance.** Is it capacity? Priority? Genuine concern about the approach?
2. **Address the root cause.** If capacity, adjust the timeline. If priority, escalate. If genuine concern, engage with their argument.
3. **Don't go around them.** Work with the team, not around them. Going around them creates long-term resentment.
4. **Escalate if needed.** If a team is blocking the initiative without valid reason, escalate to the decision-maker who approved the initiative.

### Challenge: Losing Momentum

**Problem:** The initiative started strong but is losing energy.

**Solution:**
1. **Revisit the "why."** Remind everyone why this matters. Share the data.
2. **Celebrate milestones.** "We just completed Phase 2! Here's what we've achieved."
3. **Shorten feedback loops.** Weekly updates, visible progress tracking.
4. **Check resource allocation.** Has someone been pulled off the initiative? Do we need to re-up commitment?
5. **Be honest about timeline.** If the timeline has slipped, acknowledge it and reset expectations.

## Initiative Anti-Patterns

| Anti-Pattern | What It Looks Like | Impact |
|--------------|-------------------|--------|
| **Big bang delivery** | Everything deploys at once at the end | High risk, no early feedback |
| **No single owner** | "Everyone owns it" = nobody owns it | Things fall through the cracks |
| **No communication** | Teams work in silos, no visibility | Misalignment, duplicated work |
| **Scope by committee** | Every team adds their requirements | Initiative never finishes |
| **No exit criteria** | "Done" is ambiguous | Initiative limps along indefinitely |
| **Ignoring reality** | Plan says one thing, reality says another | Plan becomes fiction, not a guide |
| **No operationalization** | Code shipped, nobody knows how to run it | Immediate tech debt |

## Measuring Initiative Success

| Metric | Why |
|--------|-----|
| **On-time delivery** | Did we deliver when we said we would? |
| **Within budget** | Did we spend what we said we would? |
| **Quality of outcome** | Did the initiative achieve its stated goals? |
| **Team satisfaction** | Was the experience positive for the teams involved? |
| **Operational readiness** | Can the operations team run this without the initiative team? |

## Case Study: Driving the GenAI Audit Logging Standard

> **Situation:** No standard for audit logging across GenAI systems. Each team logged differently, making compliance reviews painful and incident investigation difficult.
>
> **Ideation (2 weeks):** Analyzed 5 existing GenAI systems. Found 5 different logging formats. Compliance team confirmed this was their #1 pain point. Quantified: compliance reviews took 2x longer because auditors had to learn each format.
>
> **Design (3 weeks):** Wrote design doc for standard audit event schema. Reviewed with all 5 teams, security, and compliance. Addressed concerns about schema flexibility (added extension fields). Approved by security lead and engineering director.
>
> **Planning (1 week):** 3 phases: (1) Schema definition and library, (2) Migration of 3 highest-priority systems, (3) Migration of remaining systems + enforcement.
>
> **Execution (8 weeks):** Weekly updates. One team was slow to start (competing priority) — escalated to their manager, got dedicated resource. Schema library published as internal package.
>
> **Delivery (2 weeks):** All 5 systems migrated. Compliance team reported 40% reduction in review time. Enforcement added to CI pipeline (new services must use the standard library).
>
> **Total duration:** 16 weeks. **Outcome:** Standard adopted, compliance reviews faster, incident investigation easier. **Lesson:** Early engagement with the compliance team was the highest-leverage activity — their support gave the initiative credibility with other teams.

## Cross-References

- `leadership-and-collaboration/building-alignment.md` — Getting teams aligned
- `leadership-and-collaboration/influencing-without-authority.md` — Influence strategies
- `leadership-and-collaboration/managing-stakeholders.md` — Stakeholder management
- `leadership-and-collaboration/writing-strategy-docs.md` — Strategy document writing
- `engineering-culture/design-docs.md` — Design document process
- `engineering-culture/rfcs.md` — RFC process for lightweight proposals
