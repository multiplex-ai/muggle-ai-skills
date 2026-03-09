---
name: test-feature-local
description: Test a feature locally using Muggle AI local MCP. Handles the full flow of listing/creating projects, use cases, and test cases, then generates or replays test scripts. Use when the user asks to test a feature, test a user flow, or run local tests against localhost applications.
---

# Test Feature Locally

Test a feature on a local web application using Muggle AI local MCP tools.

## Workflow Overview

```
Analyze Changes (optional) → Identify impacted features
      ↓
List Projects → Find/Create Project
      ↓
List Use Cases → Find/Create Use Case(s)
      ↓
List Test Cases → Find/Create Test Case(s)
      ↓
List Test Scripts → Generate (if none) or Replay (if exists)
```

## Prerequisites

- Local MCP server running
- Target web application running (e.g., `http://localhost:3999`)
- User describes the feature to test OR has code changes to analyze

## Step 0: Analyze Current Changes (Optional but Recommended)

When user asks to "test my changes" or doesn't specify a feature, analyze the codebase changes first.

### Gather Change Context

Run these commands to understand what changed:

```bash
# See changed files
git status

# See actual code changes
git diff

# See staged changes
git diff --cached

# Recent commits if needed
git log -3 --oneline
```

### Identify Impacted Features

From the changes, extract:

1. **Modified components/pages:** Which UI components or pages changed?
2. **Modified API endpoints:** Which backend routes are affected?
3. **Modified services/logic:** What business logic changed?

### Map Changes to Test Areas

| Change Type | Likely Impacted Tests |
| :---------- | :-------------------- |
| Login/auth files | Authentication use cases |
| User profile files | User management use cases |
| Form components | Data input/validation test cases |
| API route handlers | CRUD operation test cases |
| Navigation/routing | Navigation flow use cases |
| Payment/checkout | Transaction test cases |

### Present Findings to User

After analysis, summarize:
1. What files changed
2. What features are likely impacted
3. Suggested use cases/test cases to run

**Example output:**
```
Based on your changes:
- Modified: src/components/LoginForm.tsx, src/api/auth.ts
- Impacted features: User authentication, session management
- Suggested tests:
  1. "User can log in with valid credentials"
  2. "User sees error with invalid password"
  3. "User can reset password"

Which test(s) would you like to run?
```

## Step 1: Identify or Create Project

Call `muggle_project_list` to see existing projects.

**Match logic:**
- Match by URL (exact or base domain match)
- Match by project name (case-insensitive contains)

**If match found:** Use existing project, confirm with user.

**If no match:** Call `muggle_project_create` with:
- `name`: Derived from app name or URL
- `url`: The localhost URL
- `description`: Brief description of the app

## Step 2: Identify or Create Use Case(s)

Call `muggle_use_case_list` with the `projectId`.

**Match logic:**
- Match by goal/title containing key terms from user's feature description
- Match against impacted features from change analysis (Step 0)
- Look for similar user personas

**If multiple matches found:**
1. List all matching use cases with brief descriptions
2. Ask user: "Found N use cases that may be impacted. Which would you like to test?"
3. Allow selecting multiple (e.g., "all", "1,2,3", or specific names)

**If single match found:** Confirm with user, use existing.

**If no match:** Call `muggle_use_case_save` with:
```
{
  "projectId": "<projectId>",
  "useCase": {
    "title": "<Feature name>",
    "userPersona": "<Who is performing this>",
    "goal": "<What the user wants to achieve>",
    "breakdownItems": ["<Step 1>", "<Step 2>", ...]
  }
}
```

## Step 3: Identify or Create Test Case(s)

Call `muggle_test_case_list` with `projectId` and optionally `useCaseId`.

**Match logic:**
- Match by title or goal containing feature keywords
- Match against impacted features from change analysis (Step 0)
- Match by URL if provided
- For each selected use case, find associated test cases

**If multiple matches found:**
1. List all matching test cases grouped by use case
2. Show which are likely impacted based on change analysis
3. Ask user: "Found N test cases. Run all impacted, or select specific ones?"
4. Options: "all", "impacted only", or specific IDs

**Example output:**
```
Found test cases:

Use Case: User Authentication
  [IMPACTED] tc_001 - Login with valid credentials (login form changed)
  [IMPACTED] tc_002 - Login with invalid password
  [ ] tc_003 - Remember me functionality

Use Case: User Profile
  [ ] tc_004 - Update display name

Run: [all impacted] [all] [select specific]?
```

**If single match found:** Confirm with user, use existing.

**If no match:** Call `muggle_test_case_save` with:
```
{
  "projectId": "<projectId>",
  "useCaseId": "<useCaseId>",
  "testCase": {
    "title": "<Test case name>",
    "goal": "<What this test verifies>",
    "url": "<Starting URL>",
    "precondition": "<Any setup needed>",
    "steps": ["<Step 1>", "<Step 2>", ...],
    "expectedResult": "<What should happen>"
  }
}
```

## Step 4: Generate or Replay Test Script(s)

For each selected test case, call `muggle_test_script_list` with `projectId` and `testCaseId`.

### For multiple test cases:

Process each test case and categorize:

```
Test execution plan:
  REPLAY (script exists):
    - tc_001: Login with valid credentials
    - tc_002: Login with invalid password
  
  GENERATE (no script):
    - tc_005: New password reset flow

Proceed? [yes/no]
```

### If test script exists:

1. Confirm with user: "Found existing test script. Replay it?"
2. If yes, call `muggle_execute_replay`:
   ```
   {
     "projectId": "<projectId>",
     "testScriptId": "<testScriptId>"
   }
   ```
3. Report results from the execution

### If no test script:

1. Inform user: "No test script found. Generating one..."
2. Call `muggle_execute_test_generation`:
   ```
   {
     "projectId": "<projectId>",
     "testCaseId": "<testCaseId>"
   }
   ```
3. Wait for generation to complete
4. Report the generated script details

### Batch Execution Order

When running multiple tests:

1. **Replay existing scripts first** (faster, validates current behavior)
2. **Generate new scripts after** (slower, may need user observation)
3. Track results for each test case

## Step 5: View Results

After execution, call `muggle_run_result_list` or `muggle_run_result_get` to show:
- Pass/fail status
- Screenshots captured
- Execution time
- Any errors encountered

### For Multiple Tests - Summary Report

After running multiple tests, provide a summary:

```
Test Run Summary
================
Total: 4 tests
Passed: 3
Failed: 1

Results:
  ✅ tc_001 - Login with valid credentials (2.3s)
  ✅ tc_002 - Login with invalid password (1.8s)
  ❌ tc_003 - Remember me functionality (3.1s)
     Error: Checkbox not found on page
  ✅ tc_005 - Password reset flow (4.2s) [newly generated]

Failed test details:
  tc_003: Element '#remember-me' not found. 
          Screenshot: run_xxx/step_3.png
```

### Suggested Next Actions

Based on results:

| Result | Suggested Action |
| :----- | :--------------- |
| All passed | "All tests passed. Ready to commit?" |
| Some failed | "N tests failed. Review failures or re-run?" |
| New scripts generated | "N new scripts created. Replay to verify?" |

## Example Interactions

### Example 1: Test Based on Code Changes

**User:** "Test my changes"

**Agent:**
1. `git status` → Modified: LoginForm.tsx, AuthService.ts
2. `git diff` → Analyze changes (added error handling, updated validation)
3. `muggle_project_list` → Found "My App" for localhost:3000
4. `muggle_use_case_list` → Found 5 use cases
5. Present: "Your changes affect authentication. Found 2 related use cases:
   - User Authentication (3 test cases)
   - Password Reset (2 test cases)
   Which to test?"
6. User: "all"
7. `muggle_test_case_list` → 5 test cases total
8. `muggle_test_script_list` for each → 4 have scripts, 1 doesn't
9. Present execution plan, user confirms
10. Run 4 replays, 1 generation
11. Report summary: "4 passed, 1 new script generated"

### Example 2: Test Specific Feature

**User:** "Test the login flow on my app at localhost:3000"

**Agent:**
1. `muggle_project_list` → No match for localhost:3000
2. `muggle_project_create` with name "Local App", url "http://localhost:3000"
3. `muggle_use_case_save` with goal "User can log in to the application"
4. `muggle_test_case_save` with login test details
5. `muggle_test_script_list` → Empty
6. `muggle_execute_test_generation` → Generates script
7. Report: "Test script generated with 5 steps. Ready to replay."

**User:** "Run it again"

**Agent:**
1. `muggle_test_script_list` → Found script
2. `muggle_execute_replay` → Run the script
3. Report results

### Example 3: Re-run Failed Tests

**User:** "Re-run the failed tests"

**Agent:**
1. `muggle_run_result_list` → Find recent failures
2. Identify test cases from failed runs
3. `muggle_execute_replay` for each
4. Report new results

## Quick Reference

| Action | Tool/Command |
| :----- | :----------- |
| Analyze changes | `git status`, `git diff` |
| List projects | `muggle_project_list` |
| Create project | `muggle_project_create` |
| List use cases | `muggle_use_case_list` |
| Create use case | `muggle_use_case_save` |
| List test cases | `muggle_test_case_list` |
| Create test case | `muggle_test_case_save` |
| List test scripts | `muggle_test_script_list` |
| Generate script | `muggle_execute_test_generation` |
| Replay script | `muggle_execute_replay` |
| View results | `muggle_run_result_get` |
| List recent runs | `muggle_run_result_list` |

## Notes

- **Change-aware:** When user says "test my changes", analyze git diff to identify impacted tests.
- **Batch execution:** Can run multiple tests in sequence, with summary report.
- **No publishing:** This skill focuses on local testing only. Use the `publish-to-cloud` skill for publishing.
- **Reuse entities:** Always check for existing projects/use cases/test cases before creating new ones.
- **Confirm with user:** When matches are ambiguous or multiple options exist, ask user to choose.
