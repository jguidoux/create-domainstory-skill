---
name: domain-story-to-excalidraw
description: >
  Converts a Domain Story file (produced by /create-domain-story) into a visual
  Excalidraw diagram following the Hofer & Schwentner pictographic notation.
  Actors as emoji icons, work objects as pictograms, numbered arrows.
  Invoke with /domain-story-to-excalidraw --file=reports/04_stories/[domain]_story.md
user_invocable: true
allowed-tools: Read, Write, Skill, mcp__excalidraw__clear_canvas, mcp__excalidraw__batch_create_elements, mcp__excalidraw__update_element, mcp__excalidraw__group_elements, mcp__excalidraw__set_viewport, mcp__excalidraw__get_canvas_screenshot, mcp__excalidraw__export_to_image
---

# Domain Story → Excalidraw

Reads a Domain Story Markdown file and draws it on the Excalidraw canvas following the pictographic Domain Storytelling notation: emoji icons with labels below, numbered arrows connecting actors and work objects, no shape borders.

**Prerequisite:** The Excalidraw MCP server must be running (`PORT=3000 npm run canvas`) and the canvas open at `http://localhost:3000`.

---

## Input

Reads `reports/04_stories/[domain]_story.md` — the output of `/create-domain-story`.

Extracts from the file:
- **Actors** — from the `## Actors` table
- **Work Objects** — from the `## Work Objects` table
- **Story Sequence** — from the `## Story Sequence` section (① ② ③ numbered activities)
- **Title** — from the `# Domain Story:` heading
- **Boundary Discovery** (optional) — from the `## Boundary Discovery` table

---

## Visual Rules

### Element Structure

Every actor and work object is rendered as two elements, grouped together:

```
[emoji — fontFamily:5, fontSize:36, strokeColor]   ← icon element
[label — fontFamily:4, fontSize:12, y = icon_y+50] ← label element
```

Both elements are grouped via `group_elements` so they move as one unit.

### Emoji Assignment

Match the work object or actor name against these keywords (case-insensitive, French and English):

| Keywords | Emoji |
|----------|-------|
| client, customer, user, utilisateur | 👤 |
| site, web, app, platform, plateforme | 🖥️ |
| mobile | 📱 |
| paiement, payment, stripe, bank, banque | 💳 |
| entrepôt, warehouse, depot, dépôt | 🏭 |
| catalogue, catalog, list, liste | 📋 |
| panier, cart, basket | 🛒 |
| commande, order | 📄 |
| email, mail, notification, confirmation | ✉️ |
| bon de préparation, colis, package, picking, parcel | 📦 |
| livraison, delivery, truck, camion, transport | 🚚 |
| données paiement, payment data, argent, money, finances | 💰 |
| database, db, base de données | 🗄️ |
| api, robot, bot, service, système externe | 🤖 |
| résultats, results, search, recherche | 🔍 |
| *(default)* | 📌 |

### Color Assignment

Assign colors to actors in order of appearance in the story. Work objects always use gray.

| Role | Color |
|------|-------|
| 1st actor (human initiator) | `#1971c2` (blue) |
| 2nd actor (main system) | `#9c36b5` (purple) |
| 3rd actor (external service) | `#2f9e44` (green) |
| 4th actor (secondary) | `#e8590c` (orange) |
| 5th+ (cycle) | blue → purple → green → orange |
| Work objects | `#555` (gray) |

### Arrow Rules

- Use `startElementId` / `endElementId` to bind arrows to elements (arrows follow when moved)
- Embed text directly on the arrow via the `text` property: `"① recherche"`
- Solid `strokeStyle: "solid"` = direct action
- Dashed `strokeStyle: "dashed"` = response, return, or indirect relation

### ⚠️ Critical: Drift Fix (mandatory)

**Known Excalidraw bug:** When creating bound arrows (`startElementId`/`endElementId`) pointing to emoji text elements, Excalidraw's frontend repositions those text elements after sync. This happens every time arrows are batch-created.

**Fix — always apply after step 6:**
Call `update_element` on **every icon element** to restore its intended position.
Labels do not drift and do not need correction.

---

## Layout

```
y=15   [Title]

y=50   [WorkObj]  [WorkObj]  [WorkObj]  ...   (work objects above)

y=250  [Actor]    [Actor]    [Actor]    ...   (actors — center row)

y=450  [WorkObj]  [WorkObj]  [WorkObj]  ...   (work objects below)
```

- Actors are placed in a horizontal row at `y=250`, spaced `350px` apart
- Work objects consumed by left actors → place above-left (`y=50`)
- Work objects consumed by right actors → place above-right or below
- Work objects from the warehouse / fulfillment side → place at `y=450`
- Title centered at `x=450, y=15`
- Assign element IDs as `icon-[slug]` and `label-[slug]` (e.g., `icon-client`, `label-client`)

---

## Execution Workflow

### Step 1 — Parse the story file

Read the input file. Extract:
- Story title
- Actors list (name, type)
- Work objects list (name)
- Story Sequence (numbered activities: actor → verb → work object → target actor)

### Step 2 — Assign emojis and colors

For each actor and work object, determine:
- Emoji from the keyword table above
- Color from the palette (actors by order, work objects always `#555`)

### Step 3 — Calculate layout

Place actors in a horizontal row at `y=250`.
Distribute work objects above (`y=50`) or below (`y=450`) based on which actors use them.
Record the intended `(x, y)` for every element — you will need these to fix drift later.

### Step 4 — Create icons and labels

```
batch_create_elements([
  // For each actor and work object:
  { type: "text", id: "icon-[slug]", x, y, text: "emoji",
    fontFamily: 5, fontSize: 36, strokeColor: color, ... },
  { type: "text", id: "label-[slug]", x: x-offset, y: icon_y+50,
    text: "Name", fontFamily: 4, fontSize: 12, strokeColor: color, ... },
  // Title:
  { type: "text", id: "title", x: 450, y: 15, text: "Story title",
    fontFamily: 4, fontSize: 20 }
])
```

### Step 5 — Group each icon+label pair

For each element, call `group_elements` with `[icon-id, label-id]`.

### Step 6 — Create arrows

```
batch_create_elements([
  // For each numbered activity in the Story Sequence:
  { type: "arrow", id: "arrow-[n]",
    startElementId: "icon-[source-slug]",
    endElementId: "icon-[target-slug]",
    text: "① verb",
    strokeColor: source_actor_color,
    strokeStyle: "solid" | "dashed" }
])
```

### Step 7 — Fix drift (mandatory)

After arrow creation, call `update_element` on **every icon** to restore its intended position:

```
// For each icon element:
update_element("icon-[slug]", { x: intended_x, y: intended_y })
```

Do NOT update labels — they don't drift.

### Step 8 — Fit and screenshot

```
set_viewport({ scrollToContent: true })
get_canvas_screenshot()
```

Review the screenshot. If any element is misplaced or overlapping, fix with `update_element`.

### Step 9 — Export PNG

```
export_to_image({
  format: "png",
  outputPath: "reports/04_stories/[domain]_story.png"
})
```

---

## Optional: Boundary Context Zones

If the story file contains a `## Boundary Discovery` table with bounded context candidates:

After step 7, ask the user: "Do you want to add boundary zones for the bounded contexts?"

If yes, for each bounded context candidate, create a semi-transparent background rectangle behind the relevant elements:

```
{ type: "rectangle",
  x, y, width, height,
  backgroundColor: zone_color, opacity: 15,
  strokeColor: zone_color, strokeStyle: "dashed",
  strokeWidth: 1 }
```

Add a text label at the top-left corner of each zone (not inside the rectangle — use a separate text element).

Zone colors: `#a5d8ff` (blue), `#d0bfff` (purple), `#b2f2bb` (green), `#ffd8a8` (orange)

---

## Error Handling

| Situation | Response |
|-----------|----------|
| Story file not found | Ask the user for the correct path; suggest running `/create-domain-story` first |
| Excalidraw MCP not available | Instruct user to start the canvas: `PORT=3000 npm run canvas` at `http://localhost:3000` |
| More than 9 activities | Use ⑩⑪⑫ or plain numbers `10.` `11.` for sequences beyond ⑨ |
| Element overlap after drift fix | Use `update_element` to nudge elements apart; re-screenshot to verify |
| Export path not writable | Export to `~/Desktop/[domain]_story.png` as fallback |

---

## Related Skills

| Skill | Role |
|-------|------|
| `/create-domain-story` | Produces the story file that this skill reads — always run first |
| `/excalidraw` | Generic diagramming skill — use for free-form diagrams outside Domain Storytelling |
| `event-storming` | Next step in the DDD workflow after visualizing the story |
