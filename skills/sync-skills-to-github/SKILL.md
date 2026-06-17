---
name: sync-skills-to-github
description: "Sync local Hermes skills to a GitHub repository — clone, copy, commit, push. Creates a cronnable workflow for keeping skills backed up."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [github, skills, backup, sync, automation]
    related_skills: [github-auth, github-repo-management, hermes-agent-skill-authoring]
---

# Sync Skills to GitHub

Syncs local Hermes skills (in `~/.hermes/skills/`) to a GitHub repository. Can be used ad-hoc or scheduled via cron for automatic backups.

## Prerequisites

- `gh` CLI authenticated (see `github-auth` skill)
- Write access to the target GitHub repo

## Usage

### One-shot sync

```bash
# Sync all skills to a repo (default: single sync)
SYNC_REPO="owner/repo-name"

# Clone if needed
if [ ! -d "/tmp/sync-${SYNC_REPO##*/}" ]; then
  gh repo clone "$SYNC_REPO" "/tmp/sync-${SYNC_REPO##*/}"
fi

cd "/tmp/sync-${SYNC_REPO##*/}"

# Copy all skills from local Hermes skills dirs — REPLACES repo versions
for skill_dir in ~/.hermes/skills/*/*/; do
  skill_name="$(basename "$skill_dir")"
  # Skip internal/system skills
  case "$skill_name" in
    hermes-agent|dogfood) continue ;;
  esac
  # Create or update the skill in the repo
  if [ -d "skills/$skill_name" ]; then
    cp -r "$skill_dir"/* "skills/$skill_name/"
  else
    cp -r "$skill_dir" "skills/$skill_name" 2>/dev/null || true
  fi
done

# Commit and push
git add -A
if git diff --cached --quiet; then
  echo "No changes to sync."
else
  git commit -m "Sync skills: $(date +%Y-%m-%d)"
  git push origin HEAD
fi
```

### Cron-based auto-sync

Use `cronjob` action='create' to run this daily:

```yaml
schedule: "0 6 * * *"  # daily at 6 AM
prompt: |
  Sync all Hermes skills to bowtiekreative/metabox-agent repo.
  Load the sync-skills-to-github skill and follow its instructions.
skills: [sync-skills-to-github]
```

Or use the inline script version with cronjob `no_agent=True` for zero-cost operation.

## Sync Script

Save this as a standalone script for cron use with `no_agent=True`:

```bash
#!/bin/bash
# ~/.hermes/scripts/sync-skills.sh
# Cron-ready: outputs nothing when there's nothing to sync (silent = clean)
# Outputs summary when changes are pushed.

SYNC_REPO="${1:-bowtiekreative/metabox-agent}"
WORK_DIR="/tmp/sync-${SYNC_REPO##*/}"

# Ensure clone exists
if [ ! -d "$WORK_DIR" ]; then
  gh repo clone "$SYNC_REPO" "$WORK_DIR" 2>/dev/null || exit 1
fi

cd "$WORK_DIR" || exit 1

# Update origin URL with token
REMOTE=$(git remote get-url origin)
if echo "$REMOTE" | grep -q "^https://"; then
  TOKEN=$(gh auth token)
  git remote set-url origin "https://bowtiekreative:${TOKEN}@github.com/${SYNC_REPO}.git"
fi

# Pull latest
git pull --ff-only origin HEAD 2>/dev/null || true

# Copy all local skills
UPDATED=0
for skill_dir in ~/.hermes/skills/*/*/; do
  skill_name="$(basename "$skill_dir")"
  [ -z "$skill_name" ] && continue
  mkdir -p "skills/$skill_name"
  cp -r "$skill_dir"/* "skills/$skill_name/" 2>/dev/null
  UPDATED=1
done

if [ "$UPDATED" -eq 0 ]; then
  exit 0
fi

# Commit and push
git add -A
if git diff --cached --quiet; then
  exit 0  # Silent when nothing changed
fi

git commit -m "Auto-sync skills $(date +%Y-%m-%d_%H:%M)"
git push origin HEAD 2>&1 && echo "Synced skills to ${SYNC_REPO}"
```

## Filtered Sync (specific skills only)

To sync only specific skills:

```bash
SKILLS_TO_SYNC=(
  "metabox-custom-fields"
  "metabox-views"
  "html-to-metabox-views"
)

for skill in "${SKILLS_TO_SYNC[@]}"; do
  local_path=$(find ~/.hermes/skills -name "$skill" -type d 2>/dev/null | head -1)
  if [ -n "$local_path" ]; then
    rm -rf "skills/$skill" 2>/dev/null
    cp -r "$local_path" "skills/$skill"
    echo "✓ Synced $skill"
  fi
done
```

## Notes

- The script copies the ENTIRE skill directory (SKILL.md + any references/, templates/, scripts/)
- It does NOT delete skills from the repo that no longer exist locally — those are preserved
- The cron job runs silently when nothing changed (no wasted output)
- Uses `gh auth token` for authentication — make sure `gh auth login` is active
