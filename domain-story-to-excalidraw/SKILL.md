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

### Work Object Duplication Rule (critical)

In Domain Storytelling, a work object is an **intermediary node per activity** — not a shared global node. The correct visual sentence is:

```
[Actor] --[① verb]--> [Work Object] --[à]--> [Actor target]
```

**Each time a work object appears in a different activity, create a new instance.**

- Parse WOs from `## Story Sequence` activities, **not** from the `## Work Objects` table
- For each activity that involves a WO, create a new triplet: `icon-[slug]-[seq]`, `label-[slug]-[seq]`, `anchor-[slug]-[seq]`
- If the same WO appears in N activities → N separate triplets on the canvas
- Position each copy near the actors involved in that specific activity

Example — Ticket appears in activities ⑦, ⑧, ⑨:
- `anchor-ticket-7` : created by SysBilletterie (place at y=50 near SysBilletterie)
- `anchor-ticket-8` : remis by Caissier to Spectateur (place at y=430 between them)
- `anchor-ticket-9` : presented by Spectateur to Contrôleur (place at y=430 between them)

### Arrow Rules

- Use `startElementId` / `endElementId` to bind arrows to elements (arrows follow when moved)
- Embed text directly on the arrow via the `text` property: `"① recherche"`
- Solid `strokeStyle: "solid"` = direct action
- Dashed `strokeStyle: "dashed"` = response, return, or indirect relation

### ⚠️ Invisible Rectangle Anchors (mandatory — prevents drift)

**Known Excalidraw bug:** Binding arrows to emoji text elements (fontFamily:5) causes those elements to drift after frontend sync. The `update_element` fix is unreliable — the frontend overrides it immediately.

**Solution — always use rectangle anchors:**
For each actor/work object, create an invisible rectangle at the icon position and bind arrows to it instead of to the icon:

```json
{ "type": "rectangle", "id": "anchor-[slug]",
  "x": icon_x, "y": icon_y, "width": 44, "height": 45,
  "backgroundColor": "transparent", "strokeColor": "transparent",
  "opacity": 0, "roughness": 0 }
```

Group each element as a **triplet**: `[icon-id, label-id, anchor-id]`.
Bind arrows using `startElementId: "anchor-[slug]"` — never to `icon-[slug]`.

**Benefits:** zero drift, multiple arrows auto-routed around rectangle perimeter (no pile-up when an actor has many arrows), no post-creation fix needed.

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
- Actors list (name, type) — from `## Actors` table
- Story Sequence — from `## Story Sequence` (numbered activities: actor → verb → work object → target actor)
- **Work object instances** — derived from Story Sequence, NOT from `## Work Objects` table. For each activity that mentions a WO, record: WO name, sequence number, source actor, target actor. Same WO name in different activities = different instances.

### Step 2 — Assign emojis and colors

For each actor and work object, determine:
- Emoji from the keyword table above
- Color from the palette (actors by order, work objects always `#555`)

### Step 3 — Calculate layout

Place actors in a horizontal row at `y=250`.

For each work object **instance** (one per activity):
- WO created/emitted by a system actor (flows downward to human actors) → `y=50`, x aligned on the source actor
- WO exchanged between actors (flows horizontally) → `y=430`, x positioned between the two actors involved

Multiple instances of the same WO (e.g., Ticket×3) are spread along the x-axis at their respective positions — they do not share a single node.

Record the intended `(x, y)` for every element — needed for anchor rectangle placement.

### Step 4 — Create icons, labels, and anchor rectangles

```
batch_create_elements([
  // Title:
  { type: "text", id: "title", x: 450, y: 15, text: "Story title",
    fontFamily: 4, fontSize: 20 },
  // For each actor (one per actor):
  { type: "text", id: "icon-[actor-slug]", x, y, text: "emoji",
    fontFamily: 5, fontSize: 36, strokeColor: color },
  { type: "text", id: "label-[actor-slug]", x: x-offset, y: icon_y+50,
    text: "Name", fontFamily: 4, fontSize: 12, strokeColor: color },
  { type: "rectangle", id: "anchor-[actor-slug]", x, y, width: 44, height: 45,
    backgroundColor: "transparent", strokeColor: "transparent",
    opacity: 0, roughness: 0 },
  // For each work object INSTANCE (one per activity, suffixed with seq number):
  { type: "text", id: "icon-[wo-slug]-[seq]", x, y, text: "emoji",
    fontFamily: 5, fontSize: 36, strokeColor: "#555" },
  { type: "text", id: "label-[wo-slug]-[seq]", x: x-offset, y: icon_y+50,
    text: "WO Name", fontFamily: 4, fontSize: 12, strokeColor: "#555" },
  { type: "rectangle", id: "anchor-[wo-slug]-[seq]", x, y, width: 44, height: 45,
    backgroundColor: "transparent", strokeColor: "transparent",
    opacity: 0, roughness: 0 }
])
```

### Step 5 — Group each triplet (icon + label + anchor)

For each element, call `group_elements` with `[icon-id, label-id, anchor-id]`.

### Step 6 — Create arrows (bound to anchors, not icons)

```
batch_create_elements([
  // For each numbered activity in the Story Sequence:
  { type: "arrow", id: "arrow-[n]",
    startElementId: "anchor-[source-slug]",   // ← anchor, NOT icon
    endElementId: "anchor-[target-slug]",      // ← anchor, NOT icon
    text: "① verb",
    strokeColor: source_actor_color,
    strokeStyle: "solid" | "dashed" }
])
```

**No drift fix needed** — rectangle anchors never drift with arrow binding.

### Step 7 — Fit and screenshot

```
set_viewport({ scrollToContent: true })
get_canvas_screenshot()
```

Review the screenshot. If any element is misplaced or overlapping, fix with `update_element`.

### Step 8 — Export PNG

```
export_to_image({
  format: "png",
  filePath: "/absolute/path/to/reports/04_stories/[domain]_story.png"
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
