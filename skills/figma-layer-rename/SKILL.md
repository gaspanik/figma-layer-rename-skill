---
name: figma-layer-rename
description: Accepts a Figma FRAME or SECTION URL (or, when running inside Figma's own agent environment, the current canvas selection), walks the entire node tree, detects auto-generated layer names, and renames them to semantic names via the Plugin API after user confirmation. Also offers an optional pass to differentiate duplicate generic-but-meaningful sibling names (e.g. multiple top-level nodes all named `Section` or `Container`) at a bounded depth. Use this whenever the user wants to clean up layer names across a whole frame/section/page layout. Component and component-set URLs are not supported.
argument-hint: <figma-frame-url>
allowed-tools: mcp__plugin_figma_figma__get_metadata, mcp__plugin_figma_figma__use_figma
---

# Figma Layer Rename

Analyze the target Figma FRAME or SECTION (a URL in `$ARGUMENTS`, or the current canvas selection — see Step 1), detect auto-generated layer names anywhere in its tree, propose semantic replacements, and apply them after user confirmation.

**Output language:** Respond in the same language the user is using in this conversation.

**Environment note:** This skill is written against the Figma MCP tools `get_metadata` and `use_figma`, but it also runs in environments that expose the Plugin API differently (e.g. Figma's own Design Agent, which provides an `evaluate_script` tool instead). If `get_metadata` / `use_figma` are not available, substitute whatever Plugin API execution tool the environment provides for both the tree fetch (Step 2) and the renames (Step 5) — the Plugin API code itself is identical. Likewise, if the `figma-use` skill referenced in Step 5 doesn't exist in the environment, skip loading it.

## Scope

This skill fixes two categories of issue:

- **Auto-generated layer names** — `Frame N`, `Group N`, `Rectangle N`, `Ellipse N`, `Vector N`, etc. (Step 3, always runs).
- **Duplicate generic-but-meaningful sibling names** — e.g. four different top-level sections all literally named `Section`, or a dozen unrelated `Container`s. These aren't auto-generated (the name itself is real/semantic), but sharing one name across structurally distinct siblings is just as hard to navigate. This pass is optional, bounded to a shallow depth, and only offered when duplicates are actually found (Step 3.5).

Nothing else. No variable binding, no auto-layout changes, no component property changes.

**Target node types:** FRAME or SECTION only.

- If the URL points to a `COMPONENT` or `COMPONENT_SET`, don't proceed — tell the user that component/component-set URLs aren't supported by this skill.
- Any other node type (e.g. a single `TEXT` or `RECTANGLE` node): ask the user for a frame or section URL.

**Skip `INSTANCE` subtrees entirely.** A frame will often contain instances of components used elsewhere in the file. Renaming a layer inside an instance only creates a local override on that one instance — it does not touch the main component, so the name silently diverges from every other instance and from the source of truth. When walking the tree, if a node's type is `INSTANCE`, don't inspect or rename it or any of its descendants; just skip past it.

**Treat icon-glyph containers as a single leaf — don't descend into their vector paths.** A container whose children are all primitive path/shape nodes (`VECTOR`, `BOOLEAN_OPERATION`, `STAR`, `ELLIPSE`, `RECTANGLE`, `LINE`), with no text nodes and no further nested containers, is almost always one icon glyph assembled from multiple paths (e.g. an arrow icon made of a head path and a stem path) — the individual paths are implementation detail, not separately meaningful layers. Rename the container itself if its name is auto-generated (per the regex below), then stop there: don't inspect or rename its child paths individually, even if their names also match the auto-generated pattern.

---

## Step 1: Resolve the target node

`$ARGUMENTS` can identify the target in either of two ways:

- **Figma URL** — `figma.com/design/:fileKey/...?node-id=X-Y` → extract `fileKey` and `nodeId` (convert `X-Y` to `X:Y`).
- **Canvas selection** — when running inside Figma's own agent environment, `$ARGUMENTS` often contains the selected node's `id` and `type` directly (no URL, no fileKey — you're already in the file's context). Use those as-is.

If neither a parseable URL nor a selection is present, ask the user for a Figma URL or to select the target frame/section on the canvas.

---

## Step 2: Fetch the node tree

Call `get_metadata` for the target node to retrieve the full layer hierarchy (node IDs, types, and names).

Check the root node's type:

- `FRAME` or `SECTION` → continue
- `COMPONENT` or `COMPONENT_SET` → stop and tell the user: "このURLはコンポーネントです。このスキルはコンポーネント／コンポーネントセットのURLには対応していません。" (adapt to the conversation language)
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

**Naming convention:** Don't invent a casing style — match the prevailing convention of the semantic layer names already present in the file (if existing good names look like `hero-section`, propose kebab-case; if `Hero Section`, use spaced Title Case; and so on). If the file has no clear precedent to follow, default to PascalCase (`HeroSection`).

Populate a rename list:

```
{ nodeId, currentName, proposedName, reason }
```

A large frame can easily produce dozens of candidates. If the list grows very long (roughly 30+), group it by section/region in the output table so the user can scan and approve in chunks rather than facing one undifferentiated wall of rows — and offer staged processing per Step 4.

---

## Step 3.5: Detect duplicate generic sibling names (optional pass)

This catches a different problem from Step 3. A name like `Section` or `Container` is real and semantic — it doesn't match the auto-generated regex, so Step 3 correctly leaves it alone. But when the *same* name repeats across siblings that are actually different things (a hero section, a pricing section, and a footer section all just named `Section`), the name stops being useful for telling them apart, even though it's technically meaningful.

**Scope this by depth** so it doesn't false-flag siblings that are *supposed* to share a name (e.g. repeated `List Item` in a nav, or repeated `Card` in a grid of same-shaped cards — renaming those would actively hurt, since the repetition itself is the point). How deep to check is a real tradeoff: shallow (root only) misses most real duplicates in a typical multi-section page, since `Container`/`Section` collisions are just as common 3-4 levels down as at the top. But going deeper without limit raises false-positive risk, since repeated card/list patterns get more common the deeper you go. Offer the user a depth choice rather than assuming one:

- **Depth 1**: direct children of the root node
- **Depth 2**: direct children of the root, plus their direct children
- **Depth 4**: down 4 levels from the root
- **Full tree** (unbounded, same reach as Step 3's walk): only reachable via free-text/"Other" input (e.g. the user types "全部" / "unlimited" / a specific depth like "6"), not a listed option — see the question format below.

**Collapse pass-through wrapper frames before counting depth or grouping siblings.** *This sub-rule only applies to Code to Canvas-style captures — if no node name in the tree matches `/:(margin|padding)$/`, skip this entire paragraph and use raw depth as-is.* Code to Canvas captures routinely insert a padding/margin-only wrapper around real content — a single-child frame whose name is the child's name plus a `:margin` or `:padding` suffix (e.g. `Section:margin` wrapping a `Section`, `Container:margin` wrapping a `Container`). Left uncollapsed, these wrappers silently inflate the raw depth of the content they wrap — two conceptually-sibling sections can end up 2 raw levels apart just because one happened to get wrapped and the other didn't, which makes a fixed depth cutoff miss real duplicates or catch them inconsistently. Before applying the depth scope: for any node whose name matches `/:margin$/` or `/:padding$/` and that has exactly one child, treat it as transparent — don't count it as a depth level, and for grouping purposes treat its child as if it were a direct child of the wrapper's own parent instead. Apply this recursively (a wrapper can wrap another wrapper). Do this only for Step 3.5's depth/grouping logic — it doesn't change Step 3, which already walks the full tree regardless of wrapping.

At the chosen (post-collapse) depth, group nodes by their logical parent, then by name (skip `INSTANCE` subtrees, same as Step 3). Any name shared by 2+ siblings under the same logical parent is a duplicate-name candidate — this applies regardless of whether that name matched Step 3's auto-generated regex.

**The deeper the scope, the more scrutiny each candidate needs.** At Depth 4 or Full tree, don't flag a duplicate just because the names match — check whether the siblings' children/content actually differ. Siblings that are clearly the *same kind of thing* repeated on purpose (product cards, list rows, icon slots) must be left alone even if their shared name would otherwise qualify; only flag duplicates where the siblings represent structurally or contextually different content that happens to share a generic name.

**If no duplicates are found at any depth, skip this step silently** — don't ask the question below, don't mention it in the output.

**If duplicates are found**, ask before building candidates. Keep this to a single question with at most 4 listed options (tool constraints on choice count), relying on free-text for anything beyond Depth 4:

> このフレームには、同じ名前が複数使われている箇所があります（例: `Section` が4箇所、`Container` が◯箇所）。厳密には自動生成名ではありませんが、区別しにくいので合わせてリネーム対象にしますか？
> - **含めない**（デフォルト。Step 3の結果のみ適用）
> - **ルート直下のみ**（Depth 1）
> - **2階層下まで**（Depth 2）
> - **4階層下まで**（Depth 4）
>
> （さらに深く、またはフレーム全体まで見たい場合は「その他」に「全部」「6階層まで」などと自由入力してください）

If the candidate count at the chosen depth is large (roughly 30+, same threshold as Step 3), group the confirmation table by section/region like Step 3 does, rather than presenting an undifferentiated wall of rows.

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

**Staged processing for large batches:** If the combined candidate list is large (roughly 30+, same threshold as Step 3's grouping rule), don't dump the whole table at once — first ask how the user wants to review:

> ◯件のリネーム候補が見つかりました。どう確認しますか？
> - **一括で確認**（セクション別にグループ化した1つの表で全件提示）
> - **セクションごとに順番に処理**（1セクション分ずつ提示→承認→適用を繰り返す）

If the user picks staged processing, run the Step 4 → Step 5 loop once per section (present that section's candidates, get approval, apply) before moving to the next section, and keep a running tally for the final Step 6 report. Below the threshold, skip this question and present everything in one table.

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

- If neither a parseable URL nor a canvas selection is available: ask for a valid URL or a selection
- If the node is a `COMPONENT` or `COMPONENT_SET`: tell the user component/component-set URLs aren't supported
- If the node is neither FRAME/SECTION nor COMPONENT/COMPONENT_SET: ask for a valid frame or section URL
- If `get_metadata` returns an error: report it and ask the user to verify Figma file access
- If `use_figma` fails for a specific node: report the failure inline and continue with remaining items
- If a node ID is no longer found at fix time (e.g. deleted between audit and fix): skip it and note "Node not found"
