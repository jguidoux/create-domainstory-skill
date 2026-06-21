# create-domain-story

A [Claude Code](https://claude.ai/code) skill for **Domain Storytelling** following the Hofer & Schwentner methodology. Elicits business processes as pictographic stories, builds ubiquitous language, discovers bounded context candidates, and prepares for Event Storming.

---

## What is Domain Storytelling?

Domain Storytelling is a collaborative modeling technique where domain experts tell stories about how they work. Each story is built from three elements:

- **Actors** — people or systems involved (👤 🖥️ 🏭)
- **Work Objects** — data, documents, or items exchanged (📄 📋 📦)
- **Activities** — numbered arrows connecting actors and work objects (① ② ③)

Stories are told in the domain expert's language, not the developer's. This makes them the ideal foundation for **Domain-Driven Design**.

---

## Features

- **3 facilitation modes**: Interactive (guided), Quick (extract from narrative), Document (from existing docs)
- **AS-IS / TO-BE distinction**: captures current state or desired future state, never mixed
- **Boundary Discovery**: identifies bounded context candidates from vocabulary shifts and natural handoffs
- **Glossary building**: builds ubiquitous language with collision detection
- **Event Storming integration**: maps story elements to commands, events, and aggregates
- **Mermaid diagram**: directed graph with emoji icons and borderless nodes — matches pictographic notation

### Example output diagram

```mermaid
graph LR
    classDef actor fill:none,stroke:none
    classDef wo fill:none,stroke:none

    Client["👤<br/>Client"]:::actor
    Site["🖥️<br/>E-commerce Site"]:::actor
    Payment["💳<br/>Payment Service"]:::actor
    Warehouse["🏭<br/>Warehouse"]:::actor

    Catalogue["📋<br/>Product Catalogue"]:::wo
    Cart["🛒<br/>Cart"]:::wo
    Order["📄<br/>Order"]:::wo
    PaymentData["💰<br/>Payment Data"]:::wo
    Email["✉️<br/>Confirmation Email"]:::wo
    PickingOrder["📦<br/>Picking Order"]:::wo

    Client -->|① browses| Catalogue
    Site -->|② displays results| Client
    Client -->|③ adds to| Cart
    Client -->|④ validates| Order
    Client -->|⑤ enters| PaymentData
    Site -->|⑥ requests authorization| Payment
    Payment -.->|⑦ confirms| Site
    Site -->|⑧ sends| Email
    Email -.->|to| Client
    Site -->|⑨ transmits| PickingOrder
    PickingOrder -->|to| Warehouse
```

---

## Installation

### 1. Download the skill file

```bash
curl -o ~/.claude/skills/create-domain-story/SKILL.md \
  https://raw.githubusercontent.com/jguidoux/create-domainstory-skill/master/SKILL.md
```

Or clone the repo directly into your skills folder:

```bash
git clone https://github.com/jguidoux/create-domainstory-skill.git \
  ~/.claude/skills/create-domain-story
```

### 2. Verify installation

Restart Claude Code. The skill should appear in your available skills list. You can verify with:

```
/create-domain-story --help
```

> **Note:** Skills are loaded from `~/.claude/skills/` by default. If your Claude Code installation uses a different path, adjust accordingly.

---

## Usage

### Interactive Mode (recommended)

Best for live sessions with domain experts. Claude guides you phase by phase.

```
/create-domain-story --domain=order-management --as-is
```

Claude will ask you questions at each phase:
1. Scope and objective
2. Story narrative ("tell me what happens when...")
3. Edge cases and exceptions
4. Actor identification
5. Work object cataloging
6. Boundary discovery
7. Glossary building
8. Mermaid diagram generation
9. DDD integration and next steps

---

### Quick Mode

Best when you already know the process and want fast structured output. Provide your narrative directly.

```
/create-domain-story --domain=order-management --to-be --mode=quick
```

Then describe the process in plain language — Claude extracts actors, work objects, activities, boundaries, and glossary automatically.

**Example input:**
> "A customer searches the product catalogue, adds items to their cart, validates with delivery info. The system creates an order. The customer enters payment details, the system requests authorization from Stripe, which confirms. The system sends a confirmation email to the customer and a picking order to the warehouse."

---

### Document Mode

Best for retrofitting existing systems from documentation.

```
/create-domain-story --domain=order-management --as-is --mode=document
```

Claude reads existing files (specs, wikis, API contracts) and extracts the domain story from them.

---

## Output

Each run produces a structured Markdown file at:

```
reports/04_stories/[domain]_story.md
```

The file contains:

| Section | Content |
|---------|---------|
| **Narrative Summary** | 2-3 sentence plain language overview |
| **Story Sequence** | Numbered activities ①②③ in prose |
| **Actors** | Table with type and responsibilities |
| **Work Objects** | Table with creator, consumer, description |
| **Annotations** | Implicit knowledge and exceptions |
| **Boundary Discovery** | Bounded context candidates with rationale |
| **Glossary** | Ubiquitous language with collision detection |
| **Mermaid Diagram** | Directed graph (`graph LR`) |
| **DDD Next Steps** | Domain events, bounded contexts, recommended next skill |

---

## DDD Workflow Integration

This skill fits into a broader DDD workflow:

```
/create-domain-story    →   understand "what happens"
        ↓
  event-storming        →   design "how it happens"
        ↓
  /ddd-redesign         →   define bounded contexts
        ↓
modular-architecture    →   implement the architecture
```

Story elements map to Event Storming artifacts:

| Story Element | Event Storming |
|---|---|
| Activity | Command or Domain Event |
| Work Object | Aggregate or Read Model |
| Actor | Actor (yellow sticky) |
| Boundary | Bounded Context candidate |

---

## Related Skills

| Skill | Purpose |
|-------|---------|
| `/excalidraw` | Visual Domain Storytelling diagram from this story |
| `event-storming` | Next step — design "how it happens" |
| `/analyze-system` | Extract actors and ubiquitous language as input |
| `/ddd-redesign` | Redesign bounded contexts |
| `modular-architecture` | Implement discovered bounded contexts |
| `adr-management` | Document decisions from storytelling sessions |

---

## References

- [Domain Storytelling](https://domainstorytelling.org) — official site by Hofer & Schwentner
- [Domain Storytelling (book)](https://www.domaindriven.de/domainstorytelling) — Stefan Hofer & Henning Schwentner

---

## License

MIT
