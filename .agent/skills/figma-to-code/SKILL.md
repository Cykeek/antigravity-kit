---
name: figmatocode
description: >
  Translates Figma designs into pixel-perfect, production-ready code.
  Activate when the user shares a Figma URL, asks to implement a design,
  wants to match a visual mockup, or says "pixel perfect", "design to code",
  "implement from Figma", or "extract from Figma". Covers Figma MCP setup,
  node inspection, layout translation, asset extraction, and iterative
  visual verification.
agents:
  - frontend-specialist
skills:
  - frontend-design
triggers:
  - "implement from figma"
  - "match the design"
  - "figma link"
  - "pixel perfect"
  - "design to code"
  - "extract from figma"
  - "figma url"
  - "figma file"
  - "build from design"
  - "design handoff"
---

# Figma-to-Code Skill

> **Goal:** Take any Figma design file from zero to pixel-perfect code on the first shot —
> minimal token waste, no back-and-forth guessing.

---

## Use this skill when

- The user provides a Figma file URL or node link
- The user says "implement", "build", or "match" a visual design
- The user says "pixel perfect", "design to code", or "design handoff"
- The user asks to extract assets, colors, or typography from Figma
- The user wants to verify their UI matches the original Figma mockup

## Do not use this skill when

- There is no Figma URL or design file involved
- The user is only asking about general React/CSS patterns (use `frontend-design` instead)
- The user wants to CREATE a design (not implement one)
- The task is purely backend, API, or database focused

---

## Instructions

Work through these phases in order. **Do not write a single line of code until Phases 0 and 1 are complete.**

---

### PHASE 0 — PRE-FLIGHT (Gather before acting)

**Step 1 — Collect credentials.** Ask the user for these if not already provided:
- Figma File URL → extract `fileKey` from `figma.com/design/{fileKey}/...`
- Figma API Token → format: `figd_...`
- Figma MCP token (if using Desktop Bridge plugin)

> ⛔ Never skip this. Attempting to access Figma without auth wastes 5–10 tool calls on failed requests.

**Step 2 — Verify MCP connection.** Call `figma_get_status` first.
- If connected via Desktop Bridge (WebSocket): use MCP tools preferentially.
- If not connected: fall back to REST API with `X-Figma-Token` header.

**Step 3 — Get full file structure first.**
```js
GET https://api.figma.com/v1/files/{fileKey}?depth=2
Headers: { "X-Figma-Token": "{token}" }
```
Or use `figma_get_file_data` with `verbosity: "summary"`, `depth: 1`.
Map every page and section node ID **before** designing anything.

---

### PHASE 1 — DESIGN ANALYSIS (Read everything before writing anything)

**Step 1 — Take a full screenshot first.**
Capture the entire design as a visual reference before inspecting nodes.
Use `figma_get_component_image` or `get_screenshot` on the root frame.

**Step 2 — Inspect each section node.**
For every section, call:
```
mcp_figma-dev-mode-mcp-server_get_design_context(nodeId: "SECTION_ID")
```

Record these properties for every container:

| Figma Property | CSS Output |
|---|---|
| `absoluteBoundingBox.width/height` | `width: Xpx` / `min-height: Xpx` |
| `paddingLeft/Right/Top/Bottom` | `padding: Xpx Ypx` |
| `itemSpacing` | `gap: Xpx` |
| `cornerRadius` | `border-radius: Xpx` |
| `layoutMode: HORIZONTAL` | `flex-direction: row` |
| `layoutMode: VERTICAL` | `flex-direction: column` |
| `primaryAxisAlignItems: SPACE_BETWEEN` | `justify-content: space-between` |
| `counterAxisAlignItems: CENTER` | `align-items: center` |
| `layoutSizingHorizontal/Vertical: FILL` | `flex: 1` |
| `layoutSizingHorizontal/Vertical: FIXED` | explicit `px` value |

**Step 3 — Extract typography.** Per text node record: `fontSize`, `fontWeight`, `lineHeightPx/fontSize` (ratio), `letterSpacing`, `textTransform`.

**Step 4 — Extract colors.** Figma returns RGB in 0–1 range. Convert:
```js
const toHex = ({r, g, b}) =>
  '#' + [r, g, b].map(v => Math.round(v * 255).toString(16).padStart(2, '0')).join('');
```

**Step 5 — Map the full layout hierarchy before writing code.**
```
Example:
Section (grid, 2 cols, gap 30px)
├── Left Column (flex-col, gap 10px)
│   ├── Info Card (350×319px, bg #161616, padding 30px)
│   └── Donate Card (350px, flex:1, bg #FFC821)
└── Right Column (flex:1)
    └── Form Card (stretch height, padding 40px, bg #fff)
```
This prevents structural mistakes before any HTML is written.

---

### PHASE 2 — ASSET EXTRACTION

> ⛔ **CRITICAL: Always extract raw image fills — NEVER render a section node to JPG.**
> Rendering a section bakes in text/buttons/overlays as permanent pixels → ghost/duplicate UI elements.

**Step 1 — Find image fills in node data.** Look in `fills` array for `type: "IMAGE"`:
```json
{ "type": "IMAGE", "imageRef": "9c7e9944c9b74c833e5d06911ed5de11c790b5f8" }
```

**Step 2 — Fetch the raw image URL (correct endpoint).**
```js
// ✅ CORRECT — raw source image, no UI baked in
GET https://api.figma.com/v1/files/{fileKey}/images

// ❌ WRONG for backgrounds — bakes in text/UI
GET https://api.figma.com/v1/images/{fileKey}?ids=SECTION_NODE_ID&format=jpg
```

**Step 3 — Organize assets.**
```
public/assets/
  hero/           ← hero background images
  sections/       ← per-section images
  partners/       ← logos (PNG with transparency)
  icons/          ← UI icons (SVG preferred)
```

**Step 4 — Verify downloads.** Every image must be non-zero bytes. A 0-byte file = failed download.

---

### PHASE 3 — CODE IMPLEMENTATION

**Component mapping:** Each Figma section maps to its own component file.
```
Hero Section       →  Hero.tsx + Hero.module.css
Services Section   →  Services.tsx + Services.module.css
Partners Section   →  Partners.tsx + Partners.module.css
Contact Section    →  Contact.tsx + Contact.module.css
Footer             →  Footer.tsx + Footer.module.css
```

**Full-bleed image card pattern** (hero, banners):
```css
.card { position: relative; overflow: hidden; border-radius: 30px; }
.bgImage { position: absolute; inset: 0; width: 100%; height: 100%;
           object-fit: cover; z-index: 1; }
.overlay { position: absolute; inset: 0;
           background: linear-gradient(to right, #161616 0%, rgba(22,22,22,0) 100%);
           z-index: 2; }
.content { position: relative; z-index: 3; }
```

**Two-column height-matched layout:**
```css
.split { display: grid; grid-template-columns: 350px 1fr; gap: 30px;
         align-items: stretch; }
.leftCol { display: flex; flex-direction: column; gap: 10px; }
.bottomCard { flex: 1; }
```

**Section width consistency** — all sections must use the same container:
```css
.section { padding: 0 100px; max-width: 1500px; margin: 0 auto;
           width: 100%; box-sizing: border-box; }
```

**Responsive breakpoints:**
```css
@media (max-width: 1024px) { .split { grid-template-columns: 1fr; } .section { padding: 0 40px; } }
@media (max-width: 768px)  { .section { padding: 0 20px; } }
@media (max-width: 480px)  { .section { padding: 0 16px; } }
```

---

### PHASE 4 — VISUAL VERIFICATION

After each section is coded:
1. Hard refresh the browser (`Ctrl+Shift+R`)
2. Screenshot the live rendered section
3. Screenshot the same Figma node
4. Compare — check alignment, spacing, colors, font sizes, image fills

**Per-section checklist:**
- [ ] Image fills load (not solid fallback colors)
- [ ] Column widths match exact Figma `px` values (not approximated `%`)
- [ ] Gap values exact (from `itemSpacing`, not estimated)
- [ ] Font weights match Figma (always verify, never assume)
- [ ] Border radii correct (from `cornerRadius`)
- [ ] Dark overlay opacity correct (from `fills[].opacity`)
- [ ] Column bottoms align (use `flex:1` + `align-items:stretch`)
- [ ] All section widths consistent

---

## Anti-patterns to avoid

| ❌ Wrong | ✅ Correct |
|---|---|
| Render section node as JPG for background | Use `/files/{key}/images` for raw fill URL |
| `width: 48%` for columns | Exact `Xpx` from `absoluteBoundingBox` |
| Code without inspecting node first | Always call `get_design_context` before any CSS |
| `height: 100%` without flex parent | Parent needs `display:flex; align-items:stretch` |
| Visually guessing colors | Extract hex from `fills[].color` RGB values |
| Real email in git config | Use `{user}@users.noreply.github.com` |

---

## Figma API quick reference

```bash
# Full file structure
GET /v1/files/{fileKey}?depth=2

# Specific node details (layout + size)
GET /v1/files/{fileKey}/nodes?ids={nodeId}&depth=5

# Raw image fills by imageRef ← use for backgrounds
GET /v1/files/{fileKey}/images

# Rendered node export ← use for icons/illustrations only
GET /v1/images/{fileKey}?ids={nodeId}&format=png&scale=2

# All requests require:
X-Figma-Token: {token}
```

## MCP tool priority order

When Desktop Bridge plugin is running:
1. `get_design_context` — layout specs + generated code snippets
2. `figma_get_file_data` — full document tree
3. `figma_take_screenshot` — visual verification
4. `figma_capture_screenshot` — immediate post-edit validation

Fall back to `fetch()` in Node.js when MCP is unavailable.