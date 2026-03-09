---
name: publish-to-cloud
description: Publish local test projects, use cases, test cases, and test scripts to Muggle AI cloud. Use when the user asks to publish, upload, sync to cloud, or deploy tests to production. Handles authentication, URL updates for production, and cloud ID mapping.
---

# Publish to Cloud

Publish local test artifacts to Muggle AI cloud for production testing, scheduling, and team collaboration.

## Workflow Overview

```
Check Authentication → Login if needed
      ↓
Select Project to Publish
      ↓
Update Production URL (if localhost)
      ↓
Check Existing Cloud Project → Create or Update
      ↓
Sync Use Cases, Test Cases, Test Scripts
      ↓
Confirm Publication → Return Cloud URLs
```

## Prerequisites

- Local MCP server running
- Muggle AI account
- Local project with test scripts ready to publish

## Step 1: Check Authentication

Call `muggle_auth_status` to verify login state.

**If not authenticated:**
1. Call `muggle_auth_login` to start device code flow
2. Present verification URL and code to user
3. Poll with `muggle_auth_poll` until complete
4. Confirm: "Logged in as user@example.com"

**If authenticated:** Proceed to next step.

## Step 2: Select Project to Publish

Call `muggle_project_list` to show available local projects.

**Present options:**
```
Local projects available for publishing:

1. proj_abc - "My Web App" (http://localhost:3000)
   - 3 use cases, 8 test cases, 5 test scripts
   - Last modified: 2 hours ago

2. proj_def - "Admin Portal" (http://localhost:4000)
   - 2 use cases, 4 test cases, 2 test scripts
   - Last modified: 1 day ago

Which project to publish?
```

**If user specifies project name:** Match and confirm.

## Step 3: Update Production URL

If the project URL is localhost, prompt for production URL.

**Ask user:**
```
Current project URL: http://localhost:3000

For cloud testing, provide the production/staging URL:
- Production: https://app.example.com
- Staging: https://staging.example.com

Enter the target URL for cloud tests:
```

**If URL provided:** Update locally with `muggle_project_update`.

**If user wants to keep localhost:** Warn that cloud replay won't work without public URL.

## Step 4: Check Existing Cloud Project

Call `muggle_publish_project` which internally checks for existing cloud mapping.

**Behavior:**
- If local project has `cloudProjectId` → Updates existing cloud project
- If no cloud mapping exists → Creates new cloud project

**What gets synced:**
1. Project metadata (name, description, URL)
2. All use cases under the project
3. All test cases under each use case
4. Test script references (already uploaded by electron-app during generation)

## Step 5: Publish Entities

### Publish Project

```
{
  "projectId": "<local_project_id>",
  "targetUrl": "<production_url>"  // Optional override
}
```

**Returns:**
- `cloudProjectId` - The cloud project ID
- `cloudUseCaseIds` - Mapping of local → cloud use case IDs
- `cloudTestCaseIds` - Mapping of local → cloud test case IDs
- `viewUrl` - URL to view project in Muggle AI dashboard

### Publish Individual Test Script (Optional)

If user wants to publish only specific test scripts:

```
{
  "projectId": "<local_project_id>",
  "testScriptId": "<local_test_script_id>"
}
```

**Note:** Test scripts are automatically uploaded to Firebase during generation. This tool confirms cloud status and returns the view URL.

## Step 6: Confirm Publication

After successful publish, report:

```
Publication Complete
====================

Project: "My Web App"
Cloud URL: https://app.muggle-ai.com/projects/cloud_proj_123

Published:
  ✅ 3 use cases
  ✅ 8 test cases  
  ✅ 5 test scripts

Cloud IDs mapped locally for future syncs.

Next steps:
- View in dashboard: https://app.muggle-ai.com/projects/cloud_proj_123
- Schedule automated runs
- Share with team members
```

## Selective Publishing

User can choose what to publish:

### Publish Everything

**User:** "Publish my project"

Publishes: Project + all use cases + all test cases + all test scripts

### Publish Specific Use Case

**User:** "Publish only the login use case"

1. `muggle_use_case_list` → Find matching use case
2. Publish project (if not already)
3. Publish only that use case and its test cases/scripts

### Publish Only New Content

**User:** "Publish only what's new"

1. Check cloud ID mappings
2. Identify entities without cloud IDs
3. Publish only unmapped entities

## Example Interactions

### Example 1: First-Time Publish

**User:** "Publish my tests to the cloud"

**Agent:**
1. `muggle_auth_status` → Not authenticated
2. `muggle_auth_login` → Present device code
3. User completes authentication
4. `muggle_project_list` → Show projects
5. User selects "My Web App"
6. Ask for production URL → User provides "https://app.example.com"
7. `muggle_project_update` → Update URL
8. `muggle_publish_project` → Sync to cloud
9. Report: "Published! View at https://app.muggle-ai.com/projects/..."

### Example 2: Re-publish After Changes

**User:** "Sync my project to the cloud"

**Agent:**
1. `muggle_auth_status` → Authenticated
2. `muggle_project_list` → User has 1 project with cloud mapping
3. `muggle_publish_project` → Updates existing cloud project
4. Report: "Synced 2 new test cases, 1 updated use case"

### Example 3: Publish Specific Script

**User:** "Upload the login test script"

**Agent:**
1. `muggle_test_script_list` → Find login script
2. Check if it has `actionScriptId` (already uploaded during generation)
3. If yes: "Script already in cloud. View at ..."
4. If no: "Script needs to be generated first. Run test generation?"

## Quick Reference

| Action | Tool |
| :----- | :--- |
| Check login | `muggle_auth_status` |
| Start login | `muggle_auth_login` |
| Complete login | `muggle_auth_poll` |
| List projects | `muggle_project_list` |
| Update project URL | `muggle_project_update` |
| Publish project | `muggle_publish_project` |
| Publish test script | `muggle_publish_test_script` |

## URL Handling

| Scenario | Action |
| :------- | :----- |
| localhost URL | Prompt for production URL before publishing |
| Staging URL | Use as-is, good for pre-production testing |
| Production URL | Use as-is |
| Keep localhost | Warn that cloud replay requires public URL |

## Notes

- **Authentication required:** All publish operations require Muggle AI login.
- **URL matters:** Cloud tests run against the URL stored in the project.
- **Idempotent:** Re-publishing updates existing cloud entities, doesn't duplicate.
- **Scripts auto-uploaded:** Test scripts are uploaded to Firebase during generation by electron-app.
- **Cloud ID mapping:** Local storage tracks cloud IDs for future syncs.
- **No auto-publish:** This skill only publishes when explicitly requested.
