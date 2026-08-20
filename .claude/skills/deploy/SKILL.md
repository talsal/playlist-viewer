---
name: deploy
description: Commit and push changes to the playlist-viewer repo, wait for GitHub Pages to build, work around the intermittent stuck-build issue, and verify the change is actually live. Use whenever a change to index.html (or any repo file) needs to go live on talsal.github.io/playlist-viewer.
user-invocable: true
allowed-tools:
  - Bash(git *)
  - Bash(gh *)
  - Bash(node *)
  - Bash(curl *)
  - Bash(sleep *)
---

# /deploy — playlist-viewer commit, push, and verify

This project (`talsal/playlist-viewer`, live at `talsal.github.io/playlist-viewer`)
deploys via GitHub Pages, which reliably takes 1-3 minutes to build but has an
intermittent bug: builds occasionally get stuck at `"building"` with zero
progress for 20+ minutes. This skill automates the full commit → push → wait
→ unstick-if-needed → verify sequence learned the hard way across many manual
deploys.

Arguments passed (optional): `$ARGUMENTS` — if given, use as the commit
message subject. If empty, write a commit message yourself from the actual
diff (see repo's CLAUDE.md-equivalent conventions: concise, explains *why*,
`Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>` trailer).

## Steps

1. **Check for changes to commit.**
   ```bash
   cd /Users/talsalman/Documents/tal/dev/playlist-viewer
   git status
   ```
   If the working tree is clean and HEAD is already what's live (compare
   `git rev-parse HEAD` against the deployed commit — see step 4's query),
   nothing to do; report that and stop.

2. **Validate before committing** (this repo is a single large `index.html`
   with embedded JSON-like playlist data — a syntax slip here breaks the
   whole site silently). Run:
   ```bash
   node -e "
   const fs = require('fs');
   const html = fs.readFileSync('index.html', 'utf8');
   const m = html.match(/const playlists = (\[[\s\S]*?\n\]);/);
   const playlists = eval(m[1]);
   console.log('playlists:', playlists.length);
   let total = 0;
   playlists.forEach(p => total += p.songs.length);
   console.log('songs:', total);
   "
   ```
   If this throws, stop and fix the syntax error before proceeding — do not
   push broken JSON.

3. **Commit and push.**
   ```bash
   git add -A
   git commit -m "$(cat <<'EOF'
   <message>

   Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
   EOF
   )"
   git push
   ```

4. **Poll the build status with a real timeout** — GitHub Pages builds on
   this repo normally take 1-3 minutes given the site's size (~280KB single
   file). Don't treat a 2-minute wait as "stuck." Use a timeout of at least
   8-10 minutes on the wait command itself:
   ```bash
   until [ "$(gh api repos/talsal/playlist-viewer/pages --jq .status)" != "building" ]; do sleep 10; done
   gh api repos/talsal/playlist-viewer/pages/builds/latest --jq '{status, commit, error}'
   ```

5. **If it's genuinely stuck** (still `"building"` after ~10+ real minutes,
   or `created_at` == `updated_at` on the latest build with no change across
   several checks), unstick it with an empty commit — this has worked every
   time it's been needed:
   ```bash
   git commit --allow-empty -m "$(cat <<'EOF'
   Retrigger stuck GitHub Pages build

   Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
   EOF
   )"
   git push
   ```
   Then repeat step 4.

6. **Verify the live commit hash matches** what was just pushed — the Pages
   API status can lag or report `"built"` for a stale commit while a newer
   one is mid-build. Don't trust the status alone:
   ```bash
   git rev-parse HEAD
   gh api repos/talsal/playlist-viewer/pages/builds/latest --jq .commit
   ```
   These two values must match (the full 40-char SHA) before calling it done.

7. **Confirm the actual content is live**, not just that a build succeeded —
   fetch the real page and check for something specific to the change (a
   new function name, a new playlist id, a new CSS class — whatever's
   distinctive to what was just shipped):
   ```bash
   curl -s "https://talsal.github.io/playlist-viewer/index.html" | grep -o "<distinctive string from the change>"
   ```

8. Report the result to the user with the live URL:
   `https://talsal.github.io/playlist-viewer/`

## Notes

- Never skip step 2 (syntax validation) — this repo has no CI/tests, so a
  bad push silently breaks the entire live site until caught.
- Never force-push or amend here; always a fresh commit per the global git
  safety rules.
- If step 2's validation fails, do not proceed to commit — fix the file
  first (the broken state should never reach `git add`).
