# Higgsfield Skill

A Claude Code skill that keeps Claude (and Claude-powered agents) on the rails when generating marketing images, UGC videos, and DTC ads through the [Higgsfield](https://higgsfield.ai) MCP server.

Without this skill, Claude tends to:

- Pick the wrong model (e.g. `nano_banana_2` when `marketing_studio_image` is the right call)
- Skip the required `models_explore` / `show_marketing_studio` setup calls
- Invent non-existent parameters (`product_url`, `negative_prompt`, `input_type`, `voiceover`, `image_url`, `batch_size`, etc.) that Higgsfield rejects at runtime
- Confuse `product` vs `webproduct` for URL-driven ads
- Auto-train a Soul Character on a one-off "make me an avatar" request
- Fumble the local-file upload flow (`media_upload` → PUT → `media_confirm`)

With the skill installed, decision quality on a 6-prompt benchmark jumped from **55% → 100%** (real schema correctness, no fabricated params), at a cost of ~17% more input tokens per invocation.

## What it covers

- **Three-mode routing**: free-form generation, DTC Ads via Marketing Studio, reusable Soul Characters
- **Model cheatsheet** for both images and videos with the marketing-team's actual defaults
- **Marketing Studio flow** (URL/product → `show_marketing_studio` fetch → `generate_video` with `marketing_studio_video`)
- **Reference-media handling** (the upload → confirm → `medias[]` pattern, role names like `start_image`)
- **Soul Character discipline** (when to train, when to offer the choice, when to one-off)
- **Workspace and credit awareness**
- **A gotchas list** for the most common foot-guns

## Prerequisites

1. **Claude Code** installed and running
2. **Higgsfield MCP server** connected to your Claude Code instance — the skill assumes tools like `generate_image`, `generate_video`, `show_marketing_studio`, `show_characters`, `models_explore`, `media_upload`, `media_confirm`, etc. are available

## Install

Drop the skill into your Claude Code skills directory:

```bash
git clone https://github.com/william-zou21/higgsfield-skill.git ~/.claude/skills/higgsfield
```

Or if you want to keep the repo somewhere else and symlink it:

```bash
git clone https://github.com/william-zou21/higgsfield-skill.git ~/code/higgsfield-skill
ln -s ~/code/higgsfield-skill ~/.claude/skills/higgsfield
```

That's it — restart Claude Code and the skill will appear in the available-skills list. Claude will auto-invoke it whenever you say things like "create an image", "make a UGC video", "take inspiration from this URL", or anything else the description in `SKILL.md` triggers on.

## Files

- [`SKILL.md`](./SKILL.md) — the skill itself (frontmatter + body). This is the whole thing.
- [`LICENSE`](./LICENSE) — MIT

## How to tune it for your team

Edit `SKILL.md` directly:

- The **cheatsheet** at the top is opinionated — swap the defaults to match your team's preferred models.
- The **common workflows** section maps the phrases your team actually uses ("create image", "create UGC video", "take inspiration from this URL") to specific tool calls. Add your own.
- The **gotchas list** at the bottom is the most-edited section in practice — add anything you learn from real failures.

The frontmatter `description` field is what determines whether Claude invokes the skill. If you find it under-triggering, make the description more "pushy" (list more example phrases). If it over-triggers on unrelated content gen requests, make it more specific.

## License

MIT — see [LICENSE](./LICENSE).
