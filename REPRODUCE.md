# Reproduction: dev-runtime snapshot copies nested git worktrees

This is a minimal `npx eve@latest init` project. It demonstrates that the
`eve dev` development-runtime source snapshot recurses into a **nested git
worktree** living inside the app root and copies its authored source as if it
were part of the app.

Verified with `eve@0.19.0` (includes #490).

## Steps

```bash
npm install

# Create a nested git worktree inside the repo, under a gitignored path
# (this mirrors what Claude Code does with `git worktree add`).
git worktree add .claude/worktrees/test-wt -b test-wt

# Start the dev server (snapshot is copied at boot, before any model call).
EVE_LOG_LEVEL=debug npx eve dev --no-ui
# wait for: [DEV] server listening at http://127.0.0.1:2000/
# then stop it (Ctrl-C)

# Inspect the snapshot
SNAP=$(find .eve/dev-runtime/snapshots -maxdepth 1 -mindepth 1 -type d | head -1)
find "$SNAP/source/.claude/worktrees/test-wt"
```

## Observed

`.eve/dev-runtime/snapshots/<session>/source/.claude/worktrees/test-wt/` contains
a full copy of the app's authored source:

```
.claude/worktrees/test-wt/.gitignore
.claude/worktrees/test-wt/.vercelignore
.claude/worktrees/test-wt/AGENTS.md
.claude/worktrees/test-wt/CLAUDE.md
.claude/worktrees/test-wt/agent/agent.ts
.claude/worktrees/test-wt/agent/channels/eve.ts
.claude/worktrees/test-wt/agent/instructions.md
.claude/worktrees/test-wt/package.json
.claude/worktrees/test-wt/package-lock.json
.claude/worktrees/test-wt/tsconfig.json
```

Notes:
- The worktree's own `.git` **file** is skipped (it's named `.git`, which is in
  `SNAPSHOT_SKIP_NAMES`), but nothing stops the copier from recursing into the
  rest of the worktree — confirming the skip is purely **name-based**, with no
  nested-repository-boundary detection.
- `.claude/` is gitignored (`git check-ignore .claude/worktrees/test-wt`
  succeeds) yet still copied — the snapshot does not honor `.gitignore`.

In a real repo the worktree also carries `node_modules`/`.next` (excluded by
#490) plus thousands of authored-source files, so each accumulated worktree adds
~tens of MB to every dev snapshot.

## Expected

Some combination of:
- Stop recursing into any subdirectory that contains its own `.git`
  file/directory (i.e. is itself a worktree or nested repo).
- And/or honor `.gitignore` for the snapshot.
