---
name: figma-layer-rename
description: Accepts a Figma FRAME or SECTION URL, walks the entire node tree, detects auto-generated layer names, and renames them to semantic names via the Plugin API after user confirmation. Use this whenever the user wants to clean up layer names across a whole frame/section/page layout, not a single component — for single-component or component-set renaming, use figma-component-audit-fix instead.
argument-hint: <figma-frame-url>
allowed-tools: mcp__plugin_figma_figma__get_metadata, mcp__plugin_figma_figma__use_figma
---

# Figma Layer Rename

Analyze the Figma FRAME or SECTION URL provided in `$ARGUMENTS`, detect auto-generated layer names anywhere in its tree, propose semantic replacements, and apply them after user confirmation.

**Output language:** Respond in the same language the user is using in this conversation.

## Scope

This skill fixes exactly one category of issue:

- **Auto-generated layer names** — `Frame N`, `Group N`, `Rectangle N`, `Ellipse N`, `Vector N`, etc.

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

## Step 4: Present rename candidates and ask for confirmation

If no auto-generated names are found, output:

```
✅ 自動生成されたレイヤー名は見つかりませんでした。
すべてのレイヤーがすでに意味のある名前になっています。
```

Then stop.

If candidates are found, output the following table and ask for confirmation:

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
