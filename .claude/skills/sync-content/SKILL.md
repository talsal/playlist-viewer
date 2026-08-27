---
name: sync-content
description: Check Chaim Salman's YouTube channel (@newhazzan) for videos not yet on the playlist-viewer site, correctly separate his own recordings from guest reciters and other family members, categorize new content, and add it. Use when the user asks to check for new videos, sync the site with the channel, or find what's missing.
user-invocable: true
allowed-tools:
  - Bash(yt-dlp *)
  - Bash(node *)
  - Bash(python3 *)
  - Bash(git *)
  - Bash(gh *)
  - Bash(curl *)
  - Read
  - Edit
---

# /sync-content — find and add new channel videos to the site

This project's data (`index.html`'s `playlists` array) is a hand-curated
snapshot of the father's YouTube channel, not a live feed. He periodically
uploads more. This skill re-runs the diff-and-categorize workflow that was
used to build out the site from 4 playlists / 132 songs to 47 / 1,777 songs,
so the hard-won exclusion/categorization logic doesn't need to be
reconstructed from scratch each time.

**This is a discovery + proposal skill, not an auto-commit skill.** Always
show the user what was found and get confirmation before adding anything to
`index.html` — new content sometimes needs a judgment call (new subject
category, ambiguous performer) that shouldn't be made silently.

## Step 1 — Get the current channel state

```bash
yt-dlp --flat-playlist --print "%(id)s|%(title)s|%(duration_string)s" "https://www.youtube.com/@newhazzan/videos"
```

This is far more reliable than the manual `ytInitialData` scraping approach
tried earlier in the project — use `yt-dlp` exclusively, don't reinvent the
pagination logic.

## Step 2 — Get the current site state

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const m = html.match(/const playlists = (\[[\s\S]*?\n\]);/);
const playlists = eval(m[1]);
const ids = new Set();
playlists.forEach(p => p.songs.forEach(s => ids.add(s.videoId)));
console.log(ids.size);
"
```

Diff: any channel video ID not in this set is a candidate.

## Step 3 — Exclude guest reciters and other family members

**Never attribute a guest/family recording to Chaim Salman.** Apply these
rules in order (they were derived by actually listening for named
individuals across ~1,700 videos, not guessed):

**Known guest names** (any of these appearing in a title = guest, route to
that guest's existing `guest-<slug>` playlist if titles match one already on
the site, otherwise flag as a possible new guest performer for the user to
name/decide):
```
רבי בניהו, דוד נאור, הרב אצלאן, מנשה זהבי, סלימה מראד, מרייד, מורד סלמן,
חוגי עבודי, יחזקאל סעאת, דוד שמואלי, יחזקאל יאיר, שושנה אליהו, עובדיה יוסף,
רבנו עובדיה, חכם דוד חלבי, יהודה עובדיה פתיא, חכם יצחק מחלב, שמעון דדו הלוי,
הרב אורן אליהו, נסים סלמן, הרב אייל סלמן, חכם יוסף חיים מזרחי,
החזן שלמה ראובן מעלם
```

**Generic guest pattern** (catches new/unlisted guests): a title matching
`(חכם|הרב|רבי|חזן|הרה"ג)\s+[א-ת]` is a guest **unless** "חיים סלמן" or
"משפחת סלמן" also appears in the same title (that's a false-positive risk —
verify with `yt-dlp --print "%(uploader)s"` on the specific video if
ambiguous; do not guess).

**Known other-family-member names** (not the father — route to
`family-music`, or flag if it's a name not seen before):
```
ישראל סלמן, נדב סלמן, יונתן סלמן, טל סלמן, נתנאל סלמן, נועם סלמן
```
Also: `(מנגן|נגינה על)` in a title is family/instrumental unless
"חיים סלמן" also appears in the title.

Anything not caught by the above is presumed to be the father's own
recording — this matches the heuristic already validated with the user
earlier in the project (no other name credited = his own, given it's his
personal/family channel).

## Step 4 — Categorize what's left (father's own content)

Route by keyword into the existing category structure — check
`PLAYLIST_CATEGORIES` in `index.html`'s script for the current full list of
playlist IDs and what they contain before deciding a video is "new subject."
Prefer adding to an existing playlist over creating a new one; only propose
a genuinely new playlist if the content clearly doesn't fit anything current
(e.g. how Chanukah, Purim, and Haftarot each became their own playlist during
the original build).

## Step 5 — Present findings before touching anything

Report to the user: how many new videos found, broken down by
guest/family/father's-own, and for the father's-own ones, which existing
playlist(s) they'd go into (or propose new ones). Wait for explicit
confirmation before editing `index.html`.

## Step 6 — Add and validate

Once confirmed, add the new songs into the right playlist(s) in the
`playlists` array. **Always validate before considering it done:**

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const m = html.match(/const playlists = (\[[\s\S]*?\n\]);/);
const playlists = eval(m[1]);
let total = 0;
const seen = new Set();
playlists.forEach(p => p.songs.forEach(s => {
  if (seen.has(s.videoId)) console.log('DUPLICATE:', s.videoId, p.id);
  seen.add(s.videoId);
  total++;
}));
console.log('playlists:', playlists.length, 'songs:', total);
"
```

Watch specifically for accidentally dropping an entry while hand-editing a
large array — this happened once already in this project (Pekudei got
silently lost during a merge and was only caught later by cross-referencing
a second data source).

## Step 7 — Deploy

Use this project's `/deploy` skill (or its documented procedure) to commit,
push, and verify the change is actually live — don't just push and assume.
