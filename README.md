# EnrichmentSkill

A set of LLM skills for Codex/Claude/Gemini focussed on enriching product information via OneSila.

The repo currently ships **two skills**, covering the two enrichment stages of the `NEW_PRODUCTS`
workflow: filling in a product's attributes, and writing its content.

| Skill | Does | Pulls from | Hands on to |
|---|---|---|---|
| [`fill-attributes`](skills/fill-attributes/SKILL.md) | Fills required **and** optional attribute values on a product's variations (or on the product itself if it isn't configurable), fixing duplicate variations first | `NEW_PRODUCTS_POPULATE_ATTRIBUTES` | `NEW_PRODUCTS_VERIFY_ATTRIBUTES` |
| [`write-content`](skills/write-content/SKILL.md) | Writes the default product content — name, subtitle, short description, description and bullet points — in every configured language | `NEW_PRODUCTS_WRITE_CONTENT` | `NEW_PRODUCTS_ASSIGN_TO_SALES_CHANNELS` |

They are meant to be run in that order: attributes are the checked fact base that the content
is then written from.

## Shared behaviour

Both skills work the same way, so what you learn from one carries to the other:

- **One product per run.** Each skill pulls a single product out of its source state, finishes it,
  and stops. It only loops if you explicitly ask for several.
- **Fully autonomous.** No confirmation prompts mid-run — a human reviews the result in the next
  workflow state instead.
- **Precision over coverage.** A correctly empty field is fine; a confidently wrong value is not,
  because it will slip through review. Neither skill guesses.
- **Every run leaves a note.** A plain-language comment on the product's collaboration board saying
  what was done, what was judged, and what was deliberately left — then the workflow advances.

Where they deliberately differ: on a configurable product, `fill-attributes` writes **only to the
variations** and treats the parent's properties as read-only, while `write-content` writes **only to
the parent**, because the parent is the listing customers read. Attributes belong to the variation;
content belongs to the listing.

## Requirements

Both skills expect a workflow with the code **`NEW_PRODUCTS`**, carrying these four states — matched
by **code**, not by name or ID:

| Skill | Source state | Target state |
|---|---|---|
| `fill-attributes` | `NEW_PRODUCTS_POPULATE_ATTRIBUTES` | `NEW_PRODUCTS_VERIFY_ATTRIBUTES` |
| `write-content` | `NEW_PRODUCTS_WRITE_CONTENT` | `NEW_PRODUCTS_ASSIGN_TO_SALES_CHANNELS` |

A skill finds its work by searching for a product in its source state, so a missing or differently
coded source state means it finds nothing and stops. If your instance uses different codes, edit the
Configuration table in the relevant `SKILL.md` — the skills confirm the codes with
`get_company_details(show_workflows=true)` at the start of a run, but they do not adapt on their own.

## Usage

Ask for either job in plain language — the skill descriptions match the usual phrasings:

```
populate attributes on the next product
fill attributes for the next product

write content for the next product
work the write content queue
```

## Installation

This is a Claude Code plugin. It bundles the OneSila MCP server via
[`mcp-config.json`](mcp-config.json), which needs a `ONESILA_MCP_TOKEN` in the environment:

```sh
export ONESILA_MCP_TOKEN=<your token>
```

## Configuration

Each skill has a **Configuration** table at the top of its `SKILL.md` holding the deployment-specific
values — workflow code, source and target state, and the MCP server name. The skills confirm these
against `get_company_details(show_workflows=true)` at the start of a run, but if you deploy against a
different catalogue you should edit those tables.

Two things to check before running against a different product mix:

- **The MCP server name.** Tool names in each skill's `allowed-tools` are prefixed with it
  (`mcp__onesila__get_product`). If OneSila is registered under another name in the host — a second
  instance or a demo tenant, e.g. `onesila_demo` — the prefixes match no live tool and the skill has
  nothing to call.
- **The house rules in `fill-attributes`.** The doubt defaults and deterministic rules are commercial
  policy, not OneSila behaviour, and are tuned for a costume/apparel catalogue. Country of origin in
  particular is a customs declaration, not a formatting preference.

## Known limits

SEO title, SEO description and meta keywords are **not exposed over MCP at all** — not readable, not
writable. A product leaving the content stage still has them empty, and only a human working in the
OneSila UI can fill them. `write-content` states this in its collaboration note on every run.

## Licence

MIT — see [LICENSE](LICENSE).
