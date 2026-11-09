# Power Apps Code App — Claude Code Skill

A **Claude Code skill** capturing production-proven patterns for building [Power Apps Code Apps](https://learn.microsoft.com/en-us/power-apps/developer/code-apps/overview) — React + TypeScript + Vite SPAs that connect to Microsoft Dataverse.

Every section in `SKILL.md` is a real fix discovered through building a production audit-restore app. Using this skill prevents you from hitting the same obscure SDK bugs twice.

## What this skill covers

| Section | Topic |
|---|---|
| §1 | What a Code App is (auth, connector, deploy) |
| §2 | Project scaffold & file layout |
| §3 | `WithOrganization` variants — the #1 error source |
| §4 | SDK body-wrapping bug — `"item"` vs `"body"` param name |
| §5 | Boolean header `undefined` → connector crash fix |
| §6 | `EntityDefinitions` — `$top` not supported |
| §7 | GET caching — cache-buster timestamp pattern |
| §8 | Lookup fields — 3 separate concepts (read / write / OData nav) |
| §9 | `@odata.bind` in POST rejected — two-step create+PATCH |
| §10 | Audit table patterns + `changedata` JSON parsing |
| §11 | Field name display helper (strip publisher prefix) |
| §12 | `IOperationResult` pattern |
| §13 | Deployment checklist |
| §14 | Common error → fix reference table |
| §15 | Confirmed connector limitations |

## Installation

Copy `SKILL.md` to your Claude Code commands folder:

```bash
# Mac / Linux
cp SKILL.md ~/.claude/commands/powerapp-code-app.md

# Windows
copy SKILL.md %USERPROFILE%\.claude\commands\powerapp-code-app.md
```

Then invoke it in any Code App project:

```
/powerapp-code-app
```

Claude will load all 15 sections as context before writing any code.

## Built from

This skill was extracted from building the **Audit Restore** app — a Power Apps Code App that reads Dataverse audit history and lets users restore previous field values or recreate deleted records, with full lookup re-linking via OData navigation property resolution.

## Related

- [powerplatform-architect-skill](https://github.com/powerdynamite/powerplatform-architect-skill) — senior Power Platform / Dataverse solution architect skill

---

## Built by KaizenAI

Created by Abdul Basit, founder of **[KaizenAI](https://kaizenai.systems/)**, an AI-first Microsoft consultancy for AI agents, Copilot, Dynamics 365 and the Power Platform, delivered as Results as a Service at a fixed price.

Need hands-on help with a Power Platform build? [Talk to us](https://kaizenai.systems/contact/) or see our [fixed-price packages](https://kaizenai.systems/packages/).
