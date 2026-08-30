# claude-azure-devops-skill

A Claude Code [skill](https://docs.claude.com/) that lets Claude view Azure DevOps work items (boards) and pull requests, and switch between Azure DevOps orgs/projects/tenants — via the `az` CLI + `azure-devops` extension. See `SKILL.md` for the full behavior.

Scope: **viewing** work items and PRs (branches, status, reviewers) for review purposes, plus org/project/tenant switching. Posting PR comments is out of scope (the `azure-devops` CLI extension has no native comment verb).

This repo is the source of truth; the files Claude Code actually reads are symlinks into `~/.claude/`, pointing back here. That way the skill (and its cache) are version-controlled and easy to carry to another machine.

## Files

- `SKILL.md` — the skill definition, symlinked to `~/.claude/skills/azure-devops/SKILL.md`.
- `cache.template.json` — an empty skeleton (`{"tenants": {}, "orgs": {}}`) for the cache described in `SKILL.md`: org/project/repo IDs and per-project board vocabulary (which work item types — Task/Bug/Feature/Epic/User Story/etc — each project actually uses). Tracked in git since it has no real data in it.
- `cache.json` — **not tracked in git** (see `.gitignore`). This is the real, filled-in cache — it accumulates client-identifying data (real tenant IDs, org names, project/repo names) as Claude discovers new orgs/projects, so it must stay local-only. It's symlinked to `~/.claude/skills/azure-devops/cache.json`, same as `SKILL.md`. On a fresh machine, seed it from the template (see Install below) rather than committing a filled-in copy.

  **This file previously *was* committed and pushed to this repo while it was public — treat any tenant IDs/client names from that history as already exposed** (removing the file from a later commit doesn't erase it from git history). If this repo is or was public, flip it to private now (GitHub Settings → Danger Zone, or `gh repo edit <owner>/<repo> --visibility private`); purging the old commit from history entirely requires a history rewrite (`git filter-repo` + force-push), which is a separate, destructive step to decide on deliberately.

- `settings.permissions.json` — **not symlinked**. It's a reference copy of the permission entries this skill needs in `~/.claude/settings.json`, pre-approving the read-only `az` commands (`az devops configure`, `az account show/list`, `az devops project list`, `az repos list`, `az boards work-item show`, `az boards query`, `az repos pr list/show`) so Claude isn't prompted for them every time. It can't be symlinked as-is because `~/.claude/settings.json` is a single shared file with unrelated settings (theme, model config, etc.) — the `permissions.allow` array in it needs to be merged in by hand (or ask Claude to do it, e.g. via the `update-config` skill).

## Install (new machine, or after a fresh clone)

```sh
mkdir -p ~/.claude/skills/azure-devops
ln -sf "$(pwd)/SKILL.md" ~/.claude/skills/azure-devops/SKILL.md

# seed the local, untracked cache from the template if it doesn't exist yet
[ -f cache.json ] || cp cache.template.json cache.json
ln -sf "$(pwd)/cache.json" ~/.claude/skills/azure-devops/cache.json
```

Then merge the contents of `settings.permissions.json`'s `permissions.allow` array into `~/.claude/settings.json`'s `permissions.allow` (create the key if it doesn't exist yet). Easiest way: open a Claude Code session and ask it to merge `settings.permissions.json` from this repo into your global settings.

## Requirements

- `az` CLI installed, with the `azure-devops` extension (`az extension add --name azure-devops`).
- `az login` already run for at least one tenant (`az account show` should return something).

## Switching orgs/tenants

- **Different org/project, same tenant** — Claude can do this itself: `az devops configure --defaults organization=... project=...`.
- **Different tenant** — requires an interactive browser login Claude cannot run on your behalf. Claude will ask you to run `az login --tenant <tenant-id>` yourself. If that tenant has no ARM subscriptions (common for a client tenant you only access via Azure DevOps), use `--allow-no-subscriptions`, otherwise `az` won't create an account context for it and subsequent `az devops`/`az repos`/`az boards` calls will fail with a "you need to run the login command" error even though you're logged in:
  ```sh
  az login --tenant <tenant-id> --allow-no-subscriptions
  ```
