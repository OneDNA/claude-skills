# Post-Rebase Integrity Check

Run this skill after every rebase to ensure no files were silently dropped during conflict resolution.

## Steps

1. Capture the pre-rebase SHA:

   ```bash
   git reflog | grep 'rebase (start)' | head -1
   ```

   Or use the SHA you had before rebasing (check `git reflog` for `HEAD@{N}` before the rebase started).

2. Diff the file list against the pre-rebase commit:

   ```bash
   git diff <pre-rebase-sha>..HEAD --stat
   ```

   Show the full output — list every file that was added, modified, or deleted.

3. Identify any files that were present before the rebase but are now missing:

   ```bash
   git diff <pre-rebase-sha>..HEAD --diff-filter=D --name-only
   ```

   If this outputs anything, those files were dropped. Restore them before pushing.

4. Run the local build and test suite:

   ```bash
   # Frontend
   cd client && npm run build && npm run test
   # Backend
   cd .. && pytest tests/ -q
   ```

5. Only push once all tests pass and no files are missing. Show the diff stat output before force-pushing.

## When to use

After any `git rebase`, `git rebase --continue`, or merge conflict resolution. Required before force-push.
