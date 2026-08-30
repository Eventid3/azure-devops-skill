---
name: azure-devops
description: View Azure DevOps work items (boards/backlogs) and pull requests, and switch between the user's Azure DevOps orgs/projects/tenants. Use when the user mentions work items, boards, backlog items, sprints, epics/features/tasks, pull requests / PRs, PR branches, `az boards`, `az repos`, Azure DevOps, or ADO. PR reviewing (checking out the right branch, reading the diff) is a primary use case. Does not post PR comments (out of scope).
---

# Azure DevOps

The `az` CLI with the `azure-devops` extension is installed and logged in. This skill lets you view work items and PRs across the user's orgs, and it maintains a local cache (`cache.json`, next to this file) of org/project/repo IDs and each project's board vocabulary, since that discovery is request-heavy but essentially static.

## 0. Load the cache first

Read `cache.json` before running any discovery commands (`az devops project list`, `az repos list`). Only hit `az` for information missing from the cache, then write what you learn back into `cache.json` before finishing the turn (org/project/repo IDs never change once seen; work item types only ever grow).

Cache shape:

```json
{
  "tenants": { "<tenant-guid>": { "label": "..." } },
  "orgs": {
    "<org-name>": {
      "tenant_id": "<tenant-guid>",
      "projects": {
        "<project-name>": {
          "id": "<project-guid>",
          "repos": { "<repo-name>": { "id": "<repo-guid>" } },
          "work_item_types_seen": ["Task", "Bug", "Feature", "Epic", "User Story"]
        }
      }
    }
  }
}
```

## 1. Check current context

- `az devops configure -l` — default org/project.
- `az account show` — active tenant/subscription.

Cross-reference the org/tenant against `cache.json`. If the org is unknown, look it up (`az account show`'s `tenantId`) and add it to `tenants`/`orgs` rather than guessing which tenant it belongs to.

## 2. Switching org/project (same tenant)

No login required — just update the default:

```
az devops configure --defaults organization=https://dev.azure.com/<org>/ project=<project>
```

You can do this yourself whenever the task needs a different org/project that's under the tenant already logged in.

## 3. Switching tenant

`az login --tenant <id>` requires an interactive browser flow you cannot run. When the task needs an org/directory under a different tenant than `az account show` currently reports:

1. Stop and ask the user to run it themselves, e.g. suggest:
   ```
   ! az login --tenant <tenant-id>
   ```
2. Once they confirm they're logged in, set the org/project default (step 2 above) and, if this is a new tenant, add it to `cache.json`.

Never guess a tenant ID — use one already in the cache or ask the user.

## 4. Work items (boards)

- `az boards work-item show --id <id>` — full details of one item.
- `az boards query --wiql "<WIQL>"` — search/list (e.g. items assigned to the user, items in a sprint, by type/state).

After fetching an item, check its `System.WorkItemType` (or the `fields."System.WorkItemType"` in JSON output) against that project's `work_item_types_seen` in the cache — add it if new. This is how you learn a project's board vocabulary organically (one project might use Epic → Feature → User Story → Task, another might just use Task/Bug) instead of ever needing the full process schema. When reasoning about a project's board structure, defer to what's actually in `work_item_types_seen` for that project rather than assuming a generic Agile/Scrum template.

## 5. Pull requests (viewing only — no commenting)

- `az repos pr list` — list PRs (add `--repository`/`--status` to narrow).
- `az repos pr show --id <id>` — full details: `sourceRefName`, `targetRefName`, repository, status, reviewers, work item links.

For actually reviewing a PR's diff: use `sourceRefName`/`targetRefName` from `pr show` to `git fetch`/`git checkout` the right branches in the local repo clone, then read the diff normally (same as any other branch review). This skill does not post PR comments — the `azure-devops` CLI extension has no native comment verb for that (it would require a raw `az devops invoke` REST call); flag that as a manual follow-up for the user if they ask for it.

## 6. First-time discovery for a new org/project

- `az devops project list --org https://dev.azure.com/<org>/` — project names + IDs.
- `az repos list --org https://dev.azure.com/<org>/ --project <project>` — repo names + IDs for that project.

Write the results into `cache.json` immediately so they aren't rediscovered next time.
