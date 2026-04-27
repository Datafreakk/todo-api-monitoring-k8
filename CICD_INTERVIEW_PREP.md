# CI/CD Pipeline Interview Prep

---

## 1. What is CI/CD and Why Does It Matter?

**Continuous Integration (CI)** is the practice of merging developer code into a shared repository frequently — ideally multiple times a day. Each merge triggers an automated build and test pipeline. The goal is to catch integration problems early, when they are cheap to fix, rather than at the end of a sprint when merging weeks of diverged work becomes painful.

**Continuous Delivery (CD)** extends CI by ensuring that every code change that passes automated tests is in a deployable state. The artifact can be released to production at any time with a single click or approval gate.

**Continuous Deployment** goes one step further — every change that passes all automated stages is automatically deployed to production with no human intervention at all.

**Why it matters in interviews:** Interviewers want to know you understand the *why*, not just the tooling. The business value is: faster feedback loops, reduced risk per release, higher deployment frequency, and lower mean time to recovery (MTTR).

---

## 2. Anatomy of a CI/CD Pipeline

A pipeline is a series of automated stages. Each stage has a clear purpose and a pass/fail gate. If a stage fails, the pipeline stops and the team is notified. Nothing moves forward unless quality is confirmed.

### Typical Stages

```
Code Commit → Build → Unit Tests → Code Quality/Scan → Integration Tests → Artifact Publish → Deploy to Dev → Deploy to Staging → Manual Approval → Deploy to Production
```

### Stage-by-Stage Explanation

**Source / Trigger Stage**
Every pipeline starts with a trigger. This could be a push to a branch, a pull request, a scheduled time, or a manual trigger. The trigger defines *when* the pipeline runs. A well-designed pipeline has different triggers for different purposes — e.g., PR triggers run lightweight tests only, while a merge to main runs the full pipeline.

**Build Stage**
The source code is compiled or packaged into an artifact. The artifact should be immutable — the exact same binary or container image that gets tested is what gets deployed. Never rebuild the artifact per environment. This ensures "what you tested is what you ship."

**Unit Test Stage**
Fast, isolated tests that verify individual functions and components. These should run in seconds to minutes. They give the developer immediate feedback. If unit tests fail, there is no point running slower integration tests.

**Code Quality / Security Scan Stage**
Static analysis (linting, code smell detection) and security scanning (dependency vulnerabilities, secret detection) run here. This is the stage where you enforce standards consistently — not through code reviews alone.

**Integration Test Stage**
Tests that verify the interaction between components — API contracts, database queries, service-to-service communication. These are slower and require a running environment, so they run after unit tests pass.

**Artifact Publish Stage**
The verified artifact is published to a registry (container registry, package feed, artifact storage). It gets a version tag. From this point forward, every environment deploys *this exact artifact* — not a rebuilt version.

**Deploy Stages (Dev → Staging → Production)**
Each environment is a gate. Dev is for developer validation. Staging mirrors production and is used for final acceptance testing. Production is the real thing. Each promotion is either automated (if all tests pass) or requires a manual approval gate for high-risk environments.

---

## 3. Branch Strategies and How They Connect to Pipelines

The branching strategy you choose directly shapes your pipeline design.

### Trunk-Based Development
All developers commit to a single main branch (trunk) frequently — at least once a day. Feature flags are used to hide incomplete features from users. The pipeline on main is always the full pipeline. This approach optimizes for speed and forces small, integrated changes.

*Pipeline implication:* One primary pipeline on main. Feature branches are short-lived and trigger only lightweight CI checks. This reduces merge conflicts and pipeline complexity.

### GitFlow
Long-lived branches: `main`, `develop`, `feature/*`, `release/*`, `hotfix/*`. Feature branches merge into develop. Release branches are cut from develop. Hotfixes go directly to main and back-merge to develop.

*Pipeline implication:* Multiple pipelines for different branch types. More complex to maintain. Can lead to "big bang" merges and large risk per deployment. Common in organizations with scheduled release windows.

### GitHub Flow (simplified GitFlow)
Short feature branches off main. PRs trigger CI. Merge to main triggers deployment. Simple and effective for teams doing continuous deployment.

### Branch Strategy Interview Q&A

---

**"What branching strategy do you use and why?"**

We used trunk-based development. Everyone commits to main at least once a day and we use feature flags to hide incomplete work from users. The reason I prefer it over GitFlow is that it forces small, integrated changes rather than letting branches diverge for days or weeks. The longer a branch lives, the more painful the merge — and that pain compounds with team size. The tradeoff is you need discipline around feature flags and your test coverage needs to be solid, because main is always the deployment branch. GitFlow made sense when we had scheduled release windows, but once we moved to continuous deployment it was just overhead.

---

**"How does your branching strategy connect to your deployment frequency?"**

Directly. With trunk-based development, every merge to main triggers the full pipeline and can go to production the same day. With GitFlow you have a release branch that batches changes — so your deployment frequency is capped by your release cadence, not by your team's velocity. If you're measuring DORA metrics and want to improve deployment frequency, the first thing to look at is whether your branch strategy is forcing artificial batching. Long-lived branches are usually the bottleneck people overlook when they wonder why they can only deploy weekly.

---

## 4. Release Strategies

This is one of the most important CI/CD topics in interviews. The question is: *how do you get new code to users while minimizing risk?*

### Big Bang / Recreate Deployment
Tear down the old version, deploy the new version. Simple but causes downtime. Only acceptable for non-critical, low-traffic applications or during a maintenance window.

### Rolling Deployment
Replace instances of the old version one by one (or in batches) with the new version. At any point during the rollout, some instances run the old code and some run the new code. No downtime, but you must ensure both versions can handle requests simultaneously — this means your API must be backward compatible during the transition window.

*Risk:* If the new version has a bug, some users are already affected before you detect it.

### Blue/Green Deployment
Blue-green is basically keeping two identical environments running at the same time. One is live, one is idle. You deploy your new version to the idle environment, test it, and when you're happy you just flip the traffic switch. The old environment stays warm so if anything goes wrong you roll back instantly — one command, no rebuild.

In practice we used Azure Container Apps which handles this natively through revisions. Each deployment creates a new revision with its own URL, so we could smoke test it in isolation before flipping traffic weights to 100%.

*Cost:* Requires double the infrastructure at deployment time.
*Benefit:* Zero-downtime, instant rollback, clean separation between old and new.

### Canary Deployment
Route a small percentage of real user traffic (e.g., 5%) to the new version. Monitor error rates, latency, and business metrics. If metrics look healthy, gradually increase the percentage. If something goes wrong, route all traffic back to the old version.

*Why it's powerful:* You test with real users and real traffic before full rollout. You catch issues that only appear at scale or with specific user behavior.
*Requirement:* You need feature flagging or traffic splitting at the load balancer/service mesh level, and solid observability.

### Feature Flags (Feature Toggles)
Deploy code to production but keep the feature hidden behind a flag. Enable the flag for internal users first, then beta users, then everyone. This separates *deployment* from *release* — code can be in production without being visible to users.

*Why SREs love this:* Rollback is instant — just toggle the flag off. No redeployment needed.

### Release Strategy Interview Q&A

---

**"When would you choose canary over blue-green?"**

Blue-green gives you a clean switch — you test on the idle environment and flip when you're confident. But the weakness is that your test traffic is synthetic, not real users. Canary is what you reach for when you want real production traffic to validate the change before committing to a full rollout. I'd choose canary when the change touches user-facing behaviour, involves a performance-sensitive code path, or when past incidents have been caused by things that only surface under real traffic patterns. The requirement is good observability — you need to be able to tell within minutes whether the 5% slice is behaving differently. If your monitoring isn't there yet, canary gives you a false sense of safety.

---

**"What's the risk with rolling deployments that people often miss?"**

The hidden constraint is that both versions of your app run simultaneously during the rollout window. So your API needs to be backward compatible — if the new version changes a response field or drops an endpoint, any old instance still running will break requests that hit it. The same applies to database schema changes. Teams often think rolling deployment is safe by default, but it's only safe if you've thought through compatibility. I always ask: if old and new run side by side for ten minutes, does anything break? If yes, rolling is the wrong strategy for that change — you need blue-green where you can do a hard cutover.

---

**"How do feature flags change your deployment process?"**

They decouple deployment from release, which is a big deal. You can ship code to production on a Tuesday and turn the feature on for users on a Friday after more testing — or just for internal users first. The pipeline doesn't change, the artifact is the same, but you control exposure separately. The operational benefit is that rollback becomes a flag toggle rather than a redeployment. The downside is flag debt — if you don't clean up old flags, your codebase fills up with dead code paths and conditional logic that's impossible to reason about. We treated flags as temporary scaffolding with a planned removal date, not permanent config.

---

## 5. Rollback Strategies

Rollback is what you do when a deployment goes wrong. Interviewers often ask this because it reveals how mature your pipeline thinking is.

Rollback is not always the right answer. If the bug is trivial to patch and the fix takes 10 minutes, sometimes it's faster to fix forward than to coordinate a rollback — especially if a migration has already run. I'd rollback when the blast radius is large and the fix is unknown, and fix forward when the problem is well understood and contained

### Redeploy Previous Artifact (Standard Rollback)
Trigger the pipeline again but with the previous known-good artifact version. This is safe because you are deploying a verified artifact. Downside: takes as long as a normal deployment.

### Blue/Green Instant Rollback
If you used Blue/Green, flip the load balancer back to Blue. This is the fastest rollback — seconds, not minutes. No redeployment required.

### Feature Flag Rollback
If the issue is behind a feature flag, toggle it off. No code change, no deployment, no downtime. This is the ideal rollback path when possible.

### Database Rollback — The Hard Problem
Application code is easy to roll back. Database schema changes are not. This is the most common trap in CI/CD interviews.

The rule: **database migrations must be backward compatible with both the old and new version of the application.** This means:
- Never rename or drop a column in the same release you stop using it. Deprecate it first, remove it in a later release.
- Add new columns as nullable so old application code doesn't break.
- Use the expand/contract pattern: expand the schema (add new columns), migrate data, deploy new app, contract the schema (remove old columns) in a later release.

### Rollback Interview Q&A

---

**"What's your rollback strategy if a deployment fails?"**

First thing I look at is how the failure was detected — was it a health check, an alert, or a user report. That tells me how much damage is already done. Then I go to the fastest recovery path available. If we're on blue-green, I flip traffic back immediately — that's under a minute. If not, I redeploy the previous image tag. We keep immutable versioned artifacts in our registry so there's always a known good version to point back to. In my current role on Azure Container Apps, each revision has its own URL so we could catch failures before traffic even switched.

---

**"How do you decide when to rollback vs fix forward?"**

It's a judgment call but I have a mental framework for it. I rollback when the blast radius is large, the root cause is unclear, or recovery time for a fix is unknown. I fix forward when the problem is well understood, the patch is small and low risk, and — critically — when a database migration has already run and rolling back the app would break data consistency. In an incident I always ask: is it faster and safer to revert the change or ship a fix? If I can't answer that confidently in two minutes, I rollback first and investigate after. Stability before diagnosis.

---

**"How quickly can you rollback in your current setup?"**

In Azure Container Apps, rollback is essentially a traffic weight change — we can shift 100% back to the previous revision in under a minute. The previous revision stays warm so there's no cold start delay. For our CI/CD pipeline on Azure DevOps, a full redeploy of the previous image tag takes around three to five minutes depending on the environment. We defined our RTO as five minutes for critical services, so both paths meet that target. The key was keeping old revisions alive and not pruning them immediately after a deploy.

---

**"How do you handle rollback when a database migration has already run?"**

This is the hardest part of rollback and the place most teams get caught out. My approach is to never write destructive migrations — so no dropping columns, no renaming columns, nothing that breaks the previous version of the app. Instead I use the expand-contract pattern. First deploy adds the new column but the app still reads the old one — that's the expand phase. Once the new version is fully rolled out and stable, a follow-up migration removes the old column — that's the contract phase. This means at any point during rollout, both the old and new app version are compatible with the database. If I need to rollback the app, the database state is still valid for the previous version.

---

**"What's the difference between rollback and fix forward — when do you choose each?"**

Rollback means reverting to a previous known good state — the change goes away. Fix forward means shipping a new change that corrects the bad one — you move ahead rather than back. I choose rollback when I don't understand the problem yet, when the impact is spreading, or when there's no quick fix available. I choose fix forward when the bug is obvious and the patch is a one-liner, when a rollback would cause data inconsistency due to migrations, or when the rollback itself carries risk — for example if the previous version had a known issue we'd already resolved. In practice I default to rollback during the heat of an incident and treat fix forward as something that needs a bit more confidence.

---

**"How do you test that your rollback actually works?"**

This is something a lot of teams skip and then discover in production that their rollback is broken. We treated rollback as part of the deployment pipeline — not an afterthought. Concretely that meant: keeping previous image tags in the registry and verifying they're pullable, running periodic rollback drills in staging where we deploy a bad version intentionally and time the recovery, and including rollback steps in runbooks so anyone on call can execute them without hunting for the right command. The worst time to figure out your rollback process is during an active incident.

---

**"Walk me through a real incident where you had to rollback."**

We deployed a new version of a backend service that had a subtle bug in how it handled a specific API response format from a third party integration. It wasn't caught in testing because the third party response only varied in production data. Within about four minutes of deploy our alert fired on elevated 500 errors — around 8% error rate on that endpoint. I immediately checked whether it was deployment-correlated by looking at the deployment timestamp against the error spike — it matched exactly. Rather than dig into root cause under pressure I called the rollback, flipped traffic back to the previous ACA revision, and error rate dropped within 90 seconds. We then investigated in a non-incident context, reproduced it in staging with production-like data, and fixed forward the next day. The key things that made it work: fast detection through good alerting, immutable revisions still available, and a clear decision to stabilise first and diagnose second.

---

**"What's your strategy if a rollback itself causes an outage?"**

This is a scenario you have to think about before it happens, not during. The main causes are: the previous version is incompatible with a migration that already ran, infrastructure state has changed and the old version depends on something that no longer exists, or the old version had a bug you'd already fixed and rolling back reintroduces it. My mitigation is pre-rollback validation — before I flip, I quickly check: is the previous revision still healthy in ACA, does the database state support the old app version, are there any infrastructure dependencies that changed. If rollback itself is risky, I shift to fix forward immediately with a hotfix branch. The broader answer is that a rollback causing an outage usually means your deployment process had a hidden dependency you didn't track — that's a post-incident finding worth a runbook update.

---

**"How do you define RTO and does your rollback strategy meet it?"**

RTO — recovery time objective — is the maximum acceptable time to restore service after a failure. For us critical services had a five minute RTO defined in our SLA. I mapped that against our rollback options: blue-green traffic flip was under one minute, pipeline redeploy was three to five minutes, and a full environment rebuild from scratch was out of scope for that RTO window. So the design decision was to always keep the previous revision warm and never rely on a full rebuild as the primary recovery path. RTO isn't just a number — it's a constraint that should drive how you architect your deployment and recovery process.

---

## 6. Pipeline Design Principles

These are the principles that make a pipeline production-grade, not just functional.

**Fail Fast**
Put the cheapest and most likely to fail checks first. Unit tests before integration tests. Lint before build. The developer should know within minutes if their change broke something basic.

**Idempotent Deployments**
Running the same deployment twice should produce the same result. This is critical for reliability. If a deployment fails halfway and you retry, it should not leave the system in a broken state.

**Environment Parity**
Dev, staging, and production should be as similar as possible. Differences between environments are the #1 source of "works on my machine / works in staging but fails in prod" bugs. Use the same base images, same configuration patterns, same infrastructure-as-code.

**Immutable Artifacts**
Build once, deploy everywhere. The artifact that passes tests in staging is the exact artifact deployed to production. Never allow environment-specific builds. Configuration is injected at runtime via environment variables or config maps, not baked into the artifact.

**Pipeline as Code**
Pipeline definitions live in the repository alongside the application code. This means:
- Pipeline changes go through code review
- You can see the history of pipeline changes
- You can test pipeline changes in a branch before merging
- The pipeline is version-controlled with the application it serves

**Secrets Management**
Secrets (API keys, passwords, certificates) must never be stored in the pipeline YAML or in the repository. They should be injected at runtime from a secrets manager (Key Vault, Secrets Manager, etc.). The pipeline should only have a reference to the secret, not the value.

---

## 7. Pipeline Stages in Practice (Azure DevOps as Context)

Azure DevOps uses YAML-defined pipelines stored in the repo. Here is how the concepts above map to practice — not focusing on syntax, but on design decisions.

**Stages and Jobs:** A pipeline has stages (logical phases like Build, Test, Deploy). Each stage has jobs (units of work that can run in parallel or sequence). Each job has steps (individual commands or tasks).

**Environments and Approvals:** You define environments (Dev, Staging, Production). Each environment can have approval gates — specific people or groups must approve before the pipeline proceeds to that environment. This gives humans a checkpoint before production deployments.

**Artifact promotion:** The build stage produces an artifact. Each deploy stage picks up that same artifact. The artifact is never rebuilt. This is enforced by publishing once and downloading in each subsequent stage.

**Conditional stages:** A hotfix pipeline might skip the full integration test suite and go straight to staging after unit tests, because speed is critical. Conditions on stages let you encode this logic in the pipeline definition.

---

## 8. Key Metrics — How You Measure Pipeline Health

Interviewers at SRE-focused companies will ask how you measure and improve pipelines. These are the DORA metrics — the industry standard for measuring software delivery performance.

**Deployment Frequency:** How often you successfully deploy to production. High performers deploy multiple times per day. This measures how often you are delivering value.

**Lead Time for Changes:** Time from code commit to running in production. Shorter lead time means faster feedback and less risk per change.

**Change Failure Rate:** Percentage of deployments that cause a production incident or require a hotfix. A high failure rate means your pipeline is not catching enough problems before production.

**Mean Time to Recovery (MTTR):** How long it takes to recover from a production incident. Good pipelines enable fast rollback, which directly reduces MTTR.

The goal is: high frequency, low lead time, low failure rate, low MTTR.

### DORA Metrics Interview Q&A

---

**"How do you measure the health of your CI/CD pipeline?"**

I use the DORA metrics as the baseline — deployment frequency, lead time for changes, change failure rate, and MTTR. Each one tells you something different. Deployment frequency tells you if your process is actually enabling fast delivery or just promising it. Lead time tells you how much friction is in the pipeline — if it's two hours, something is wrong. Change failure rate tells you if your testing is catching the right things before production. MTTR tells you how well your rollback and incident response works. In practice I track these in a dashboard and treat a spike in change failure rate as a signal to invest in test coverage, and a spike in lead time as a signal to look at slow pipeline stages.

---

**"Your deployment frequency is low — how do you diagnose and fix it?"**

I start by looking at where the time actually goes. Usually it's one of three things: a slow pipeline stage that everyone waits on, a manual approval process that creates a queue, or a batching habit where teams accumulate changes before deploying because deploys feel risky. For slow stages I profile each step and look for parallelisation opportunities — integration tests are often serialised when they could run concurrently. For manual approvals I ask whether the gate is adding real value or just creating delay — sometimes it's habit. For batching I address the root cause: teams batch when deploys feel risky, so the fix is reducing risk per deploy through better testing and smaller changes, not pushing people to deploy more often.

---

## 9. Common Interview Questions and How to Answer Them

---

**"What is the difference between Continuous Delivery and Continuous Deployment?"**

Continuous Delivery means every change is always in a deployable state — it's passed all your tests and gates and could go to production right now. But a human makes the call on when. Continuous Deployment removes that last human step — if it passes everything, it ships automatically. The difference sounds small but the organisational implication is significant. Continuous Deployment requires very high confidence in your test suite and rollback capability, because there's no human checkpoint catching things before they hit users. Most teams I've worked with sit at Continuous Delivery — automated all the way to staging, then a lightweight approval before production.

---

**"How do you handle database migrations in a CI/CD pipeline?"**

The key rule is that you never deploy a breaking schema change at the same time as the application change that depends on it. If you do, you either have a window where the new app is running against the old schema, or the old app is running against the new schema — either way something breaks. The pattern I use is expand-contract: first deploy adds new columns without removing old ones, the application is updated to write to both, then a follow-up migration drops the old column once the new version is fully rolled out and stable. It means database changes take two releases instead of one, but it means rollback is always safe because the old app version is always compatible with the current database state.

---

**"How do you prevent secrets from leaking in a pipeline?"**

Three layers. First, secrets never go in the repo or pipeline YAML — they live in a secrets manager like Key Vault or AWS Secrets Manager and are referenced by name, not value. Second, the pipeline runtime masks secret values in logs — most CI systems do this automatically if you declare variables as secret, but you need to verify it. Third, access is scoped: not every pipeline can access every secret, you use RBAC to limit which environments and service identities can fetch which secrets. The thing people miss is the third layer — they protect the secret at rest but give every pipeline broad read access, so a compromised pipeline can exfiltrate anything.

---

**"A deployment broke production. Walk me through your response."**

First I stabilise — rollback before I investigate. The fastest path depends on the setup: blue-green flip, feature flag toggle, or redeploy the previous image tag. The goal is to restore service, not to understand the root cause under pressure. Once the error rate is back to baseline, I look at what changed: deployment timestamp vs alert timestamp, which service, what commit. Then I reproduce it in a lower environment if I can. The post-mortem question I care most about is not just what went wrong but why the pipeline didn't catch it — what test or check was missing that would have caught this class of failure. That's the output that actually makes the next incident less likely.

---

## 10. Core CI/CD Interview Questions

These are the questions that come up most frequently, grouped by what the interviewer is actually testing.

---

**"Walk me through your CI/CD pipeline end to end"**

This is almost always the opener. They want to see if you understand the full picture, not just one piece.

In our setup we use Azure DevOps. A developer pushes to a feature branch, which triggers a CI pipeline — that runs unit tests, builds a Docker image, and pushes it to our container registry with an immutable tag based on the commit SHA. On merge to main, the CD pipeline picks up that image tag and deploys to staging via Helm on AKS. We run smoke tests against staging automatically. Promotion to production requires a manual approval gate, after which the same image — not rebuilt — gets deployed to prod. We use the same artifact across all environments so there's no drift between what was tested and what's live.

Key things to land: immutable artifacts, same image across envs, approval gates, smoke tests.

---

**"How do you handle secrets in your pipeline?"**

We never store secrets in the pipeline YAML or in the repo. In ADO we use Variable Groups linked to Azure Key Vault — the pipeline fetches secrets at runtime using a managed identity, so no credential is ever hardcoded. For Kubernetes workloads we use the CSI Secret Store Provider to mount Key Vault secrets as volumes directly into pods. The principle is: secrets are pulled, never pushed, and the pipeline identity only has the minimum permissions it needs.

---

**"How do you ensure the same artifact is deployed across environments?"**

We build once and promote the same image. The Docker image is tagged with the Git commit SHA at build time and pushed to the registry. Every subsequent stage — staging, production — references that exact tag. We never rebuild for a different environment. This guarantees what you tested in staging is exactly what runs in prod. Environment-specific config is handled separately via Helm values files or Key Vault, not baked into the image.

---

**"How do you handle a failed deployment in your pipeline?"**

The pipeline stops and raises an alert. Because we use immutable image tags and Helm, rollback is a helm rollback command or a traffic flip in ACA back to the previous revision. We don't automatically rollback — a human confirms the decision — but the mechanism is fast, under a minute for ACA revision flips. Post-incident we look at why the smoke tests didn't catch it and update the test suite.

---

**"What are your pipeline stages and what does each gate check?"**

We have four main stages. CI runs on every PR — lint, unit tests, image build, vulnerability scan. On merge to main the image is pushed to the registry and deployed to staging automatically. Staging runs integration and smoke tests. Production requires a manual approval and then deploys using the same image. Each gate has a quality threshold — if unit test coverage drops below a threshold or the container scan finds a critical CVE, the pipeline fails and the change doesn't progress.

---

**"How do you manage pipeline as code?"**

All our pipelines are defined in YAML committed to the repo alongside the application code. That means pipeline changes go through the same PR review process as code changes — you can't modify the pipeline without a reviewer approving it. We use ADO templates to share common stages across multiple pipelines so we're not copy-pasting YAML. Changes to the pipeline are versioned and auditable.

---

**"How do you prevent bad code from reaching production?"**

Layered gates. PR level: branch policies enforce at least one reviewer approval and a passing CI build before merge. Pipeline level: unit tests, integration tests, container vulnerability scanning, and smoke tests against staging. Infrastructure level: ACA health checks and readiness probes mean a bad container that fails to start never receives traffic. Monitoring level: error rate and latency alerts fire within minutes of a bad deploy reaching prod, which triggers our rollback decision. No single gate is enough — you need defence in depth.

---

**"What's a pipeline bottleneck you've identified and how did you fix it?"**

They want to see you treat the pipeline as a system to be optimised — classic SRE toil reduction.

Our Docker builds were slow because we weren't using layer caching properly. Every build was pulling base images and reinstalling dependencies from scratch. We restructured the Dockerfile to copy dependency files first and install packages before copying application code — so the dependency layer only rebuilds when dependencies actually change. We also enabled ADO pipeline caching for the Docker layer cache between runs. Build time dropped significantly. The principle is the same as any performance work — measure first, find the bottleneck, fix the right thing.

---

**"How do you manage multiple environments in ADO?"**

We use ADO Environments with deployment jobs scoped to each environment. Each environment has its own approval rules — staging is automatic, production requires manual sign-off from a senior engineer. We use Helm values files per environment for config differences and the same image tag across all of them. ADO tracks deployment history per environment so you can see at a glance what version is running where.

---

**"What security checks do you run in your pipeline?"**

Three main ones. Static analysis on the code itself using a linter and SAST tool. Container image scanning for known CVEs — we use Trivy which integrates cleanly into ADO and fails the build on critical severity findings. And dependency scanning to catch vulnerable packages in our requirements files. We also enforce that the pipeline runs with a service principal that has the minimum required permissions — least privilege on the identity, not just the code.

---

### Likely Follow-up Questions

- **"How do you handle a flaky test in your pipeline?"** — Quarantine it, track it as toil, fix or delete it, never just ignore it. A flaky test that gets ignored trains engineers to ignore all failures.
- **"What's the difference between a deployment and a release?"** — Deployment is the technical act of getting code onto a server. Release is making it available to users. They can be fully decoupled with feature flags.
- **"How do you monitor your pipeline health itself?"** — Pipeline duration trends, failure rates, queue times. Treat the pipeline as a production system with its own SLOs.
- **"Have you used pipeline templates or reusable components in ADO?"** — Yes, template YAML files shared across repos via a central pipeline library repo avoids copy-paste drift across teams.

---

### Meta-Answer — Works as a Closer for Almost Any CI/CD Question

The way I think about CI/CD as an SRE is that the pipeline is a production system in its own right. It has SLOs — how fast should a commit reach production, what's the acceptable failure rate of the pipeline itself. If engineers start working around the pipeline because it's too slow or too flaky, you've lost the safety guarantees it was supposed to provide. So we treat pipeline reliability the same way we treat service reliability.

---

## 11. Summary Cheat Sheet

| Concept | One-Sentence Essence |
|---|---|
| CI | Merge and validate frequently to catch problems early |
| CD (Delivery) | Every change is deployable; humans decide when to release |
| CD (Deployment) | Every passing change is automatically released |
| Rolling deploy | Replace instances gradually; requires backward compatibility |
| Blue/Green | Two environments, instant traffic switch, instant rollback |
| Canary | Real traffic to small % first, monitor, then expand |
| Feature flags | Separate deployment from release; instant rollback without redeploy |
| Immutable artifact | Build once, deploy everywhere — configuration injected at runtime |
| Expand/contract | Safe DB migration pattern: add before remove, across multiple releases |
| DORA metrics | Frequency, Lead Time, Change Failure Rate, MTTR |
