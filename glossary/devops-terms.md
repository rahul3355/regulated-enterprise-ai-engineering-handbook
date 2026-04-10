# DevOps and CI/CD Terms

> Essential DevOps and CI/CD terminology for GenAI platform engineers. Each term includes definition, banking/GenAI example, related concepts, and common misunderstandings.

## Glossary

### 1. CI/CD (Continuous Integration / Continuous Delivery)
**Definition:** CI is the practice of frequently integrating code changes into a shared repository with automated builds and tests. CD extends this to automatically delivering changes to staging or production environments.
**Example:** Every push to the GenAI chat service triggers a CI pipeline: lint → test → security scan → build image → deploy to staging → run smoke tests.
**Related Concepts:** Pipeline, automation, DevOps, GitOps
**Common Misunderstanding:** CI/CD is not a tool — it's a practice. Jenkins, GitHub Actions, and Tekton are tools that implement CI/CD.

### 2. Pipeline
**Definition:** A sequence of automated stages that code changes pass through from commit to production.
**Example:** The GenAI platform pipeline: lint → unit test → integration test → security scan → build container → push to registry → deploy to staging → smoke test → deploy to production.
**Related Concepts:** Stage, job, artifact, gate
**Common Misunderstanding:** A pipeline is not just "build and deploy." It includes quality gates (tests, security scans, compliance checks) that can block promotion.

### 3. Artifact
**Definition:** A file produced by a build process that can be deployed or distributed (container image, JAR wheel, etc.).
**Example:** The CI pipeline produces a Docker container image tagged with the Git SHA, which is pushed to the internal container registry for deployment.
**Related Concepts:** Build, registry, immutability, versioning
**Common Misunderstanding:** Artifacts should be immutable — once built, they're never modified. If you need to change something, build a new artifact.

### 4. Container Image
**Definition:** A lightweight, standalone, executable package that includes everything needed to run an application: code, runtime, libraries, and configuration.
**Example:** The GenAI chat service's container image includes the Python runtime, application code, dependencies, and startup command — built from a Dockerfile.
**Related Concepts:** Docker, OCI, registry, layer, multi-stage build
**Common Misunderstanding:** Container images are not virtual machines. They share the host OS kernel and are much more lightweight than VMs.

### 5. Infrastructure as Code (IaC)
**Definition:** Managing and provisioning infrastructure through machine-readable definition files rather than manual processes.
**Example:** The GenAI platform's Kubernetes resources (deployments, services, ConfigMaps) are defined in YAML manifests stored in Git, deployed using kubectl or Helm.
**Related Concepts:** Terraform, Helm, GitOps, declarative configuration
**Common Misunderstanding:** IaC is not just about initial provisioning. It's also about ongoing management — changes to infrastructure are made by changing the code, not by clicking in a console.

### 6. GitOps
**Definition:** An operational framework where Git is the single source of truth for both application code and infrastructure, with automated reconciliation.
**Example:** ArgoCD watches the Git repository for changes to the GenAI platform manifests. When a new image tag is committed, ArgoCD automatically syncs the cluster to match.
**Related Concepts:** IaC, ArgoCD, Flux, declarative, reconciliation
**Common Misunderstanding:** GitOps is not just "deploy from Git." It requires automated reconciliation (the system continuously ensures the actual state matches the desired state in Git).

### 7. Blue-Green Deployment
**Definition:** A deployment strategy where two identical production environments (blue and green) exist. Traffic is switched from one to the other for zero-downtime deployments.
**Example:** The current GenAI chat service runs on "blue." A new version is deployed to "green." After validation, traffic is switched from blue to green. If issues arise, traffic switches back to blue.
**Related Concepts:** Canary, rolling update, feature flag, rollback
**Common Misunderstanding:** Blue-green requires double the infrastructure resources (two full environments). It's more expensive than rolling updates but safer.

### 8. Canary Deployment
**Definition:** A deployment strategy where a new version is released to a small subset of users first, and gradually expanded if metrics look healthy.
**Example:** The new GenAI chat version is deployed to 5% of users. After 30 minutes of monitoring (error rate, latency, user satisfaction), it's expanded to 25%, then 50%, then 100%.
**Related Concepts:** Blue-green, progressive delivery, feature flag, A/B testing
**Common Misunderstanding:** Canary deployments are not A/B tests. Canaries test the same functionality with different code versions. A/B tests compare different features or designs.

### 9. Feature Flag
**Definition:** A conditional toggle in code that enables or disables a feature without requiring a deployment.
**Example:** The GenAI response caching feature is behind a feature flag. It's deployed to production but disabled. The team enables it for 10% of users, monitors, then gradually enables for all.
**Related Concepts:** Toggle, canary, A/B testing, configuration
**Common Misunderstanding:** Feature flags are not a substitute for proper testing. Code behind a flag must still be tested. Flags add complexity (combinatorial explosion of enabled/disabled states).

### 10. Rollback
**Definition:** Reverting a deployment to a previous known-good version when issues are detected.
**Example:** After deploying GenAI chat v2.3, the error rate spikes to 5%. The team rolls back to v2.2 within 2 minutes using `kubectl rollout undo`.
**Related Concepts:** Roll forward, blue-green, canary, deployment strategy
**Common Misunderstanding:** Rollback is not always the right response. If the issue is caused by a database migration, rolling back the code may not fix the problem.

### 11. Smoke Test
**Definition:** A quick set of tests that verify the most critical functionality works after a deployment.
**Example:** After deploying the GenAI chat service, the smoke test: (1) hits the health endpoint, (2) sends a test query, (3) verifies a response is returned, (4) checks the audit log was written.
**Related Concepts:** Integration test, health check, deployment verification
**Common Misunderstanding:** Smoke tests are not comprehensive tests. They verify "is it alive and basically working?" — not "is every feature correct?"

### 12. Shift Left
**Definition:** Moving testing and quality checks earlier in the development process (left on the timeline), catching issues before they reach production.
**Example:** The GenAI team runs security scanning (Bandit, Safety) as a pre-commit hook, not just in CI. Issues are caught before code is even pushed.
**Related Concepts:** CI/CD, pre-commit, test pyramid, quality gates
**Common Misunderstanding:** Shift Left doesn't mean "developers do everything." It means quality checks happen as early as possible in the pipeline.

### 13. Test Pyramid
**Definition:** A testing strategy with many fast, cheap unit tests at the base, fewer integration tests in the middle, and even fewer slow, expensive end-to-end tests at the top.
**Example:** The GenAI platform has: 500 unit tests (milliseconds each), 50 integration tests (seconds each), 5 end-to-end tests (minutes each).
**Related Concepts:** Unit test, integration test, E2E test, testing strategy
**Common Misunderstanding:** The pyramid is about proportion, not prohibition. You still need some E2E tests — just not as many as unit tests.

### 14. Dependency Scanning
**Definition:** Automatically checking project dependencies for known security vulnerabilities.
**Example:** The GenAI CI pipeline runs `safety check` against requirements.txt and fails the build if any CRITICAL or HIGH vulnerabilities are found.
**Related Concepts:** SCA, CVE, SBOM, Dependabot, Renovate
**Common Misunderstanding:** Dependency scanning only finds KNOWN vulnerabilities (with CVEs). It cannot find vulnerabilities that haven't been publicly disclosed.

### 15. SBOM (Software Bill of Materials)
**Definition:** A complete inventory of all components, libraries, and dependencies in a software artifact.
**Example:** The GenAI platform generates an SBOM for each container image, listing every Python package, system library, and their versions — used for vulnerability impact analysis.
**Related Concepts:** Dependency scanning, SPDX, CycloneDX, supply chain
**Common Misunderstanding:** An SBOM doesn't fix vulnerabilities — it just makes them visible. You still need a process to act on SBOM findings.

### 16. Observability
**Definition:** The ability to understand a system's internal state by examining its external outputs (logs, metrics, traces).
**Example:** The GenAI platform is observable through: Grafana dashboards (metrics), Elasticsearch (logs), and Jaeger (distributed traces). When latency spikes, engineers can trace it from the API gateway through each service.
**Related Concepts:** Monitoring, logging, tracing, SLO, alerting
**Common Misunderstanding:** Observability is not monitoring. Monitoring tells you when something is wrong. Observability helps you understand WHY it's wrong.

### 17. SLO (Service Level Objective)
**Definition:** A target value for a specific service metric that the team commits to meeting.
**Example:** The GenAI chat service SLOs: 99.9% availability (measured monthly), p95 latency < 1.5s, error rate < 0.1%.
**Related Concepts:** SLI, SLA, error budget, burn rate
**Common Misunderstanding:** SLOs are not SLAs. SLAs are contractual commitments to customers (with penalties). SLOs are internal targets that should be stricter than SLAs.

### 18. Error Budget
**Definition:** The amount of acceptable failure within an SLO period, calculated as 100% minus the SLO target.
**Example:** With a 99.9% availability SLO, the error budget is 0.1% = ~43 minutes of downtime per month. If the service has already been down 40 minutes this month, the remaining 3 minutes are all the budget left.
**Related Concepts:** SLO, burn rate, reliability, risk appetite
**Common Misunderstanding:** An error budget is not a target to hit. It's a resource to spend on innovation. If you have budget remaining, you can take more risks.

### 19. Burn Rate
**Definition:** The rate at which the error budget is being consumed.
**Example:** If the GenAI service experiences a 2-hour outage (consuming 28% of the monthly error budget), the burn rate for that period was 28% in 2 hours vs. the normal rate of ~3.3% per day.
**Related Concepts:** Error budget, SLO, multi-window alerting
**Common Misunderstanding:** Burn rate is not about the current error rate — it's about how fast you're consuming your error budget relative to the total.

### 20. Runbook
**Definition:** A documented set of procedures for handling common operational scenarios (alerts, deployments, troubleshooting).
**Example:** The GenAI platform runbook includes procedures for: high latency troubleshooting, LLM API failover, database connection issues, and certificate renewal.
**Related Concepts:** SOP, playbook, on-call, operational readiness
**Common Misunderstanding:** A runbook is not documentation for developers. It's operational instructions for whoever is on-call — who may not be the person who wrote the runbook.

### 21. Chaos Engineering
**Definition:** The practice of intentionally injecting failures into a system to test its resilience and improve operational readiness.
**Example:** The GenAI team runs chaos experiments monthly: killing random pods, adding latency to database connections, and simulating LLM API outages — verifying that the system handles them gracefully.
**Related Concepts:** Resilience testing, game day, fault injection, resilience
**Common Misunderstanding:** Chaos engineering is not random destruction. It's controlled, measured experimentation with hypotheses about system behavior.

### 22. Immutable Infrastructure
**Definition:** An approach where infrastructure components are never modified after creation — they are replaced entirely when changes are needed.
**Example:** Instead of patching a running GenAI service container, the team builds a new container image with the fix and deploys it, replacing the old pods.
**Related Concepts:** Cattle vs. pets, configuration drift, IaC
**Common Misunderstanding:** Immutable infrastructure doesn't mean data is immutable. Persistent data (databases) still needs in-place updates. It's the compute layer that's immutable.

### 23. Configuration Drift
**Definition:** The gradual divergence of actual infrastructure configuration from the desired/defined state, typically due to manual changes.
**Example:** An engineer manually increases the memory limit on a GenAI pod to fix an issue but forgets to update the deployment manifest. The next deployment reverts the change, and the issue returns.
**Related Concepts:** IaC, GitOps, immutable infrastructure
**Common Misunderstanding:** Configuration drift is not just about infrastructure. It also applies to application configuration, database schemas, and security policies.

### 24. Secret Rotation
**Definition:** The periodic replacement of secrets (API keys, passwords, certificates) to limit the impact of compromise.
**Example:** The GenAI platform's LLM API key is rotated every 90 days. The rotation is automated: a new key is created in Kubernetes Secrets, the deployment is updated, and the old key is revoked.
**Related Concepts:** Secrets management, Vault, key lifecycle
**Common Misunderstanding:** Secret rotation is not just about changing the secret. The old secret must be revoked, and all consumers must be updated — or there's a window where both are valid.

### 25. Golden Image
**Definition:** A pre-configured, tested, and approved template for creating new infrastructure instances.
**Example:** The bank's golden container image includes: approved base OS, security patches, monitoring agent, logging agent, and security configurations. All GenAI services are built on top of this golden image.
**Related Concepts:** Base image, CIS benchmark, compliance, standardization
**Common Misunderstanding:** Golden images are not static. They must be regularly updated with security patches and re-tested to remain trustworthy.
