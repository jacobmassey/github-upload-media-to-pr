# GitHub Upload Media to PR

An AI agent skill that uploads local images and videos to a GitHub PR and embeds them in the description or comments — automatically, just by asking.

Forked from [tonkotsuboy/github-upload-image-to-pr](https://github.com/tonkotsuboy/github-upload-image-to-pr) with added video support. This skill is opinionated: it uses [`agent-browser`](https://github.com/vercel-labs/agent-browser) exclusively for browser automation.

## Installation

```bash
npx skills add jacobmassey/github-upload-media-to-pr
```

### Dependencies

This skill requires [`agent-browser`](https://github.com/vercel-labs/agent-browser):

```bash
npx skills add vercel-labs/agent-browser -g -y
npm i -g agent-browser && agent-browser install
```

## Usage

Trigger the skill with phrases like:

- "Attach this screenshot to the PR"
- "Add images to the PR description"
- "Upload test results to the PR"
- "Put this screenshot in the PR"
- "Embed before/after images in the PR"
- "Attach this video to the PR"
- "Upload the recording to the PR"
- "Add a demo video to the PR"

## Supported Media Types

- **Images:** png, jpg, jpeg, gif, webp
- **Videos:** mp4, webm, mov

## How It Works

### Why not just use the GitHub API?

GitHub does **not** provide a public REST API endpoint for uploading media attachments to embed in PR descriptions or comments. The official GitHub API allows creating/editing PR content as markdown text, but has no endpoint for uploading binary files like images or videos.

As a workaround, this skill uses **`agent-browser`** to upload media the same way a human would through the GitHub web UI.

### The mechanism

1. **Open the PR page** in a browser via `agent-browser`
2. **Locate the comment textarea** at the bottom of the PR conversation
3. **Upload the media file** using the hidden file input inside the drop zone — this triggers GitHub's internal upload pipeline and generates a persistent `https://github.com/user-attachments/assets/...` URL
4. **Extract the URL** from the textarea value before submitting anything
5. **Clear the textarea** (the media URL remains valid even without posting the comment)
6. **Update the PR description** via `gh pr edit`, embedding the media as markdown

This approach works because GitHub's media hosting is separate from comment submission — files are persisted the moment they're uploaded, regardless of whether the comment is ever posted.

## Requirements

- An AI agent that supports skills (e.g., [Codex](https://openai.com/codex/), [Claude Code](https://claude.ai/claude-code))
- [`agent-browser`](https://github.com/vercel-labs/agent-browser) — for browser automation
- [GitHub CLI (`gh`)](https://cli.github.com/) — for updating the PR description via API

## License

MIT

Copyright 2026 tonkotsuboy, Jacob Massey
