---
name: github-upload-media-to-pr
description: >-
  Upload local images and videos to a GitHub PR and embed them in the description or comments.
  Use when asked to "attach screenshots to PR", "add images to PR", "upload test results to PR",
  "embed screenshots in PR description", "add before/after images to PR", "attach UI screenshots",
  "show test results in PR", "add visual evidence to PR", "attach video to PR",
  "upload recording to PR", "add demo video to PR", or any request involving media and PRs.
  Always use this skill when the user wants to visually document changes in a pull request,
  even if they don't use the word "upload" — phrases like "put the screenshot in the PR" or
  "show the recording in the PR" should trigger this skill.
  Supports images (png, jpg, gif, webp) and videos (mp4, webm, mov).
  Uses agent-browser (vercel-labs/agent-browser) for browser automation.
dependencies:
  - vercel-labs/agent-browser
allowed-tools: Bash(agent-browser:*), Bash(gh:*), Bash(npx:*), Bash(cp:*), Bash(file:*), Read, Glob, Write
---

# Upload Media to PR

Upload local images and videos to a GitHub PR and embed them in the description or comments using `agent-browser`.

This skill is opinionated: it uses **`agent-browser`** (via the `vercel-labs/agent-browser` skill) as its browser automation backend. No other browser tools are needed.

## Dependencies

This skill requires **`agent-browser`**. If not installed:

```bash
npx skills add vercel-labs/agent-browser -g -y
npm i -g agent-browser && agent-browser install
```

## How It Works

Since the GitHub API does not support direct media uploads, this skill uses the **PR comment textarea as a staging area for GitHub's media hosting** — uploading files there to obtain persistent `user-attachments/assets/` URLs, then updating the PR description or posting a comment via the `gh` CLI.

**Supported media types:**
- **Images:** png, jpg, jpeg, gif, webp
- **Videos:** mp4, webm, mov

## Step 0: Resolve PR context and validate media files

If the user didn't specify a PR number or URL, auto-detect it:

```bash
gh pr view --json number,url -q '"\(.number) \(.url)"'
```

If multiple repos or branches are involved, confirm with the user which PR to target.

Normalize the media paths to absolute paths. If a path contains special characters (e.g., Unicode narrow spaces from CleanShot X), copy the file to `/tmp/` first:

```bash
cp /path/to/CleanShot*keyword*.png /tmp/screenshot.png
```

Verify the file type:

```bash
file --mime-type /path/to/media
```

## Step 1: Verify agent-browser is available

```bash
agent-browser --version
```

If not found, install it: `npm i -g agent-browser && agent-browser install`

## Step 2: Navigate to PR page and check login state

Follow the `vercel-labs/agent-browser` skill's core workflow: **navigate → snapshot → interact → re-snapshot**.

```bash
# Navigate with persistent GitHub auth profile
agent-browser --headed --profile ~/.agent-browser-github open "https://github.com/{owner}/{repo}/pull/{number}"

# ALWAYS snapshot after navigating to get element refs
agent-browser snapshot -i
```

**If SSO authentication screen appears:** Look for a "Continue" button ref in the snapshot and click it.

**If NOT logged in:**
1. Navigate to `https://github.com/login`
2. Ask the user to log in manually in the headed browser window
3. Wait for user confirmation, then navigate back to the PR page
4. Re-snapshot to verify login

## Step 3: Locate the file upload area

Scroll to the bottom of the page to find the comment area and confirm the upload zone is present.

**How GitHub's upload zone works:** The visible "Paste, drop, or click to add files" button (`<file-attachment>` custom element) appears in snapshots and confirms you're in the right place. However, `agent-browser upload` only works on `<input type="file">` elements — and the actual file input (`#fc-new_comment_field`) is **hidden** inside the drop zone, so it won't appear in snapshot output.

**Workflow:** Use the snapshot to confirm the drop zone button is visible, then use the hidden input's CSS selector for the actual upload.

```bash
# Scroll to bottom to reveal the comment form
agent-browser scroll down 3000
agent-browser snapshot -i

# Look for: button "Paste, drop, or click to add files" [ref=e__]
# This confirms the upload zone is present and you're in the right place.

# Verify the hidden file input exists (it's inside the drop zone)
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify((() => {
  const fa = document.querySelector('file-attachment.js-upload-markdown-image');
  if (!fa) return { found: false, reason: 'no file-attachment element' };
  const input = fa.querySelector('input[type="file"]');
  if (!input) return { found: false, reason: 'no file input inside file-attachment' };
  return { found: true, id: input.id, selector: '#' + input.id };
})())
EVALEOF
```

The expected result is `{ found: true, id: "fc-new_comment_field", selector: "#fc-new_comment_field" }`. Use this CSS selector for uploads — **not** the drop zone button's `@e` ref (clicking it opens an OS file picker that can't be automated, and `upload` requires an `<input type="file">`).

## Step 4: Upload media files one by one

Upload each media file. Wait **5 seconds** after each image upload and **10 seconds** after each video upload for GitHub to process.

For multiple files, upload them all to the same comment textarea before extracting URLs.

```bash
# Upload image
agent-browser upload "#fc-new_comment_field" /absolute/path/to/media.png

# Wait for GitHub to process
agent-browser wait 5000

# Upload video and wait longer
agent-browser upload "#fc-new_comment_field" /absolute/path/to/recording.webm
agent-browser wait 10000
```

**Important:** Always use absolute file paths.

## Step 5: Retrieve uploaded media URLs

Read the textarea value. GitHub injects markdown into the textarea:
- **Images:** `![description](https://github.com/user-attachments/assets/...)`
- **Videos:** `https://github.com/user-attachments/assets/...` (plain URL)

```bash
agent-browser eval --stdin <<'EVALEOF'
(() => {
  const ta = document.getElementById('new_comment_field')
          || document.querySelector('textarea[id*="comment"]');
  return ta ? ta.value : 'textarea not found';
})()
EVALEOF
```

Extract all media URLs/markdown from the textarea value before clearing it.

## Step 6: Clear the textarea (do not submit the comment)

```bash
agent-browser eval --stdin <<'EVALEOF'
(() => {
  const ta = document.getElementById('new_comment_field')
          || document.querySelector('textarea[id*="comment"]');
  if (ta) { ta.value = ""; return "cleared"; }
  return "textarea not found";
})()
EVALEOF
```

## Step 7: Embed media in the PR

**Option A — Update PR description** (default):

```bash
EXISTING_BODY=$(gh pr view {PR_NUMBER} --json body -q .body)

# For images:
gh pr edit {PR_NUMBER} --body "$(printf '%s\n\n## Screenshots\n\n%s' "$EXISTING_BODY" "![screenshot](https://github.com/user-attachments/assets/...)")"

# For videos (plain URL renders as inline player on GitHub):
gh pr edit {PR_NUMBER} --body "$(printf '%s\n\n## Demo\n\n%s' "$EXISTING_BODY" "https://github.com/user-attachments/assets/...")"

# For mixed media, use "## Media" as the section header
```

**Option B — Post as a new comment**:

```bash
gh pr comment {PR_NUMBER} --body "## Media

![screenshot](https://github.com/user-attachments/assets/...)

https://github.com/user-attachments/assets/...  (video)"
```

Use Option A by default unless the user explicitly asks for a comment, or if the PR description is already long.

## Step 8: Verify the result

Reload the page and take a screenshot to confirm the media is displayed correctly:

```bash
agent-browser open "https://github.com/{owner}/{repo}/pull/{number}"
agent-browser wait --load networkidle
agent-browser screenshot ./pr-verification.png
```

## Tips

- **Image sizing**: Control display size via HTML `<img>` tags: `<img width="800" alt="description" src="..." />`
- **Video sizing**: Videos render inline on GitHub — no special sizing needed
- **Multiple files**: Upload all media in one session to the same textarea; extract all URLs before clearing
- **Login persistence**: Use `--profile ~/.agent-browser-github` to persist GitHub login across sessions
- **Recording demos**: Use `agent-browser record start ./demo.webm` to record your automation workflow (see `vercel-labs/agent-browser` video recording reference)
- **Snapshots are essential**: Always run `agent-browser snapshot -i` after navigating or clicking — refs are invalidated on page changes
- **Command chaining**: Chain independent commands with `&&` for efficiency: `agent-browser open URL && agent-browser wait --load networkidle && agent-browser snapshot -i`

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Not logged in | Use `--headed` flag, navigate to login page, ask user to log in manually. For SSO, take snapshot and click "Continue" button |
| Browser window not visible | Ensure `--headed` flag is used |
| File path with special characters | Copy file to `/tmp/` with a simple name: `cp /path/CleanShot*keyword*.png /tmp/screenshot.png` |
| File upload fails | Ensure the file path is absolute |
| Textarea doesn't contain URLs yet | Wait 5s (images) or 10s (videos) after upload; retry once if needed |
| Textarea selector not found | GitHub UI changes occasionally — use the JS eval in Step 3 to find the current element |
| Video upload seems stuck | Large videos take longer — wait up to 15s; GitHub limits most formats to 10MB |
| agent-browser not found | `npm i -g agent-browser && agent-browser install` |
| File input not in snapshot | The upload input is hidden inside the `<file-attachment>` drop zone — use CSS selector `"#fc-new_comment_field"` instead of `@e` refs. The visible "Paste, drop, or click" button confirms the zone exists but can't be used for `upload` directly |
| Ref not found error | Re-snapshot with `agent-browser snapshot -i` — refs invalidate on page changes |
| PR not found / 404 | Private repos return 404 for unauthenticated users — check login state |

## Notes

- GitHub `user-attachments/assets/` URLs are **persistent** — media remains accessible even without submitting the comment
- Editing the description directly in the browser UI is fragile — updating via `gh pr edit` is strongly preferred
- Multiple media files can be uploaded in a single session before extracting URLs
- Videos on GitHub render as inline players when embedded as plain URLs in markdown
- This skill relies on the `vercel-labs/agent-browser` skill for browser automation patterns and best practices
