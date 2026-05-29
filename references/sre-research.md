# SRE & Reliability Research — Industry Practices for Solo Dev

Sources: Nubank Engineering Blog, Google SRE Book, DORA Research (10 years), ThoughtWorks Tech Radar, SRE community.

## Nubank — Why We Killed Our E2E Test Suite

**Context:** Fintech, 1000+ engineers, Clojure microservices, critical money flows.

**Problem:** E2E tests caused progressive slowdown:
- Queueing theory predicted E2E suite would take INFINITE time by 2021
- Teams bartered to skip ahead in queue
- Deploy time: 2+ hours minimum, often > 1 day
- High context-switching cost

**Solution: Contract Testing (Sachem)**
- Consumer-Driven Contract (CDC) testing
- Leveraged existing Clojure Schema library (schemas already existed in codebase)
- Built custom framework (Sachem) instead of Pact — less intrusive
- Contract tests catch structural incompatibilities
- Acceptance tests (in-memory, single JVM) catch behavioral issues

**Results:**
- Deploy pipeline: 2+ hours → ~20 minutes (predictable)
- Deploys per week: significantly increased
- Both metrics align with DORA's four key metrics for high-performing orgs

**Lesson for solo dev:** Don't over-invest in E2E. Contract tests (zod schemas) + focused integration tests are more reliable and faster. E2E only for 2-3 critical money flows.

## DORA Metrics — 4 Keys (Google Research)

Research spanning 10 years, statistically significant link between these metrics and high-performing teams:

| Metric | Measures | Elite Performance |
|--------|----------|-------------------|
| Lead Time for Changes | commit → production | < 1 hour |
| Deployment Frequency | how often deploy | On demand (multiple/day) |
| Change Failure Rate | % deploys causing incidents | < 5% |
| MTTR | time to recover from failure | < 1 hour |

**Key finding (2024/25 report):** AI tools speed up low-level tasks but haven't translated into significant DORA metric gains. Psychological safety consistently correlates with better performance across all 4 metrics.

**Solo dev implementation:** Deploy log (.deploy-log), postmortem folder, monthly review. No dashboard needed.

## Google SRE — Production Readiness Review (PRR)

From "Google SRE Book" Appendix E (Launch Coordination Checklist, circa 2005):

**Required before launch:**
- Architecture sketch, server types, request types
- Traffic/bandwidth estimates, launch spike, 6-month projection
- Load test, end-to-end test, capacity per datacenter at max latency
- For each backend: dependencies, failure modes, retry logic
- Monitoring internal state + end-to-end behavior
- Methods and change control for updates

**Error Budget concept:**
- SLO = promise (e.g., 99.9% availability)
- Error budget = 1 - SLO (e.g., 43 min/month downtime allowed)
- As long as budget not spent, dev team free to launch
- Budget exhausted = stop features, focus reliability
- Eliminates structural tension between dev and SRE

## Chaos Engineering — Small Team Applicability

**Netflix (Chaos Monkey):** Kill random instances in production.
**Small teams:** Don't need automated chaos — need TESTS for specific failure scenarios.

**Practical chaos scenarios for SaaS:**
- Stripe webhook arrives late/duplicated
- DB connection drops mid-transaction
- Concurrent checkout requests for same user
- Neon cold start timeout (3s+)
- Network offline during payment submit

**Free tools:** Steadybit, Harness CE, or just write tests that simulate failures.

## Feature Flags — Trunk-Based Development

**Industry standard:** Deploy code inactive, activate gradually. Kill switch without redeploy.

**Free implementation for solo dev:** Environment variables. Simple, effective, zero dependencies.

**Rules from industry:**
- Flags should be short-lived (< 30 days for stable features)
- Test both paths (ON and OFF)
- Default OFF for new features
- Money features MUST have kill switch

## What Does NOT Apply to Solo Dev

- Canary/Blue-Green deployment (Vercel handles this)
- On-call rotation (it's just you)
- Distributed tracing (Jaeger/Zipkin) — overkill for monolith
- Chaos Monkey automation — manual tests sufficient
- Scrum ceremonies — you're a team of one
- Microservices — monolith is better for solo dev
- Custom contract testing framework — zod schemas suffice
