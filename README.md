# EMAP Signup Skills

A ready-to-use AI agent skill for building the EMAP merchant signup form.

## Installation

**1. Get the skill**

*Option A: `npx skills` (recommended)*

```bash
npx skills add arpit-dapton/easypaydirect-signup-skill
```

This is the [open agent skills CLI](https://skills.sh): it detects which coding agents you have installed (Cursor, Codex, Junie, and 70+ others), lets you pick which to install to, and symlinks or copies `skills/signup/` into the right place for each. Use `--list` first to preview what it finds without installing anything:

```bash
npx skills add arpit-dapton/easypaydirect-signup-skill --list
```

*Option B: copy it manually*

```bash
git clone https://github.com/arpit-dapton/easypaydirect-signup-skill.git easypaydirect-signup-skill
cp -r easypaydirect-signup-skill/skills/signup <your-agent's-skills-dir>/signup
```

(Where that is depends on your agent: see the CLI's [Supported Agents](https://github.com/vercel-labs/skills#supported-agents) table for the exact path. Any folder your agent can read works, even outside that convention.)

**2. Point your agent at it**

```
Generate the signup form using this specification:
path/to/signup/SKILL.md
```

`SKILL.md` is the single entry point. It links out to every step file and reference doc it needs, so the agent never has to guess where the rest of the spec lives.

## Answer the Gate Questions

Before it writes any code, the skill stops and asks:

1. *Do you have a partner API key?* Determines whether `partner_key` gets sent in the signup payload (email variant only).
2. *How do you want the sign-up process to work for merchants on your site?* Asked in plain language, no internal jargon. Either way, your site builds **only** the short Step 1 form — the merchant finishes on EMAP. The two options:

   1. **Quick start, then continue on EMAP**: the merchant enters their basic contact info on your website. As soon as they submit, they're taken straight to EMAP's own website to finish the rest of their application. *(Internally: Variant 1 — build only Step 1, make no API call, and redirect the browser to EMAP's hosted signup with the Step 1 values as query params.)*
   2. **Quick start, then we email you a link**: the merchant enters their basic contact info on your website, and instead of being redirected, they get an email with a link to continue on EMAP whenever they're ready. *(Internally: Variant 2 — build only Step 1, `POST /api/v1/signup`, then `POST /api/v1/signup/resume-link`, then show a "check your email" confirmation.)*

These aren't optional: they're one-way doors (a signup submitted without `partner_key` can never be attributed to a partner after the fact, and the flow choice determines how the merchant is handed off to EMAP), so the skill is written to refuse to proceed until a human actually answers. See the gate question and its full implementation table in [`skills/signup/SKILL.md`](skills/signup/SKILL.md).

## Folder Structure

```
easypaydirect-signup-skill/
├── README.md ...................................... This file
│
└── skills/
    └── signup/ ..................................... The skill
        ├── SKILL.md ................................ ⭐ Entry point: navigation hub & spec
        │
        ├── steps/ .................................. Step 1 field + conditional docs
        │
        └── reference/ .............................. Supporting docs
            ├── api-examples.md
            ├── DROPDOWNS_REFERENCE.md
            └── BACKEND_DEPENDENCIES.md
```

This is the flat layout the `npx skills` CLI expects (`skills/<name>/SKILL.md`), so any skill placed under `skills/` here is installable with `npx skills add` out of the box.

## Current Skills

| Skill | Entry point |
|-------|-------------|
| EMAP Merchant Signup | [`skills/signup/SKILL.md`](skills/signup/SKILL.md) |

---

**Last updated**: 2026-09-03
