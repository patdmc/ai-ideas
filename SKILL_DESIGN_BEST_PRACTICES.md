# Skill Design Best Practices

> Meta-guide for designing skills that make developers more effective

**Audience**: Humans who write skills for AI agents
**Orthogonal to**: SKILL_BEST_PRACTICES.md (patterns for agents executing skills)

## Philosophy

**Skills should optimize for developer effectiveness, not agent efficiency.**

Good skill design:
- ✅ Gives developer control at key decision points
- ✅ Makes agent behavior predictable
- ✅ Surfaces important information clearly
- ✅ Provides recovery paths when things fail
- ✅ Teaches as it executes (developer learns patterns)
- ✅ Enables deterministic, reproducible executions

Bad skill design:
- ❌ Agent runs autonomously without checkpoints
- ❌ Developer doesn't know what's happening or why
- ❌ Failures are mysterious (no recovery guidance)
- ❌ No learning transfer (developer dependent on agent)
- ❌ Non-deterministic behavior (context-dependent)

---

## Design Pattern 1: Clean Context Execution

**Pattern**: Design skills to start with clean, predictable context

**Why**: Context contamination is the #1 cause of non-deterministic agent behavior

### Implementation

```yaml
---
name: your-skill
follows_patterns:
  - clean-context-execution

context:
  mode: hybrid  # Start here for most skills
  boundaries:
    repo_change: true              # Clear on repo switch
    time_gap_seconds: 1800         # Clear after 30 min
    domain_change: true            # Clear on domain switch
    explicit_keywords: ["new task"]

  preserve:
    - auto_memory: true            # Keep pattern rules
    - skill_references: true       # Keep evidence files
    - user_last_n_messages: 3      # Keep recent context

  clear:
    - previous_task_context: true  # Drop prior work
    - intermediate_results: true   # Drop cached builds
    - cached_searches: true        # Drop Glean results
---
```

### When to Use Each Mode

**isolated**: Always clear (pure functions, validators, generators)
```yaml
# Example: validate-skill-patterns
context:
  mode: isolated
```

**continuous**: Never clear (debugging, iterative refinement)
```yaml
# Example: debug-test-failures
context:
  mode: continuous
```

**hybrid**: Smart detection (migrations, development tasks) - **RECOMMENDED DEFAULT**
```yaml
# Example: upgrade-java-to-17
context:
  mode: hybrid
  boundaries:
    repo_change: true
    time_gap_seconds: 1800
```

### Anti-Patterns

❌ **No context management**: Agent inherits messy state
❌ **Always isolated**: Breaks iterative workflows
❌ **Always continuous**: Context bloat, compaction, cross-contamination
❌ **Wrong boundaries**: Too aggressive (disrupts flow) or too loose (contamination)

---

## Design Pattern 2: Progressive Disclosure

**Pattern**: Reveal complexity as needed, not all upfront

**Why**: Reduces cognitive load, prevents information overload, improves scanability

### Good: Layered Information

```markdown
## Step 1: Dependency Resolution

Map HomeAway Dropwizard artifacts to ExpediaGroup equivalents.

> 📖 See references/eg-dw-dependency-mappings.json for verified mappings
> 🔧 See references/bundle-migration-guide.md for bundle-specific patterns
> ⚠️  See references/troubleshooting.md if issues arise

**Quick Check**:
```bash
grep "com.homeaway.dropwizard" pom.xml
```

**Detailed Steps**: [Click to expand if needed]
```

### Bad: Wall of Text

```markdown
## Step 1: Dependency Resolution

Read pom.xml. Check for HomeAway artifacts. Map to ExpediaGroup.
Note: Some bundles add .bundle. namespace but others don't.
Metrics and monitoring don't get .bundle.
Never-migrated bundles stay as com.homeaway.*
These include dropwizard-mybatis, dropwizard-l5dtracing-bundle...
[500 more lines of details]
```

### Implementation Tips

✅ **Layer 1**: One-sentence summary
✅ **Layer 2**: Quick validation command
✅ **Layer 3**: Link to detailed reference
✅ **Layer 4**: Troubleshooting (when needed)

---

## Design Pattern 3: Explicit Checkpoints

**Pattern**: Insert decision points where user control matters

**Why**: Prevents scope drift, gives user control, surfaces state clearly

### Where to Add Checkpoints

1. **After major phases complete**
2. **Before risky operations** (push to CI, deploy, delete)
3. **When multiple valid next steps exist**
4. **When user input would change approach**

### Checkpoint Structure

```markdown
**CHECKPOINT**: [Phase Name] Complete

Status:
- ✅ What succeeded
- ✅ What was validated
- ⚠️  Any warnings/notes

Next options:
1. [Option 1] - [Brief description] (Recommended)
2. [Option 2] - [Brief description]
3. [Option 3] - [Brief description]

Which approach? [1/2/3]

[Agent WAITS for user response - does NOT proceed automatically]
```

### Example: Java 17 Migration

```markdown
**CHECKPOINT**: Dependencies Resolved

Status:
- ✅ 15/15 artifact mappings applied (references/eg-dw-dependency-mappings.json)
- ✅ 8/8 API changes fixed (main + test code)
- ✅ Local compile passes (0 errors)
- ✅ No upper-bound conflicts detected

Next options:
1. Push to CI for validation (Recommended)
   - Will trigger Jenkins build
   - Monitor with gh run list
   - Takes 10-15 minutes

2. Additional local testing first
   - Run full test suite locally
   - Check for integration test issues
   - Adds 5-10 minutes

3. Review changes in detail before pushing
   - Show git diff
   - Explain each change
   - Verify nothing unexpected

Which approach? [1/2/3]
```

### Anti-Patterns

❌ **No checkpoints**: Agent continues indefinitely
❌ **Too many checkpoints**: Disrupts flow (<5 min between)
❌ **Vague checkpoints**: "Ready to continue?" (what completed? what's next?)
❌ **No status summary**: User doesn't know current state

---

## Design Pattern 4: Constrained Autonomy

**Pattern**: Give agent freedom within bounds, checkpoints at boundaries

**Why**: Balances efficiency (agent works independently) with safety (user controls risky operations)

### Autonomy Levels

```markdown
## Phase 1: Local Analysis (HIGH AUTONOMY)
Agents can:
- ✅ Read any files
- ✅ Try different approaches
- ✅ Iterate on fixes
- ✅ Run local builds

Agent works independently - no checkpoints needed

---

**CHECKPOINT**: Ready to push changes to CI?
[This crosses into LOW AUTONOMY zone]

---

## Phase 2: Remote Operations (LOW AUTONOMY)
Agent must:
- ⚠️  Confirm before pushing
- ⚠️  Show git diff before commit
- ⚠️  Wait for CI, don't assume success
- ⚠️  Ask before force-push or destructive ops

Checkpoints required at each step
```

### Risk-Based Boundaries

**Low Risk (High Autonomy)**:
- Reading files
- Local compiles
- Analyzing logs
- Searching documentation

**Medium Risk (Moderate Autonomy)**:
- Editing code
- Running tests
- Local git commits
- Creating branches

**High Risk (Low Autonomy)**:
- Pushing to remote
- Deleting files
- Force operations
- Production changes

### Implementation

```yaml
---
autonomy_zones:
  local_work:
    level: high
    checkpoints: optional
    confirmation: none

  remote_operations:
    level: low
    checkpoints: required
    confirmation: explicit  # User must say "yes"

  destructive_operations:
    level: minimal
    checkpoints: always
    confirmation: double_check  # Ask twice
---
```

---

## Design Pattern 5: Decision Amplification

**Pattern**: Surface high-leverage decisions to user with clear options

**Why**: Agent shouldn't make architectural decisions without user input

### When to Amplify

- Multiple valid approaches exist
- Trade-offs are significant
- User preferences matter
- Long-term impact (hard to reverse)

### Structure

```markdown
**DECISION POINT**: [What needs deciding]

Context: [Why this matters, what's at stake]

Option A: [Approach name]
✅ Pros: [2-3 key benefits]
❌ Cons: [2-3 key drawbacks]
⏱️  Impact: [Time/effort estimate]

Option B: [Approach name]
✅ Pros: [2-3 key benefits]
❌ Cons: [2-3 key drawbacks]
⏱️  Impact: [Time/effort estimate]

Recommended: [Option X] for [reason]
Your choice: [A/B/other]
```

### Example: Migration Strategy

```markdown
**DECISION POINT**: Migration Approach

Context: Found HomeAway Dropwizard 3.x app. Two paths to Java 17.

Option A: Two-Step Migration (Namespace → Java 17)
✅ Pros:
  - Smaller changes per PR (easier review)
  - Easier debugging (isolate namespace vs JDK issues)
  - Lower risk per step

❌ Cons:
  - Two PRs (2x review overhead)
  - Longer total timeline (5-7 days)
  - Two CI validation cycles

⏱️  Impact: 2 PRs, 5-7 days total

Option B: Combined Migration (Namespace + Java 17)
✅ Pros:
  - One PR (less overhead)
  - Faster (3-4 days total)
  - Single validation cycle

❌ Cons:
  - Large changeset (harder review)
  - Harder debugging (mixed failures)
  - Higher risk if rollback needed

⏱️  Impact: 1 PR, 3-4 days total

Recommended: Option A for first-time migrations (clearer debugging)
Your choice: [A/B]
```

### Anti-Patterns

❌ **Agent decides architecture**: "I chose option B because..."
❌ **No trade-off analysis**: Just lists options
❌ **Buried in text**: Decision point not visible
❌ **No recommendation**: User forced to research

---

## Design Pattern 6: Transparent State

**Pattern**: Make current state visible so user knows where they are

**Why**: Reduces confusion, enables recovery, improves trust

### Progress Indicators

```markdown
## Migration Progress

[█████████░░░░░░░░░░] 45% complete

Completed:
✅ Step 0: Prerequisites validated
✅ Step 1: Dependencies resolved (15/15)
✅ Step 2: API changes applied (8/8)
✅ Step 3: Local build passed

In Progress:
🔄 Step 4: CI validation
   - Jenkins build #42 running
   - Estimated 8 minutes remaining
   - Polling every 30 seconds

Remaining:
⏳ Step 5: Create PR
⏳ Step 6: Request review

Current: Polling Jenkins (attempt 3/30)
Next: Create PR after CI passes
```

### State Persistence

For multi-day workflows:

```markdown
## Session State

**Session ID**: java17-api-availability-20260228
**Started**: 2026-02-28 09:00
**Last Checkpoint**: Step 3 (local build passed)
**Elapsed**: 2.3 hours
**State File**: .skill-state/java-17-migration/state.yaml

To resume: `/resume-migration --session java17-api-availability-20260228`
```

### State Visualization

```yaml
# .skill-state/java-17-migration/state.yaml
session_id: java17-api-availability-20260228
skill: upgrade-java-to-17
repo: api-availability-service
started: 2026-02-28T09:00:00Z
last_updated: 2026-02-28T11:18:00Z

progress:
  current_step: 4
  total_steps: 6
  percentage: 67

completed_steps:
  - step: 0
    name: Prerequisites
    completed_at: 2026-02-28T09:15:00Z
  - step: 1
    name: Dependencies
    completed_at: 2026-02-28T10:30:00Z
    artifacts: 15
  - step: 2
    name: API Changes
    completed_at: 2026-02-28T10:45:00Z
    changes: 8
  - step: 3
    name: Local Build
    completed_at: 2026-02-28T11:00:00Z

current_step:
  step: 4
  name: CI Validation
  started_at: 2026-02-28T11:05:00Z
  jenkins_build: 42
  status: running
```

---

## Design Pattern 7: Explicit Expectations

**Pattern**: Tell user what agent will and won't do

**Why**: Sets realistic expectations, reduces frustration, builds trust

### Structure

```markdown
## What This Skill Does
✅ Updates pom.xml dependencies automatically
✅ Applies known API changes from reference file
✅ Runs local builds and captures errors
✅ Monitors CI and reports status
✅ Creates PR with migration summary

## What This Skill Won't Do
❌ Doesn't fix custom code errors (you'll need to debug)
❌ Doesn't handle multi-module repos yet (single module only)
❌ Doesn't modify CI config (GitHub Actions manual update)
❌ Doesn't deploy to production (just creates PR)
❌ Doesn't merge PRs (requires human review)

## When to Use Human Intervention
⚠️  Custom business logic errors (agent will flag, you fix)
⚠️  Complex multi-module dependencies (agent will detect, you resolve)
⚠️  Infrastructure issues (Kubernetes, Artifactory)
```

### Scope Boundaries

```markdown
## Scope

**In Scope**:
- Java 11 → 17 source compatibility
- EG Dropwizard 4.x parent POM updates
- Standard dependency migrations
- Known API breaking changes

**Out of Scope**:
- Java 8 → 17 (requires intermediate upgrade to 11 first)
- EG Dropwizard 5.x (Jakarta, requires /migrate-to-eg-dropwizard-2)
- Custom plugin migrations (handle manually)
- Application-specific refactoring

**Delegation**:
- HomeAway DW namespace → Use /migrate-to-eg-dropwizard first
- Core-API v1 → v2 migration → Use /migrate-core-api first
```

---

## Design Pattern 8: Failure Recovery Paths

**Pattern**: Document what to do when things go wrong

**Why**: Failures are common, recovery guidance saves time

### Structure

```markdown
## Common Issues & Recovery

### Issue: Upper-Bound Dependency Conflicts

**Symptom**:
```
[ERROR] Rule requireUpperBoundDeps failed:
  Require upper bound dependencies error for:
    com.fasterxml.jackson.core:jackson-databind
```

**Root Cause**: Multiple versions of jackson-databind in dependency tree

**Recovery**:
1. Run locally: `mvn enforcer:enforce -Drules=requireUpperBoundDeps`
2. Capture ALL conflicts: `mvn enforcer:enforce 2>&1 | grep "Require upper"`
3. Fix all in dependencyManagement (pom.xml):
   ```xml
   <dependencyManagement>
     <dependency>
       <groupId>com.fasterxml.jackson.core</groupId>
       <artifactId>jackson-databind</artifactId>
       <version>2.13.5</version>
     </dependency>
   </dependencyManagement>
   ```
4. Verify: `mvn clean install -DskipTests`

**Prevention**: Run enforcer check BEFORE pushing (proactive-validation pattern)

**Evidence**: See references/eg-dw-dependency-mappings.json for common fixes
```

### Recovery Decision Tree

```markdown
## If CI Fails

1. **Check build logs**:
   ```bash
   gh run view <run-id> --log-failed
   ```

2. **Categorize error**:
   - Compilation error → See "Compilation Failures" below
   - Test failure → See "Test Failures" below
   - Upper-bound conflict → See "Dependency Conflicts" below
   - Infrastructure → Contact DevOps

3. **Apply fix**:
   - Use reference file patterns when available
   - Run locally to verify
   - Push fix

4. **If stuck**:
   - Search Glean for similar errors
   - Ask in #mpaas-java-migrations
   - Escalate to skill maintainer
```

---

## Design Pattern 9: Teaching Through Execution

**Pattern**: Help developer learn patterns, don't just do the work

**Why**: Builds competence, reduces dependency, enables knowledge transfer

### Explain as You Go

```markdown
## Step 1: Dependency Resolution

**What we're doing**: Mapping HomeAway artifacts to ExpediaGroup equivalents

**Why this matters**: EG Dropwizard uses different groupIds and some have
namespace changes (.bundle. added to some packages but not others)

**How to verify**:
```bash
# Before:
grep "com.homeaway.dropwizard" pom.xml

# After (should find com.expediagroup):
grep "com.expediagroup.dropwizard" pom.xml
```

**What you're learning**: Dependency migration patterns that apply to
any HA → EG migration, not just Dropwizard
```

### Teach Validation

```markdown
## Success Criteria

You'll know this step succeeded when:

1. **Local build passes**:
   ```bash
   mvn clean install
   [INFO] BUILD SUCCESS
   ```

2. **No upper-bound conflicts**:
   ```bash
   mvn enforcer:enforce -Drules=requireUpperBoundDeps
   No errors
   ```

3. **Tests pass**:
   ```bash
   mvn test
   Tests run: 42, Failures: 0, Errors: 0
   ```

**Pro tip**: Run these locally BEFORE pushing to catch issues early
(saves 10-15 min per CI cycle)
```

### Document Patterns

```markdown
## Pattern You Just Applied

**Pattern Name**: Proactive Error Capture

**What you did**:
- Compiled once to capture ALL errors (not one at a time)
- Categorized by type (dependencies, API changes, tests)
- Fixed all in parallel (one pass through code)
- Verified with single final compile

**Why it works**:
- 20-30 compile cycles → 1 compile cycle (95% reduction)
- Reveals related issues with shared root causes
- Saves 1-2 hours of iterative debugging

**When to use**: Any time you're fixing compilation errors
(Java, TypeScript, Swift, Kotlin - universal pattern)

**Learn more**: See SKILL_BEST_PRACTICES.md#1-proactive-error-capture
```

---

## Design Pattern 10: Evidence-Based Design

**Pattern**: Design skills based on real execution data, not assumptions

**Why**: Empirical validation, continuous improvement, institutional knowledge

### Track Evidence

```yaml
evidence_base:
  repos_analyzed: 139
  success_rate: 94.2%
  last_executed: 2026-02-27

  recent_execution:
    repo: cl-partner-notification-publisher
    status: success
    duration_seconds: 8280  # 2.3 hours
    issues_encountered: 6
    fixes_applied:
      - upper-bound-dependencies-14-fixes
      - maven-363-build-agent-workaround
      - github-actions-java-17-workflow-fix

  common_issues:
    - issue: upper-bound-conflicts
      frequency: 78%  # 108/139 repos
      fix: references/eg-dw-dependency-mappings.json

    - issue: maven-363-build-agent
      frequency: 23%  # 32/139 repos
      fix: memory/java17-build-agents.md
```

### Update from Experience

```markdown
## Version History

### v6.2 (2026-02-27)
- Added proactive Jenkins agent validation (Step 5d-3)
- Evidence: 32/139 repos hit Maven 3.6.3 issue
- Impact: Prevents 15-20 min of CI debugging

### v5.0 (2026-02-15)
- Added evidence-based-references pattern
- Created references/eg-dw-dependency-mappings.json
- Evidence: Reduced dependency resolution time 90% (5-10 hrs → 30 min)

### v3.0 (2026-02-01)
- Added proactive-error-capture pattern
- Evidence: Reduced compile cycles 95% (20-30 → 1-2)
```

### Capture What Works

```markdown
## Known Fixes

### Fix: AliveBundle API Change
**Issue**: `new AliveBundle<>(getClass(), config::getAlive)` fails compilation
**Fix**: Remove first argument: `new AliveBundle<>(config::getAlive)`
**Evidence**: Validated in 15+ repos (auth-service, user-service, booking-service)
**Reference**: references/eg-dw-dependency-mappings.json#apiChanges

### Fix: Upper-Bound jackson-databind
**Issue**: Require upper bound dependency error
**Fix**: Add `2.13.5` to dependencyManagement
**Evidence**: Required in 78% of migrations (108/139 repos)
**Reference**: references/eg-dw-dependency-mappings.json#upperBoundFixes
```

---

## Anti-Patterns to Avoid

### 1. Kitchen Sink Skill

**Problem**: Tries to handle every edge case, becomes unmaintainable

**Symptom**:
- 2000+ line skill file
- "If repo has X, then Y, but if Z, then Q, unless..."
- Endless special cases

**Fix**:
- Extract sub-skills for variants
- Use delegation: "If X, use /other-skill first"
- Focus on 80% case, document 20% exceptions

### 2. Assumption-Driven Design

**Problem**: Assumes structure without verification

**Symptom**:
- "pom.xml is always in root"
- "Tests are in src/test"
- "CI uses Jenkins" (might be GitHub Actions)

**Fix**:
- Add code-first-analysis: verify by reading
- Step 0: Detect actual structure
- Document variants, route accordingly

### 3. No User Control

**Problem**: Agent runs autonomously, no checkpoints

**Symptom**:
- Agent does 2 hours of work user didn't want
- Scope creep (keeps going beyond intent)
- User can't redirect without interrupting

**Fix**:
- Add checkpoints at phase boundaries
- Give options at decision points
- Wait for confirmation before risky operations

### 4. Mysterious Failures

**Problem**: When it fails, user doesn't know why or how to recover

**Symptom**:
- "Build failed" (no details)
- No recovery guidance
- User stuck, can't continue

**Fix**:
- Add "Common Issues & Recovery" section
- Document symptoms, causes, fixes
- Provide decision tree for categorizing errors

### 5. Not Deterministic

**Problem**: Same inputs, different outputs (context-dependent)

**Symptom**:
- Works for developer A, fails for developer B
- Works on Monday, fails on Tuesday
- "It worked yesterday" (context changed)

**Fix**:
- Add clean-context-execution pattern
- Use isolated or hybrid mode
- Make behavior reproducible

---

## Skill Design Checklist

Use this when creating or updating skills:

### Structure
- [ ] Clear "When to Use" section
- [ ] Step 0: Prerequisites / Triage
- [ ] Numbered steps with validation commands
- [ ] Success criteria checklist
- [ ] Common issues & recovery section

### Context Management
- [ ] Declares context mode (isolated/continuous/hybrid)
- [ ] Appropriate boundaries for hybrid mode
- [ ] Preserves necessary context (auto_memory, references)
- [ ] Clears contaminating context (previous task, cache)

### User Experience
- [ ] Checkpoints at major phases
- [ ] Decision amplification for high-leverage choices
- [ ] Transparent state (progress visible)
- [ ] Explicit expectations (what it does/doesn't do)
- [ ] Recovery paths for failures

### Quality
- [ ] Evidence-based references (not repeated searches)
- [ ] Tracks execution metrics (success rate, duration)
- [ ] Documents common issues with fixes
- [ ] Updates from real execution data
- [ ] Teaches patterns (not just executes them)

### Patterns
- [ ] Declares which patterns it follows
- [ ] Implements patterns correctly
- [ ] Passes validation (/validate-skill-patterns)
- [ ] Documents agent errors (if known)
- [ ] Includes examples from real executions

---

## Measuring Design Quality

Good skills have:

**High Success Rate**: >90% of executions succeed
**Fast Execution**: Time improves with pattern adoption
**Low Intervention**: <3 user corrections per execution
**Reproducible**: Same repo → same behavior
**Teachable**: Developers learn patterns, become independent

**Evidence**:
```yaml
skill_quality:
  success_rate: 94.2%
  avg_duration_seconds: 8280  # 2.3 hours
  interventions_per_execution: 1.2
  determinism_score: 0.94
  learning_transfer: 78%  # Developers can replicate without agent
```

---

## Examples of Well-Designed Skills

### Excellent: upgrade-java-to-17
- ✅ Clean context execution (hybrid mode)
- ✅ Strong evidence base (139 repos, 94.2% success)
- ✅ Comprehensive reference files (dependency mappings)
- ✅ Clear checkpoints (4 decision points)
- ✅ Recovery guidance (common issues documented)
- ✅ Teaching orientation (explains patterns)

### Excellent: migrate-to-eg-dropwizard
- ✅ Detailed triage (3 namespace patterns)
- ✅ Per-bundle guidance (not generic)
- ✅ Evidence-based (137+ migrations)
- ✅ Domain-specific patterns (14 patterns applied)
- ✅ Never-migrated catalog (exceptions documented)

### Good: shadow-test-pubsub
- ✅ Clean phase boundaries (1, 2, 3, 4)
- ✅ Multiple tools orchestrated
- ✅ Order-independent comparison (auto-sort)
- ⚠️  Could add more checkpoints
- ⚠️  Could document common failures better

---

## Conclusion

**Skills are interfaces between humans and AI agents.**

Good skill design:
- Respects that humans make decisions
- Makes agent behavior predictable
- Teaches patterns as it executes
- Provides recovery when things fail
- Improves from evidence

**The best skills**:
- Have clean context boundaries (deterministic)
- Surface decisions at the right moments
- Track evidence and iterate
- Make developers more effective, not just faster

**Remember**: You're not writing instructions for agents. You're designing *tools that make developers more effective at working with agents*.

That's a higher bar.

---

**Related Documents**:
- SKILL_BEST_PRACTICES.md - Patterns for agents executing skills
- GETTING_STARTED_WITH_BEST_PRACTICES.md - Quick start guide

**Questions?** Review existing well-designed skills as examples, or ask in #skill-builder.
