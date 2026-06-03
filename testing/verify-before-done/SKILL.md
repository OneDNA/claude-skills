# Verify Before Claiming Done

Run this skill before declaring any task complete. Prevents premature success claims by enforcing end-to-end validation.

## Steps

### For code / API changes

1. Hit the actual endpoint and show the response:

   ```bash
   curl -s https://<url>/api/<endpoint> | jq .
   ```

2. Check the container logs for errors:

   ```bash
   az containerapp logs show --name <ca-name> --resource-group <rg> --tail 20
   ```

### For docs deploys

1. Fetch the live nav and confirm the expected page titles are present:

   ```bash
   curl -s https://<docs-url>/ | grep -o '<title>[^<]*</title>'
   ```

2. Spot-check at least one newly added or changed page by curling its URL.

### For data syncs (MEMORY.md, kanban board, roadmap)

1. Count records in both source and destination and show the diff:

   ```bash
   # Example: count Next Up items in MEMORY.md vs board.json
   grep -c '^\d\.' C:/Users/srely/Repos/portfolio/docs/planning/wip.md
   cat .claude/board.json | jq '.in_progress | length'
   ```

2. List any items present in source but absent from destination.

### For feature flags / config

1. Confirm the flag value via the live endpoint or App Config:

   ```bash
   az appconfig kv show --name dna-portfolio-we-appcs --key <flag-name> --label <label>
   ```

### General rule

- Do NOT write "Done" or "Complete" until at least one of the above checks has been run and the output shown.
- If a check cannot be run (e.g., no network access), explicitly say so and describe what would need to be verified manually.

## When to use

Before closing any task, before writing a PR description, before writing a handover doc, and whenever you are about to say the words "done", "complete", "deployed", or "synced".
