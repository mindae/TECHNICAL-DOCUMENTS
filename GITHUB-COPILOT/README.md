# GitHub Copilot Handbook — IntelliJ, Java & Enterprise Development

> **Audience:** Senior Java/Spring Boot developers and emerging architects using GitHub Copilot in real enterprise codebases.  
> **Environment used:** macOS, IntelliJ IDEA 2026.2, GitHub Copilot plugin, independent Spring Boot microservices.  
> **Learning approach:** Breadth first; understand the useful 80%, practise on real code, revisit advanced capabilities only when a project requires them.  
> **Bootcamp:** Two-day, roughly 8–10-hour learning target; notes consolidated from hands-on sessions in October 2026.

## Table of contents

1. [Goals and guiding principles](#1-goals-and-guiding-principles)
2. [Setup: IntelliJ and GitHub Copilot](#2-setup-intellij-and-github-copilot)
3. [Workspace for independent microservices](#3-workspace-for-independent-microservices)
4. [Choose the right Copilot capability](#4-choose-the-right-copilot-capability)
5. [Inline completion and inline chat](#5-inline-completion-and-inline-chat)
6. [Ask: understand an unfamiliar application](#6-ask-understand-an-unfamiliar-application)
7. [Plan: design before changing code](#7-plan-design-before-changing-code)
8. [Agent: implement, validate and correct](#8-agent-implement-validate-and-correct)
9. [Production incident investigation](#9-production-incident-investigation)
10. [Repository instructions and context engineering](#10-repository-instructions-and-context-engineering)
11. [Reusable prompt files](#11-reusable-prompt-files)
12. [Custom agents and agent skills](#12-custom-agents-and-agent-skills)
13. [Model Context Protocol (MCP)](#13-model-context-protocol-mcp)
14. [Git, code review and multi-repository delivery](#14-git-code-review-and-multi-repository-delivery)
15. [AI credits, quotas and cost awareness](#15-ai-credits-quotas-and-cost-awareness)
16. [Enterprise security and operating practices](#16-enterprise-security-and-operating-practices)
17. [Hands-on case studies and lessons learned](#17-hands-on-case-studies-and-lessons-learned)
18. [Reusable prompt cookbook](#18-reusable-prompt-cookbook)
19. [Daily quick reference and troubleshooting](#19-daily-quick-reference-and-troubleshooting)
20. [Progress log, maintenance and next passes](#20-progress-log-maintenance-and-next-passes)
21. [References](#21-references)

---

## 1. Goals and guiding principles

**Practical outcomes:** Use Copilot to understand an existing Java project, trace API flows, work with unfamiliar classes, implement a new feature, accept/reject suggestions, diagnose production issues, create tests, perform a meaningful code review, and use approved external tools.

**Working rules:**

- **Explore → Plan → Implement → Test → Review → Commit/PR.** Don't start by asking an agent to rewrite code it hasn't inspected.
- Use the **smallest suitable capability**: completion for a line, inline chat for a method, Ask for explanation, Plan for design, Agent for multi-file execution.
- Keep human ownership of API contracts, architecture, security, migrations, approvals and final merges.
- Demand **actual classes, methods, paths, tests and logs**; distinguish observed evidence from inference.
- A successful agent summary does **not** replace reviewing Git diffs or running an application.
- Prefer concrete tasks on the existing microservices over throwaway toy projects.
- Avoid overengineering the learning sandbox. Confirm the real requirement before adding a general-purpose framework, custom allocator, etc.

## 2. Setup: IntelliJ and GitHub Copilot

1. IntelliJ IDEA → **Settings** (`⌘ ,`) → **Plugins** → **Installed**; search `GitHub Copilot`.
2. If absent: **Marketplace** → `GitHub Copilot` (publisher GitHub) → Install → restart if prompted.
3. Use **Tools → GitHub Copilot → Login to GitHub** or the Copilot sign-in action. Complete browser authorization with the intended account.
4. Open Copilot **Chat** on the right; verify **Ask / Plan / Agent** in the mode selector.
5. Type a comment and begin a Java method to verify a grey inline suggestion, then accept with `Tab`.
6. Verify the active GitHub account, subscription and enterprise entitlement before assuming a specific feature is included.

**Observed working setup:** Copilot Chat, Ask, Plan, Agent, inline completion, inline chat, custom agents and GitHub MCP all worked in IntelliJ 2026.2 on a personal Mac.

### Useful shortcuts (Mac)

| Action | Shortcut or method | Notes |
|---|---|---|
| Accept inline suggestion | `Tab` | Only after reviewing it |
| Reject suggestion | `Esc` | No code accepted |
| Alternate suggestions | `Option + ]` / `Option + [` | May do nothing if alternatives aren't available; keymap varies |
| Open Copilot Inline Chat | **`Ctrl + Shift + I`** | **Actually worked in our IntelliJ setup**; don't assume another Mac shortcut |
| Find IntelliJ action | `⌘ + Shift + A` | Helpful for discovering commands |
| IntelliJ Settings | `⌘ ,` | Keymap and Copilot Customizations |

**Key lesson:** Plugin UI and shortcuts change. Validate against the installed version and actual IDE keymap; do not rely blindly on screenshots or instructions for another IDE.

## 3. Workspace for independent microservices

Our original projects lived in separate directories and repositories:

```text
~/IdeaProjects/
├── employee-svc/        # 8081; H2, Kafka/outbox, department calls
├── department-svc/      # 8082; department H2/JPA CRUD
├── api-gateway/         # 9090; routes requests
├── discovery-svc/       # 8761; Eureka server
├── hr-mgmt-system/      # separate multi-module learning project
└── microservices-workspace/  # IntelliJ multi-project workspace
```

**IntelliJ Multi-Project Workspace:** Install/enable the JetBrains Multi-Project Workspace plugin if necessary. Create a workspace and add each **existing** project root. Projects remain in their original locations, maintain independent Maven builds and independent `.git` histories.

**Critical distinction:** *An IDE displaying four projects* is not proof that Copilot's current session can inspect all four. Test access explicitly:

```text
Inspect all accessible workspace roots. For employee-svc, department-svc,
api-gateway and discovery-svc, locate pom.xml, the Spring Boot main class,
application.yaml and server port. Give actual paths; say what is inaccessible.
Do not modify files.
```

This verification succeeded for all four repositories. Avoid treating a multi-project workspace as a Git monorepo or Maven reactor.

## 4. Choose the right Copilot capability

| Need | Start with | Why |
|---|---|---|
| Continue writing Java | Inline Completion | Fast, minimal context switching |
| Correct a small method | Inline Chat | Scoped edit, easy diff |
| Understand classes / dependencies | Ask | Explanation without code changes |
| Trace an incident | Ask + logs and code | Evidence-based diagnosis |
| Design a multi-file change | Plan | Clarify contracts, affected files and risks |
| Implement a bounded change | Agent | Read/edit/test iteratively |
| Repeat API reviews | `.prompt.md` | Standardized on-demand task |
| Specialized ongoing reviewer | `.agent.md` | Role + instructions + tool constraints |
| Read live GitHub issues/commits | Agent + MCP | External authenticated context |
| Delegate remote repository task | Cloud coding agent (when enabled) | Isolated remote execution and PR workflow |

Ask, Plan and Agent are distinct modes. **Auto** is model selection, not an agent mode. The exact list of modes/models is installation- and account-dependent.

## 5. Inline completion and inline chat

### Inline completion

Example comment to influence a suggestion:

```java
// Validate department name: reject null or whitespace-only strings;
// minimum 3 characters after trimming.
public boolean isValidDepartmentName(
```

Pause, inspect the grey suggestion and accept with `Tab` only if it meets requirements. Copilot once suggested code equivalent to `name.length() >= 3`, which misses null/blank handling.

### Inline Chat

Select an existing method → press **`Ctrl + Shift + I`** (verified on our Mac) → request a narrow edit:

```text
Improve only the selected method. Reject null, blank and whitespace-only
names. Trim before checking length. Preserve its signature and dependencies.
Do not modify other classes.
```

Review the inline diff before keeping the change. **Inline Completion** predicts next code; **Inline Chat** performs an instructed edit on selected code.

## 6. Ask: understand an unfamiliar application

### Architecture investigation prompt

```text
Analyze this Spring Boot application. Explain the responsibility of each
module, API request flow, cross-service calls, configuration, error handling,
and persistence. Cite actual classes and methods. Identify any inaccessible
files or unverified assumptions. Do not modify anything.
```

### Trace a particular endpoint

```text
Trace GET /api/v1/employees from the controller through services, HTTP
clients, Eureka/load balancing and downstream department response creation.
Show the actual file and method for each stage. Distinguish code facts from
runtime assumptions. Do not modify files.
```

**Practised:** In `DepartmentServiceTest.java`, Copilot identified Mockito `when(...).thenReturn(...)`, the successful `assertEquals`, missing negative-case tests, no explicit `verify(...)`, and a need for Spring-context tests to verify Resilience4j aspects.

**Warning:** “Looks plausible” is not evidence. Verify file references and method signatures; ask which files were actually inspected.

## 7. Plan: design before changing code

A good plan states **scope, current behavior, acceptance criteria, compatibility, risks, files, tests and non-goals**.

### Example: Department write API

```text
Inspect department-svc and plan POST/PUT/DELETE using existing JPA/H2.
Keep GET /api/v1/departments (single object) and /all (collection) unchanged.
POST 201, PUT 200, DELETE 204. Invalid names 400; absent target 404.
Use DTOs, validation and focused tests. Explain ID strategy and migration
risks. Touch department-svc only. Do not implement yet.
```

**Plan questions are valuable:** Copilot asked whether IDs should be client-provided or server-generated, and whether a protected row should yield `409`, `400` or `403`. Answer architectural questions intentionally; Copilot's “Recommended” isn't necessarily right for the real requirement.

**Lessons from our project:**

- We preserved existing GET response shapes rather than silently converting objects to collections.
- We narrowed a large two-service plan to a **Department Phase 1** implementation.
- We challenged a custom ID allocator and discovered that preserving arbitrary H2 state was more complex than our true goal of having convenient seed data.
- Clarifying that sample data is a convenience changed the design priorities. Avoid making sample IDs permanent domain rules without business justification.

Plan-mode `plan.md` was written under `~/.copilot/session-state/`; a “1 file changed” notice referred to the plan artifact, not application code. **Always inspect what changed.**

## 8. Agent: implement, validate and correct

### Agent task contract

```text
Implement the approved department-svc change only.
Acceptance criteria: <explicit endpoints/statuses/behavior>.
Read existing implementation first; preserve API contracts.
Run focused tests, then full module tests if practical.
Report changed files, test results and known limitations.
Do not edit other repositories, delete existing data, commit or push.
```

### Recommended workflow

1. Check `git status` in the **specific** repository and make/preserve a feature branch.
2. Review Plan; hand only the approved task to Agent.
3. Inspect every proposed write or terminal action before approving it.
4. Agent reports success → inspect **Git diff and untracked files**, not only summary.
5. Run `./mvnw test`, `git diff --check`, and representative `curl` calls.
6. Ask a scoped review question; fix findings in a short follow-up rather than regenerate everything.
7. **Keep All** means retain proposed edits in working files; it does **not** commit or push.
8. Commit and merge only the verified code, each repository separately.

### Example runtime checks

```bash
curl -i http://localhost:8082/api/v1/departments
curl -i http://localhost:8082/api/v1/departments/all
curl -i http://localhost:8081/api/v1/employees
curl -i http://localhost:8081/api/v1/employees/all
```

### Seed-data learning

We ended with **local-profile** idempotent SQL seeding for 3 departments and 4 sample employees, preserving pre-existing H2 rows and avoiding Kafka/outbox events during initialization. The Employee collection endpoint `/api/v1/employees/all` was added while the legacy single-object GET remained intact. Sample records are a *developer convenience*, not production reference data.

**Data consistency caveat:** We observed older Employee rows containing department names such as `Tech`, `tech` and `Technology` while the Department collection showed Finance, Engineering and Human Resources. The two services own separate databases; this discrepancy was documented, not silently “fixed.”

## 9. Production incident investigation

### Real scenario from our lab

Employee GET invokes a load-balanced `RestTemplate` call to `http://department-svc/api/v1/departments`. Eureka resolves the logical service name. Resilience4j Retry and CircuitBreaker wrap the employee-side service call.

When Department was stopped, Employee returned HTTP 200 with a synthetic department:

```json
{"id":101,"name":"mindae","department":{"id":0,"name":"circuitbreaker: dept temp unavail"}}
```

A later CircuitBreaker event contained:

```text
type=ERROR
errorMessage=java.lang.IllegalStateException: No instances available for department-svc
stateTransition=null
```

**What this proves:** The named CircuitBreaker fallback response was returned and the breaker recorded an error. **What it does not prove:** A transition to OPEN or the number of retry attempts. The Retry events array was empty for the queried interval.

### Evidence-first incident prompt

```text
Act as a senior Java production-support engineer.
Given the failing endpoint, sanitized logs and repository code:
1. Trace controller → service → downstream client.
2. Separate the immediate exception from underlying cause.
3. Check configured and actually observed retry/breaker/fallback behavior.
4. Rank hypotheses by evidence, noting unknowns.
5. Propose a minimal fix and regression test.
Do not modify files.
```

### Resilience4j / Actuator lab

Employee API listens on **8081**. Agent initially used **8082** for a separate management server—collision risk because Department already uses 8082. Corrected local-only management listener to **`127.0.0.1:8086`**. Activate IntelliJ `local` profile, restart and verify:

```bash
curl -i http://127.0.0.1:8086/actuator/health
curl -s http://127.0.0.1:8086/actuator/retryevents/retryDepartmentService
curl -s http://127.0.0.1:8086/actuator/circuitbreakerevents/departmentService
curl -s 'http://127.0.0.1:8086/actuator/metrics/resilience4j.circuitbreaker.calls?tag=name:departmentService'
```

**Important:** In Docker, `127.0.0.1` inside a container is not the Mac. A loopback-bound management server won't automatically be reachable from a Prometheus container. Account for networking/host bindings when deployment changes.

### Spring local profile in IntelliJ

Run → Edit Configurations → select `EmployeeSvcApplication` / `DepartmentSvcApplication` → set **Active profiles** to `local`, or program arguments:

```text
--spring.profiles.active=local
```

Verify startup log: `The following 1 profile is active: "local"`. Do this in **both** independent run configurations.

## 10. Repository instructions and context engineering

**Location:** `.github/copilot-instructions.md` in each repository. Copilot can include it as persistent repository context; verify via response References, not just the model claiming it read it.

Example:

```markdown
# Department Service — Copilot Instructions

- Independent Spring Boot service using Spring Data JPA and H2.
- Prefer constructor injection and thin controllers.
- Preserve existing REST response contracts.
- Use existing entities, DTOs, repositories and conventions.
- Use JUnit 5/Mockito for unit tests; integration tests for HTTP/JPA.
- Never report tests as passed unless executed.
- Do not change unrelated services or commit/push without authorization.
- Never add secrets or generated database files to Git.
```

Context types:

| Type | Examples | Usage |
|---|---|---|
| Automatic | Active file, discoverable workspace source | Helpful but not exhaustive |
| Explicit | Attached class, selected method, logs | Best for bounded questions |
| Persistent | `copilot-instructions.md` | Repeated project-wide rules |
| External | MCP tool data | Live GitHub/issue systems when authorized |

**Efficiency:** Start a new conversation for a new objective; retain a thread for the same feature; attach only relevant context; ask Copilot to identify gaps instead of guessing.

## 11. Reusable prompt files

**Location:** `.github/prompts/review-spring-api.prompt.md`  
**Invoke:** `/review-spring-api` in Copilot Chat (when discovered by the plugin).

Example content:

```markdown
# Review Spring Boot API Changes
Act as a senior Java/Spring Boot reviewer.
Review changed files and relevant context for:
- Correctness, HTTP contracts and backward compatibility
- Validation, error handling and status codes
- Transactions, concurrency and persistence
- Security, logging and sensitive data
- Missing tests and unrelated modifications
Report P1/P2/P3, file/line, evidence and smallest correction.
Distinguish confirmed defects from possible risks.
Do not modify files.
```

**Real result:** This prompt identified a default-profile POST defect: the Department ID allocator table had been created only by `data-local.sql`. It also flagged missing maximum name-length validation. Agent moved allocator schema/bootstrap setup to `schema.sql`, preserved local-only sample rows, added a 255-character limit and a default-profile POST test. Tests reported **14 passed**.

When to make a prompt file: Once a task is repeatedly useful and the instructions have stabilized. Do **not** create dozens proactively.

## 12. Custom agents and agent skills

### Custom Agent

**Location:** `.github/agents/spring-reviewer.agent.md`

```markdown
---
name: spring-reviewer
description: Reviews Spring Boot APIs for defects and compatibility risks.
---

# Spring Boot Reviewer
Read relevant code and tests. Check API responses, validation,
transactions, error mapping, security, profile initialization and tests.
Report P1/P2/P3 findings with paths/lines and evidence.
Distinguish confirmed defects from conditional risks.
Do not modify files, run commands, commit or push.
```

**Note:** The exact supported frontmatter/tool names and enforcement depend on the plugin. Select appropriate read/search permissions through **Configure Tools** where offered. Instructions alone aren't a security sandbox.

In IntelliJ 2026.2 the verified route was **Settings → Tools → GitHub Copilot → Customizations → Agents**, which lists agent-file search locations (including `.github/agents`). This panel configures *locations*, not necessarily a “Create Agent” button. We created the file manually, then selected `spring-reviewer` beside Ask/Plan/Agent. The specialized agent found a genuine default-profile issue with the legacy Department GET that depended on sample ID 100. We chose to document it rather than make sample data a production requirement.

### Agent Skill

**Agent = specialist/role; Skill = reusable procedure** with instructions and optionally scripts/templates/reference materials. Example skill: investigate a microservice incident via logs, dependency chain, retries and regression tests. We covered the concept, **not** a full practical skill installation.

## 13. Model Context Protocol (MCP)

**MCP** allows compatible AI clients to discover and call tools from external servers. The client/server needs explicit authorization and permitted tools. It does not grant automatic universal access.

### Two easy-to-confuse IntelliJ settings

| Setting | Direction |
|---|---|
| **Tools → MCP Server → Exposed Tools** | IntelliJ offers IDE operations to outside MCP clients |
| **Tools → GitHub Copilot → Model Context Protocol** | GitHub Copilot connects to external MCP servers |
| **Tools → External Tools** | IntelliJ runs external commands; not the Copilot MCP client |

### What worked on our Mac

1. Copilot Settings → **Model Context Protocol (MCP)**; built-in read-only GitHub server option was enabled.
2. Click **Configure** to open local `mcp.json`, under `~/.config/github-copilot/intellij/` in our setup.
3. Example configuration for a remote server (confirm current support/authentication in your plugin; do **not** hard-code tokens):

```json
{
  "servers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/"
    }
  }
}
```

4. Authorize in the browser when prompted. Start a **new Agent chat**.
5. **Configure Tools was the sliders icon next to `Auto` in the Copilot chat composer**—not the Ask/Plan/Agent dropdown.
6. In the tool picker, `github` showed **Running**, 28 tools, 2 prompts and 6 resources; select only needed tools, click **Apply**.
7. If a current session can't see a newly enabled tool, open a fresh Agent session; optionally **Restart** the server in the tools panel.
8. A successful remote tool call was observed:

```text
github-list_branches(owner="mindae", repo="department-svc", perPage=100, page=1)
github-list_commits(owner="mindae", repo="department-svc", perPage=5, page=1, ...)
```

The remote default branch was `master`; we retrieved its latest commits (including an allocator/validation correction). This **proved actual tool execution**, unlike a response that merely claims “MCP connected.”

**Operating tips:** A running server alone doesn't necessarily consume AI credits; model/tool-processing turns do. Avoid duplicate built-in and manually configured GitHub servers. Prefer read-only scopes, minimal tools and approved providers. Never put personal access tokens in committed `mcp.json`.

## 14. Git, code review and multi-repository delivery

Each microservice is a separate repo. One cross-service feature therefore has **separate branches, commits and possibly PRs**.

```bash
cd ~/IdeaProjects/department-svc
git status
git branch --show-current
git switch -c feature/my-change  # only from a clean intended baseline
# Agent edits → review → tests
git diff --stat
git diff
git diff --cached
./mvnw test
git diff --check
```

After staging only intended paths:

```bash
git add pom.xml src/ .github/  # only if these paths contain intended changes
git diff --cached --stat
git commit -m "feat(department): describe change"
git switch master
git merge --no-ff feature/my-change
git push origin master
```

Review the *actual* branch and uncommitted changes before switching/merging. In a real enterprise project follow PR, CI and branch-protection policy; local merge/push was a learning shortcut, not the default corporate approach.

### Generated H2 files — recurring pitfall

```gitignore
data/*.mv.db
data/*.trace.db
```

If a DB file was already tracked, `.gitignore` alone is insufficient:

```bash
git rm --cached data/department-db.mv.db
```

`--cached` retains the local file while removing it from index. **Removing it in a later commit does not erase it from previous Git history.** Check for sensitive data before publishing and clean history if necessary.

### Copilot code review options

- Ask/custom reviewer: scoped analysis of selected classes or Git diff.
- Dedicated local code review: may appear in the commit workflow, but **was not found in our IntelliJ 2026.2 UI** despite investigation; don't conflate “Generate Commit Message” with “Review Code Changes.”
- GitHub PR review: separate workflow with PR and supported GitHub entitlement; we discussed it but did **not** fully exercise formal PR review.

## 15. AI credits, quotas and cost awareness

The October 2026 IntelliJ UI showed **GitHub Copilot Free** with separate usage meters:

| Meter | Observed | Renewal shown |
|---|---:|---|
| AI credits | 11.6% used | Nov 1, 2026 |
| Inline suggestions | 45.1% used | Nov 1, 2026 |

**Do not infer a fixed numeric Free AI-credit limit from percentages alone.** Check the current GitHub account/billing UI and official documentation; limits and plans change. At the time of our lesson, the published Free inline completion allowance was 2,000 per month, so 45.1% corresponds to about 902 completions, assuming that allowance applies. Free AI-credit total wasn't shown numerically in our screenshot.

- Credits reflect metered model usage; heavier Agent tasks and repeated iterations can use more than a short Ask answer.
- **Inline completion** and **chat/agent credits** may use separate counters and plan rules.
- A per-turn line such as `0.3 Credits` isn't necessarily a direct out-of-pocket payment; compare it with plan allowance, budgets and current billing terms.
- In the observed interface: GitHub Copilot status → **AI Credit Usage** showed percentages and reset dates, plus usage notifications.
- Keep **Auto** as a good starting model choice. Use stronger reasoning models selectively for complex bugs/architecture; avoid model switching as a substitute for better context.
- Short tasks, focused attachments, new chat for a new objective, and only necessary MCP tools help manage token/context costs.
- Corporate Business/Enterprise accounts may have pooled allowances, policy restrictions and organizational budgets different from a personal Free account.
- GitHub Copilot and ChatGPT subscriptions are independent.

## 16. Enterprise security and operating practices

**Before delegating:** Identify the repository and branch, define allowed files/commands, set acceptance criteria and data-handling restrictions.

**While Agent works:** Inspect shell commands, database operations, migrations, network calls and credentials exposure. “Default approvals” are safer for initial use than blanket auto-approval. A role description like “read-only reviewer” is not a security boundary if tools grant write access.

**Afterward:** Check diffs, tests, build, curl results, accidental changes, missing negative cases, log sensitivity and dependencies. Don't commit/push automatically merely because tests passed. For production, require standard reviews and CI.

**MCP:** Use only authorized external servers, least-privilege tool sets and approved authentication. Don't paste secrets into prompts/config files or send confidential project code to unapproved services.

**Cloud coding agents:** Useful when permitted for isolated, well-scoped remote repository tasks; confirm environment, permissions, data access and PR/CI process. We covered this at a conceptual level, not as a hands-on run.

## 17. Hands-on case studies and lessons learned

| Exercise | What we learned |
|---|---|
| Inspect `DepartmentServiceTest` | Ask explains Mockito and finds coverage gaps |
| Multi-project architecture scan | Prove actual access to four independently versioned projects |
| Department H2/JPA phase 1 | Plan scope; Agent multi-file implementation; preserve API contract |
| Seed script used H2 `MERGE ... KEY(id)` | “Idempotent” can still overwrite existing data |
| H2 console enabled globally | Dev convenience must not automatically be a production exposure |
| First H2 database committed | Generated binary data must be ignored and untracked |
| Employee fallback investigation | Response message and logs do not prove retry count or breaker OPEN state |
| Agent moved management to 8082 | Local fix can conflict with another service's port |
| Local profile for management 8086 | Runtime context differs between IntelliJ run configs and Agent tests |
| H2 lock during Maven tests | Use isolated test DB; stop running process only as short-term workaround |
| Department write API | Acceptance criteria: 201/200/204/400/404/409, validation, compatibility |
| Custom ID allocator | Challenge complexity; understand existing data/migration constraints |
| Local employee/department seed data | SQL initializes convenient examples without Kafka outbox side effects |
| Reusable review prompt | Caught allocator schema initialization missing in default profile |
| Custom review agent | Found legacy GET depending on local-only sample ID 100 |
| GitHub MCP | Server Running ≠ tools available in a session; verify real call |
| Credits UI | Free plan showed % used and renewal, not numeric free-credit total |

## 18. Reusable prompt cookbook

### A. Understand an unfamiliar module

```text
Explain the responsibilities and execution flow of <module>. Name real
controllers, services, repositories, configurations and downstream calls.
Distinguish observations from assumptions; do not modify files.
```

### B. Plan a safe feature

```text
Inspect the existing implementation and plan <feature>.
Acceptance criteria: <criteria>.
Preserve <API/contracts>; scope to <repository>; identify affected files,
migrations, risks and tests. Don't implement yet.
```

### C. Implement only the approved plan

```text
Implement the reviewed plan within <repo>. Keep changes minimal.
Run tests; show files changed and any limitations. Do not commit or push.
```

### D. Diagnose incident

```text
Given <symptom/logs>, trace the actual code path and dependencies.
Identify verified observations, competing hypotheses and missing evidence.
Recommend a minimal fix with regression tests. Do not change code.
```

### E. Review a diff

```text
Review the diff for correctness, API compatibility, security, validation,
transactions, failures and tests. Report significant P1/P2/P3 findings with
file/line and evidence. Distinguish confirmed defects from risks. No edits.
```

### F. Verify external MCP data

```text
Use only <named MCP read tools> to retrieve <resource>. Report the exact
tool calls used. Do not use local substitutes or modify remote resources.
If tools are unavailable, say so clearly.
```

## 19. Daily quick reference and troubleshooting

### First 5 minutes on an unfamiliar repository

```text
1. git status + actual branch
2. Ask: identify architecture, entry points, tests and conventions
3. Read copilot-instructions.md and relevant documentation
4. Trace target API end-to-end
5. Agree on change scope and acceptance criteria
```

### Common troubleshooting

| Symptom | Check |
|---|---|
| No inline suggestion | Plugin status, file type, completion settings, keymap; try a comment and partial signature |
| No alternate completion | No alternatives may be available |
| Inline Chat doesn't open | In our IntelliJ setup `Ctrl + Shift + I` worked |
| Copilot sees only one service | Verify IntelliJ multi-project workspace **and** actual agent file access |
| Custom agent missing | `.github/agents/*.agent.md`; Settings → Copilot → Customizations → Agents search paths; new chat |
| MCP menu confusion | Sliders next to `Auto` → **Configure Tools**, not Agent selector |
| GitHub MCP Running, tools unavailable | Check boxes + **Apply**; fresh Agent session; optionally Restart server |
| Port 8086 connection refused | Restart Employee with `local` profile; verify management Tomcat startup |
| Employee fallback has HTTP 200 | Check fallback implementation and logs; do not assume downstream success |
| H2 DB unexpectedly staged | `.gitignore`, `git ls-files data/`, `git rm --cached ...` |
| Test H2 file locked | Use in-memory test database; don't share runtime DB with test suite |
| Copilot says tests passed | Check actual command output and tests; run locally |
| `#` lines fail in interactive zsh | `setopt interactivecomments` or paste command-only blocks |

### Local endpoints in this lab

```text
Employee API:          http://localhost:8081/api/v1/employees
Employee collection:   http://localhost:8081/api/v1/employees/all
Department API:        http://localhost:8082/api/v1/departments
Department collection: http://localhost:8082/api/v1/departments/all
Gateway:               http://localhost:9090
Eureka:                http://localhost:8761
Employee Actuator:     http://127.0.0.1:8086/actuator/health (local profile)
```

**Ports and paths reflect our local learning environment, not general Spring defaults.**

## 20. Progress log, maintenance and next passes

### Bootcamp coverage (October 2026)

| Topic | Status |
|---|---|
| Installation, authentication, inline completion | **Hands-on complete** |
| Inline Chat selected edit | **Hands-on complete** |
| Multi-project IntelliJ workspace | **Hands-on complete** |
| Ask, Plan and Agent | **Hands-on complete** |
| Runtime debugging and tests | **Hands-on complete** |
| Repository instructions | **Hands-on complete** |
| Prompt file `/review-spring-api` | **Hands-on complete** |
| Custom agent `spring-reviewer` | **Hands-on complete** |
| GitHub MCP read-only tools | **Hands-on complete** |
| Git branch/commit/merge/push | **Practised** |
| Dedicated local Copilot Code Review UI | **Explored; button not found** |
| Formal GitHub PR review/CI | **Overview; hands-on later** |
| Agent Skills | **Conceptual overview; hands-on later** |
| Cloud coding agents and subagents | **Overview; hands-on later** |
| Enterprise admin controls/billing | **Basic awareness; revisit in corporate environment** |

### Second-pass backlog (only when needed)

- Review a genuine pull request with Copilot and CI checks.
- Create one reusable **Agent Skill** for incident investigation.
- Try a cloud coding agent on a low-risk isolated repo task, if corporate policy allows.
- Explore precise model/credit metering for the organization-provided Copilot plan.
- Improve consistent data references across Employee/Department without treating sample IDs as permanent business rules.
- Revisit native JPA sequence/identity migration if database portability or permanent ID non-reuse becomes important.
- Configure corporate-approved MCP services (issue tracker, docs, observability) with least privilege.

### How we'll maintain this handbook

Whenever we cover a useful new Copilot topic, say:

> **Update our GitHub Copilot Markdown notes.**

Consolidate the newly learned commands, confirmed UI paths, examples, mistakes, corrected practices and progress into this `README.md`. Preserve earlier lessons, revise information that has become outdated, and clearly separate **tested behavior** from **overview-only features**. Keep the document practical and suitable for reading offline or from an enterprise environment with limited ChatGPT access.

Suggested Git location:

```text
TECHNICAL-DOCUMENTS/
├── AI-LEARNING/
├── NETWORKING/
└── GITHUB-COPILOT/
    └── README.md
```

## 21. References

These official links are useful for checking feature behavior, especially as the IntelliJ plugin and GitHub billing models evolve.

- [GitHub Copilot documentation](https://docs.github.com/en/copilot)
- [Repository custom instructions in an IDE](https://docs.github.com/en/copilot/how-tos/copilot-in-your-ide/customize-copilot/configure-custom-instructions/add-repository-instructions-in-your-ide)
- [Extending Copilot Chat with MCP in JetBrains](https://docs.github.com/en/copilot/customizing-copilot/extending-copilot-chat-with-mcp?tool=jetbrains)
- [GitHub Copilot billing and usage](https://docs.github.com/en/copilot/concepts/billing-and-usage)
- [GitHub CLI](https://cli.github.com/manual/)
- [JetBrains IDE documentation](https://www.jetbrains.com/help/idea/)
- [Our Technical Documents repository](https://github.com/mindae/TECHNICAL-DOCUMENTS)

---

**Bottom line:** Copilot is most effective when treated as a capable implementation and investigation partner: provide context, control scope, require evidence, review changes, test behavior and keep developer responsibility for decisions and delivery.
