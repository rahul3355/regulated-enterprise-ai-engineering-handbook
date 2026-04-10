# First 90 Days: Ownership and Leadership

> **Audience:** Senior Software Engineer, GenAI Platform Team (Days 61-90)
> **Purpose:** Transition from contributor to owner — service ownership, mentoring, project leadership
> **Prerequisites:** Completed `first-60-days.md` milestones, at least 1 successful on-call shift

---

## Overview

Days 61-90 are where you stop being "the new senior engineer" and start being "a senior engineer on the team." The expectations shift:

| Metric | Days 1-30 | Days 31-60 | Days 61-90 |
|--------|-----------|------------|------------|
| PRs per week | 1-2 (small) | 2-3 (medium) | 3-5 (mix of sizes) |
| On-call | Shadow only | With backup | Independent |
| Incidents | Observe | Participate | Lead response |
| Code reviews | Receive only | Give + receive | Give thorough reviews |
| Design input | None | Informal | Formal design docs |
| Mentoring | None | None | Mentor someone |
| Project ownership | None | Task ownership | Service ownership |

In a regulated banking environment, Day 90 is also when you are expected to understand compliance well enough to self-govern. Your manager should not need to remind you about change management, audit logging, or security review requirements.

---

## Week 9-10: Service Ownership

### What Service Ownership Means

When you own a service, you are responsible for:

1. **Operational health** — SLOs, error budgets, incident response
2. **Code quality** — Architecture, testing, technical debt
3. **Documentation** — Runbooks, API docs, architecture decision records
4. **Roadmap** — What needs to change and why
5. **Consumer support** — Onboarding new consumers, debugging their issues

**Services Available for Ownership:**

| Service | Current Owner | Complexity | Why You Might Own It |
|---------|--------------|------------|---------------------|
| Embedding Service | ML Platform Engineer | Medium | You have ML background |
| API Gateway Consumer Config | API Gateway Engineer | Low | You understand consumer needs |
| Guardrails Policy Engine | Security Engineer | High | You have security interest |
| Audit Logging Pipeline | SRE Engineer | Medium | You understand observability |
| RAG Document Indexer | Data Engineer | High | You understand data pipelines |
| Cost Tracking Service | FinOps Engineer | Low | You want a gentle start |

**Discuss with your Engineering Manager** which service is the best fit based on your background and growth goals.

### Service Ownership Checklist

When you take ownership of a service, work through this checklist:

```
SERVICE OWNERSHIP ONBOARDING
Service: ___________________

1. CODEBASE
   [ ] Clone the repository, verify it builds locally
   [ ] Read the README and architecture docs
   [ ] Identify the top 5 files by change frequency (git log --oneline | head)
   [ ] Identify areas with low test coverage
   [ ] List all dependencies and their versions
   [ ] Note any TODO/FIXME/HACK comments in the code

2. INFRASTRUCTURE
   [ ] Identify all environments (dev, staging, prod)
   [ ] Check resource allocations (CPU, memory, storage)
   [ ] Review autoscaling configuration
   [ ] Verify backup and disaster recovery procedures
   [ ] Check network policies and security groups
   [ ] Identify all external dependencies (databases, APIs, queues)

3. OBSERVABILITY
   [ ] Review existing dashboards
   [ ] Check all active alerts
   [ ] Verify alert thresholds are appropriate
   [ ] Check logging completeness and quality
   [ ] Verify distributed tracing is enabled
   [ ] Test alert routing (fire a test alert)

4. DEPLOYMENT
   [ ] Walk through the deployment pipeline
   [ ] Verify rollback procedures work
   [ ] Check deployment frequency and success rate
   [ ] Review the last 5 deployments for issues
   [ ] Understand feature flag usage
   [ ] Verify canary/blue-green deployment is configured

5. DOCUMENTATION
   [ ] Review runbooks (are they current?)
   [ ] Check API documentation (OpenAPI/Swagger)
   [ ] Verify Architecture Decision Records (ADRs) are complete
   [ ] Check service catalog entry is accurate
   [ ] Review the incident history and post-mortems
   [ ] Identify documentation gaps

6. COMPLIANCE
   [ ] Verify audit logging meets retention requirements
   [ ] Check data classification labels are correct
   [ ] Review access controls (who has admin access?)
   [ ] Verify encryption in transit and at rest
   [ ] Check PII handling procedures
   [ ] Review last security assessment findings

7. ROADMAP
   [ ] List known technical debt
   [ ] Identify performance bottlenecks
   [ ] Note any upcoming regulatory changes that affect the service
   [ ] Plan capacity for next 2 quarters
   [ ] Identify opportunities for improvement
   [ ] Draft a 30/60/90 day plan for the service
```

### Architecture Decision Records (ADRs)

As a service owner, you will write and maintain ADRs. Here is the format we use:

```markdown
# ADR-0147: Migrate Embedding Service to Multi-Model Architecture

## Status
Accepted

## Context
The embedding service currently uses a single model (bank-internal/multilingual-embedding-v3)
for all consumers. This creates problems:

1. Different consumers have different quality/latency requirements. 
   Investment research needs high-quality embeddings (768-dim) but can tolerate 100ms latency.
   Chat consumers need fast embeddings (128-dim, <50ms latency) but can accept lower quality.

2. The current model is optimized for English, but we are onboarding consumers in 
   Mandarin and Arabic. The multilingual extension reduces quality by 15% for English queries.

3. Cost: The 768-dim model costs 3x more per inference than the 128-dim model.
   Chat consumers are paying for quality they do not need.

## Decision
Migrate to a multi-model architecture where:
- Consumers specify their embedding requirements in their config
- The service routes to the appropriate model based on requirements
- Default model remains the 768-dim multilingual model for backward compatibility

Model tiers:
- TIER_1: 128-dim English model (latency-optimized, cost-optimized)
- TIER_2: 256-dim multilingual model (balanced)
- TIER_3: 768-dim multilingual model (quality-optimized, current default)

## Consequences

### Positive
- 40% cost reduction for chat consumers (TIER_1)
- Improved latency for chat consumers (35ms vs 80ms P50)
- Better quality for non-English consumers (dedicated multilingual models)
- Clear quality/cost trade-offs for consumers to choose

### Negative
- Increased complexity in service routing logic
- Need to maintain 3 model deployments instead of 1
- Consumers must understand and choose their tier (education required)
- Migration requires consumer config updates

### Risk Mitigation
- Default tier remains unchanged (no breaking change)
- Consumers are notified and supported through migration
- Monitoring added to detect incorrect tier selection
- Rollback plan: single model config can be restored in 5 minutes

## Compliance Notes
- All tiers use models approved by the AI Governance Board (ref: AIG-2025-0034)
- Model performance metrics logged to compliance dashboard
- No PII exposure differences between tiers

## Reviewers
- Tech Lead: @sarah.chen
- ML Platform: @james.rodriguez
- Security: @priya.patel
- Compliance: @marcus.wright (approved)

## Date
2025-03-15
```

---

## Week 11: Mentoring

### Why Mentoring Matters

As a senior engineer, mentoring is not optional — it is part of your job description. Mentoring:

1. **Scales your impact** — You multiply your effectiveness through others
2. **Improves team resilience** — More people who understand a system means fewer single points of failure
3. **Develops your leadership** — Teaching forces you to articulate and clarify your thinking
4. **Builds team culture** — How you treat newer engineers sets the tone for the team

### Your First Mentee

You will be assigned a mentee — likely a mid-level engineer who joined recently or an intern starting their placement.

**Mentoring Framework:**

```
WEEKLY MENTORING STRUCTURE

1. Weekly 1:1 (45 minutes)
   - 15 min: What did you accomplish this week?
   - 15 min: What are you blocked on or struggling with?
   - 15 min: Career development and growth discussion

2. Async Support
   - Mentee can DM you with questions
   - Response SLA: 4 hours during business hours
   - For urgent issues, use the team channel instead

3. Code Review Mentorship
   - Review mentee's PRs with detailed explanations
   - Explain WHY something should change, not just WHAT
   - Encourage them to review your PRs and learn from feedback

4. Pair Programming (biweekly)
   - Work on a complex task together
   - Let them drive, you navigate
   - Share your debugging process and heuristics

MENTEE PROGRESS TRACKING (for your 1:1 with manager)
Week 1: Establish goals and expectations
Week 2: First code contribution with guidance
Week 4: Independent contribution to a feature
Week 6: Leading a small feature end-to-end
Week 8: Reviewing others' code
Week 10: Mentoring someone newer (yes, really)
```

**Mentoring Anti-Patterns:**

| Anti-Pattern | What It Looks Like | What to Do Instead |
|-------------|-------------------|-------------------|
| The Helicopter | Hovering over every line of code, fixing everything yourself | Let them struggle productively. Ask questions, do not give answers. |
| The Ghost | "My door is always open" but never actually available | Proactively schedule check-ins. Be predictably available. |
| The Critic | Every review is a list of things wrong, nothing positive | Balance critique with praise. Explain what they did right. |
| The Answer Machine | Always giving the answer immediately | Ask "What have you tried?" and "What do you think?" first. |
| The Scope Creep | Assigning work that is far beyond their level | Start small. Increase complexity as they grow. |

### Story: When Mentoring Went Wrong

> In 2023, a senior engineer (let us call them Sam) was assigned to mentor a junior engineer (Jordan). Sam was brilliant but impatient. When Jordan struggled with Kubernetes concepts, Sam would fix the configuration themselves and say "just copy this pattern."
>
> After 3 months, Jordan could deploy changes but had no idea why the deployment worked. When a production incident occurred and Sam was on vacation, Jordan was unable to troubleshoot because they had never learned the debugging process.
>
> **What went wrong:**
> - Sam optimized for short-term productivity over long-term growth
> - Jordan learned procedures but not principles
> - No one checked Jordan's understanding until it was too late
>
> **What should have happened:**
> - Sam should have spent 30 minutes explaining the Kubernetes networking model
> - Jordan should have been given a safe environment (dev cluster) to break things and learn
> - Weekly check-ins with the engineering manager should have assessed Jordan's actual understanding, not just their output
>
> **Outcome:** Jordan transferred to a different team after 6 months, citing "lack of growth opportunities." The team lost a capable engineer because mentoring failed.

---

## Week 12-13: Leading a Project

### Project Selection

By Day 60-75, you should identify and lead a project that:

1. **Delivers value** — Solves a real problem for consumers or the team
2. **Is scoped appropriately** — Achievable in 4-6 weeks by you + 1-2 engineers
3. **Has stakeholder buy-in** — Someone cares about the outcome
4. **Is visible** — The team and leadership will see the results

**Good First Projects:**

| Project | Impact | Complexity | Stakeholders |
|---------|--------|------------|-------------|
| Implement per-consumer token budgeting | High (cost control) | Medium | FinOps, Product Owners |
| Add response caching for common queries | High (latency + cost) | Low | All consumers |
| Create automated document freshness checker | Medium (quality) | Medium | Data Engineering, Compliance |
| Build guardrails A/B testing framework | Medium (quality measurement) | High | Security, ML Platform |
| Improve RAG retrieval quality for multilingual queries | High (consumer satisfaction) | High | ML Platform, International teams |
| Add distributed tracing across all services | High (observability) | Medium | SRE, All engineering |

### Project Execution Framework

```
PROJECT LIFECYCLE

PHASE 1: DEFINITION (Week 1)
├── Problem statement (1 paragraph)
├── Success metrics (how will we know it worked?)
├── Constraints (budget, timeline, regulatory)
├── Stakeholders and their interests
├── Risks and mitigation strategies
└── Go/No-Go criteria

PHASE 2: DESIGN (Week 2)
├── Architecture diagram
├── Data flow description
├── API contracts (if applicable)
├── Infrastructure requirements
├── Security and compliance review (early!)
├── ADR drafted
└── Design review meeting with team

PHASE 3: IMPLEMENTATION (Weeks 3-4)
├── Sprint planning with scoped tasks
├── Daily progress updates (standup)
├── Weekly stakeholder updates
├── Code reviews as you go
├── Tests written alongside code
└── Documentation updated continuously

PHASE 4: DEPLOYMENT (Week 5)
├── Staging deployment and validation
├── Performance testing (latency, throughput, cost)
├── Security review completion
├── CAB submission (if required)
├── Feature flag configuration
├── Runbook creation/update
├── Monitoring and alerting setup
└── Rollback plan tested

PHASE 5: ROLLOUT AND RETROSPECTIVE (Week 6)
├── Progressive rollout (canary -> full)
├── Monitor success metrics
├── Stakeholder demo
├── Retrospective: What went well? What could improve?
├── Handover documentation (if service ownership transfers)
└── Post-project write-up for the team
```

### Example Project: Per-Consumer Token Budgeting

**Problem Statement:**

Currently, all consumers share a single token budget across all AI provider accounts. When one consumer experiences a traffic spike (e.g., during a marketing campaign), it consumes tokens that were budgeted for other consumers. This causes other consumers to hit rate limits or receive degraded service. We need per-consumer token budgets with the ability to allocate, monitor, and adjust token usage at the consumer level.

**Success Metrics:**

| Metric | Current | Target |
|--------|---------|--------|
| Cross-consumer impact incidents | 3 per quarter | 0 per quarter |
| Token budget predictability | None (shared pool) | Per-consumer allocation with 95% accuracy |
| Cost attribution | Platform-level only | Per-consumer cost reporting |
| Emergency rate limit applications | Reactive | Proactive (budget-based) |

**Architecture:**

```
┌──────────────────────────────────────────────────────┐
│                Token Budget Manager                   │
├──────────────────────────────────────────────────────┤
│                                                       │
│  ┌─────────────┐    ┌─────────────┐    ┌───────────┐ │
│  │  Budget     │    │  Usage      │    │  Alert    │ │
│  │  Allocator  │───▶│  Tracker    │───▶│  Engine   │ │
│  │             │    │             │    │           │ │
│  │ Per-consumer│    │ Real-time   │    │ Threshold │ │
│  │ allocations │    │ token count │    │ breach    │ │
│  │ per-provider│    │ per-consumer│    │ detection │ │
│  └─────────────┘    └─────────────┘    └───────────┘ │
│         │                  │                  │       │
│         │                  │                  │       │
│         ▼                  ▼                  ▼       │
│  ┌─────────────────────────────────────────────────┐ │
│  │               Redis (Hot Storage)                │ │
│  │  consumer:investment-research-bot:tokens:used    │ │
│  │  consumer:wealth-management-chat:tokens:used     │ │
│  │  consumer:compliance-assistant:tokens:used       │ │
│  └─────────────────────────────────────────────────┘ │
│                                                       │
│  ┌─────────────────────────────────────────────────┐ │
│  │          PostgreSQL (Cold Storage / Audit)       │ │
│  │  Daily aggregates, historical trends, forecasts  │ │
│  └─────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

**Implementation: Budget Enforcement Middleware**

```python
# genai_platform/api_gateway/middleware/token_budget.py
"""
Token Budget Enforcement Middleware

Intercepts requests and checks if the consumer has remaining token budget.
If budget is exhausted, the request is rejected with a 429 status code.

Budget calculation:
- Each consumer has a monthly token allocation per provider
- Usage is tracked in real-time via Redis
- Budget resets at 00:00 UTC on the 1st of each month
- Consumers can request budget top-ups (requires FinOps approval)

Compliance: All budget decisions are logged for audit.
"""

import time
from datetime import datetime, timezone
from typing import Optional
import redis.asyncio as redis
import structlog

logger = structlog.get_logger()

class TokenBudgetEnforcer:
    def __init__(
        self,
        redis_client: redis.Redis,
        budget_prefix: str = "token_budget",
    ):
        self.redis = redis_client
        self.budget_prefix = budget_prefix
    
    def _budget_key(self, consumer_id: str, provider: str, month: str) -> str:
        return f"{self.budget_prefix}:{consumer_id}:{provider}:{month}"
    
    def _usage_key(self, consumer_id: str, provider: str, month: str) -> str:
        return f"{self.budget_prefix}:usage:{consumer_id}:{provider}:{month}"
    
    async def check_budget(
        self,
        consumer_id: str,
        provider: str,
        estimated_tokens: int,
    ) -> tuple[bool, dict]:
        """
        Check if the consumer has sufficient token budget.
        
        Args:
            consumer_id: The consumer making the request
            provider: The AI provider being used (azure-openai, internal-llm, etc.)
            estimated_tokens: Estimated token usage for this request
        
        Returns:
            Tuple of (allowed, metadata)
        """
        now = datetime.now(timezone.utc)
        month_key = now.strftime("%Y-%m")
        
        budget_key = self._budget_key(consumer_id, provider, month_key)
        usage_key = self._usage_key(consumer_id, provider, month_key)
        
        # Get budget allocation (cached in Redis, refreshed hourly)
        budget = await self.redis.get(budget_key)
        if budget is None:
            # Budget not set — this is a configuration error
            # Default to ALLOW but log the issue
            logger.warning(
                "token_budget_not_configured",
                consumer_id=consumer_id,
                provider=provider,
                action="allowing_by_default",
            )
            return True, {
                "reason": "budget_not_configured",
                "action": "allowed_with_warning",
            }
        
        budget = int(budget)
        
        # Get current usage
        usage = await self.redis.get(usage_key)
        usage = int(usage) if usage else 0
        
        remaining = budget - usage
        
        if remaining >= estimated_tokens:
            # Budget sufficient — record the estimated usage
            await self.redis.incrby(usage_key, estimated_tokens)
            await self.redis.expire(usage_key, 30 * 24 * 3600)  # 30 days TTL
            
            return True, {
                "reason": "budget_sufficient",
                "remaining_tokens": remaining - estimated_tokens,
                "budget": budget,
                "usage": usage + estimated_tokens,
                "utilization_pct": ((usage + estimated_tokens) / budget) * 100,
            }
        
        # Budget would be exceeded
        utilization_pct = (usage / budget) * 100
        
        logger.warning(
            "token_budget_exceeded",
            consumer_id=consumer_id,
            provider=provider,
            budget=budget,
            usage=usage,
            remaining=remaining,
            estimated_tokens=estimated_tokens,
            utilization_pct=utilization_pct,
        )
        
        return False, {
            "reason": "budget_exceeded",
            "remaining_tokens": remaining,
            "budget": budget,
            "usage": usage,
            "utilization_pct": utilization_pct,
            "retry_after": self._calculate_retry_after(consumer_id, provider),
        }
    
    def _calculate_retry_after(self, consumer_id: str, provider: str) -> int:
        """
        Calculate seconds until the next budget reset.
        For monthly budgets, this is the time until the 1st of next month.
        """
        now = datetime.now(timezone.utc)
        if now.month == 12:
            next_month = now.replace(year=now.year + 1, month=1, day=1, 
                                     hour=0, minute=0, second=0, microsecond=0)
        else:
            next_month = now.replace(month=now.month + 1, day=1,
                                     hour=0, minute=0, second=0, microsecond=0)
        return int((next_month - now).total_seconds())
```

**Stakeholder Communication Plan:**

| Stakeholder | Interest | Communication | Frequency |
|------------|----------|--------------|-----------|
| FinOps team | Cost attribution and predictability | Budget allocation spreadsheet, usage reports | Weekly during project, monthly after |
| Product Owners | Consumer experience impact | Impact assessment per consumer | At design review and before rollout |
| Security team | Audit logging of budget decisions | Security review of the enforcement logic | At design phase |
| Engineering Manager | Project progress and risks | Written update in weekly 1:1 | Weekly |
| Tech Lead | Architecture quality | Design review, code review | At milestones |
| Consumer teams | How to configure and use their budget | Documentation, office hours | Before and after rollout |

### Project Retrospective Template

```
PROJECT RETROSPECTIVE: [Project Name]
Date: ____
Participants: ____

WHAT WENT WELL:
(List things that should be repeated in future projects)
- 
- 
- 

WHAT COULD IMPROVE:
(List things that should change in future projects)
- 
- 
- 

ACTION ITEMS:
(Specific changes to implement, with owners and deadlines)
- [ ] ____ Owner: ____ Deadline: ____
- [ ] ____ Owner: ____ Deadline: ____
- [ ] ____ Owner: ____ Deadline: ____

SURPRISES:
(What happened that nobody expected?)
- 
- 

METRICS:
(Did we hit our targets?)
| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
|        |        |        |        |

LESSONS LEARNED:
(What will we do differently next time?)
- 
- 
- 
```

---

## Week 14: Interview Readiness

### Why Interview Readiness Matters

By Day 90, you should be able to articulate your contributions to the team in an interview context. This is not about preparing to leave — it is about:

1. **Performance reviews** — You need to describe your impact clearly
2. **Promotion cases** — Senior engineers must demonstrate scope and ownership
3. **Internal mobility** — Other teams may want to recruit you
4. **External opportunities** — Even if you are not looking, being ready is good

### Documenting Your Impact

Create a "Brag Document" that captures your contributions:

```
BRAG DOCUMENT: [Your Name]
Period: [Start Date] — [Current Date]
Role: Senior Software Engineer, GenAI Platform Team

IMPACT SUMMARY:
- Owned [Service Name] end-to-end, maintaining 99.95% SLO
- Led [Project Name], delivering [specific measurable outcome]
- Mentored [Mentee Name], who independently contributed [their outcome]
- Responded to [N] incidents, led response to [specific incident]
- Contributed [N] PRs, reviewed [N] PRs from other team members
- Improved [specific process or system], resulting in [measurable improvement]

TECHNICAL CONTRIBUTIONS:
1. [Feature/System] — Description, impact, complexity
   - Technologies: Python, Kubernetes, Redis, Azure OpenAI
   - Outcome: Reduced latency by 40%, saved $X/month in token costs
   - Challenge: [What made this hard?]
   - Approach: [How did you solve it?]

2. [Guardrails improvement]
   - Technologies: Python, ML models, OWASP LLM Top 10
   - Outcome: Reduced injection detection false positives by 15%
   - Compliance: Approved by Security team, passed audit

3. [On-call improvement]
   - Technologies: Grafana, PagerDuty, Runbook updates
   - Outcome: Reduced mean time to recovery (MTTR) by 20%
   - Impact: Updated 8 runbooks, created 2 new ones

LEADERSHIP CONTRIBUTIONS:
1. Mentored [Name] — weekly 1:1s, code review guidance, pair programming
2. Led design review for [Project] — facilitated discussion, drove consensus
3. Presented [Topic] at team knowledge sharing — 20-person audience

COMPLIANCE AND PROCESS:
1. Passed security review for all changes (0 findings)
2. Maintained 100% change management compliance
3. Contributed to [N] post-mortems, tracked [N] remediation actions to completion

GROWTH AREAS:
1. Deepened knowledge of Kubernetes networking and service mesh
2. Learned the bank's change management and compliance processes
3. Developed mentoring and project leadership skills

GOALS FOR NEXT QUARTER:
1. Lead a cross-team initiative (involving 2+ teams)
2. Contribute to the AI governance and policy process
3. Present at an internal engineering conference
4. Improve the onboarding process based on my experience
```

### Interview Preparation: Common Questions and How to Answer

**Question: "Tell me about a time you dealt with a production incident."**

Structure your answer using the STAR method:

```
Situation:
"In my second month, the guardrails injection detection rate dropped from 96% to 78% 
at 3 AM. This was a P1 incident because it meant potentially malicious prompts were 
getting through to the LLM without detection."

Task:
"As the on-call engineer (with backup), my job was to restore service to normal 
as quickly as possible, following the runbook."

Action:
"I followed the runbook for guardrails detection rate drops. The first step was to 
check recent deployments. I found that a new guardrails model (v2.4) had been 
deployed at 02:30. I checked the model evaluation report and found that its accuracy 
was 94.2%, below our 95% threshold. The deployment had been allowed through because 
the accuracy gate was configured as a warning rather than a blocking check — a change 
made in a previous PR that had not been reviewed by the Security team.

I made the decision to roll back to v2.3 immediately. The rollback took 5 minutes, 
and the detection rate returned to 95.5% within 3 minutes of the rollback completing.

After the incident, I filed the incident report, scheduled the post-mortem, and 
proposed a fix: changing the accuracy gate from WARNING to BLOCKING. I also identified 
that the PR which changed the gate type had been approved by a contractor who was 
not aware of the policy — so I recommended adding a policy check to the PR template."

Result:
"The rollback restored service within 16 minutes of the alert firing. The post-mortem 
produced 4 remediation actions, all of which were completed within 2 sprints. The 
policy check I recommended was added to the PR template and has prevented similar 
issues since. My engineering manager cited my response as an example of good 
incident handling in the team retrospective."
```

**Question: "Describe a technical decision you made and its impact."**

```
Situation:
"When I took ownership of the embedding service, I noticed that all consumers were 
using the same 768-dimensional multilingual model, even though some consumers 
(chat interfaces) only needed fast, simple embeddings."

Task:
"I needed to reduce costs for chat consumers without degrading quality for 
research consumers."

Action:
"I designed a multi-model architecture with three tiers. I wrote the ADR, 
presented it at design review, and incorporated feedback from the ML Platform 
team and the Security team. I implemented the routing logic and configured 
the additional model deployments. I worked with each consumer team to assess 
their needs and migrate them to the appropriate tier."

Result:
"Chat consumers saw a 40% cost reduction and 55% latency improvement (35ms vs 80ms P50).
Research consumers were unaffected. The platform saved approximately $12,000/month 
in AI provider costs. The ADR was cited as an example of good architectural decision-making 
in the team's engineering excellence review."
```

---

## End of Month 3: Self-Assessment

### Ownership Check

**Can you say yes to all of these?**

- [ ] I own at least one service end-to-end and can speak to every aspect of it
- [ ] I have completed the full service ownership checklist for my service
- [ ] I have written at least one ADR that was accepted
- [ ] I have updated at least 3 runbooks based on my operational experience
- [ ] I can deploy and roll back my service without assistance
- [ ] I can debug any alert related to my service without looking at the runbook

### Mentoring Check

- [ ] I have a mentee and we have a regular 1:1 cadence
- [ ] My mentee has made at least 5 independent contributions
- [ ] I have received feedback that my mentoring is helpful
- [ ] I have helped my mentee grow in at least one measurable way

### Project Leadership Check

- [ ] I have led at least one project from definition to retrospective
- [ ] The project delivered measurable value (I can cite the numbers)
- [ ] Stakeholders were satisfied with the outcome
- [ ] The retrospective produced actionable improvements

### Interview Readiness Check

- [ ] I have a current brag document
- [ ] I can answer "tell me about your role" in 2 minutes
- [ ] I can describe at least 3 technical contributions with impact metrics
- [ ] I can describe at least 1 leadership contribution
- [ ] I have practiced answering incident response questions using STAR
- [ ] I could pass an interview for my current role today

### Trust Check

**Signs you are now a trusted team member:**

- Other engineers come to you for advice on your area of ownership
- You are included in architecture discussions before implementation begins
- Your opinions in sprint planning influence task prioritization
- You are the first escalation for certain types of issues
- Newer engineers ask you for help before asking the team lead
- Your engineering manager delegates decisions to you
- You are invited to meetings that are not strictly about your work

---

## What Comes Next

After 90 days, you are a full member of the team. The next milestones are:

1. **6 Months** — Cross-team project leadership, AI governance contributions
2. **1 Year** — Staff engineer readiness, platform strategy input
3. **Ongoing** — Continuous improvement, mentoring the next wave of engineers

**Remember:** The first 90 days are about proving you can do the job. The rest of your time here is about deciding what job you want to do next.

---

*Last updated: April 2025 | Document owner: Engineering Enablement Team | Review cycle: Quarterly*
