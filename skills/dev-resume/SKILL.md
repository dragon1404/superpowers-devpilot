---
name: dev-resume
description: "Resume a DevPilot workflow from the last completed stage. Running this command while in an approval-wait state counts as approving that stage. Usage: /dev-resume <workItemId>"
---

# DevPilot: Resume Workflow

**Announce:** "Resuming DevPilot workflow for work item {workItemId}."

## Step 1 — Extract Work Item ID

Extract the workItemId from the user's message (the integer following `/dev-resume`).

If no workItemId is provided, stop and say: "Please provide a work item ID. Usage: /dev-resume <workItemId>"

## Step 2 — Read State File

Read `.devpilot/state/{workItemId}.json`.

If the file does not exist, stop and say: "No DevPilot workflow found for work item {workItemId}. Start one with `/dev-workitem {workItemId}`."

Store all fields from the state file for use in subsequent steps.

## Step 3 — Branch on Status

Read the `status` field and jump to the matching section below.

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
- id: {workItemId}
- text:
  ```
  [DevPilot] Design Approved — Starting Implementation Plan
  ```

**3d.** Continue to **[STAGE 3: Implementation Plan]** below.

---

## [STAGE 3: Implementation Plan]

**3e.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- id: {workItemId}
- text:
  ```
  [DevPilot] Stage Started: Implementation Plan
  ```

**3f.** Read the design document from `docs/design/{workItemId}-design.md`.

**3g.** Invoke `superpowers:writing-plans` using the Skill tool, passing the design document content as the `args` parameter. The plan must be saved to `docs/plan/{workItemId}-plan.md`.

**3h.** Commit:
```bash
git add docs/plan/{workItemId}-plan.md
git commit -m "docs: add implementation plan for work item {workItemId}"
```

**3i.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- id: {workItemId}
- text:
  ```
  [DevPilot] Stage Completed: Implementation Plan
  Document: docs/plan/{workItemId}-plan.md
  ```

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
- id: {workItemId}
- text:
  ```
  [DevPilot] Plan Approved — Starting Implementation
  ```

**4d.** Continue to **[STAGE 4: Implementation]** below.

---

## [STAGE 4: Implementation]

**4e.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- id: {workItemId}
- text:
  ```
  [DevPilot] Stage Started: Implementation
  ```

**4f.** Read the plan from `docs/plan/{workItemId}-plan.md`.

**4g.** Invoke `superpowers:subagent-driven-development` using the Skill tool, passing the plan content as the `args` parameter.

**4h.** After implementation completes:

Call `mcp__azure-devops__wit_add_work_item_comment`:
- id: {workItemId}
- text:
  ```
  [DevPilot] Stage Completed: Implementation
  ```

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
- id: {workItemId}
- text:
  ```
  [DevPilot] Stage Started: Code Review
  ```

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

**5e.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- id: {workItemId}
- text:
  ```
  [DevPilot] Stage Completed: Code Review
  Document: docs/review/{workItemId}-review.md
  ```

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
- id: {workItemId}
- text:
  ```
  [DevPilot] Stage Started: Testing
  ```

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

**6e.** Call `mcp__azure-devops__wit_add_work_item_comment`:
- id: {workItemId}
- text:
  ```
  [DevPilot] Stage Completed: Testing
  Document: docs/testing/{workItemId}-testing.md
  ```

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
- id: {workItemId}
- text:
  ```
  [DevPilot] Stage Started: Pull Request
  ```

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
- id: {workItemId}
- text:
  ```
  [DevPilot] Stage Completed: Pull Request
  PR: {prUrl}
  ```

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
