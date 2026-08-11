---
name: fill-attributes
description: "Fill out product attribute values for one OneSila product sitting in the populate-attributes workflow state, then hand it on for verification. Use whenever the user wants to enrich, complete, or fill in attributes / properties — e.g. \"populate attributes on the next product\", \"fill attributes for the next product\", \"work the new products queue\", \"do an attribute-filling pass\", \"complete the properties on the next product\", \"grab a product from populate attributes and finish it\". Fully autonomous: pulls one product, fixes duplicate variations if needed, fills required and optional attribute values on the variations (or the product itself if it is not configurable), leaves a human-readable note on the collaboration board, and advances the workflow. Processes exactly one product per run unless the user explicitly asks for more."
argument-hint: "[sku]"
allowed-tools:
  - mcp__onesila__get_company_details
  - mcp__onesila__search_products
  - mcp__onesila__get_product
  - mcp__onesila__search_properties
  - mcp__onesila__get_property
  - mcp__onesila__search_property_select_values
  - mcp__onesila__get_property_select_value
  - mcp__onesila__create_property_select_values
  - mcp__onesila__upsert_products
  - mcp__onesila__create_product_collaboration_entry
  - WebSearch
---

# Attribute filling

A focused, autonomous workflow for completing the attribute values on **one** product in a OneSila PIM. Pull a single product out of the source state, get its attributes right, write a short note explaining what you did, and move it to the review state.

This runs end-to-end without stopping for confirmation. The whole point is that a human reviews the result afterwards — so your job is to be **precise and reliable**, never to guess wildly. A correct empty optional field is fine; a confidently wrong value is not, because it will slip through review.

**Process exactly one product per run.** When you have moved that one product on, stop and report. Only loop if the user explicitly asks you to do several or to keep going.

---

## Configuration

These are the deployment-specific values. Confirm them against `get_company_details(show_workflows=true)` at the start of a run; adjust this block when deploying against a different catalogue.

| Setting | Value |
|---|---|
| Workflow code | `NEW_PRODUCTS` |
| Source state (pull from) | `NEW_PRODUCTS_POPULATE_ATTRIBUTES` |
| Target state (hand on to) | `NEW_PRODUCTS_VERIFY_ATTRIBUTES` |
| MCP server name | `onesila` |

The tool names in `allowed-tools` are prefixed with the MCP server name, so they read `mcp__onesila__get_product` and so on. That matches the server this plugin ships in `mcp-config.json`. If the OneSila server is registered under a different name in the host — a second instance, a demo tenant, anything like `onesila_demo` — the prefixes in `allowed-tools` no longer match any live tool and the skill has nothing to call. Update the frontmatter to the name actually in use.

House rules — the doubt defaults and deterministic rules in Step 3 — are commercial policy, not OneSila behaviour. They are tuned for a costume/apparel catalogue. Review them before running against a different product mix; **country of origin in particular is a customs declaration**, not a formatting preference.

---

## The one rule you must never break

**On a configurable (parent) product, its own properties are READ-ONLY. Never write, change, or clear a property on the configurable parent.**

You only ever write to:

- the **variations** (child products) of a configurable product, or
- the product **itself** when it is a simple, non-configurable product.

Two things that are *not* "properties" and so are allowed on the parent: posting a collaboration-board comment, and moving the parent's workflow state. Everything in the parent's `properties` is off-limits.

---

## Step 0 — Pull one product

Find one product currently in the source state:

```
search_products(workflow_state_code="NEW_PRODUCTS_POPULATE_ATTRIBUTES", limit=1)
```

If nothing comes back, there is no work to do — say so and stop. Otherwise take the first result's SKU. That summary row already includes the product's `type` / `type_label` (so you can tell configurable from simple) and its current `workflows` (workflow code + state code), which you will need when you advance it at the end.

Once, at the start, confirm the workflow's state codes are what you expect:

```
get_company_details(show_workflows=true)
```

This returns each workflow with its code and the list of valid states, so you know the exact `workflow` code and that the target state is a real downstream state before you try to move the product into it.

Now load the product in detail:

```
get_product(sku=<sku>, show_inspector=true, show_variations=true, show_properties=true, show_property_requirements=true, show_images=true, show_translations=true)
```

- `show_inspector` tells you what is missing or blocking — read its issue list carefully.
- `show_variations` lists each child (sku, name, active, missing-info flags) for configurable products, and is empty for simple ones. This is how you decide which path to take.
- `show_property_requirements` gives the full required/optional property map for the product type — your checklist of what to fill.
- `show_properties` shows what is already set (read-only context on the parent; the working set on a simple product). Already-populated attribute values are strong evidence for the ones still missing.
- `show_images` returns the image URLs, titles and descriptions, and `show_translations` returns the product content (names, subtitles, descriptions, bullet points). These are primary evidence — see Step 3. Pull them in here so you have the full media picture before you start filling.

---

## Step 1 — If configurable: fix duplicate variations first

This step only applies to configurable products **and only when the inspector reports something like "Some variations are duplicates."** If the inspector is happy with the variations, skip straight to Step 2.

### Why duplicates happen

Each variation of a configurable product must have a **unique combination of its configurator (variation-defining) values** — typically things like **Size, Colour, Style, Pack size**. A "duplicate" means two or more variations currently resolve to the same combination, usually because a distinguishing value is missing or wrong (e.g. two variations both have `Size` empty, or both say `Red`).

You must resolve these **before** filling other attributes, because the variations cannot be told apart until their configurator values are right.

### How to fix each clashing variation

For each variation involved in a duplicate, load it and look at its configurator property values:

```
get_product(sku=<variation_sku>, show_properties=true, show_inspector=true)
```

Then, for the property that is colliding, do one of two things, in this order of preference:

1. **Infer the correct value from the variation's name or SKU and set it.** This is almost always possible — variation SKUs and names usually encode the variant (e.g. SKU `TSHIRT-RED-L` → `Size = L`; name "... – Navy" → `Colour = Navy`). Set it with `upsert_products` on the **variation** SKU (see Step 3 for how to resolve/create the select value). This is by far the common case: a typical duplicate is several variations left on the same default value (e.g. three sizes all stuck on "M"), and the SKU suffix tells you the right one for each.
2. **If you genuinely cannot infer it, flag it for manual clearing — do not guess.** Blanking a value is not reliably available through `upsert_products`, so treat "leave it empty" as a manual action in the UI rather than something this skill performs. When you can't determine the correct value, leave the field as-is and call it out clearly in your collaboration note so a human can clear or correct it. Never invent a configurator value just to break the tie — a wrong size/colour is worse than a flagged one.

Re-check with `get_product(sku=<parent_sku>, show_inspector=true)` and confirm the duplicate warning is gone before moving on.

---

## Step 2 — Decide what to fill and where

- **Configurable product** → you fill attribute values on **every child variation — active AND inactive**. Walk the full list from `show_variations`. The product is not finished until every child is resolved; inactive variations get reactivated, and an unfilled one resurfaces as a problem later. Prioritise the active ones (read each individually, full treatment); for large sets of inactive siblings of the same family it is acceptable to apply the family-uniform values and SKU-derivable values (colour, pack quantity) in bulk, skipping any field you've seen vary deliberately between siblings — and say in the note which variations got the bulk treatment.
- **Simple (non-configurable) product** → you fill attribute values on the **product itself**.

**Variations aren't always "simple" products.** A configurable's children can come back as Simple, **Bundle**, or **Alias** product types. You still fill their attribute properties the same way (they hold their own properties), so treat them uniformly for attribute purposes. But Bundle/Alias products also have *structural* issues that are **not attributes and not yours to fix** — e.g. "Bundle Products with Inactive Items", "Bundle Products Missing Items", "Items with Missing Mandatory Information". Leave those for the relevant step and mention them in the note; don't try to resolve them here.

**When the legacy data is broadly inconsistent, stay in safe-fix mode.** Some products (especially old multi-channel ones) have variations that disagree with each other on the same field — different size conventions ("80" vs "80CM"), nonsense styles ("Sweatpants" on a hula skirt), wrong materials ("Cobalt"), mixed-language values ("polyester" vs "Poliestere"). When you see this much divergence, do not silently rewrite everything to one convention — that's a normalisation decision the user should sign off on. Instead: fill the genuine *gaps* in required fields with the clearly-correct value, leave the inconsistent existing values in place, and flag the mess prominently in the note so a human can normalise it. Reserve in-place corrections for values that are unambiguously wrong against the product's own evidence (e.g. a gift-wrap flag set to Yes when we never gift-wrap).

Use the property requirement map (`show_property_requirements`) as your checklist. The variations of one product share a product type, so you can read the requirement map once from the parent (or one variation) and reuse it; check each variation's own inspector for what *that* variation is still missing.

### Required AND optional — the optional pass is not skippable

Fill **both required and optional** properties. The required ones unblock the product; the optional ones are where most of the value is, and skipping them is the main way this job gets done badly. So treat the optional list as a deliberate pass, not an afterthought: walk every optional property the requirement map / inspector reports as missing and ask "can I determine this reliably from the name, SKU, description, images, attached docs, or the sibling variations?" If yes, fill it. If no, leave it.

The bar is the same for required and optional: only write a value you are confident is correct. The difference is effort, not standard — go through the *whole* optional list rather than stopping at the blockers.

**You will not — and should not — empty the optional list.** A product type carries dozens of optional fields that simply don't apply to the product in hand (battery chemistry, lens colour, golf-club flex, ring size, scent… on a fabric costume). The inspector keeps listing these as "missing optional" forever. That is fine and expected. Fill what is *applicable and knowable*; ignore the rest. A still-long optional list is not a failure.

### Correcting obvious errors in existing data

The source data is not always right, and you are allowed to fix clearly-wrong values — including on fields that already have a value, and including values you'd otherwise copy from a "complete" sibling. Only correct things that are *obviously* wrong against the product's own evidence, e.g. a kids' costume tagged Department = "Adults", or a "recommended age 1–14 years" that contradicts the size's stated age range. Set the corrected value the same way you set any other (Step 3), and record what you changed and why in the note. Don't relabel things that are merely stylistic choices or that you can't prove wrong — this is for genuine errors, not preference.

Because the sibling variations are your main reference, sanity-check them before trusting them: if a "complete" sibling carries an obviously wrong value, fix it there too rather than propagating it to the others.

---

## Step 3 — Fill attribute values precisely

For each target (each variation, or the simple product), work through the required and optional properties and set the ones you can determine reliably.

**Read every variation before you write to it — never extrapolate one variation's data from another.** It's tempting, on a configurable with many children, to inspect one or two variations and assume the rest match. Don't. Variations in these catalogues routinely disagree (different wrong values, different missing fields, mixed languages), so a value that's right for the sibling can be wrong — or already correct, making your write pointless or harmful — on the one you didn't read. Call `get_product(sku=<variation>, show_properties=true, show_inspector=true)` for **each** variation you intend to change, and base every write on that variation's own data. Equally important, your collaboration note must only state "before" values you actually observed on that specific SKU — don't describe a variation you didn't read.

### Where correct values come from

Read everything already attached to this product before you decide a value — the answer is usually already in the listing somewhere. Draw on **any media and content on the product**: the images, any attached documents, the written content, and the attribute values already populated. In roughly this order of trust:

1. The product/variation **name and SKU** — these usually encode size, colour, material, pack quantity, model, etc.
2. The **attribute values already populated** on the product, and (for a variation) values shared consistently across the variant family — these are reliable and often imply the missing ones.
3. The **content / translations** — the description, subtitle, and bullet points frequently state material, dimensions, capacity, care instructions, certifications, and so on in plain text.
4. The **images** — actually look at them (or the available meta data on them). Pack shots, labels, swing tags, spec panels, and on-product printing reveal colour, material, model number, dimensions, certifications, and country of origin. Fetch and view the image URLs from `show_images` rather than guessing from the filename.
5. Any **attached documents** (spec sheets, manuals, datasheets, safety/compliance docs) — these are authoritative for technical attributes. Read them when present.
6. The product **type** and category — constrains what attributes mean (a "Material" on a mug ≠ material on a t-shirt) — together with the **brand**.
7. **Web search** to confirm a concrete spec when the product is identifiable by brand + model and the attribute is factual (e.g. dimensions, capacity, certified material). Use this to *verify*, not to invent.

If, after working through the attached media and content, a value is still genuinely unknown, prefer leaving an optional property empty over guessing. For a required property you cannot determine, apply the doubt defaults below where they fit; otherwise leave it and flag it in your note so the reviewer can finish it.

### Doubt defaults (use only when genuinely uncertain)

House policy — see Configuration. Deliberate fallbacks for fields that are often unknowable from a listing but rarely wrong in this catalogue. Don't reach for them when the name, SKU, or description actually tells you the answer.

- **Country of origin** — if in doubt, set it to **China**. It is a select, so resolve the value to an ID and write it with `value_is_id: true`.
- **Material**, for textile / apparel products — if in doubt, assume it is **polyester / polyester-based**. It is a multiselect, so write an array of IDs and carry over any existing values you mean to keep (see "How to write a value").

### Merchandising flags — leave these to a human

Product types carry a set of boolean flags that record a **commercial decision rather than a fact about the product**: "New", "Sale", "Eco Collection", "Performance Fabric", staff-pick badges like "Erin Recommends", and their equivalents. Nothing in the listing can tell you whether the buyer means to run this product as a sale line or badge it as new — that's a merchandising call, and a wrong flag shows up on the storefront.

**Leave them empty and name them in your note**, even though the inspector will go on reporting them as missing optional. This is the one part of the optional pass where "I could plausibly pick one" is not a reason to fill it.

Two things this does not cover:

- **Flags that are genuinely factual** — an "Organic", "Contains battery", or "Made in EU" style flag that you can verify from the material, the description, or a spec sheet. Those are attributes like any other; fill them on the evidence.
- **The deterministic rules above.** Where a house rule already dictates a boolean (gift wrap and gift messaging are always No), the rule wins — set it and move on.

### Deterministic rules (always apply)

House policy — see Configuration. These are not judgement calls: when the condition holds, set the value, and correct it if an existing value disagrees.

They apply to **every product that carries the property**. Product types differ in which of these they expose — the list below leans on the Amazon-facing apparel types, and a leaner type may carry only a handful of them, or none. So check each rule against the product's own requirement map, apply it to every listed property that is there, and skip the ones that aren't. A rule that doesn't appear on the product in front of you is not thereby retired: it applies in full on the next product that has the property. Never go looking for a property that isn't on the type, and never create one to satisfy a rule.

- **Amazon Product Tax Code** (`product_tax_code`): a **child / kids' item → `A_CLTH_CHILD`**; **anything else (adult or non-child) → `A_GEN_STANDARD`**. Decide "child item" from the product itself — kids/children's product type, the description ("Childs", "Kids", age range like 4–6 years), Department/Age fields, etc. Fill it wherever it's missing, and switch `A_GEN_STANDARD ↔ A_CLTH_CHILD` if the existing one is on the wrong side of the child/adult line. Zero-rated / no-tax codes (`A_GEN_NOTAX`, `G_GEN_NOTAX`) **may be overwritten** with the correct code per this rule — they are not protected values. The property is a select: the codes above are the value *text* to search for, so resolve to an ID and write with `value_is_id: true`.
- **Gift wrap & gift messaging**: we never offer these. Set **Is Gift Wrap Available** (`is_gift_wrap_available`) → **No/false** and **Offering Can Be Gift Messaged** (`offering_can_be_gift_messaged`) → **No/false** on every product/variation, regardless of what the source data says. Both are booleans — write the literal, not an ID.
- **Model Number** (`model_number`, a text field): set it to the **SKU** of that product/variation. Free text, so write the SKU string directly — no select-value lookup needed. Do not mirror it into **Model Name**, which is a separate free-text field (see the caution below).

### How to write a value

**Every ID is resolved at run time. Never hard-code one, and never carry one between products, runs, or tenants.** Property IDs and select-value IDs are per-tenant and change without notice — an ID that was right last week, or right on another company's catalogue, will silently address the wrong property here. Refer to a property by its `internal_name` and look the ID up on the run that uses it:

1. `search_properties(internal_name="<internal_name>")` → the property's `id` and its `type`. If it returns nothing, the product type doesn't carry that property; skip it (see the deterministic-rules preamble).
2. For a select or multiselect, `search_property_select_values(property_id=<id>, search="<text>")` → the select-value `id`. Use `get_property(property_id=<id>)` to see all of a property's values if you prefer.
3. If the value does **not** exist, create it with `create_property_select_values({value: "<text>", property_id: <id>})` and use the returned `id` — but see the caution below before creating.

The same goes for the house rules below: they name properties by `internal_name` on purpose. Don't add a rule keyed to a numeric ID.

Then use `upsert_products` (each item goes in the `products` list; a single object is accepted as a one-item batch), identifying the target by its `sku` (the variation SKU, or the simple product's SKU — **never the configurable parent**), with a `properties` array. Each property item is `{property_id, value, value_is_id}`:

- **Text / number / boolean property** → pass the literal in `value` (e.g. `value: "325 ml"`, `value: 2`, `value: false`). Leave `value_is_id` off (false).
- **Select property** → `value` is a **single select-value ID**, with `value_is_id: true`:
  `{property_id: <colour_property_id>, value: <select_value_id>, value_is_id: true}`
- **Multiselect property** → `value` is an **array of select-value IDs**, still with `value_is_id: true`:
  `{property_id: <material_property_id>, value: [<select_value_id_1>, <select_value_id_2>], value_is_id: true}`
  A multiselect you're setting to one value still takes a one-item array: `value: [<select_value_id>]`, not `value: <select_value_id>`. Check the property's `type` (`SELECT` vs `MULTISELECT`) from the lookup above or in the requirement map before you write, since the two look identical in the inspector's missing-property list.

  Treat a multiselect write as **replacing** the property's whole value list, not appending to it. So when you are adding to a multiselect that already holds values, pass the existing IDs you want to keep alongside the new ones — read the variation's current values first (you're doing that anyway) and build the full list you want to end up with.

**Be careful creating new select values.** A created value is added permanently to that property's company-wide list. That's fine for genuinely shared, low-cardinality values (a colour like "Green", a size like "L", "Not Applicable"). It's a problem for values that are effectively unique per product, because you'd grow the list by one for every product forever. So:

- Only create a value when you'd expect it to be reused across products.
- If the "right" value is essentially per-product unique (e.g. a model name that equals the SKU, a part/offer number), do **not** create a select value for it. Reuse an exact existing match if there is one, otherwise leave the field and flag it.
- **Model Name** specifically: it is **free text**, not a select, so there is no value list to bloat — but that is not a reason to fill it. Write it only when the listing actually states a model name. Never default it to the SKU; that is what Model Number is for.
- **If none of the existing values actually describes the product, leave the field empty.** Don't pick the nearest-looking one, and don't create a new value just so the field isn't blank. A garment whose neckline is a hooded henley placket is not "Crew" because Crew is the closest option on the list.
- **Treat clearing a value as unavailable.** You can overwrite a value with another valid one; do not rely on being able to blank one. If a field holds something wrong and you have no correct replacement, leave it and flag it in the note for manual clearing (see Step 1).

You can batch multiple properties for one SKU in a single `upsert_products` call, and you can update up to 10 products (variations) per call — but keep each item correct; one invalid item fails the whole call. Because of that all-or-nothing behaviour, when you're about to push the same shape of payload across many variations, **write one variation first and re-read it** to confirm the values landed as you intended, then batch the rest. It costs one call and saves you a failed batch or a silently wrong write repeated fifteen times.

---

## Step 4 — Verify before you hand it on

Re-fetch and confirm your work actually landed and nothing is broken:

```
get_product(sku=<parent_or_simple_sku>, show_inspector=true, show_variations=true)
```

Check that:

- the "duplicates" warning (if there was one) is resolved,
- the missing-required-**properties** issue is cleared on the variations / product (this is the one labelled *"Products Missing Required Properties" / "Add the following required properties…"*),
- you never wrote to the configurable parent's own properties.

**The inspector may stay red even after your attributes are complete — that's often fine.** "Missing required information" lumps in things that are *not* attributes and are not yours to fix in this step, such as:

- missing **price overrides** on a manual price list (a pricing step),
- missing required **marketplace documents** (e.g. "UK Agent", "English Agency Information" — a documents step),
- missing **images**.

Do not try to invent prices, documents, or images to force the inspector green. Your remit is the property/attribute values. If the only remaining blockers are non-attribute ones like these, the product is still ready to advance — just list them explicitly in your note so the reviewer and the later workflow steps pick them up. Likewise, if a genuine *attribute* is still unresolvable, say so rather than guessing.

---

## Step 5 — Leave a human-readable note on the collaboration board

Post one short, plain-language comment so the next person understands what happened and why — not a robotic dump of IDs.

```
create_product_collaboration_entry(sku=<parent_or_simple_sku>, comment="<note>")
```

A good note covers:

- what you did (e.g. "Filled required + optional attributes across all 6 variations"),
- the notable judgement calls and *why* (e.g. "Set Size per the SKU suffix; set Colour from the variation names"),
- where you applied a doubt default (e.g. "Country of origin set to China and material to polyester as the listing didn't specify"),
- anything you deliberately left empty or couldn't resolve, so the reviewer knows where to look,
- any duplicate variations you fixed and how.

Write it the way you'd brief a colleague. Example:

> Filled attributes on all 4 variations. Fixed two duplicate variations — both were missing Size; set them to M and L from the SKU suffixes. Colour taken from the variation names (Black / Charcoal). Material wasn't stated so I set it to polyester (apparel default); country of origin set to China (not specified). Left "GSM weight" empty on all variations — couldn't find a reliable figure. Ready for checking.

---

## Step 6 — Advance the workflow

Advance the workflow on the product you pulled (the configurable parent, or the simple product). Moving workflow state is allowed even on a configurable parent — it is not a property.

```
upsert_products(products=[{
  sku: "<parent_or_simple_sku>",
  workflows: [{ workflow_code: "NEW_PRODUCTS", state_code: "NEW_PRODUCTS_VERIFY_ATTRIBUTES" }]
}])
```

Both fields are required and must be named exactly `workflow_code` and `state_code` — the call is rejected otherwise. Use the codes from Configuration, having confirmed them in Step 0 (from the product's `workflows` summary or from `get_company_details(show_workflows=true)`).

Then stop and give the user a one or two sentence summary: which SKU you completed, the headline of what you filled, and that it has moved on. Do not pick up another product unless asked.

---

## Tool reference

| Task | Tool |
|---|---|
| Find a product in the source state | `search_products` (with `workflow_state_code`) |
| Confirm workflow code + valid states | `get_company_details` (`show_workflows=true`) |
| Load product detail / inspector / variations / requirements | `get_product` |
| Resolve a property ID + type from its `internal_name` | `search_properties` |
| Find an existing select value | `search_property_select_values` |
| Inspect a property and all its values | `get_property` |
| Look up one select value by id | `get_property_select_value` |
| Create a missing select value | `create_property_select_values` |
| Write attribute values / move workflow | `upsert_products` |
| Post the collaboration-board note | `create_product_collaboration_entry` |
| Verify factual specs (sparingly) | `WebSearch` |

## Guardrails recap

- One product per run. Stop after advancing it.
- Read every variation before writing to it — never copy assumptions from a sibling, and only note "before" values you actually saw.
- Never touch the configurable parent's properties — variations only (or the simple product itself).
- Fix duplicate variations before filling other attributes.
- Do the optional pass properly — walk every applicable optional, not just the required blockers. But don't try to empty the optional list; most type-default optionals don't apply.
- Leave merchandising flags (New, Sale, Eco Collection, staff picks) to a human, and say so in the note.
- Fix obviously-wrong existing values (incl. on sibling references); leave anything you can't prove wrong.
- Only write values you're confident are correct; empty beats wrong. Don't rely on blanking a value — flag those for manual clearing.
- Never hard-code or reuse an ID. Resolve properties by `internal_name` and select values by text, every run.
- Multiselects take an **array** of select-value IDs and replace the whole list; selects take a single ID. Prove the payload on one variation, then batch.
- Doubt defaults: country of origin → China; apparel material → polyester.
- Always-true rule: Amazon Product Tax Code → `A_CLTH_CHILD` for kids' items, `A_GEN_STANDARD` for everything else — on every product type that carries the property.
- Always leave a readable note, then advance the workflow.
