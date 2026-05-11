# skills

Reusable [Claude Code](https://claude.com/claude-code) skills.

A skill is a markdown file with YAML frontmatter that Claude Code can invoke when its `description` matches what the user is asking for. Drop these into `~/.claude/skills/` (user-wide) or `<repo>/.claude/skills/` (project-scoped) to make them available.

## Skills in this repo

| Skill | What it does |
| --- | --- |
| [`video-export`](video-export/SKILL.md) | Download a video from a public web URL (YouTube, LinkedIn, X, TikTok, …) via yt-dlp, hand the file back. |
| [`video-analyze`](video-analyze/SKILL.md) | Transcribe + summarize the content of a web video. yt-dlp → faster-whisper → Claude. Optionally sample frames for visual demos. |

## Install

Single skill:

```bash
mkdir -p ~/.claude/skills
cp -r video-export ~/.claude/skills/
```

All skills:

```bash
git clone https://github.com/jeanclawd/skills.git ~/.claude/skills-repo
ln -s ~/.claude/skills-repo/video-export ~/.claude/skills/video-export
# repeat per skill, or symlink the whole dir if you want every skill
```

After installing, restart Claude Code (or run `/skill-list`) to pick up the new skill.

## Skill format

Each skill lives in its own directory with a `SKILL.md` file:

```
my-skill/
└── SKILL.md
```

`SKILL.md` starts with frontmatter:

```markdown
---
name: my-skill
description: One- or two-sentence description that tells Claude when to invoke this skill. Be specific about triggers.
---

# my-skill

Body — instructions Claude follows when the skill fires.
```

The `description` is the trigger. Make it specific (what phrases, what URLs, what file types) and explicit about when **not** to use the skill.
