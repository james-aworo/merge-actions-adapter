---
name: merge-actions-adapters
version: 0.1.2
description: |
  Consolidates thin Actions boundary wrappers after the first Actions refactor pass.

  Use when Actions has multiple adapter or use case wrappers that should be merged into one wrapper per foreign domain.
  The skill can consolidate outgoing adapters, incoming use cases, or both, starting from a selected Actions service or selected domain.

  This skill preserves behavior. It does not redesign services, rewrite business logic, rename service methods,
  change repositories, change DTOs, or perform full hexagonal architecture. It only consolidates wrapper classes
  and updates call sites to use the consolidated wrapper.

triggers:
  - consolidate actions adapters
  - merge actions adapters
  - consolidate actions use cases
  - merge actions use cases
  - actions adapter cleanup
  - actions wrapper consolidation
  - actions domain adapter
  - merge action adapters by domain
  - consolidate boundary wrappers

allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - MultiEdit
  - Glob
  - Grep
  - AskQuestion
  - Jetbrains
---

# merge-actions-adapters

## 0. Role

You are a guarded Actions boundary-wrapper consolidation controller.

Your job is to consolidate already-created thin Actions wrappers into clearer domain-level wrappers while preserving behavior exactly.

This is not the first Actions relocation skill.
This is not a general refactoring skill.
This is not a service cleanup skill.
This is not a full hexagonal architecture skill.
This is not a “delete old files because grep looked empty once” skill.

The skill exists because the first relocation pass may create multiple safe but noisy wrappers. This skill turns those wrappers into a cleaner boundary shape: one wrapper per foreign domain, when safe.

Core rule:

```text
Audit first.
Consolidate only selected wrapper type and selected scope.
Preserve behavior.
Use one wrapper per foreign domain.
Delete only when the user explicitly selects a delete-capable mode.
Record uncertainty in review.jsonl.
```

Definitions:

```text
Outgoing adapter:
  An Actions-owned wrapper used by Actions services to call foreign-domain code.

Incoming use case:
  An Actions-owned wrapper used by foreign-domain code to call Actions-owned code.

Source wrapper:
  Existing adapter/use case that may be merged into a consolidated wrapper.

Consolidated wrapper:
  The domain-level adapter/use case that should remain after consolidation.

Foreign domain:
  The owning domain/package outside Actions, such as Target, Alert, User, Incident, ReportFile, Permission, Score, Marketplace, Collection, Perspective, SelfAssessment, etc.
```

---

## 1. Operating Principle

The first Actions refactor optimized for safe relocation and explicit dependency boundaries. That may produce several adapter classes around the same foreign domain.

This skill optimizes the wrapper shape after that pass.

It must keep the operation mechanical:

```text
multiple thin wrappers for same domain
-> one domain-level wrapper
-> same forwarding methods
-> same call order
-> same behavior
```

Do not make the code “better” while consolidating. Better is how refactors become archaeological digs with test failures wearing hats.

---

## 2. Required Human Input

The user must specify what to consolidate and where to start.

Required fields:

```text
wrapper: adapters | usecases | both
start: service:<ActionsServiceName> | domain:<ForeignDomainName> | wrapper:<ExistingWrapperName>
mode: audit_only | apply | apply_allow_delete
```

Examples:

```text
wrapper: adapters
start: service:ActionService
mode: audit_only
```

```text
wrapper: adapters
start: domain:Target
mode: apply
```

```text
wrapper: usecases
start: service:ActionService
mode: apply_allow_delete
```

If required input is missing, ask one focused question:

```text
What should I consolidate? Provide wrapper type, starting point, and mode, for example: `wrapper: adapters`, `start: service:ActionService`, `mode: audit_only`.
```

Do not guess mode.
Do not assume deletion is allowed.

---

## 3. Modes

### 3.1 audit_only

```text
- inspect wrappers
- group by foreign domain
- propose consolidation plan
- write audit JSON
- do not edit application files
- do not delete files
```

### 3.2 apply

```text
- create or update consolidated wrappers
- move/copy forwarding methods into consolidated wrappers
- update injections/imports/call sites
- run checks where available
- write patch report
- write review tasks
- do not delete old wrapper files
```

Old wrappers may become unused. Leave them in place and record cleanup candidates.

### 3.3 apply_allow_delete

```text
- perform everything in apply
- delete obsolete old wrapper files only after all delete gates pass
```

Deletion requires all of these:

```text
mode == apply_allow_delete
AND source wrapper has zero usages after consolidation
AND source wrapper is an adapter/use case wrapper only
AND source wrapper is under domains/actions/adapter or domains/actions/usecase
AND source wrapper contains no business logic
AND source wrapper is not a service, repository, controller, data type, mapper, validator, config, or test
AND compile/checks pass before deletion or the failure is proven unrelated
```

If any delete gate fails, do not delete. Record a cleanup review task.

---

## 4. Naming Conventions

### 4.1 Outgoing adapters

Use this standard name:

```text
<Domain>ActionAdapter
```

Examples:

```text
TargetActionAdapter
UserActionAdapter
AlertActionAdapter
IncidentActionAdapter
QuestionnaireActionAdapter
ReportFileActionAdapter
PermissionActionAdapter
ScoreActionAdapter
MarketplaceActionAdapter
CollectionActionAdapter
PerspectiveActionAdapter
```

Do not use these new names for consolidated domain-level adapters:

```text
TargetServiceActionsAdapter
TargetMetaServiceActionsAdapter
TargetJpaRepositoryActionsAdapter
TargetActionsAdapter
TargetAdapter
TargetProxy
TargetGateway
```

If an old wrapper already uses a non-standard name, it may be a source wrapper. The consolidated target must use `<Domain>ActionAdapter` unless the human explicitly says otherwise.

### 4.2 Incoming use cases

Use this standard name:

```text
<Domain>ActionUseCase
```

Examples:

```text
AlertActionUseCase
TargetActionUseCase
MarketplaceActionUseCase
UserActionUseCase
IncidentActionUseCase
```

Use case names describe the foreign domain that calls into Actions, not the internal Actions service.

---

## 5. Target Package Layout

Canonical packages:

```text
domains/actions/adapter
domains/actions/usecase
```

Rules:

```text
Outgoing adapters -> domains/actions/adapter
Incoming use cases -> domains/actions/usecase
```

Do not move services, repositories, controllers, data, mappers, validators, or configs as part of this skill.

---

## 6. Bootstrap: Run First

Run this before auditing, editing, or deleting.

```bash
set -euo pipefail

RUN_ID="$(date -u +%Y%m%dT%H%M%SZ)-$$"
REPO_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || true)"

if [ -z "$REPO_ROOT" ]; then
  echo "BLOCKED: not_a_git_repository"
  exit 0
fi

cd "$REPO_ROOT"
REPO_ROOT="$(pwd -P)"
BRANCH="$(git branch --show-current 2>/dev/null || echo detached)"
HEAD_SHA="$(git rev-parse HEAD 2>/dev/null || echo unknown)"

STATE_DIR=".cursor/plans/merge-actions-adapters/state"
AUDIT_DIR="$STATE_DIR/audits"
PATCH_DIR="$STATE_DIR/patches"
TMP_DIR="$STATE_DIR/tmp"
LOG_FILE="$STATE_DIR/runs.jsonl"
REVIEW_FILE="$STATE_DIR/review.jsonl"

mkdir -p "$AUDIT_DIR" "$PATCH_DIR" "$TMP_DIR"
touch "$LOG_FILE"
touch "$REVIEW_FILE"

RUN_ID="$RUN_ID" \
REPO_ROOT="$REPO_ROOT" \
BRANCH="$BRANCH" \
HEAD_SHA="$HEAD_SHA" \
LOG_FILE="$LOG_FILE" \
python3 - <<'PY'
import json
import os
from pathlib import Path
from datetime import datetime, timezone

event = {
  "ts": datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ"),
  "event": "started",
  "skill": "merge-actions-adapters",
  "version": next(
      (
          line.split(":", 1)[1].strip()
          for path in (
              Path(os.environ.get("SKILL_FILE", "")),
              Path.home() / ".cursor/skills/merge-actions-adpter/SKILL.md",
          )
          if path.is_file()
          for line in path.read_text(encoding="utf-8").splitlines()
          if line.startswith("version:")
      ),
      "unknown",
  ),
  "runId": os.environ["RUN_ID"],
  "repoRoot": os.environ["REPO_ROOT"],
  "branch": os.environ["BRANCH"],
  "head": os.environ["HEAD_SHA"],
}
log_file = Path(os.environ["LOG_FILE"])
log_file.parent.mkdir(parents=True, exist_ok=True)
with log_file.open("a", encoding="utf-8") as fh:
    fh.write(json.dumps(event, separators=(",", ":")) + "\n")
PY

echo "RUN_ID=$RUN_ID"
echo "REPO_ROOT=$REPO_ROOT"
echo "BRANCH=$BRANCH"
echo "HEAD=$HEAD_SHA"
echo "STATE_DIR=$STATE_DIR"
echo "AUDIT_DIR=$AUDIT_DIR"
echo "PATCH_DIR=$PATCH_DIR"
echo "REVIEW_FILE=$REVIEW_FILE"
```

If bootstrap prints `BLOCKED`, stop and report the blocker.

Expected ignored state path:

```text
.cursor/plans/merge-actions-adapters/state
```

If this path is not ignored, warn in the final response.

---

## 7. Workflow

Follow this order exactly:

```text
Step 0: Bootstrap repository and run context
Step 1: Parse user input: wrapper, start, mode
Step 2: Resolve scope
Step 3: Find candidate wrappers
Step 4: Group wrappers by foreign domain
Step 5: Build consolidation plan
Step 6: Write audit JSON
Step 7: Stop unless mode allows apply
Step 8: Create or update consolidated wrapper
Step 9: Move forwarding methods into consolidated wrapper
Step 10: Update constructor injection, imports, and call sites
Step 11: Verify old wrappers have zero usages if deletion is requested
Step 12: Delete obsolete wrappers only in apply_allow_delete mode and only after delete gates pass
Step 13: Run checks
Step 14: Write patch report
Step 15: Append review tasks
Step 16: Report result
```

The audit step cannot edit files.
The apply step cannot delete files.
Only `apply_allow_delete` may delete files.

---

## 8. Scope Resolution

### 8.1 Start from service

Input:

```text
start: service:ActionService
```

Behavior:

```text
- resolve the Actions service class
- inspect constructor dependencies
- find adapter/use case wrappers injected into that service
- find wrapper method calls from that service
- group only wrappers reached from that service unless the human expands scope
```

Do not scan every Actions service unless explicitly requested.

### 8.2 Start from domain

Input:

```text
start: domain:Target
```

Behavior:

```text
- find wrappers under domains/actions/adapter and domains/actions/usecase that appear to belong to the Target domain
- include wrappers whose constructor dependencies, imports, class names, or package names point to Target
- group them into TargetActionAdapter or TargetActionUseCase depending on wrapper type
```

### 8.3 Start from wrapper

Input:

```text
start: wrapper:TargetServiceActionsAdapter
```

Behavior:

```text
- resolve that wrapper
- infer its foreign domain from constructor dependency/import/package/class name
- find sibling wrappers in the same wrapper type that belong to the same foreign domain
- propose consolidation
```

If domain inference is ambiguous, stop as `BLOCKED` or ask the human to provide `domain:<DomainName>`.

---

## 9. Wrapper Discovery

Search only under Actions wrapper packages unless the user explicitly expands scope:

```text
domains/actions/adapter
domains/actions/usecase
```

Wrapper candidates usually match:

```text
*ActionAdapter
*ActionsAdapter
*ServiceActionsAdapter
*RepositoryActionsAdapter
*JpaRepositoryActionsAdapter
*EvaluatorActionsAdapter
*ActionUseCase
*ActionsUseCase
```

Do not treat these as wrapper candidates:

```text
controllers
services
repositories
data classes
payloads
responses
mappers
validators
configs
tests
generated files
```

If a class has business logic, it is not safe to consolidate automatically. Record a review task.

---

## 10. Domain Grouping Rules

Group wrappers by foreign domain using evidence in this order:

```text
1. Explicit user-provided domain
2. Constructor dependency type
3. Imported package path
4. Existing wrapper class name
5. Current package/folder name
6. Call-site usage context
```

Examples:

```text
TargetServiceActionsAdapter           -> Target
TargetMetaServiceActionsAdapter       -> Target
TargetJpaRepositoryActionsAdapter     -> Target
TargetLogoServiceActionsAdapter       -> Target
TargetPermissionEvaluatorActionsAdapter -> Target
```

Consolidated adapter:

```text
TargetActionAdapter
```

Examples:

```text
UserJpaRepositoryActionsAdapter -> UserActionAdapter
IncidentServiceActionsAdapter   -> IncidentActionAdapter
ReportFileServiceActionsAdapter -> ReportFileActionAdapter
ScoreContextServiceActionsAdapter -> ScoreActionAdapter
```

If one wrapper clearly belongs to a different domain, do not merge it just because the selected service uses it.

When uncertain:

```text
- do not guess
- do not merge
- write a review task
```

---

## 11. Consolidation Rules

### 11.1 Outgoing adapters

When consolidating adapters:

```text
- create or update <Domain>ActionAdapter
- copy/move forwarding methods from source adapters into the consolidated adapter
- preserve method names and signatures
- preserve return types
- preserve nullability
- preserve suspend/operator/inline modifiers if any
- preserve annotations if they exist and are necessary
- inject original foreign dependencies into the consolidated adapter
- update Actions services to inject the consolidated adapter
- update call sites to use the consolidated adapter
```

Allowed method body:

```kotlin
fun fetchParents(targets: List<Target>, customerId: Int, organizationId: Int) =
    targetService.fetchParents(targets, customerId, organizationId)
```

Forbidden method body:

```kotlin
fun fetchParents(targets: List<Target>, customerId: Int, organizationId: Int) {
    if (targets.isNotEmpty()) {
        targetService.fetchParents(targets, customerId, organizationId)
    }
}
```

### 11.2 Incoming use cases

When consolidating use cases:

```text
- create or update <Domain>ActionUseCase
- copy/move forwarding methods from source use cases into the consolidated use case
- preserve method names and signatures
- preserve return types
- preserve nullability
- inject original Actions-owned services into the consolidated use case
- update foreign callers to inject and call the consolidated use case
```

Do not change the Actions service method being called.
Do not rename foreign caller methods.
Do not change foreign-domain logic.

---

## 12. One-Call Forwarding Rule

Every method in a consolidated wrapper must contain exactly one forwarded method invocation.

Allowed:

```kotlin
fun findById(id: Int) =
    userJpaRepository.findById(id)
```

Allowed when Kotlin block body is required:

```kotlin
fun publish(event: ActionEvent) {
    applicationEventPublisher.publishEvent(event)
}
```

Forbidden:

```kotlin
fun findById(id: Int): User? {
    val user = userJpaRepository.findById(id)
    return user.orElse(null)
}
```

Forbidden operations inside wrappers:

```text
- if / when branching
- loops
- try/catch
- null fallback
- collection mapping
- filtering
- sorting
- validation
- permission checks
- logging newly introduced by the skill
- event publishing plus extra logic
- combining multiple foreign calls into one method
```

If a source wrapper already contains logic, do not consolidate automatically. Record a review task.

---

## 13. Delete Safety Rules

Deletion is forbidden unless `mode: apply_allow_delete`.

In `apply`, old wrappers remain in place even if unused.

Before deleting any source wrapper, verify zero usages with at least two evidence paths:

```text
1. IDE/symbol usage search if available
2. grep/ripgrep textual search
```

Suggested search:

```bash
rg "SourceWrapperName" . \
  --glob '!build/**' \
  --glob '!out/**' \
  --glob '!dist/**' \
  --glob '!node_modules/**' \
  --glob '!generated/**'
```

Delete only if remaining matches are limited to:

```text
- the source wrapper file itself
- audit/patch/review state files
- git diff metadata
```

Never delete:

```text
- controllers
- services
- repositories
- data classes
- tests
- configs
- mappers
- validators
- generated files
- wrappers outside domains/actions/adapter or domains/actions/usecase
- wrappers with remaining usages
- wrappers with logic
```

If deletion is skipped, append a cleanup review task.

---

## 14. Audit JSON

Before editing, write audit JSON to:

```text
.cursor/plans/merge-actions-adapters/state/audits/<RUN_ID>.audit.json
```

Shape:

```json
{
  "runId": "string",
  "timestamp": "ISO-8601",
  "request": {
    "raw": "string",
    "wrapper": "adapters | usecases | both",
    "start": "service:<name> | domain:<name> | wrapper:<name>",
    "mode": "audit_only | apply | apply_allow_delete"
  },
  "repository": {
    "root": "string",
    "branch": "string",
    "head": "string",
    "dirty": false
  },
  "skill": {
    "name": "merge-actions-adapters",
    "version": "<skill version from SKILL.md frontmatter>"
  },
  "scope": {
    "startKind": "service | domain | wrapper",
    "startName": "string",
    "wrapperType": "adapters | usecases | both",
    "confidence": "high | medium | low"
  },
  "groups": [
    {
      "domain": "string",
      "targetWrapper": "string",
      "sourceWrappers": ["string"],
      "foreignDependencies": ["string"],
      "callSites": ["string"],
      "confidence": "high | medium | low"
    }
  ],
  "plannedChanges": {
    "createOrUpdateWrappers": ["string"],
    "moveForwardingMethods": [
      {
        "from": "string",
        "to": "string",
        "methods": ["string"]
      }
    ],
    "updateCallSites": ["string"],
    "deleteCandidates": ["string"]
  },
  "classification": {
    "risk": "LOW_RISK | MEDIUM_RISK | HIGH_RISK | BLOCKED",
    "allowedToApply": false,
    "deleteAllowed": false,
    "requiresBehaviorChange": false,
    "wrapperHasLogic": false,
    "domainGroupingAmbiguous": false,
    "remainingUsagesFound": false,
    "requiresServiceLogicChange": false,
    "requiresRepositoryChange": false,
    "requiresBuildChange": false
  },
  "decision": {
    "summary": "string",
    "reasons": ["string"],
    "stopReason": "string | null",
    "nextSafeAction": "string"
  }
}
```

### 14.1 allowedToApply Rule

Set `allowedToApply=true` only when:

```text
risk == LOW_RISK
AND scope.confidence == high
AND each group confidence == high
AND no wrapper method requires logic
AND no behavior change is required
AND no service logic change is required
AND no repository change is required
AND no build change is required
AND target wrapper names follow the naming convention
```

Set `deleteAllowed=true` only when:

```text
allowedToApply == true
AND mode == apply_allow_delete
AND every delete candidate passes delete safety rules
```

---

## 15. Stop Gate

After writing audit JSON:

```text
IF mode == audit_only:
  report audit result
  do not edit files
  stop

IF allowedToApply != true:
  report audit result
  do not edit files
  stop
```

Allowed stop reasons:

```text
not_a_git_repository
missing_required_input
start_service_not_found
start_wrapper_not_found
wrapper_package_not_found
domain_grouping_ambiguous
source_wrapper_has_logic
consolidated_wrapper_would_need_logic
requires_behavior_change
requires_service_logic_change
requires_repository_change
requires_build_change
existing_uncommitted_changes_conflict
```

---

## 16. Apply Rules

When applying:

```text
- create/update the consolidated wrapper first
- preserve all forwarding method signatures
- inject required original dependencies into the consolidated wrapper
- update service/use-case/caller injection to use the consolidated wrapper
- update imports
- preserve call order
- do not change business logic
- do not rename service methods
- do not rename repository methods
- do not rename data types
- do not reformat unrelated code
```

When moving methods from source wrappers:

```text
- if method name conflicts with an existing method with identical signature and body, reuse existing method
- if method name conflicts but body differs, stop and record a review task
- if two source wrappers expose same method name for different dependencies, preserve behavior by keeping distinct method names or stop if ambiguity exists
```

Do not consolidate two wrappers if doing so requires changing call semantics.

---

## 17. Review Task Output

Use machine-readable JSONL for follow-up tasks.

File:

```text
review.jsonl
```

Each line must be one valid JSON object.
Append only.
Do not overwrite previous tasks.
Do not write Markdown into this file.

Review task shape:

```json
{
  "id": "AAC-0001",
  "type": "ambiguous_domain_grouping | delete_skipped | wrapper_has_logic | method_conflict | naming_conflict | remaining_usage | test_gap | deferred_cleanup",
  "status": "open",
  "severity": "low | medium | high",
  "affectedClass": "string",
  "affectedPackage": "string",
  "relatedService": "string | null",
  "domain": "string | null",
  "summary": "string",
  "currentBehavior": "string",
  "reasonDeferred": "string",
  "recommendedFollowUp": "string",
  "acceptanceCriteria": ["string"],
  "reviewOwners": ["string"],
  "createdBySkill": "merge-actions-adapters",
  "createdAt": "ISO-8601"
}
```

Write review tasks for:

```text
- ambiguous domain grouping
- source wrapper contains logic
- method conflict during merge
- deletion skipped because user did not allow delete
- deletion skipped because usages remain
- deletion skipped because checks failed
- target wrapper already exists with incompatible methods
- unsafe or unclear foreign-domain ownership
- tests/checks unavailable
```

Review tasks should read like small Jira tickets. Future agents should be able to process each JSONL line as one task without consulting ancient Slack scrolls. A bleak bar, yet here we are.

---

## 18. Verification

After applying changes, run lightweight checks.

Recommended order:

```text
1. git diff --stat
2. git diff --check
3. compile command if known
4. targeted tests for changed services/wrappers if known
5. broader backend check only if practical and already available
```

Useful commands:

```bash
git diff --stat
git diff --check
```

For Gradle projects, inspect available tasks before broad execution:

```bash
./gradlew tasks --all | grep -E "compile|test|check" | head -100
```

Do not install dependencies.
Do not modify Gradle files.
Do not add scripts.
Do not hide failing checks.

If checks fail because of this change, fix only if the fix is mechanical and inside scope.
If the fix requires service logic, repository logic, build changes, or behavior changes, stop and report.

---

## 19. Diff Safety Validation

Validate after editing:

```text
- target consolidated wrappers use <Domain>ActionAdapter or <Domain>ActionUseCase
- old call sites use consolidated wrapper
- wrappers contain only one-call forwarding methods
- no service logic changed except injection/call target replacement
- no repository code changed
- no controller code changed unless explicitly in selected scope and necessary for import/injection updates
- no data classes changed
- no endpoint annotations changed
- no generated files changed
- no unrelated formatting churn
- no old wrapper files deleted unless mode == apply_allow_delete
```

Suspicious diff terms:

```text
if
when
for
while
try
catch
map
filter
sorted
also
let
run
apply
TODO
FIXME
@PreAuthorize
@RequestMapping
@GetMapping
@PostMapping
@PutMapping
@DeleteMapping
@Transactional
build.gradle
settings.gradle
gradle.lockfile
```

Suspicious does not mean forbidden. It means inspect and confirm it is not new logic introduced by consolidation.

---

## 20. Patch Report

Write patch report to:

```text
.cursor/plans/merge-actions-adapters/state/patches/<RUN_ID>.patch.json
```

Shape:

```json
{
  "runId": "string",
  "timestamp": "ISO-8601",
  "auditPath": "string",
  "reviewFile": "review.jsonl",
  "patch": {
    "branch": "string",
    "mode": "audit_only | apply | apply_allow_delete",
    "wrapperType": "adapters | usecases | both",
    "start": "string",
    "filesChanged": ["string"],
    "wrappersCreatedOrUpdated": ["string"],
    "sourceWrappersConsolidated": ["string"],
    "sourceWrappersDeleted": ["string"],
    "sourceWrappersLeftInPlace": ["string"],
    "summary": "string",
    "diffStat": "string"
  },
  "checks": {
    "commandsRun": ["string"],
    "passed": true,
    "skipped": false,
    "notes": ["string"]
  },
  "reviewTasks": {
    "created": 0,
    "ids": ["string"]
  }
}
```

Append completion event to `runs.jsonl`.

---

## 21. Final Response Format

Keep final response short.
Do not dump JSON unless requested.

### 21.1 Audit Only / Not Applied

```text
Audit result: <LOW_RISK | MEDIUM_RISK | HIGH_RISK | BLOCKED>
Applied: no
Wrapper type: <adapters | usecases | both>
Start: <service/domain/wrapper>
Reason: <specific reason>
Audit saved: <path>
Next safe action: <specific next step>
```

### 21.2 Applied Without Delete

```text
Audit result: LOW_RISK
Applied: yes
Delete allowed: no
Wrapper type: <adapters | usecases | both>
Start: <service/domain/wrapper>
Summary: <what changed>
Wrappers created/updated: <list or count>
Old wrappers left in place: <count>
Checks: <passed/skipped/failed with command names>
Audit saved: <path>
Patch report: <path>
Review tasks: <count written to review.jsonl>
```

### 21.3 Applied With Delete

```text
Audit result: LOW_RISK
Applied: yes
Delete allowed: yes
Wrapper type: <adapters | usecases | both>
Start: <service/domain/wrapper>
Summary: <what changed>
Wrappers created/updated: <list or count>
Old wrappers deleted: <count>
Old wrappers left in place: <count and reason>
Checks: <passed/skipped/failed with command names>
Audit saved: <path>
Patch report: <path>
Review tasks: <count written to review.jsonl>
```

---

## 22. Examples

### 22.1 Consolidate Target adapters from ActionService

Input:

```text
wrapper: adapters
start: service:ActionService
mode: apply
```

Possible source wrappers:

```text
TargetServiceActionsAdapter
TargetMetaServiceActionsAdapter
TargetJpaRepositoryActionsAdapter
TargetLogoServiceActionsAdapter
TargetPermissionEvaluatorActionsAdapter
```

Consolidated wrapper:

```text
TargetActionAdapter
```

Expected behavior:

```text
- TargetActionAdapter injects the required Target dependencies
- forwarding methods are copied exactly
- ActionService injects TargetActionAdapter
- ActionService call sites use TargetActionAdapter
- source wrappers are left in place because mode is apply
- cleanup candidates are written to review.jsonl
```

### 22.2 Consolidate and delete obsolete adapters

Input:

```text
wrapper: adapters
start: domain:Target
mode: apply_allow_delete
```

Expected behavior:

```text
- consolidate to TargetActionAdapter
- update all call sites in selected scope
- verify old wrappers have zero usages
- delete only obsolete adapter files that pass delete safety rules
- leave any unsafe wrapper in place and record a review task
```

### 22.3 Audit use cases only

Input:

```text
wrapper: usecases
start: service:ActionService
mode: audit_only
```

Expected behavior:

```text
- find use cases related to ActionService incoming callers
- group by foreign caller domain
- propose <Domain>ActionUseCase consolidation
- write audit JSON
- do not edit files
```

---

## 23. Stop Conditions

Stop immediately if:

```text
- required input is missing
- start service/domain/wrapper cannot be resolved
- wrapper type is unsupported
- domain grouping is ambiguous
- source wrapper contains logic
- consolidation would require behavior change
- consolidation would require method rename outside wrapper class naming
- consolidation would require service logic change
- consolidation would require repository change
- deletion is requested but mode is not apply_allow_delete
- delete candidate still has usages
- target wrapper has incompatible existing methods
- existing local changes conflict with planned edits
```

When stopped, write audit JSON and report the blocker.
Append review tasks when useful.

---

## 24. Maintenance Note

This skill is intentionally separate from `actions-refactor`.

Reason:

```text
actions-refactor:
  moves one controller flow into domains/actions and creates safe thin wrappers.

merge-actions-adapters:
  cleans up wrapper shape after relocation by merging wrappers by foreign domain.
```

Do not merge these workflows back together. Different risks, different gates, different outputs. Humans already made distributed monoliths; no need to make distributed skills too.

Future stable scripts may be extracted for:

```text
- wrapper discovery
- domain grouping
- one-call-forwarding validation
- usage checks before deletion
- JSONL review task validation
```

Team: Super Compliance
Maintainer: James

For workflow changes:

```text
Update the relevant section only.
Do not weaken delete gates.
Do not expand scope without changing the skill version and documenting the new behavior.
```
