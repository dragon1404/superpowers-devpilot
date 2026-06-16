---
name: dev-resume
description: "Resume a DevPilot workflow from the last completed stage. Running this command while in an approval-wait state counts as approving that stage. Usage: /dev-resume [workItemId]"
---

# DevPilot: Resume Workflow

**Announce:** "Resuming DevPilot workflow for work item {workItemId}."

## Step 1 — Extract Work Item ID

Extract the workItemId from the user's message (the integer following `/dev-resume`).

If no workItemId is provided:

1. Run:
   ```bash
   ls .devpilot/state/ 2>/dev/null
   ```
2. Collect all filenames matching `<integer>.json`. Strip the `.json` suffix to get candidate IDs.
3. If exactly one candidate exists → use it as `workItemId` and announce: "No work item ID provided — resuming work item {workItemId} from state file."
4. If multiple candidates exist → list them and ask the developer which one to resume:
   > "Multiple DevPilot workflows found. Which work item do you want to resume?
   > {numbered list of IDs}
   > Reply with the number or work item ID."
   Wait for the response. Use the chosen ID as `workItemId`.
5. If no candidates exist → stop and say: "Please provide a work item ID. Usage: `/dev-resume <workItemId>`  
   Or start a new workflow with `/dev-workitem <workItemId>`."

## Step 2 — Read State File

Read `.devpilot/state/{workItemId}.json`.

If the file does not exist, stop and say: "No DevPilot workflow found for work item {workItemId}. Start one with `/dev-workitem {workItemId}`."

Store all fields from the state file for use in subsequent steps.

## Step 2b — Surface Worktree Path

If `worktreePath` is not null, tell the developer:

> This workflow has a git worktree at: `{worktreePath}`
> Ensure you are running from that directory before continuing.

If `worktreePath` is null, continue without comment.

## Step 2c — Detect Gitignore Settings

Run:
```bash
git check-ignore -q docs/ 2>/dev/null && echo "IGNORED" || echo "TRACKED"
git check-ignore -q .devpilot/ 2>/dev/null && echo "IGNORED" || echo "TRACKED"
```

Store:
- `docsGitignored` = true if `docs/` is IGNORED, false otherwise
- `stateGitignored` = true if `.devpilot/` is IGNORED, false otherwise

**Global commit rule (applies for the rest of this skill):**
- If `docsGitignored` is true — skip all `git add docs/...` commands and their associated commits
- If `stateGitignored` is true — skip all `git add .devpilot/...` commands and their associated commits

## Step 3 — Branch on Status

Read the `status` field and jump to the matching section below.

---

## [STATUS: WAITING_FOR_DESIGN_CLARIFICATION]

The developer has updated the work item description with clarification answers. Re-read the work item and proceed to design.

**Ca.** Call `mcp__azure-devops__wit_get_work_item` with id: {workItemId}. Read the updated title, description, acceptance criteria, and repro steps (`Microsoft.VSTS.TCM.ReproSteps`).

**Cb.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Clarifications received — resuming Design stage
  ```

If the tool call fails or errors, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post clarification comment — {error}. Continuing."

**Cc.** Update `.devpilot/state/{workItemId}.json`:
- Set `status` to `"DESIGNING"`
- Set `lastUpdated` to current ISO 8601 timestamp

**Cd.** Commit:
```bash
git add .devpilot/state/{workItemId}.json
git commit -m "chore: devpilot state — design clarifications received for {workItemId}"
```

**Ce.** Continue to **[STAGE 2: Design]** below.

---

## [STATUS: DESIGNING]

Design was in progress when the session ended. Post ADO comment:
```
[DevPilot] Resuming Design stage
```

Re-fetch the work item (`mcp__azure-devops__wit_get_work_item` with id: {workItemId}) to get the latest title, description, acceptance criteria, and repro steps (`Microsoft.VSTS.TCM.ReproSteps`).

Then continue to **[STAGE 2: Design]** below.

---

## [STAGE 2: Design]

**2a.** Invoke `superpowers:brainstorming` using the Skill tool, passing:

> **Work Item {workItemId}: {title}**
>
> **Description:**
> {description}
>
> **Acceptance Criteria:**
> {acceptanceCriteria}
>
> {if reproSteps is not empty: "**Repro Steps:**\n> {reproSteps}\n>"}
>
> **Instructions for brainstorming:** Treat this work item as the feature requirement. The design document must include: Work Item ID and title, assumptions, impacted modules, database impact, API impact, and testing impact. Save the design to `docs/design/{workItemId}-design.md`.
>
> **Before asking any clarifying question in this session:** First post it as a comment to work item {workItemId} using `mcp__azure-devops__wit_add_work_item_comment` with the comment:
> ```
> [DevPilot] Design Question
>
> {your question}
> ```
> Then ask the same question in the session. After the developer answers, post their answer as another comment:
> ```
> [DevPilot] Design Answer
>
> {developer's answer}
> ```
> This ensures the full Q&A is preserved in the work item if the session is lost.

**2b.** After brainstorming completes, verify that `docs/design/{workItemId}-design.md` exists.

**2b2.** Review `docs/design/{workItemId}-design.md` for any open questions or unresolved assumptions that require a developer decision before the design can be finalised.

If you identify any questions:

1. Format them as a numbered list. For each question, include your own suggested answer.
2. Commit the draft design:
   ```bash
   git add docs/design/{workItemId}-design.md
   git commit -m "docs: add draft design document for work item {workItemId}"
   ```
3. Call `mcp__azure-devops__wit_add_work_item_comment` with:
   - workItemId: {workItemId}
   - comment:
     ```
     [DevPilot] Design Clarifications Needed

     The following questions need answers before the design can be finalised. Suggested answers are provided — update the work item description with your decisions, then run `/dev-resume {workItemId}`.

     {numbered list of questions with suggested answers}
     ```
4. Update `.devpilot/state/{workItemId}.json`:
   - Set `status` to `"WAITING_FOR_DESIGN_CLARIFICATION"`
   - Set `lastUpdated` to current ISO 8601 timestamp
5. Commit:
   ```bash
   git add .devpilot/state/{workItemId}.json
   git commit -m "chore: devpilot state — waiting for design clarification on {workItemId}"
   ```
6. Tell the developer:
   > Design clarifications needed for work item {workItemId}.
   >
   > Questions have been posted as a comment on the ADO work item. Update the work item description with your decisions, then run `/dev-resume {workItemId}` to continue.

   **STOP.**

If you have no open questions, continue to **2c**.

**2c.** Commit:
```bash
git add docs/design/{workItemId}-design.md
git commit -m "docs: add design document for work item {workItemId}"
```

**2c2.** Upload design document to ADO: read the full content of `docs/design/{workItemId}-design.md` and call `mcp__azure-devops__wit_add_work_item_comment` with:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Design Document

  {full content of docs/design/{workItemId}-design.md}
  ```
If the tool call fails, print a warning and continue: "⚠️ [DevPilot] Warning: Could not upload design document — {error}."

**2d.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Stage Completed: Design
  Document: docs/design/{workItemId}-design.md
  ```

If the tool call fails or errors, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post 'Design Completed' comment — {error}. Continuing."

**2e.** Update `.devpilot/state/{workItemId}.json`:
- Set `designCompleted` to `true`
- Set `status` to `"WAITING_FOR_DESIGN_APPROVAL"`
- Set `lastUpdated` to current ISO 8601 timestamp

**2f.** Commit:
```bash
git add .devpilot/state/{workItemId}.json
git commit -m "chore: devpilot state — waiting for design approval on {workItemId}"
```

**2g.** Tell the developer:

> Design complete for work item {workItemId}.
>
> Review the design document at: `docs/design/{workItemId}-design.md`
>
> When you are satisfied with the design, run `/dev-resume {workItemId}` to approve it and continue to the Implementation Plan stage.

**STOP — wait for developer to run `/dev-resume` again.**

---

## [STATUS: WAITING_FOR_DESIGN_APPROVAL]

Running `/dev-resume` here means the developer has reviewed and approved the design document.

**3a.** Update `.devpilot/state/{workItemId}.json`:
- Set `designApproved` to `true`
- Set `status` to `"PLANNING"`
- Set `lastUpdated` to current ISO 8601 timestamp

**3b.** Commit:
```bash
git add .devpilot/state/{workItemId}.json
git commit -m "chore: devpilot state — design approved for {workItemId}"
```

**3c.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Design Approved — Starting Implementation Plan
  ```

If the tool call fails or errors, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post 'Design Approved' comment — {error}. Continuing."

**3d.** Continue to **[STAGE 3: Implementation Plan]** below.

---

## [STAGE 3: Implementation Plan]

**3e.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Stage Started: Implementation Plan
  ```

If the tool call fails or errors, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post 'Implementation Plan Started' comment — {error}. Continuing."

**3f.** Read the design document from `docs/design/{workItemId}-design.md`.

**3g.** Invoke `superpowers:writing-plans` using the Skill tool, passing the design document content as the `args` parameter. The plan must be saved to `docs/plan/{workItemId}-plan.md`.

**3g2.** Review `docs/plan/{workItemId}-plan.md` for any open questions or unresolved assumptions that require a developer decision before implementation can begin.

If you identify any questions:

1. Format them as a numbered list. For each question, include your own suggested answer.
2. Commit the draft plan:
   ```bash
   git add docs/plan/{workItemId}-plan.md
   git commit -m "docs: add draft implementation plan for work item {workItemId}"
   ```
3. Call `mcp__azure-devops__wit_add_work_item_comment` with:
   - workItemId: {workItemId}
   - comment:
     ```
     [DevPilot] Plan Clarifications Needed

     The following questions need answers before the plan can be finalised. Suggested answers are provided — update the work item description with your decisions, then run `/dev-resume {workItemId}`.

     {numbered list of questions with suggested answers}
     ```
4. Update `.devpilot/state/{workItemId}.json`:
   - Set `status` to `"WAITING_FOR_PLAN_CLARIFICATION"`
   - Set `lastUpdated` to current ISO 8601 timestamp
5. Commit:
   ```bash
   git add .devpilot/state/{workItemId}.json
   git commit -m "chore: devpilot state — waiting for plan clarification on {workItemId}"
   ```
6. Tell the developer:
   > Plan clarifications needed for work item {workItemId}.
   >
   > Questions have been posted as a comment on the ADO work item. Update the work item description with your decisions, then run `/dev-resume {workItemId}` to continue.

   **STOP.**

If you have no open questions, continue to **3h**.

**3h.** Commit:
```bash
git add docs/plan/{workItemId}-plan.md
git commit -m "docs: add implementation plan for work item {workItemId}"
```

**3h2.** Upload implementation plan to ADO: read the full content of `docs/plan/{workItemId}-plan.md` and call `mcp__azure-devops__wit_add_work_item_comment` with:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Implementation Plan

  {full content of docs/plan/{workItemId}-plan.md}
  ```
If the tool call fails, print a warning and continue: "⚠️ [DevPilot] Warning: Could not upload implementation plan — {error}."

**3i.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Stage Completed: Implementation Plan
  Document: docs/plan/{workItemId}-plan.md
  ```

If the tool call fails or errors, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post 'Implementation Plan Completed' comment — {error}. Continuing."

**3j.** Update `.devpilot/state/{workItemId}.json`:
- Set `planCompleted` to `true`
- Set `status` to `"WAITING_FOR_PLAN_APPROVAL"`
- Set `lastUpdated` to current ISO 8601 timestamp

**3k.** Commit:
```bash
git add .devpilot/state/{workItemId}.json
git commit -m "chore: devpilot state — waiting for plan approval on {workItemId}"
```

**3l.** Tell the developer:

> Implementation plan complete for work item {workItemId}.
>
> Review the plan at: `docs/plan/{workItemId}-plan.md`
>
> When you are satisfied with the plan, run `/dev-resume {workItemId}` to approve it and begin implementation.

**STOP — wait for developer to run `/dev-resume` again.**

---

## [STATUS: WAITING_FOR_PLAN_CLARIFICATION]

The developer has updated the work item description with plan clarification answers. Re-read the work item and proceed to planning.

**Pa.** Call `mcp__azure-devops__wit_get_work_item` with id: {workItemId}. Read the updated title, description, acceptance criteria, and repro steps (`Microsoft.VSTS.TCM.ReproSteps`).

**Pb.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Clarifications received — resuming Implementation Plan stage
  ```

If the tool call fails or errors, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post clarification comment — {error}. Continuing."

**Pc.** Update `.devpilot/state/{workItemId}.json`:
- Set `status` to `"PLANNING"`
- Set `lastUpdated` to current ISO 8601 timestamp

**Pd.** Commit:
```bash
git add .devpilot/state/{workItemId}.json
git commit -m "chore: devpilot state — plan clarifications received for {workItemId}"
```

**Pe.** Continue to **[STAGE 3: Implementation Plan]** step 3e above.

---

## [STATUS: PLANNING]

A crash occurred after design was approved but before the implementation plan was completed. Resume by continuing to **[STAGE 3: Implementation Plan]** step 3e above.

Post ADO comment first:
```
[DevPilot] Resuming Implementation Plan generation
```

---

## [STATUS: WAITING_FOR_PLAN_APPROVAL]

Running `/dev-resume` here means the developer has reviewed and approved the implementation plan.

**4a.** Update `.devpilot/state/{workItemId}.json`:
- Set `planApproved` to `true`
- Set `status` to `"IMPLEMENTING"`
- Set `lastUpdated` to current ISO 8601 timestamp

**4b.** Commit:
```bash
git add .devpilot/state/{workItemId}.json
git commit -m "chore: devpilot state — plan approved for {workItemId}"
```

**4c.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Plan Approved — Starting Implementation
  ```

If the tool call fails or errors, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post 'Plan Approved' comment — {error}. Continuing."

**4d.** Continue to **[STAGE 4: Implementation]** below.

---

## [STAGE 4: Implementation]

**4e.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Stage Started: Implementation
  ```

If the tool call fails or errors, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post 'Implementation Started' comment — {error}. Continuing."

**4f.** Read the plan from `docs/plan/{workItemId}-plan.md`.

**4g.** Invoke `superpowers:subagent-driven-development` using the Skill tool, passing the plan content as the `args` parameter.

**4h.** After implementation completes:

Call `mcp__azure-devops__wit_add_work_item_comment`:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Stage Completed: Implementation
  ```

If the tool call fails or errors, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post 'Implementation Completed' comment — {error}. Continuing."

Update `.devpilot/state/{workItemId}.json`:
- Set `implementationCompleted` to `true`
- Set `status` to `"REVIEWING"`
- Set `lastUpdated` to current ISO 8601 timestamp

Commit:
```bash
git add .devpilot/state/{workItemId}.json
git commit -m "chore: devpilot state — implementation complete for {workItemId}"
```

**4i.** Continue immediately to **[STAGE 5: Code Review]** below.

---

## [STAGE 5: Code Review]

**5a.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Stage Started: Code Review
  ```

If the tool call fails or errors, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post 'Code Review Started' comment — {error}. Continuing."

**5b.** Invoke `superpowers:requesting-code-review` using the Skill tool for all files modified during implementation.

**5c.** Save the review findings to `docs/review/{workItemId}-review.md`. The document must include:
- Summary of files reviewed
- Architecture compliance findings
- Coding standards findings
- Security concerns
- Performance concerns
- Overall assessment

**5d.** Commit:
```bash
git add docs/review/{workItemId}-review.md
git commit -m "docs: add code review report for work item {workItemId}"
```

**5d2.** Upload code review to ADO: read the full content of `docs/review/{workItemId}-review.md` and call `mcp__azure-devops__wit_add_work_item_comment` with:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Code Review

  {full content of docs/review/{workItemId}-review.md}
  ```
If the tool call fails, print a warning and continue: "⚠️ [DevPilot] Warning: Could not upload code review — {error}."

**5e.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Stage Completed: Code Review
  Document: docs/review/{workItemId}-review.md
  ```

If the tool call fails or errors, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post 'Code Review Completed' comment — {error}. Continuing."

**5f.** Update `.devpilot/state/{workItemId}.json`:
- Set `reviewCompleted` to `true`
- Set `status` to `"TESTING"`
- Set `lastUpdated` to current ISO 8601 timestamp

Commit:
```bash
git add .devpilot/state/{workItemId}.json
git commit -m "chore: devpilot state — review complete for {workItemId}"
```

**5g.** Continue immediately to **[STAGE 6: Testing]** below.

---

## [STAGE 6: Testing]

**6a.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Stage Started: Testing
  ```

If the tool call fails or errors, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post 'Testing Started' comment — {error}. Continuing."

**6b.** Invoke `superpowers:test-driven-development` using the Skill tool to run the full test suite and ensure all tests pass. If tests are missing for acceptance criteria, write them first using TDD, then verify all pass.

**6c.** Write a testing report to `docs/testing/{workItemId}-testing.md`. The document must include:
- Test execution results (pass/fail counts)
- Coverage summary
- Acceptance criteria coverage mapping (each criterion: covered / not covered)
- Risks identified

**6d.** Commit:
```bash
git add docs/testing/{workItemId}-testing.md
git commit -m "docs: add testing report for work item {workItemId}"
```

**6d2.** Upload testing report to ADO: read the full content of `docs/testing/{workItemId}-testing.md` and call `mcp__azure-devops__wit_add_work_item_comment` with:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Testing Report

  {full content of docs/testing/{workItemId}-testing.md}
  ```
If the tool call fails, print a warning and continue: "⚠️ [DevPilot] Warning: Could not upload testing report — {error}."

**6e.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Stage Completed: Testing
  Document: docs/testing/{workItemId}-testing.md
  ```

If the tool call fails or errors, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post 'Testing Completed' comment — {error}. Continuing."

**6f.** Update `.devpilot/state/{workItemId}.json`:
- Set `testingCompleted` to `true`
- Set `status` to `"CREATING_PR"`
- Set `lastUpdated` to current ISO 8601 timestamp

Commit:
```bash
git add .devpilot/state/{workItemId}.json
git commit -m "chore: devpilot state — testing complete for {workItemId}"
```

**6g.** Continue immediately to **[STAGE 7: Pull Request]** below.

---

## [STAGE 7: Pull Request]

**7a.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Stage Started: Pull Request
  ```

If the tool call fails or errors, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post 'Pull Request Started' comment — {error}. Continuing."

**7b.** Invoke `superpowers:finishing-a-development-branch` using the Skill tool.

The PR must include:
- Title: `[{workItemId}] {work item title}`
- Work Item ID and link
- Design summary (from `docs/design/{workItemId}-design.md`)
- Implementation summary
- Testing summary (from `docs/testing/{workItemId}-testing.md`)
- Risks identified in code review and testing

**7c.** After the PR is created, record the PR URL.

**7d.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- workItemId: {workItemId}
- comment:
  ```
  [DevPilot] Stage Completed: Pull Request
  PR: {prUrl}
  ```

If the tool call fails or errors, print a warning and continue: "⚠️ [DevPilot] Warning: Could not post 'Pull Request Completed' comment — {error}. PR URL: {prUrl}"

**7e.** Update `.devpilot/state/{workItemId}.json`:
- Set `prCreated` to `true`
- Set `prUrl` to the PR URL string
- Set `status` to `"COMPLETED"`
- Set `lastUpdated` to current ISO 8601 timestamp

Commit:
```bash
git add .devpilot/state/{workItemId}.json
git commit -m "chore: devpilot state — workflow complete for {workItemId}"
```

**7f.** Tell the developer:

> DevPilot workflow complete for work item {workItemId}!
>
> Pull Request: {prUrl}
>
> All 7 stages completed. The Azure DevOps work item has been updated with a full progress trail.

---

## [STATUS: IMPLEMENTING]

Implementation was in progress when the session ended.

Post ADO comment:
```
[DevPilot] Resuming Implementation
```

Then continue from **[STAGE 4: Implementation]** step 4e above (re-post the Stage Started comment, then read the plan and re-invoke the implementation skill).

---

## [STATUS: REVIEWING]

Code review was in progress. Post ADO comment:
```
[DevPilot] Resuming Code Review
```

Then resume from **[STAGE 5: Code Review]** step 5a above.

---

## [STATUS: TESTING]

Testing was in progress. Post ADO comment:
```
[DevPilot] Resuming Testing
```

Then resume from **[STAGE 6: Testing]** step 6a above.

---

## [STATUS: CREATING_PR]

PR creation was in progress. Post ADO comment:
```
[DevPilot] Resuming Pull Request Creation
```

Then resume from **[STAGE 7: Pull Request]** step 7a above.

---

## [STATUS: COMPLETED]

This workflow is already complete.

Tell the developer: "Work item {workItemId} workflow is already completed. PR: {prUrl}"

---

## [STATUS: UNKNOWN]

If the `status` field does not match any of the sections above, stop and say:

"The DevPilot state file for work item {workItemId} contains an unrecognized status: '{status}'. The state file may be corrupted. Check `.devpilot/state/{workItemId}.json` and correct the status field manually before retrying."
