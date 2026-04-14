# KEY ESG AI Skills

Pre-built instruction sets for Claude that connect to your KEY ESG data via the [Model Context Protocol (MCP)](https://modelcontextprotocol.io). Each skill guides Claude through a specific ESG workflow — which tools to call, in what order, and how to present the results.

Skills follow the [Agent Skills Specification](https://agentskills.io/specification) and work with Claude Code, claude.ai, and Claude Desktop.

## Prerequisites

1. A KEY ESG account with Company Admin or Fund Manager Admin role.
2. The KEY ESG MCP server connected to Claude. Follow the setup guide: **[KEY ESG MCP Docs](https://api.keyesg.com/docs/mcp)**

## Available skills

| Skill | Description |
| --- | --- |
| [ESG Questionnaire](skills/esg-questionnaire/SKILL.md) | Answer LP, vendor, or regulatory ESG questionnaires using live KEY ESG data. |
| [Portfolio Data Query](skills/portfolio-data-query/SKILL.md) | Ask natural-language questions about portfolio ESG performance across funds and years. |
| [ESG Commitment Gap Analysis](skills/esg-commitment-gap-analysis/SKILL.md) | Identify off-track targets, unmitigated commitments, and missing action plan coverage. |
| [ESG Reporting Brief](skills/esg-reporting-brief/SKILL.md) | Draft board packs, investor updates, or annual report sections from your ESG data. |

## Installing a skill

### Claude Code

Copy the skill directory into your personal or project skills folder. Claude Code will discover it automatically.

```bash
# Personal (available across all projects):
cp -r skills/esg-questionnaire ~/.claude/skills/esg-questionnaire

# Project-local:
cp -r skills/esg-questionnaire .claude/skills/esg-questionnaire
```

Once installed, Claude invokes the skill automatically when your prompt matches its description, or you can invoke it directly with `/esg-questionnaire`.

See the [Claude Code skills docs](https://code.claude.com/docs/en/skills) for more detail.

### claude.ai / Claude Desktop

1. Download the `SKILL.md` file from the skill directory you want to use.
2. In Claude, go to **Customize → Skills → + Create Skill → Upload a skill**.
3. Select the `SKILL.md` file and save.

Claude will automatically invoke the skill when your prompt matches its description.

## Support

- MCP setup guide: [api.keyesg.com/docs/mcp](https://api.keyesg.com/docs/mcp)
- Knowledge base: [intercom.help/key-esg](https://intercom.help/key-esg/en/)
- Email: [support@keyesg.com](mailto:support@keyesg.com)
