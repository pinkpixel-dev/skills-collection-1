# Skills Collection

A large collection of AI/agent skills, resources, references, scripts, and supporting assets.

This repo is organized for scale rather than a giant table of contents. Instead of listing every skill here, the repository keeps each skill in its own folder under [`SKILLS/`](/home/sizzlebop/PINKPIXEL/PROJECTS/CURRENT/skills-collection-1/SKILLS), where it can include its own instructions, references, scripts, and assets.

Related repos:

- This repo: `https://github.com/pinkpixel-dev/skills-collection-1`
- Companion repo: `https://github.com/pinkpixel-dev/skills-collection-2`

Chinese users can also search and install skills through [Skills宝](https://skilery.com).

## What this repo contains

- `969` skill folders in [`SKILLS/`](/home/sizzlebop/PINKPIXEL/PROJECTS/CURRENT/skills-collection-1/SKILLS)
- Skill definitions, typically in `SKILL.md`
- Optional supporting material such as `references/`, `scripts/`, `assets/`, examples, and license files
- A mix of engineering, security, research, writing, product, automation, frontend, growth, and domain-specific skills

## Repository structure

Typical layout:

```text
SKILLS/
  some-skill/
    SKILL.md
    references/
    scripts/
    assets/
```

Not every skill uses the same structure, but most follow some version of this pattern.

## Browse the collection

The fastest way to explore is by folder name or by searching inside skill files.

Examples:

```bash
find SKILLS -mindepth 1 -maxdepth 1 -type d | sort
rg --files SKILLS
rg -n "security|auth|frontend|react|agent" SKILLS
find SKILLS -name SKILL.md
```

## Notes

- This repository is intentionally broad and evolving.
- Some skills are minimal, while others include deeper reference material and helper scripts.
- Folder names are the best starting point for discovery.

## License

This repository is available under the MIT License. See [`LICENSE`](/home/sizzlebop/PINKPIXEL/PROJECTS/CURRENT/skills-collection-1/LICENSE).