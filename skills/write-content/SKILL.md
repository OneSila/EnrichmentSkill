---
name: write-content
description: "Write the default product content — name, subtitle, short description, description and bullet points — in every configured language for one OneSila product sitting in the write-content workflow state, then hand it on. Use whenever the user wants to write, generate, or complete product content / copy / descriptions — e.g. \"write content for the next product\", \"do a content pass\", \"work the write content queue\", \"write the descriptions on the next product\", \"grab a product from write content and finish it\". Fully autonomous: pulls one product, reads its attributes, existing copy, images and brand voice, writes the missing languages, leaves a human-readable note on the collaboration board, and advances the workflow. Processes exactly one product per run unless the user explicitly asks for more."
argument-hint: "[sku]"
allowed-tools:
  - mcp__onesila__get_company_details
  - mcp__onesila__search_products
  - mcp__onesila__get_product
  - mcp__onesila__search_properties
  - mcp__onesila__get_property
  - mcp__onesila__search_property_select_values
  - mcp__onesila__get_property_select_value
  - mcp__onesila__upsert_products
  - mcp__onesila__create_product_collaboration_entry
  - WebSearch
---

# Write content

A focused, autonomous workflow for writing the **default** product content on **one** product in a OneSila PIM, in every configured language. Pull a single product out of the source state, write what's missing, leave a short note, and move it on.

This runs end-to-end without stopping for confirmation, so the standard is the same as attribute filling: **never state a fact you cannot support**. Marketing copy invites invention — a plausible-sounding "water-resistant coating" or "ethically sourced cotton" that no source supports is a false product claim on a live storefront, and some of them are legally actionable. Write vividly about what is true; say nothing about what you don't know.

**Process exactly one product per run.** When you have moved that one product on, stop and report.

---

## Configuration

Confirm against `get_company_details(show_workflows=true, show_languages=true)` at the start of a run.

| Setting | Value |
|---|---|
| Workflow code | `NEW_PRODUCTS` |
| Source state (pull from) | `NEW_PRODUCTS_WRITE_CONTENT` |
| Target state (hand on to) | `NEW_PRODUCTS_ASSIGN_TO_SALES_CHANNELS` |
| MCP server name | `onesila` |

Languages come from `get_company_details(show_languages=true)` — never hard-code the list. Read `enabled_languages` and note which is `is_default`.

---

## Scope — what this skill writes

**Default content only.** Omit `sales_channel_id` when writing a translation. That produces the default content every channel falls back to. Channel-specific overrides are a later concern and not yours.

**These five fields**, per language:

| Field | Notes |
|---|---|
| `name` | The product title. |
| `subtitle` | Short supporting line. |
| `short_description` | Plain summary, a sentence or two. |
| `description` | **HTML** — see the formatting rule below. |
| `bullet_points` | Array of strings, one claim each. |

`url_key` is returned when you read a product but is **not writable** through `upsert_products`. Don't try to set it; slugs are managed elsewhere.

**On a configurable product, write the parent only.** The parent is the listing customers read; variations inherit it. Read the variations for evidence — they carry the sizes, colours and materials worth mentioning — but write content only to the parent.

> Note this is the **opposite** of the attribute-filling rule, where the parent's properties are strictly read-only and all writes go to the variations. Attributes belong to the variation; content belongs to the listing. Don't carry the other skill's habit across.

---

## Step 0 — Pull one product

```
search_products(workflow_state_code="NEW_PRODUCTS_WRITE_CONTENT", limit=1)
```

Nothing back means no work — say so and stop. Otherwise take the first SKU and load it in full:

```
get_product(sku=<sku>, show_translations=true, show_properties=true, show_brand_voice=true,
            show_images=true, show_variations=true, show_inspector=true)
```

- `show_translations` — the existing copy, one entry per language, each with a `sales_channel` (`null` = default). **This is what tells you which languages are already done.**
- `show_properties` — the attribute values, your fact base.
- `show_brand_voice` — the brand's writing directive, or `null`. See Step 2.
- `show_images` — look at them; they show colour, finish, styling and use context that no attribute captures.
- `show_variations` — the range (sizes, colours) worth mentioning in the parent listing.

For a configurable, also read a couple of variations with `get_product(sku=<variation>, show_properties=true)` so the range you describe is real.

---

## Step 1 — Work out which *fields* are missing

Build a grid: every configured language × the five fields, filled from the translations where `sales_channel` is `null`. **The unit of work is a field, not a language.** A language that already has a `name` and a `description` is not "done" — its empty `subtitle`, `short_description` and `bullet_points` are gaps like any other.

- **An empty field → fill it.** Whether or not other fields in that language are populated.
- **A field that already holds content → leave it alone.** Do not rewrite, retranslate, or "improve" it. Existing copy may be hand-tuned, legally reviewed, or channel-agreed, and silently replacing it is the one genuinely destructive thing this skill could do.

This keeps the non-destructive guarantee exactly — an empty field cannot be destroyed — while avoiding the trap of leaving a partly-populated language half-written. Judging gaps per language rather than per field is how you end up with three complete languages and two carrying only a name and a description.

When you fill some fields of a language that already has others, **match what is there**: same register, same terminology, same product framing. You are completing someone else's page, not starting a competing one.

**Read the existing content, and flag it if it's weak.** Thin, obviously machine-translated, truncated, or contradicting the product's own attributes — say so in your note, name the language and field, and say what's wrong. A human decides whether to replace it. Judgement, not a rewrite.

If every language has every field, write nothing, flag anything weak, and advance the product. That is a successful run.

---

## Step 2 — Establish the voice

If `brand_voice` is present, its `default_prompt` is a direct instruction from the brand owner — follow it. It governs tone, register and emphasis. It does **not** license claims the product evidence doesn't support: a luxury voice justifies writing warmly about materials, never inventing them.

If `brand_voice` is `null`, write in clear, concrete, unfussy commercial English (and the equivalent register per language). Describe the product; don't sell at the reader.

---

## Step 3 — Gather the facts

Three sources, with a clear precedence between them.

1. **The images** (`show_images`) — direct evidence of the physical product. Actually look at them; don't skim the filenames.
2. **The attribute values** (`show_properties`) — filled and reviewed in the previous workflow step. Treat them as the **checked facts**: material, dimensions, capacity, colour, pack quantity, compliance.
3. **The existing copy** in any language — supplier or import text. Good for substance, phrasing and detail the attributes don't carry (use cases, feel, construction detail).

**Attributes beat existing copy.** If the imported English says "genuine leather" and the Material attribute says polyester, the attribute is the checked value; write polyester and flag the contradiction in your note. Never split the difference or quietly drop the conflict.

**But an attribute the images plainly contradict is not a fact — it's a defect.** Attributes are entered by people and imports, and they are wrong often enough to matter. When the images clearly disagree with an attribute, **write neither claim, and flag it.** Do not "resolve" it by trusting the attribute, and do not describe what you see as though it were recorded data.

> Seen in practice: a daytrip backpack whose `features_bags` said *Wheeled* while both images showed padded shoulder straps and no wheels, trolley sleeve or pull handle; and a `color` of *Black* on a bag whose entire front face photographs pink and grey. Following "attributes win" literally would have put a wheeled-luggage claim and a wrong headline colour on a live listing. The right move was to say nothing about wheels or colour and flag both.

This is deliberately narrower than second-guessing the data. It applies when the images are **unambiguous** — a countable feature that is present or absent, a colour that dominates the product. A judgement call about shade or finish is not a contradiction; leave those to the attribute.

Also usable: any attached documents, and the product type and brand. `WebSearch` only to confirm a factual spec on an identifiable brand + model — to verify, never to invent.

**Anything you cannot support from those sources does not go in the copy.** No invented certifications, care claims, origin, warranty, sustainability or performance language. If the product is thin on facts, write shorter — a short honest description beats a padded one.

---

## Step 4 — Write each missing language natively

Write each language **from the product evidence directly**, not by translating the English. The point is copy that reads as though written for that market: local idiom, natural search terms, the register a buyer there expects. A literal translation of an English title is usually a worse title.

Native does not mean divergent. Every language must state the **same facts** — same material, same dimensions, same pack quantity. Vary the phrasing, never the substance.

Practical rules:

- **`description` and `short_description` are both HTML.** Match what the catalogue already uses: `<p>` for prose, `<strong>` for lead-ins, `<ul>`/`<ol>` for feature lists, `<h3>` for section headings. No inline styles, no scripts. If a language already has content, mirror its markup shape — but do not reproduce editor artifacts you find there (`<span class="ql-ui">`, `data-list` attributes, stray `__markdown__` tokens); write clean markup.
- **`bullet_points` is an array of plain strings** — one claim per entry, no HTML, no leading dashes or bullet characters. Keep them scannable.
- **Names should be specific and stable** — product, key distinguishing feature, brand where it belongs. Don't stuff keywords, don't bury it in adjectives.
- **Don't invent a unit convention.** Use the units already on the product's attributes, localised in the normal way for the language.
- **Don't write variation-specific copy on a configurable parent.** Describe the range ("available in four sizes"), not one child's size.

---

## Step 5 — Write it

One `upsert_products` call per product, with all missing languages in the `translations` array:

```
upsert_products(products=[{
  sku: "<sku>",
  translations: [
    { language: "de", name: "...", subtitle: "...", short_description: "...",
      description: "<p>...</p>", bullet_points: ["...", "..."] },
    { language: "es", name: "...", subtitle: "...", short_description: "...",
      description: "<p>...</p>", bullet_points: ["...", "..."] }
  ]
}])
```

- **Omit `sales_channel_id`** — that is what makes it default content.
- **Include only the fields you are filling.** A translation entry may carry just the one or two fields that were empty — you do not have to send a whole language. Never echo back a field that already had content; there is no reason to rewrite it and every reason not to.
- One invalid item fails the whole call. If you are writing several languages, **write one first and re-read it** to confirm the shape landed — especially the HTML and the bullet-point array — then send the rest.

---

## Step 6 — Verify

```
get_product(sku=<sku>, show_translations=true)
```

Confirm each language you wrote is present with `sales_channel: null`, that the HTML came back intact rather than escaped, and that you did not disturb a language you meant to leave alone.

**Reads are sparse: empty fields are omitted entirely.** A field missing from the response means "not set", not "failed" — and `subtitle` has never been observed coming back at all. So absence is not proof your write failed. Verify what you can see, and don't retry a write just because a field didn't appear.

---

## Step 7 — Leave a note

```
create_product_collaboration_entry(sku=<sku>, comment="<note>")
```

**Say what you did, not what you wrote.** The note is a change summary for the next person — never paste the copy into it. The content is already on the product; repeating it makes the board unreadable.

Cover: which languages you wrote and which you left; whether a brand voice applied; any attribute-vs-existing-copy conflict and how you resolved it; anything you deliberately left vague for lack of evidence; and any existing content you think needs a human look, with the reason.

> Wrote de, es and nl from the attributes and the English copy; left en and fr alone as they already had content. No brand voice configured, so standard commercial register. The English description says "genuine leather" but the Material attribute is polyester — I wrote polyester in all three new languages, please check which is right. Nothing in the listing supports a care instruction so I left that out of the copy entirely. Ready for channel assignment.

---

## Step 8 — Advance

```
upsert_products(products=[{
  sku: "<sku>",
  workflows: [{ workflow_code: "NEW_PRODUCTS", state_code: "NEW_PRODUCTS_ASSIGN_TO_SALES_CHANNELS" }]
}])
```

Then stop and give a one or two sentence summary: the SKU, which languages you wrote, and that it has moved on. Do not pick up another product unless asked.

---

## Tool reference

| Task | Tool |
|---|---|
| Find a product in the source state | `search_products` (with `workflow_state_code`) |
| Confirm workflow codes and enabled languages | `get_company_details` (`show_workflows`, `show_languages`) |
| Load content, attributes, brand voice, images, variations | `get_product` |
| Resolve a property by `internal_name` | `search_properties` |
| Read a property's values | `get_property`, `search_property_select_values`, `get_property_select_value` |
| Write translations / move workflow | `upsert_products` |
| Post the collaboration-board note | `create_product_collaboration_entry` |
| Verify a factual spec (sparingly) | `WebSearch` |

## Guardrails recap

- One product per run. Stop after advancing it.
- Never invent a product claim. Unsupported facts stay out of the copy, however good they'd sound.
- Fill every empty field in every configured language. Gaps are per field, not per language — a language with a name and a description still needs its subtitle, short description and bullet points.
- Never overwrite a field that already holds content — flag it instead.
- Default content only: omit `sales_channel_id`.
- Attributes beat existing copy on any factual conflict, and the conflict goes in the note.
- But an attribute the images plainly contradict is a defect, not a fact: state neither, flag both.
- Brand voice governs tone, never facts. No voice configured → plain commercial register.
- Author each language natively; same facts everywhere, phrasing free.
- `description` is HTML; `bullet_points` is an array of plain strings; `url_key` is not writable.
- Configurables: parent only — the reverse of the attribute-filling rule.
- The note says what you changed, never the copy itself.
- Reads omit empty fields; a missing field is "not set", not a failed write.
- Always leave a readable note, then advance the workflow.
