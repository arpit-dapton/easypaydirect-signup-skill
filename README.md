# EasyPayDirect Signup Skills

A ready-to-use AI agent skill for building the EasyPayDirect merchant signup form.

## Purpose

This skill is for **partners** who want to earn commission by referring merchants to EasyPayDirect. Give it to your AI agent and it builds a branded referral signup page that you can host anywhere (Vercel, Netlify, your own site, etc.). Every merchant who signs up through your page is attributed to you via your partner key, so the signups you drive earn you commission.

## Installation

**What you'll need**

- **A partner API key (optional)** — from the [EasyPayDirect partner portal](https://emap.easypaydirect.com) (see [Answer the Gate Questions](#answer-the-gate-questions) below). It's what attributes signups to you for commission, but you can start without it and add it later.
- **An AI coding assistant** (Cursor, Claude Code, ChatGPT, etc.) — this is what actually builds the page from the skill.
- **Node.js** — only for the `npx` install method below; not needed if you copy the skill manually. Get it at [nodejs.org](https://nodejs.org).

> Not comfortable with a terminal? You can skip the commands entirely — open your AI assistant, give it the skill (paste the link to `SKILL.md`), and ask it to build the page for you.

**1. Get the skill**

*Option A: `npx skills` (recommended)*

> **Before you run this, you need Node.js installed** — it's what provides the `npx` command. Download it from [nodejs.org](https://nodejs.org) (pick the "LTS" version and click through the installer), then reopen your terminal. To check it worked, run `node --version`; if it prints a version number you're set. If `npx` still isn't found after installing, close and reopen the terminal.

```bash
npx skills add arpit-dapton/easypaydirect-signup-skill
```

This is the [open agent skills CLI](https://skills.sh): it detects which coding agents you have installed (Cursor, Codex, Junie, and 70+ others), lets you pick which to install to, and symlinks or copies `skills/signup/` into the right place for each. Use `--list` first to preview what it finds without installing anything:

```bash
npx skills add arpit-dapton/easypaydirect-signup-skill --list
```

*Option B: download it (no terminal needed)*

1. Open the [GitHub repo](https://github.com/arpit-dapton/easypaydirect-signup-skill).
2. Click the green **Code** button → **Download ZIP**, then unzip it.
3. The skill is the `skills/signup` folder inside. Point your agent at it (next step), or drop that folder wherever your agent reads skills from.

Any folder your agent can read works. If you're not sure where that is for your agent, just use the [Answer the Gate Questions](#answer-the-gate-questions) step below — you can hand the agent the `SKILL.md` file directly.

**2. Point your agent at it**

```
Generate the signup form using this specification:
path/to/signup/SKILL.md
```

`SKILL.md` is the single entry point. It links out to every step file and reference doc it needs, so the agent never has to guess where the rest of the spec lives.

## Answer the Gate Questions

Before it writes any code, the skill stops and asks:

1. *Do you have a partner API key?* Determines whether `partner_key` gets sent in the signup payload (email variant only). You get the key from the [EasyPayDirect partner portal](https://emap.easypaydirect.com) under **Integration → API Integration** — one key per partner account, reused for every signup. Not a partner yet? [Register as a partner](https://emap.easypaydirect.com/signup/partner) first, then grab the key from that same page. (The skill walks you through this step by step and lets you **Skip** if you don't have one yet — signups just won't be attributed to you until you add it.)
2. *How do you want the sign-up process to work for merchants on your site?* Asked in plain language, no internal jargon. Either way, your site builds **only** the short Step 1 form — the merchant finishes on EasyPayDirect. The two options:

   1. **Quick start, then continue on EasyPayDirect**: the merchant enters their basic contact info on your website. As soon as they submit, they're taken straight to EasyPayDirect's own website to finish the rest of their application. *(Internally: Variant 1 — build only Step 1, make no API call, and redirect the browser to EasyPayDirect's hosted signup with the Step 1 values as query params.)*
   2. **Quick start, then we email you a link**: the merchant enters their basic contact info on your website, and instead of being redirected, they get an email with a link to continue on EasyPayDirect whenever they're ready. *(Internally: Variant 2 — build only Step 1, `POST /api/v1/signup`, then `POST /api/v1/signup/resume-link`, then show a "check your email" confirmation.)*

These aren't optional: they're one-way doors (a signup submitted without `partner_key` can never be attributed to a partner after the fact, and the flow choice determines how the merchant is handed off to EasyPayDirect), so the skill is written to refuse to proceed until a human actually answers. See the gate question and its full implementation table in [`skills/signup/SKILL.md`](skills/signup/SKILL.md).

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
| EasyPayDirect Merchant Signup | [`skills/signup/SKILL.md`](skills/signup/SKILL.md) |

---

**Last updated**: 2026-09-03
