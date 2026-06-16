# KEY ESG AI Skills

**Your next LP report starts with a prompt, not a spreadsheet.**

Pre-built AI workflows for ESG teams, powered by live, verified data from [KEY ESG](https://www.keyesg.com) via the [Model Context Protocol (MCP)](https://modelcontextprotocol.io). Each skill teaches your AI assistant a complete ESG workflow — which tools to call, in what order, what to verify, and how to present the result — so you get a finished deliverable instead of prompting step by step.

Every answer is grounded in the data your organisation has already collected and verified in KEY ESG. The connector is read-only: your AI assistant can query and summarise, never change, your records.

Skills follow the open [Agent Skills format](https://agentskills.io/specification) and work with Claude (claude.ai, Desktop, Code), ChatGPT, Cursor, Mistral, and any other assistant that supports skills.

## Quick install

If you use a coding agent (Claude Code, Cursor, Codex, and ~70 others):

```bash
# All KEY ESG skills
npx skills add key-esg/ai-skills

# A single skill
npx skills add key-esg/ai-skills --skill esg-questionnaire
```

For chat assistants (claude.ai, ChatGPT, Le Chat), see [Installing a skill](#installing-a-skill) below.

## Available skills

### For fund managers

| Skill | What it does | |
| --- | --- | --- |
| [ESG Questionnaire](skills/esg-questionnaire/SKILL.md) | Complete LP, vendor, or regulatory ESG questionnaires from your live portfolio data — with coverage stated per answer and low-confidence items flagged for human review. | [ZIP](https://github.com/key-esg/ai-skills/releases/latest/download/esg-questionnaire.zip) |
| [Portfolio Data Query](skills/portfolio-data-query/SKILL.md) | Ask plain-language questions about portfolio ESG performance — compare funds, years, and companies, and drill into a single portfolio company. | [ZIP](https://github.com/key-esg/ai-skills/releases/latest/download/portfolio-data-query.zip) |

### For companies (standalone or portfolio companies)

| Skill | What it does | |
| --- | --- | --- |
| [ESG Questionnaire](skills/esg-questionnaire/SKILL.md) | Complete customer, lender, or regulatory ESG questionnaires from your own organisation's live data. | [ZIP](https://github.com/key-esg/ai-skills/releases/latest/download/esg-questionnaire.zip) |
| [Carbon Data Audit](skills/carbon-data-audit/SKILL.md) | Audit the activity data behind your footprint — find sites and months with missing entries, outlier quantities, and possible duplicates before the numbers go into a report. | [ZIP](https://github.com/key-esg/ai-skills/releases/latest/download/carbon-data-audit.zip) |

### For both

| Skill | What it does | |
| --- | --- | --- |
| [ESG Reporting Brief](skills/esg-reporting-brief/SKILL.md) | Draft board packs, investor updates, or annual report sections — with a mandatory data-transparency section that flags every gap. | [ZIP](https://github.com/key-esg/ai-skills/releases/latest/download/esg-reporting-brief.zip) |
| [ESG Commitment Gap Analysis](skills/esg-commitment-gap-analysis/SKILL.md) | Surface off-track targets, commitments with no action plan behind them, and targets that can't be assessed because data is missing. | [ZIP](https://github.com/key-esg/ai-skills/releases/latest/download/esg-commitment-gap-analysis.zip) |
| [Metric Breakdown Analysis](skills/metric-breakdown-analysis/SKILL.md)     | Break a metric down over time (monthly/quarterly) or across entities, or portfolio companies, reconciling each split against the yearly total. |

The **ZIP** links are ready-to-upload skill packages for chat assistants — rebuilt automatically from [the latest release](https://github.com/key-esg/ai-skills/releases/latest) whenever a skill changes.

Each skill starts by calling `who_am_i`, so your assistant always knows which organisation it is acting for and which tools apply to your account type.

## Prerequisites

1. A KEY ESG account. Tool access follows your KEY ESG role — see the [tools reference](https://api.keyesg.com/docs/mcp/tools).
2. The KEY ESG MCP server connected to your AI assistant (sign in with your own KEY ESG account; OAuth, read-only). Setup guide: **[KEY ESG MCP Docs](https://api.keyesg.com/docs/mcp)**

## Installing a skill

### CLI (Claude Code, Cursor, and other coding agents)

```bash
npx skills add key-esg/ai-skills
```

The [`skills` CLI](https://www.skills.sh) detects your agent and installs into the right folder. Or copy manually:

```bash
# Claude Code — personal (all projects)
cp -r skills/esg-questionnaire ~/.claude/skills/esg-questionnaire

# Claude Code — project-local
cp -r skills/esg-questionnaire .claude/skills/esg-questionnaire
```

Once installed, your agent invokes the skill automatically when your prompt matches its description, or invoke it directly with `/esg-questionnaire`.

### claude.ai / Claude Desktop

1. Download the ready-made **ZIP** for your skill from the [Available skills](#available-skills) table above.
2. In Claude, go to **Customize → Skills → + Create Skill → Upload a skill**.
3. Upload the ZIP and save.

### ChatGPT

1. Connect the KEY ESG MCP server first — see the [ChatGPT setup guide](https://api.keyesg.com/docs/mcp/chatgpt).
2. Open your profile menu → **Skills**, then upload the skill ZIP from the table above (or paste `SKILL.md` into the skill editor).

### Other assistants

See the [skills installation guide](https://api.keyesg.com/docs/mcp/skills) for Cursor, Mistral Vibe, and Le Chat.

## Good to know

- **Read-only by design.** Skills query and summarise; they never create or edit records in KEY ESG.
- **No fabricated numbers.** Every skill instructs the assistant to flag missing data rather than estimate it, and to state coverage (e.g. "18 of 22 in-scope companies") alongside portfolio figures.
- **Role-aware.** If your KEY ESG role doesn't grant access to a tool, the skill says so instead of guessing.

## Support

- MCP setup guide: [api.keyesg.com/docs/mcp](https://api.keyesg.com/docs/mcp)
- Knowledge base: [intercom.help/key-esg](https://intercom.help/key-esg/en/)
- Email: [support@keyesg.com](mailto:support@keyesg.com)
