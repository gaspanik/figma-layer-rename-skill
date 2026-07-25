# figma-layer-rename — Semantic Layer Naming for Figma Frames

A Claude Code skill that walks an entire Figma FRAME or SECTION, finds auto-generated layer names (`Frame 12`, `Group 4`, `Rectangle 7`, bare `Frame`/`Line`, etc.), infers a semantic name for each from its type, position, and content, and applies the renames via the Plugin API after user confirmation. It can also, on request, spot duplicate-but-real names (four different sections all literally named `Section`) and differentiate those too.

---

## What this is

Figma files generated quickly — by hand under time pressure, or by tools like Figma's Design Agent (`generate_figma_design`) — tend to keep whatever name the canvas assigned by default. A frame full of `Frame`, `Frame 2`, `Group 14` is hard to read, hard to hand off, and hard for any AI tool (including Claude) to reason about later. This skill does the tedious part: read the whole tree, propose a name for every generic layer based on its role in the layout, and apply only what you approve.

```
Figma FRAME/SECTION URL
        │
        ▼
Walk tree, skip INSTANCE subtrees
        │
        ▼
Detect auto-generated names (Frame N, Group N, Rectangle N, bare Frame/Line, ...)
        │
        ▼
[optional] Detect duplicate generic-but-real sibling names (Section x4, Container x12, ...)
           at a user-chosen depth, collapsing `:margin`/`:padding` wrapper frames first
        │
        ▼
Propose semantic names from type + position + sibling/children context
        │
        ▼
Confirmation table  ──▶  "yes" or a list of numbers
        │
        ▼
Apply via use_figma  ──▶  Result report
```

INSTANCE subtrees are skipped entirely — renaming inside an instance only creates a local override on that one instance, silently diverging from the main component and every other instance.

---

## Scope

- **In scope:** auto-generated layer names anywhere in the tree — `Frame N`, `Group N`, `Rectangle N`, `Ellipse N`, `Vector N`, `Polygon N`, `Star N`, `Line N`, `Arrow N`, `Boolean N`, `Component N`, `Instance N`, with or without a trailing number.
- **Also in scope (optional pass):** duplicate generic-but-meaningful sibling names — e.g. four top-level sections all literally named `Section`. These names aren't auto-generated (they're real, semantic strings), but sharing one name across structurally distinct siblings is just as hard to navigate. This pass only runs when duplicates are actually found, and only after you pick a depth (root only / 2 levels / 4 levels / free-text for deeper or the full tree). Single-child `:margin`/`:padding`-suffixed wrapper frames — a common Code to Canvas artifact — are treated as transparent when counting depth and grouping siblings, so a section wrapped in a padding frame is still compared against its unwrapped counterparts.
- **Out of scope:** variable binding, auto-layout changes, component properties. Nothing about the frame's structure or values changes — only names.
- **Not for components.** `COMPONENT` and `COMPONENT_SET` URLs are not supported.

---

## Repo structure

```
skills/
  figma-layer-rename/
    SKILL.md          — skill definition loaded by Claude Code
    LICENSE
```

---

## Getting started

**1. Clone this repo**

```bash
git clone https://github.com/gaspanik/figma-layer-rename-skill
```

**2. Install the skill into Claude Code**

```bash
cp -r skills/figma-layer-rename ~/.claude/skills/
```

**3. Verify the Figma MCP is connected**

This skill requires the official Figma MCP server and calls `use_figma` to apply renames.

**4. Run the skill**

```
/figma-layer-rename https://www.figma.com/design/<fileKey>/...
```

```
このFigmaフレームのレイヤー名を整理して: https://www.figma.com/design/...
```

```
Clean up the generic layer names in this frame: https://www.figma.com/design/...
```

---

## When the skill is triggered automatically

The skill description instructs Claude to use it whenever the user wants to clean up layer names across a whole frame, section, or page layout. Component and component-set URLs are not supported.

---

Built by Masaaki Komori - [@cipher](https://x.com/cipher) · Skill for [Claude Code](https://claude.ai/code) + [Figma MCP](https://github.com/figma/mcp-server-guide)
