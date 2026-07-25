---
name: figma-layer-rename
description: Accepts a Figma FRAME or SECTION URL, walks the entire node tree, detects auto-generated layer names, and renames them to semantic names via the Plugin API after user confirmation. Also offers an optional pass to differentiate duplicate generic-but-meaningful sibling names (e.g. multiple top-level nodes all named `Section` or `Container`) at a bounded depth. Use this whenever the user wants to clean up layer names across a whole frame/section/page layout, not a single component — for single-component or component-set renaming, use figma-component-audit-fix instead.
argument-hint: <figma-frame-url>
allowed-tools: mcp__plugin_figma_figma__get_metadata, mcp__plugin_figma_figma__use_figma
---

# Figma Layer Rename

Analyze the Figma FRAME or SECTION URL provided in `$ARGUMENTS`, detect auto-generated layer names anywhere in its tree, propose semantic replacements, and apply them after user confirmation.

**Output language:** Respond in the same language the user is using in this conversation.

## Scope

This skill fixes two categories of issue:

- **Auto-generated layer names** — `Frame N`, `Group N`, `Rectangle N`, `Ellipse N`, `Vector N`, etc. (Step 3, always runs).
- **Duplicate generic-but-meaningful sibling names** — e.g. four different top-level sections all literally named `Section`, or a dozen unrelated `Container`s. These aren't auto-generated (the name itself is real/semantic), but sharing one name across structurally distinct siblings is just as hard to navigate. This pass is optional, bounded to a shallow depth, and only offered when duplicates are actually found (Step 3.5).

Nothing else. No variable binding, no auto-layout changes, no component property changes.

**Target node types:** FRAME or SECTION only.

- If the URL points to a `COMPONENT` or `COMPONENT_SET`, don't proceed — tell the user to use the `figma-component-audit-fix` skill instead. That skill already covers layer renaming for components, and its rename logic also cross-checks variant siblings, which this skill doesn't do.
- Any other node type (e.g. a single `TEXT` or `RECTANGLE` node): ask the user for a frame or section URL.

**Skip `INSTANCE` subtrees entirely.** A frame will often contain instances of components used elsewhere in the file. Renaming a layer inside an instance only creates a local override on that one instance — it does not touch the main component, so the name silently diverges from every other instance and from the source of truth. When walking the tree, if a node's type is `INSTANCE`, don't inspect or rename it or any of its descendants; just skip past it.

---

## Step 1: Parse the URL

Extract from `$ARGUMENTS`:

- `figma.com/design/:fileKey/...?node-id=X-Y` → `fileKey` and `nodeId` (convert `X-Y` to `X:Y`)

If extraction fails, ask the user for a valid Figma URL.

---

## Step 2: Fetch the node tree

Call `get_metadata` for the target node to retrieve the full layer hierarchy (node IDs, types, and names).

Check the root node's type:

- `FRAME` or `SECTION` → continue
- `COMPONENT` or `COMPONENT_SET` → stop and tell the user: "このURLはコンポーネントです。レイヤー名の修正には `figma-component-audit-fix` スキルを使ってください。" (adapt to the conversation language)
- anything else → ask for a valid frame or section URL

---

## Step 3: Detect auto-generated layer names

Walk the node tree depth-first from the root. At each node:

- If the node's type is `INSTANCE`, skip it and all of its descendants — do not descend into it.
- Otherwise, check whether its name matches:

```
/^(Frame|Group|Rectangle|Ellipse|Vector|Polygon|Star|Line|Arrow|Boolean|Component|Instance)(\s+\d+)?$/i
```

The trailing number is optional: real files (especially those generated via Figma Make or similar AI tooling) often contain bare default names like `Frame` or `Ellipse` with no number at all — typically zero-size spacer frames used for gap sizing, or plain ellipses used as bullet/divider dots. These carry just as little meaning as `Frame 12` and should be renamed too (e.g. a 10px-wide empty frame between two text nodes → `Spacer`; a small ellipse next to a list item → `BulletDot`).

For each match, infer a semantic name from:

- Node type and its position in the tree (root container, header, icon slot, text group, divider, footer, etc.)
- Sibling and parent context (e.g. a `Rectangle` right below a node named `Card` → `Divider`; a `Group` next to a `Nav` at the top → `Logo`)
- Children content (e.g. a `Group` containing only text nodes → `LabelGroup`; a `Frame` containing an image and a heading → `HeroContent`)

Populate a rename list:

```
{ nodeId, currentName, proposedName, reason }
```

A large frame can easily produce dozens of candidates. If the list grows very long (roughly 30+), group it by section/region in the output table so the user can scan and approve in chunks rather than facing one undifferentiated wall of rows.

---

## Step 3.5: Detect duplicate generic sibling names (optional pass)

This catches a different problem from Step 3. A name like `Section` or `Container` is real and semantic — it doesn't match the auto-generated regex, so Step 3 correctly leaves it alone. But when the *same* name repeats across siblings that are actually different things (a hero section, a pricing section, and a footer section all just named `Section`), the name stops being useful for telling them apart, even though it's technically meaningful.

**Scope this narrowly** so it doesn't false-flag siblings that are *supposed* to share a name (e.g. repeated `List Item` in a nav, or repeated `Card` in a grid of same-shaped cards — renaming those would actively hurt, since the repetition itself is the point). Only check at a bounded, shallow depth from the root passed in `$ARGUMENTS`:

- **Depth 1**: direct children of the root node
- **Depth 2**: direct children of the root, plus their direct children

At the chosen depth, group nodes by parent, then by name (skip `INSTANCE` subtrees, same as Step 3). Any name shared by 2+ siblings under the same parent is a duplicate-name candidate — this applies regardless of whether that name matched Step 3's auto-generated regex.

**If no duplicates are found at either depth, skip this step silently** — don't ask the question below, don't mention it in the output.

**If duplicates are found**, ask before building candidates:

> このフレームには、同じ名前が複数使われている箇所があります（例: `Section` が4箇所、`Container` が◯箇所）。厳密には自動生成名ではありませんが、区別しにくいので合わせてリネーム対象にしますか？
> - **含めない**（デフォルト。Step 3の結果のみ適用）
> - **ルート直下のみ含める**（Depth 1）
> - **ルート直下+1階層下まで含める**（Depth 2）

If the user opts in, infer a semantic name for each duplicate using the same content-based reasoning as Step 3 (position in the tree, sibling/parent context, children content — e.g. the `Section` containing a countdown timer and a coupon code → `PricingSection`; the `Container` holding three stat numbers → `StatsRow`). Merge these into the same candidate list as Step 3's results, tagging the reason so the user can tell the two categories apart (e.g. "重複名の差別化" vs. an auto-generated-name reason).

If the user declines, proceed to Step 4 with only Step 3's candidates (if any).

---

## Step 4: Present rename candidates and ask for confirmation

If no auto-generated names were found in Step 3, and Step 3.5 found no duplicates (or the user declined to include them), output:

```
✅ 自動生成されたレイヤー名は見つかりませんでした。
すべてのレイヤーがすでに意味のある名前になっています。
```

Then stop.

If candidates are found (from Step 3, Step 3.5, or both), output the following table and ask for confirmation:

---

### Rename candidates

| # | Node ID | Current name | Proposed name | Reason |
|---|---------|-------------|---------------|--------|
| 1 | 123:456 | Frame 12 | HeroSection | Root container holding the hero heading and CTA |
| 2 | 123:789 | Group 4 | LabelGroup | Groups a heading and subheading text pair |
| 3 | 123:901 | Rectangle 7 | Divider | Thin rectangle separating two content blocks |

→ Type **"yes"** to apply all, or list the numbers you want (e.g. `1 3`).

---

## Step 5: Apply fixes

Before calling `use_figma`, load the `figma-use` skill to ensure correct Plugin API usage.

Apply only the items the user approved. Run all approved changes in a single `use_figma` call when possible to minimize round-trips.

```js
// Rename approved layers
const renames = [
  { id: '123:456', name: 'HeroSection' },
  { id: '123:789', name: 'LabelGroup' },
];

const renamedIds = [];
for (const { id, name } of renames) {
  const node = await figma.getNodeByIdAsync(id);
  if (node) {
    node.name = name;
    renamedIds.push(id);
  }
}

return { renamedCount: renamedIds.length, renamedIds };
```

---

## Step 6: Output the result report

After applying all approved renames, output:

---

### Rename result

| # | Node | Change | Status |
|---|------|--------|--------|
| 1 | 123:456 | Frame 12 → HeroSection | ✅ Applied |
| 3 | 123:901 | Rectangle 7 → Divider | ✅ Applied |
| 2 | 123:789 | Group 4 → LabelGroup | ⏭ Skipped (user choice) |

**Applied: N / Skipped: N**

---

## Error handling

- If fileKey or nodeId cannot be parsed: ask for a valid URL
- If the node is a `COMPONENT` or `COMPONENT_SET`: redirect the user to `figma-component-audit-fix`
- If the node is neither FRAME/SECTION nor COMPONENT/COMPONENT_SET: ask for a valid frame or section URL
- If `get_metadata` returns an error: report it and ask the user to verify Figma file access
- If `use_figma` fails for a specific node: report the failure inline and continue with remaining items
- If a node ID is no longer found at fix time (e.g. deleted between audit and fix): skip it and note "Node not found"
