# How to Earn Trust on the Team

> **Audience:** All engineers joining the GenAI Platform Team
> **Purpose:** Practical advice for becoming a reliable, trusted team member
> **Prerequisites:** None — read this in Week 1, revisit monthly

---

## What Trust Looks Like

Trust on an engineering team is not about being the smartest person in the room. It is about being **predictably reliable**. A trusted engineer:

1. **Does what they say they will do** — If they commit to something, it gets done
2. **Communicates proactively** — No one has to chase them for updates
3. **Owns their mistakes** — Errors are admitted quickly, fixed thoroughly, documented honestly
4. **Helps others** — Knowledge is shared, not hoarded
5. **Respects the process** — Follows procedures even when they feel slow

**Trust is earned in drops and lost in buckets.** Every reliable delivery adds a drop. Every missed commitment, every hidden mistake, every skipped process drains the bucket.

---

## The Trust Timeline

### Days 1-30: Prove You Are Safe

In your first month, the team wants to know:

- Can you follow instructions?
- Will you ask before doing something risky?
- Do you respect the process?
- Are you paying attention?

**How to build trust in Month 1:**

```
DO:
- Read the documentation before asking questions
- Ask questions after reading the documentation (shows you tried)
- Follow the PR process exactly as documented
- Admit when you do not understand something
- Show up on time for meetings, prepared
- Take notes when someone explains something to you
- Write down processes so you do not ask the same question twice
- Thank people who help you

DO NOT:
- Push to production without approval
- Skip the PR review process
- Assume you know better than the documentation
- Ask questions that are answered in the README
- Complain about process being slow (it feels slow for a reason)
- Make changes to areas you do not understand yet
```

**Specific trust-building actions in Month 1:**

1. **Fix a documentation gap.** You are the freshest pair of eyes the docs will get for months. If something confused you, the next person will be confused too. Fix it.

2. **Add tests to an area with low coverage.** This is low-risk, high-value work that shows you care about quality.

3. **Review a PR thoroughly.** Not just "looks good to me" — actually read the code, test it locally, find something the author missed.

4. **Show up to an incident as an observer.** Do not try to help (you do not know enough yet). Just watch, learn, and take notes.

### Days 31-60: Prove You Are Competent

In your second month, the team wants to know:

- Can you work independently?
- Do you understand the system well enough to change it safely?
- Will you catch your own mistakes before they reach production?
- Can you be trusted with production systems?

**How to build trust in Month 2:**

```
DO:
- Take ownership of a small feature end-to-end
- Catch your own mistakes in code review before reviewers point them out
- Proactively update stakeholders on your progress
- Offer to help with on-call tasks (even if you are shadowing)
- Share what you have learned with the team (write a doc, give a short talk)
- Review PRs from other team members and provide useful feedback

DO NOT:
- Deploy to production without following the process
- Ignore an alert because you assume someone else is handling it
- Hide a mistake you made (admit it immediately)
- Blame tools, processes, or other people for problems
- Over-commit and under-deliver
```

**Specific trust-building actions in Month 2:**

1. **Complete your first on-call shift without incident.** If an alert fires, follow the runbook. If you are unsure, escalate. Your backup will remember how you handled it.

2. **Lead a production deployment.** From the CAB submission through the post-deployment monitoring. Get it right and the team will trust you with production.

3. **Identify and fix a production issue.** Not a bug in code — an operational issue. A misconfigured alert, an outdated runbook, a missing dashboard. These show you care about the system, not just the code.

4. **Mentor someone.** Even if it is just helping another new engineer with their setup. Teaching proves understanding.

### Days 61-90: Prove You Are a Leader

In your third month, the team wants to know:

- Can you lead a project?
- Do you make good technical decisions?
- Do other engineers seek you out for advice?
- Are you invested in the team's success, not just your own work?

**How to build trust in Month 3:**

```
DO:
- Propose a project and drive it from definition to delivery
- Make decisions based on evidence, not opinion
- Give credit to others for their contributions
- Take blame when things go wrong on your project
- Invest in the team's processes and tools
- Push back respectfully when you disagree with a decision

DO NOT:
- Hoard knowledge to make yourself indispensable
- Dismiss others' concerns without consideration
- Make unilateral decisions that affect other people
- Complain about the team in public (or private)
- Focus only on your work and ignore team needs
```

**Specific trust-building actions in Month 3:**

1. **Deliver a project that matters.** Something that improves the platform, saves money, or reduces risk. Make it visible.

2. **Handle an incident well.** When something breaks, be the person who stays calm, follows the process, and gets the service back online.

3. **Improve a team process.** The onboarding docs, the PR template, the deployment checklist — something that makes the team better after you joined.

4. **Be the person someone else trusts.** When a newer engineer asks you for help, help them well. Trust is transitive — if the team trusts you and you trust someone, the team starts trusting them too.

---

## Trust-Building Behaviors (Detailed)

### 1. Communication

**The single most important trust-building behavior is proactive communication.**

```
GOOD COMMUNICATION:
- "I am working on PROJ-1234. I expect to have the PR ready by Thursday."
- "I ran into an issue with the vector DB connection. Investigating now. Will update by 2 PM."
- "I will not make the Friday deadline. I need until Monday. Here is what is blocking me."
- "I made a mistake in the deployment. Here is what happened, what I am doing to fix it, and how I will prevent it."

BAD COMMUNICATION:
- Silence until someone asks for an update
- "Almost done" for three days in a row
- "It works on my machine" as an explanation for a production failure
- "I thought someone else was handling it"
```

**The 3-Update Rule:**

For any task that takes more than 2 days, provide at least 3 updates:

1. **Starting** — "I am starting work on this. Here is my plan."
2. **Progress** — "Here is where I am. Here is what I found."
3. **Done (or blocked)** — "Here is the result" or "Here is what is blocking me."

### 2. Ownership

**Ownership means seeing something through, not just completing your task.**

```
TASK OWNER:
"I will do the task" → "I will make sure the task is done"

The difference:
- Doing the task: writing the code, submitting the PR
- Making sure the task is done: writing the code, getting the PR reviewed, 
  deploying it, verifying it works, updating the docs, monitoring for issues

Engineers who "make sure the task is done" are trusted more than engineers 
who "do the task."
```

**Example of real ownership:**

```
SCENARIO: You are asked to add a new guardrails check.

Minimum ownership:
- Write the check
- Write tests
- Submit PR
- Get it merged

Full ownership:
- Write the check
- Write tests
- Submit PR
- Get it merged
- Update the guardrails documentation
- Configure the alert for the new check
- Update the runbook
- Brief the on-call team on what to look for
- Monitor the check in production for the first week
- Report on its effectiveness (detection rate, false positive rate)

The engineer who demonstrates full ownership becomes a trusted team member.
```

### 3. Mistake Handling

**How you handle mistakes determines your trust level more than never making mistakes.**

```
THE MISTAKE HIERARCHY:

Best: Catch your own mistake before anyone else sees it
      "I noticed an issue in my PR. Pushing a fix."
      Trust impact: POSITIVE (shows diligence)

Good: Catch your own mistake after it is merged but before production
      "I found a bug in what I merged yesterday. Filing a PR to fix it."
      Trust impact: NEUTRAL (everyone makes mistakes)

Acceptable: Someone else catches your mistake in code review
      "Good catch. I will fix that."
      Trust impact: SLIGHTLY NEGATIVE (but expected for new engineers)

Bad: The mistake reaches production
      Trust impact: NEGATIVE (but recoverable with good response)

Worst: The mistake reaches production and you hide it
      Trust impact: SEVERE (may be unrecoverable)

The worst outcome is not the mistake — it is the hiding.
```

**Real Example: The Hidden Configuration Error**

> An engineer (let us call them Chris) deployed a configuration change that accidentally disabled rate limiting for one consumer. Chris noticed the issue 20 minutes after deployment but thought "no one will notice, I will fix it quietly."
>
> Someone noticed. The consumer's traffic spiked 10x and caused increased costs ($2,000 in excess token usage over 3 hours). The spike was detected by the cost tracking system.
>
> When the incident was investigated, Chris admitted the configuration error but did not mention knowing about it for 3 hours. The investigation revealed that Chris had noticed and said nothing.
>
> **Consequence:** Chris was put on a performance improvement plan. Not for the mistake — for the hiding. The mistake would have been a learning moment. The hiding was a trust violation.
>
> **Lesson:** If you make a mistake, report it immediately. The longer you wait, the worse it gets.

### 4. Helping Others

**Trust is a team sport. You cannot be trusted if no one trusts you personally.**

```
WAYS TO HELP:

Code reviews:
- Review PRs within 24 hours (not 4 days later)
- Provide specific, actionable feedback (not "this looks wrong")
- Explain WHY something should change, not just WHAT
- Balance critique with praise ("Good approach here, one suggestion...")

Knowledge sharing:
- Write documentation when you learn something non-obvious
- Give brown-bag talks on topics you understand well
- Answer questions in Slack helpfully (not "read the docs")
- Create runbooks for operational tasks you perform

Mentoring:
- Volunteer to mentor newer engineers
- Be available and responsive
- Share your mistakes as learning opportunities
- Celebrate your mentee's successes publicly

Operational support:
- Volunteer for on-call shifts (even when not required)
- Update runbooks when you use them
- Fix flaky tests (everyone hates them, everyone will love you)
- Improve monitoring dashboards
```

**The "Bus Factor" Test:**

If you were hit by a bus tomorrow (metaphorically), how much knowledge would leave with you?

- If the answer is "a lot" — you are hoarding knowledge. Share it.
- If the answer is "not much" — you are doing a good job of documenting and sharing.

### 5. Respecting Process

**Process exists because something went wrong in the past. You do not know what went wrong. Respect the process.**

```
PROCESS VIOLATIONS AND THEIR IMPACT:

Skipping code review:
- "It is a one-line change, I will just push it."
- Impact: The one-line change could introduce a security vulnerability.
  In 2023, a one-line change to the authentication middleware disabled
  MFA for 200 users for 6 hours.

Skipping testing:
- "The tests take too long, I will just test manually."
- Impact: Manual testing misses edge cases. Automated tests catch them.
  A skipped integration test missed a database migration bug that 
  corrupted 1,000 consumer configurations.

Skipping CAB:
- "It is just a config change, I do not need CAB."
- Impact: Config changes can be as impactful as code changes.
  A config change to the rate limiting threshold caused one consumer
  to exhaust the platform's token budget in 2 hours.

Skipping documentation:
- "The code is self-documenting."
- Impact: In 6 months, no one (including you) will understand the code.
  An undocumented guardrails check was disabled during an incident
  because the on-call engineer did not know it existed.
```

**When to Challenge Process:**

Process should be challenged — but through the right channels:

```
GOOD: "I noticed this CAB process takes 3 days for low-risk changes. 
       Can we propose an expedited process for standard changes?"

BAD: "CAB is pointless. I am just deploying this."

The good approach improves the process.
The bad approach violates trust.
```

---

## Trust Signals (What People Notice)

### Positive Signals

| Behavior | What It Signals |
|----------|----------------|
| Responding to messages within 4 hours | You are reliable and available |
| Updating your JIRA tickets daily | You are organized and transparent |
| Admitting mistakes immediately | You are honest and accountable |
| Reviewing PRs within 24 hours | You respect your colleagues' time |
| Writing thorough PR descriptions | You communicate well |
| Volunteering for on-call shifts | You are a team player |
| Updating runbooks after incidents | You invest in team knowledge |
| Asking thoughtful questions | You are engaged and learning |
| Helping others without being asked | You care about the team |
| Meeting deadlines consistently | You are predictable |

### Negative Signals

| Behavior | What It Signals |
|----------|----------------|
| Missing standup without notice | You are unreliable |
| PRs sitting unreviewed for days | You do not respect others' work |
| Defensive responses to feedback | You cannot grow |
| Blaming tools or processes | You avoid accountability |
| Skipping tests or reviews | You cut corners |
| Hiding behind "it works on my machine" | You do not care about production |
| Complaining about process publicly | You undermine the team |
| Taking credit for others' work | You are not trustworthy |
| Not documenting your work | You hoard knowledge |
| Missing commitments without communication | You are unpredictable |

---

## Trust Recovery

### If You Have Lost Trust

It happens. Everyone makes mistakes that damage trust. Here is how to recover:

```
STEP 1: Acknowledge the mistake
- Do not minimize it ("it was not that bad")
- Do not deflect ("the process was unclear")
- Say: "I made a mistake. Here is what happened."

STEP 2: Fix the immediate problem
- Focus on restoring service / fixing the issue
- Document what you did and why

STEP 3: Prevent recurrence
- "Here is how I will make sure this does not happen again."
- Actually implement the prevention

STEP 4: Rebuild through consistency
- Trust is rebuilt the same way it was built originally: 
  drop by drop, through consistent reliable behavior
- Expect it to take longer to rebuild than it took to lose

STEP 5: Ask for feedback
- "What can I do differently?"
- Listen without defending
- Act on the feedback
```

### Story: Trust Recovery Done Right

> A senior engineer (Morgan) deployed a change without full testing that caused the RAG retrieval pipeline to return no documents for 45 minutes. The wealth management chat was effectively down.
>
> Morgan's response:
> 1. Immediately acknowledged the incident in the channel: "This is my deployment. I am rolling back now."
> 2. Rolled back within 5 minutes.
> 3. Filed the incident report within 1 hour.
> 4. In the post-mortem, took full responsibility: "I did not run the full e2e test suite because it takes 20 minutes and I was impatient. That was my error."
> 5. Proposed a fix: "The deployment pipeline should block deployment if the e2e suite has not been run on the exact commit being deployed. I will implement this."
> 6. Implemented the fix within 2 weeks.
> 7. Presented the incident and fix at the team retrospective as a learning opportunity.
>
> **Result:** Morgan's trust was not permanently damaged. The team respected the honesty, the speed of the fix, and the systemic improvement. The post-mortem was cited as an example of good incident handling.
>
> **Key insight:** Morgan lost trust for 45 minutes (the outage). Morgan regained trust over the following 2 weeks through transparent, accountable behavior. The net trust impact was slightly positive.

---

## Trust and the Banking Context

### Why Trust Matters More Here Than at a Startup

At a startup, a bad deployment might lose customers some data. At a bank, a bad deployment can:

- Violate regulatory requirements (fines, license risk)
- Expose customer data (GDPR violations)
- Produce discriminatory outcomes (fair lending violations)
- Provide incorrect financial advice (consumer harm, reputational damage)

**The trust you build is not just with your team — it is with the bank's risk management function, the compliance team, the auditors, and ultimately the regulators.**

### The Trust Chain

```
You → Your team → Your manager → The department head → The CTO → The Board → The Regulator

If any link in this chain breaks, the regulator asks:
"How did this happen? Who approved it? Who tested it? Who was responsible?"

Your name will be on that list. Make sure the answer is:
"They followed the process, did the right thing, and can prove it."
```

---

## Final Advice

### The 5 Rules of Trust

1. **Say what you will do. Do what you say. Say what you did.** This is the entire game.
2. **Admit mistakes immediately.** The cover-up is always worse than the crime.
3. **Help others succeed.** Your success is measured partly by your output and partly by your impact on the team.
4. **Respect the process.** It exists for reasons you may not know. Challenge it through the right channels.
5. **Be the engineer you would want on your team at 3 AM during an incident.** Calm, competent, communicative, honest.

---

## Further Reading

- `first-30-days.md` — Building trust in your first month
- `first-60-days.md` — Deepening trust through production contributions
- `first-90-days.md` — Trust through ownership and leadership
- `common-mistakes-new-engineers-make.md` — Mistakes that damage trust
- `incident-management/` — How incident handling affects trust

---

*Last updated: April 2025 | Document owner: Engineering Enablement Team | Review cycle: Quarterly*
