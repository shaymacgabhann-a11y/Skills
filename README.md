# ScalePad Skills

Shareable AI skills that automate tasks and workflows in ScalePad Lifecycle Manager. Each skill is a folder with a `SKILL.md` file: plain instructions that an AI client loads when a request matches. Skills work with any client that supports the Agent Skills format, including Claude (Claude.ai, Claude Desktop, Claude Code) and OpenAI Codex.

## Skills

| Skill | What it does | Writes to Lifecycle Manager? |
| --- | --- | --- |
| [scalepad-assessment-evaluator](skills/scalepad-assessment-evaluator/SKILL.md) | Answers Lifecycle Manager assessment questions (for example, the Technology Alignment Assessment) from live ScalePad evidence, and leaves an internal comment explaining every answer. | Yes: selected criteria and internal comments |
| [lm-microsoft-license-renewals](skills/lm-microsoft-license-renewals/SKILL.md) | Lists a client's Microsoft licenses with seat counts and expiration dates, flags renewals, trials and unassigned seats, and writes a customer-facing renewal summary. | No: read-only |

## Before you start

These skills read and write your ScalePad data, so your AI client needs a connection to ScalePad first.

1. Connect the ScalePad MCP server in your AI client, using an API key from your ScalePad account.
2. Check that the client can see your Lifecycle Manager data, for example by asking it to list your clients.
3. For skills that write data, try them first on a test or demo client and review the results before you use them on real client records.

## Install

### Claude Code

Add this repo as a plugin marketplace, then install the skills:

```bash
claude plugin marketplace add shaymacgabhann-a11y/Skills
```

```bash
claude plugin install lifecycle-manager-skills@scalepad-skills
```

Or copy a single skill folder into `~/.claude/skills/` (just for you) or `.claude/skills/` in a project (shared with that project).

### Claude.ai and Claude Desktop

1. Download the skill folder (for example `skills/scalepad-assessment-evaluator`) and zip it, keeping `SKILL.md` at the top of the folder.
2. In Claude, open **Settings > Capabilities > Skills** and upload the zip.
3. On Team and Enterprise plans, an admin can upload the skill once for the whole organization.

### OpenAI Codex

Copy the skill folder into your Codex skills directory, `~/.codex/skills/` for personal use or `.agents/skills/` in a repo.

### Other AI clients

If your client doesn't support skills natively, you can still paste the contents of `SKILL.md` into a project's custom instructions or system prompt. The client will need its own connection to the ScalePad MCP server.

## Repo layout

```
.claude-plugin/
  marketplace.json      Claude Code plugin marketplace listing
skills/
  <skill-name>/
    SKILL.md            The skill: name, trigger description, instructions
    references/         Optional: longer reference material the skill loads when needed
    scripts/            Optional: helper scripts the skill can run
```

## Adding a skill

1. Create `skills/<skill-name>/SKILL.md`. Use lowercase, hyphenated names that start with the product, for example `lm-qbr-prep`.
2. Start the file with frontmatter containing `name` (matching the folder name) and `description`. The description is what the AI uses to decide when to load the skill, so say what it does and the phrases that should trigger it.
3. Write the instructions client-neutral: refer to "the ScalePad MCP tools" rather than tool names specific to one AI client.
4. If the skill writes data, make it confirm the target client and tenant before writing, and report exactly what it changed.
5. Add the skill to the table above and to `skills` in `.claude-plugin/marketplace.json`.
