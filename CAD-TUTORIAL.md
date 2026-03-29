# Claude Code pro CAD inzenyry — Prakticke lekce se Solvinou

> Naucte se Claude Code tim, ze ho pouzijete na skutecnem projektu:
> rozsireni produkcniho 2D CAD enginu Solvina.

**Ucebnice**: [claude-howto](https://github.com/anthropics/claude-howto) — referencni materialy ke kazdemu tematu
**Laborator**: [Solvina](../solvina/) — React 19 + TypeScript + Vite aplikace pro stavebni inzenyrstvi

---

## Obsah

| # | Lekce | Claude Code feature | CAD vystup |
|---|-------|---------------------|------------|
| 01 | [Slash Commands — Orientace v projektu + prvni zmeny](#lekce-01-slash-commands--orientace-v-projektu--prvni-zmeny) | /init, /help, /commit, /diff, /compact, vlastni skill | `/scaffold-cad-tool` skill |
| 02 | [Memory / CLAUDE.md — Zdokumentovani architektury](#lekce-02-memory--claudemd--zdokumentovani-architektury) | CLAUDE.md hierarchie, .claude/rules/, #, @ importy | CLAUDE.md + rules pro Solvinu |
| 03 | [Skills — Novy kresici nastroj](#lekce-03-skills--novy-kresici-nastroj) | SKILL.md format, auto-invocation, $ARGUMENTS, context: fork | EllipseTool pres cad-tool-generator |
| 04 | [Subagents — Paralelni vyvoj nove feature](#lekce-04-subagents--paralelni-vyvoj-nove-feature) | .claude/agents/, paralelni delegace, tool restrictions, worktree | Entity grouping (3 agenti) |

---

## Uvod

### Pro koho je tento tutorial

- Vyvojari, kteri chteji poznat Claude Code na realnem projektu (ne na TODO appce)
- CAD/stavebni inzenyri, kteri chteji automatizovat rozsireni vlastniho nastroje
- Kdokoli, kdo se uci lepe "rukou" nez ctenim dokumentace

### Co budete potrebovat

| Pozadavek | Minimum |
|-----------|---------|
| Claude Code | v2.1.80+ (`claude --version`) |
| Node.js | 20+ |
| Git | 2.38+ (kvuli worktrees) |
| Solvina projekt | naklonovan a `npm install` hotov |
| Textovy editor | VS Code doporucen, ale neni nutny |

### Jak lekce funguji

Kazda lekce ma dve roviny:

1. **Claude Code feature** — naucite se konkretni nastroj nebo koncept
2. **CAD uloha** — pouzijete ho na realne zmene v Solvine

Lekce stavaji na sobe — vystupy z Lekce 01 pouzivate v Lekci 02 atd. Pokud uz nektery feature znate, preskocte teorii a jdete rovnou na praktickou cast.

### Struktura Solvina projektu

Pro orientaci — dulezite casti CAD enginu, se kterymi budeme pracovat:

```
solvina/
├── packages/cad-engine/src/
│   ├── core/
│   │   ├── types.ts          # Vec2, Entity typy, Layer, CadDocument, ViewportState
│   │   ├── engine.ts         # CadEngine trida — canvas, viewport, tools, undo/redo
│   │   └── document.ts       # Immutable operace: addEntity, removeEntity, updateEntity
│   ├── tools/
│   │   ├── tool-interface.ts  # Tool interface, ToolContext, ToolResult, createEntity()
│   │   ├── line-tool.ts       # Vzorovy nastroj — state machine pattern
│   │   ├── select-tool.ts
│   │   ├── rectangle-tool.ts
│   │   ├── circle-tool.ts
│   │   └── ... (12 nastroju celkem)
│   ├── render/
│   │   ├── viewport.ts       # worldToScreen, screenToWorld, zoomAt, pan
│   │   └── renderer.ts       # Canvas 2D rendering s requestAnimationFrame
│   ├── snap/
│   │   └── snap-engine.ts    # findSnap() — endpoints, midpoints, centers, grid
│   ├── geometry/
│   │   └── math.ts           # distance, midpoint, vec, segmentSegmentIntersection
│   └── io/
│       ├── dxf-import.ts
│       ├── dxf-export.ts
│       └── svg-export.ts
└── src/modules/cad/
    ├── index.tsx              # CadPage — hlavni stranka
    └── components/
        ├── CadCanvas.tsx      # Canvas wrapper
        ├── Toolbar.tsx        # Panel nastroju
        ├── LayerPanel.tsx     # Sprava vrstev
        └── PropertyPanel.tsx  # Panel vlastnosti
```

---

## Lekce 01: Slash Commands — Orientace v projektu + prvni zmeny

**Cas**: 30 min | **Obtiznost**: Beginner

### Co se naucite

- **Claude Code**: Zakladni slash commands (`/init`, `/help`, `/commit`, `/diff`, `/compact`) a tvorba vlastniho skillu jako slash command
- **CAD**: Prozkoumani architektury Solviny a vytvoreni `/scaffold-cad-tool` skillu, ktery generuje nove nastroje podle existujiciho vzoru

### Predpoklady

- Nainstalovany Claude Code (`npm install -g @anthropic-ai/claude-code`)
- Solvina projekt stazeny a pripraveny: `cd ~/solvina && npm install`
- Otevren terminal v projektu: `cd ~/solvina && claude`

### Teorie: Claude Code

**Slash commands** jsou zkratky, ktere ovladaji chovani Claude Code behem interaktivni session. Existuji tri druhy:

1. **Vestavene** — dodavane Claude Code (`/help`, `/clear`, `/model`, `/compact`)
2. **Skills** — uzivatelsky definovane prikazy jako `SKILL.md` soubory
3. **Plugin/MCP commands** — prikazy z pluginu a MCP serveru

Dulezite vestavene prikazy pro zacatek:

| Prikaz | K cemu slouzi |
|--------|---------------|
| `/init` | Vytvori CLAUDE.md — zaklad projektove pameti |
| `/help` | Zobrazi napovedu a dostupne prikazy |
| `/commit` | Inteligentni git commit s kontextem |
| `/diff` | Interaktivni prohlizec uncommitted zmen |
| `/compact` | Zkomprimuje konverzaci — usetri kontext |
| `/cost` | Ukazuje spotrebu tokenu |

Vlastni slash commands se dnes tvorili pres `.claude/commands/`, ale **doporuceny zpusob je skill** v `.claude/skills/<nazev>/SKILL.md`. Skill je slozka s instrukci pro Claude — kdyz ho zavolate pres `/nazev`, Claude nacte SKILL.md a ridi se jim.

> Podrobnosti viz [claude-howto/01-slash-commands/README.md](01-slash-commands/README.md)

### Teorie: CAD

Kazdy kresici nastroj v Solvine implementuje `Tool` interface z `tool-interface.ts`. Vsechny nastroje sdileji spolecny vzor — **state machine** s metodami `onPointerDown`, `onPointerMove`, `onPointerUp`, `onKeyDown` a `cancel`. Pridani noveho nastroje vyzaduje:

1. Definovat novy `EntityType` v `types.ts` (pokud jeste neexistuje)
2. Vytvorit soubor `<nazev>-tool.ts` v `tools/`
3. Implementovat `Tool` interface se stavovym automatem
4. Zaregistrovat nastroj v `engine.ts`
5. Pridat tlacitko do `Toolbar.tsx`

Tento postup budeme opakovat vicekrat — proto si ho automatizujeme pres skill.

### Prakticka cast

#### Krok 1: Spustte Claude Code v Solvine

```bash
cd ~/solvina
claude
```

#### Krok 2: Prozkoumejte projekt pomoci zakladnich prikazu

Zadejte tyto prompty do Claude Code (kazdy na novy radek, stiskni Enter):

```
/help
```

Prohlednete si seznam prikazu. Pak:

```
Prozkoumej architekturu CAD enginu v packages/cad-engine/src/.
Zajima me: jake entity typy existuji, jak vypada Tool interface,
a jak LineTool implementuje state machine pattern.
```

Claude projde soubory a vysvetli architekturu. Sledujte, jak pouziva `Read`, `Grep` a `Glob` nastroje.

#### Krok 3: Inicializujte projektovou pamet

```
/init
```

Claude vytvori zakladni `CLAUDE.md`. Zatim ho nechte v defaultnim stavu — v Lekci 02 ho budeme podrobne upravovat.

#### Krok 4: Zkomprimujte konverzaci

Po pruzkumu architektury mate v kontextu hodne textu. Usetrete tokeny:

```
/compact Zachovej informace o Tool interface a LineTool vzoru
```

Claude zkomprimuje historii, ale zachova klicove informace, ktere jste specifikovali.

#### Krok 5: Vytvorte skill `/scaffold-cad-tool`

Vytvorte adresarovou strukturu:

```
Vytvor slozku .claude/skills/scaffold-cad-tool/ a v ni SKILL.md
```

Obsah SKILL.md by mel byt presne takovy:

```yaml
---
name: scaffold-cad-tool
description: Scaffold a new CAD drawing tool for the Solvina engine following the existing state machine pattern. Use when creating new tools like EllipseTool, SplineTool, etc.
argument-hint: <tool-name>
allowed-tools: Read, Write, Grep, Glob
---

# Scaffold CAD Tool: $ARGUMENTS

## Context

Read the following files to understand the existing patterns:

1. @packages/cad-engine/src/tools/tool-interface.ts — Tool interface and ToolContext
2. @packages/cad-engine/src/tools/line-tool.ts — Reference implementation (state machine)
3. @packages/cad-engine/src/core/types.ts — Entity types and Vec2

## Current tools

!`ls packages/cad-engine/src/tools/*.ts`

## Instructions

Generate a new tool called **$ARGUMENTS** following these steps:

### Step 1: Entity type

Check if the entity type for "$ARGUMENTS" already exists in `core/types.ts`.
If not, add it to the `Entity` union type and `EntityType` type.

### Step 2: Tool file

Create `packages/cad-engine/src/tools/$ARGUMENTS-tool.ts` with:

- A `<Name>State` type with at least `idle` and `placing` states
- A `<Name>Tool` class implementing `Tool`
- The `type` property set to the matching `ToolType`
- `onPointerDown` — starts or completes the shape based on state
- `onPointerMove` — updates preview geometry while placing
- `onPointerUp` — no-op (consistent with other tools)
- `onKeyDown` — handle Escape to cancel
- `cancel()` — reset to idle state
- Use `createEntity()` from `tool-interface.ts` for entity creation
- Use `addEntity()` from `document.ts` to commit to document

### Step 3: Engine registration

In `packages/cad-engine/src/core/engine.ts`:
- Import the new tool
- Add it to the `tools` Map in the constructor

### Step 4: Toolbar button

In `src/modules/cad/components/Toolbar.tsx`:
- Add a button for the new tool with an appropriate icon/label
- Wire it to `setActiveTool('<tool-type>')`

### Step 5: Summary

Print a summary of all files created or modified.
```

#### Krok 6: Otestujte novy skill

Zkuste skill vyvolat (zatim jen nasucho — nemusime entitu implementovat):

```
/scaffold-cad-tool pentagon
```

Claude nacte SKILL.md, doplni `$ARGUMENTS` za "pentagon" a provede vsech 5 kroku. Sledujte, jak pouziva `@` importy k nacteni realnych souboru a `!`command`` k zjisteni existujicich nastroju.

#### Krok 7: Podivejte se na zmeny a commitnete

```
/diff
```

Prohlednete si zmeny. Pokud vypadaji dobre:

```
/commit
```

Claude vytvori commit s popisnou zpravou na zaklade zmenenych souboru.

### Cviceni

1. **Prozkoumejte dalsi vestavene prikazy**: Zkuste `/cost` (kolik tokenu jste utratili), `/context` (vizualizace vyuziti kontextu) a `/status` (verze, model, ucet).

2. **Upravte skill**: Pridejte do `scaffold-cad-tool` SKILL.md dalsi krok — "Step 6: Snap support" — ktery prida snap points pro novy tvar do `snap/snap-engine.ts`. Zvladte to sami, bez navodu.

### Overeni

- [ ] `ls .claude/skills/scaffold-cad-tool/SKILL.md` — soubor existuje
- [ ] V Claude Code zadejte `/scaffold-cad-tool` a overete, ze se zobrazi v nabidce
- [ ] Skill pri spusteni nacte `tool-interface.ts`, `line-tool.ts` a `types.ts` jako kontext
- [ ] Vysledkem je novy soubor v `packages/cad-engine/src/tools/` a upraveny `engine.ts` a `Toolbar.tsx`

### Co jste pridali do Solvina

| Soubor | Zmena |
|--------|-------|
| `.claude/skills/scaffold-cad-tool/SKILL.md` | **Novy** — skill pro generovani CAD nastroju |
| `CLAUDE.md` | **Novy** — zakladni projektova pamet (z `/init`) |
| `packages/cad-engine/src/tools/pentagon-tool.ts` | **Novy** — PentagonTool (testovaci vystup skillu) |
| `packages/cad-engine/src/core/engine.ts` | **Upraven** — registrace PentagonTool |
| `src/modules/cad/components/Toolbar.tsx` | **Upraven** — tlacitko pro pentagon |

---

## Lekce 02: Memory / CLAUDE.md — Zdokumentovani architektury

**Cas**: 25 min | **Obtiznost**: Beginner

### Co se naucite

- **Claude Code**: Hierarchie CLAUDE.md souboru, `.claude/rules/` pro modularna pravidla, `#` quick rules, `@` importy
- **CAD**: Vytvorite CLAUDE.md zachycujici architekturu Solviny — immutable document model, tool state machine vzor, entity typy, konvence pojmenovani

### Predpoklady

- Dokoncena Lekce 01
- Otevren Solvina projekt: `cd ~/solvina && claude`

### Teorie: Claude Code

**Memory** v Claude Code je system souboru `CLAUDE.md`, ktere Claude automaticky nacita pri kazdem spusteni. Funguje v hierarchii (od nejdulezitejsiho):

| Uroven | Umisteni | Sdileni |
|--------|----------|---------|
| Managed Policy | `/Library/Application Support/ClaudeCode/CLAUDE.md` | Organizace |
| Project Memory | `./CLAUDE.md` nebo `./.claude/CLAUDE.md` | Tym (pres git) |
| Project Rules | `./.claude/rules/*.md` | Tym (pres git) |
| User Memory | `~/.claude/CLAUDE.md` | Osobni (vsechny projekty) |
| Local Memory | `./CLAUDE.local.md` | Osobni (jen tento projekt) |

**Klicove mechanismy:**

- **`#` quick rules** — napiste `# Vzdy pouzivej async/await` a Claude to ulozi do memory
- **`@` importy** — v CLAUDE.md napiste `@docs/architecture.md` a Claude nacte obsah souboru
- **`.claude/rules/*.md`** — modularna pravidla s volitelnym `paths:` frontmatter pro scope na konkretni soubory
- **`/memory`** — otevre CLAUDE.md soubory v editoru

> Podrobnosti viz [claude-howto/02-memory/README.md](02-memory/README.md)

### Teorie: CAD

Solvina pouziva nekolik architekturalnich vzoru, ktere je dulezite zakotvit v CLAUDE.md, aby je Claude dodrzoval pri kazde zmene:

1. **Immutable document model** — `document.ts` nikdy nemutuje stav, vzdy vraci novy `CadDocument`. Funkce `addEntity()`, `removeEntity()`, `updateEntity()` jsou ciste funkce.

2. **Tool state machine** — kazdy nastroj ma definovane stavy (napr. `idle | placing | dragging`) a prechody mezi nimi. Stav se meni jen v `onPointerDown`, `onKeyDown` a `cancel`.

3. **Entity union type** — vsechny entity jsou soucasti diskriminovaneho unionu v `types.ts`. Pridani nove entity vyzaduje rozsireni unionu A vsech switch/if retezcu, ktere ho pouzivaji (renderer, snap engine, DXF export...).

4. **Viewport transformace** — svet (world) vs. obrazovka (screen). Kazdy nastroj pracuje ve svetovych souradnicich a pouziva `viewport.worldToScreen()` / `viewport.screenToWorld()`.

### Prakticka cast

#### Krok 1: Prepiste CLAUDE.md

Nahradte obsah CLAUDE.md (vytvoreneo v Lekci 01) detailni dokumentaci architektury:

```
Prepis CLAUDE.md s nasledujicim obsahem (zachovej to presne jak ti ho dam):
```

Obsah CLAUDE.md:

```markdown
# Solvina — 2D CAD Engine pro stavebni inzenyrstvi

## Prehled projektu
- **Tech stack**: React 19 + TypeScript + Vite
- **Ucel**: Strukturalni inzenyrstvi — 2D vykresy, kotovani, DXF import/export
- **Monorepo**: packages/cad-engine (jadro) + src/modules/cad (React UI)

## Architektura

### Immutable Document Model
Soubor `packages/cad-engine/src/core/document.ts` obsahuje ciste funkce.
NIKDY nemutuj CadDocument primo — vzdy pouzij:
- `addEntity(doc, entity)` — vraci novy CadDocument
- `removeEntity(doc, id)` — vraci novy CadDocument
- `updateEntity(doc, id, updates)` — vraci novy CadDocument
- `findEntity(doc, id)` — read-only lookup

### Tool State Machine Pattern
Kazdy nastroj v `packages/cad-engine/src/tools/` implementuje `Tool` interface:
- Definuj typ stavu: `type MyToolState = 'idle' | 'placing' | ...`
- `onPointerDown` / `onPointerMove` / `onPointerUp` — prechodove metody
- `onKeyDown` — handle Escape (cancel) a Enter (confirm)
- `cancel()` — reset do idle
- Pouzij `createEntity()` z `tool-interface.ts`
- Reference implementace: `line-tool.ts`

### Entity System
Entity typy (discriminated union v `core/types.ts`):
line | circle | arc | rectangle | polyline | text | dimension | xline | image | hatch

Pridani nove entity vyzaduje zmeny v:
1. `core/types.ts` — Entity union + EntityType
2. `render/renderer.ts` — vykreslovani
3. `snap/snap-engine.ts` — snap points
4. `io/dxf-export.ts` — DXF export
5. `io/dxf-import.ts` — DXF import (pokud je relevantni)

### Souradnicovy system
- Vsechny entity ulozeny ve SVETOVYCH souradnicich
- `viewport.worldToScreen(point)` pro vykreslovani
- `viewport.screenToWorld(point)` pro vstup z mysi
- `geometry/math.ts` — utility funkce (distance, midpoint, vec, closestPointOnSegment)

## Dulezite prikazy

| Prikaz | Ucel |
|--------|------|
| `npm run dev` | Spusti dev server |
| `npm run build` | Build pro produkci |
| `npm run test` | Spusti testy |
| `npm run lint` | Linting |

## Konvence

### Pojmenovani
- Soubory nastroju: `<nazev>-tool.ts` (kebab-case)
- Tridy: `<Nazev>Tool` (PascalCase)
- Stavove typy: `<Nazev>State` (PascalCase)
- Entity typy v union: lowercase (`'ellipse'`, `'spline'`)

### Git
- Commit zpravy: conventional commits (feat, fix, refactor, docs)
- Branch: `feature/<popis>` nebo `fix/<popis>`
- Vzdy `npm run lint` pred commitem

## Reference
@packages/cad-engine/src/core/types.ts
@packages/cad-engine/src/tools/tool-interface.ts
```

#### Krok 2: Vytvorte .claude/rules/ pro path-scoped pravidla

Vytvorte pravidlo specificke pro geometricke operace:

```
Vytvor soubor .claude/rules/geometry.md s nasledujicim obsahem:
```

Obsah `.claude/rules/geometry.md`:

```yaml
---
paths: packages/cad-engine/src/geometry/**/*.ts
---

# Geometricka pravidla

- Vsechny geometricke funkce musi byt CISTE (pure functions) — zadne side effects
- Pouzivej typ Vec2 z core/types.ts, nevytvarej vlastni {x, y} objekty
- Kazda funkce musi mit JSDoc s @param a @returns
- Ciselne porovnani pouzivej s EPSILON toleranci (1e-10)
- Uhly vzdy v RADIANECH, ne ve stupnich
- Testy pro geometrii v __tests__/ vedle zdrojoveho souboru
```

#### Krok 3: Vytvorte pravidlo pro nastroje

```
Vytvor soubor .claude/rules/tools.md s nasledujicim obsahem:
```

Obsah `.claude/rules/tools.md`:

```yaml
---
paths: packages/cad-engine/src/tools/**/*.ts
---

# Pravidla pro CAD nastroje

- Kazdy nastroj MUSI implementovat Tool interface z tool-interface.ts
- Stav nastroje definuj jako union type (napr. type EllipseState = 'idle' | 'placing')
- NIKDY nepristupuj k canvasu primo — pouzij ToolContext
- Pouzij createEntity() z tool-interface.ts — nerob new entity rucne
- Preview geometrii vrat v ToolResult.toolPreview, nemodifikuj document
- onPointerUp typicky vraci { needsRedraw: false } — akce jsou v onPointerDown
- cancel() musi vzdy resetovat stav na 'idle' a vycistit preview
```

#### Krok 4: Vyzkousejte quick rule

Pridejte pravidlo primo z konverzace:

```
# V tomto projektu pouzivame vzdy named exports, nikdy default exports
```

Claude vas pozada o vyber memory souboru — vyberte Project Memory (`./CLAUDE.md`).

#### Krok 5: Overte, ze @ importy funguji

Podivejte se, zda Claude spravne nacita referencovane soubory:

```
Ukazmi obsah Tool interface — podle informaci v CLAUDE.md
```

Claude by mel pouzit `@` import z CLAUDE.md a nacist `tool-interface.ts` automaticky.

### Cviceni

1. **Vytvorte osobni memory**: Vytvorte `~/.claude/CLAUDE.md` s vasimi osobnimi preferencemi (napr. preferovany jazyk, styl komentaru, oblibene patterny). Overte, ze se nacita vedle projektove memory.

2. **Pridejte dalsi rule**: Vytvorte `.claude/rules/renderer.md` s pravidly pro `render/` slozku — napr. "Pouzivej ctx.save()/ctx.restore() kolem kazdeho entity renderu" a "Barvu entity resolvuj pres resolveEntityColor(), ne primo z entity.color".

### Overeni

- [ ] `cat CLAUDE.md` — obsahuje sekce Architektura, Entity System, Konvence
- [ ] `cat .claude/rules/geometry.md` — ma `paths:` frontmatter
- [ ] `cat .claude/rules/tools.md` — ma `paths:` frontmatter
- [ ] V Claude Code zadejte `Jak pridam novou entitu?` — Claude by mel odpovedet podle CLAUDE.md (5 kroku: types, renderer, snap, DXF export, DXF import)
- [ ] V Claude Code zadejte `Napis geometrickou funkci pro vypocet plochy trojuhelniku` — Claude by mel pouzit Vec2, cisty pristup, EPSILON, JSDoc (diky `geometry.md` pravidlu)

### Co jste pridali do Solvina

| Soubor | Zmena |
|--------|-------|
| `CLAUDE.md` | **Prepsan** — detailni architektura, konvence, reference |
| `.claude/rules/geometry.md` | **Novy** — pravidla pro geometricke funkce |
| `.claude/rules/tools.md` | **Novy** — pravidla pro CAD nastroje |

---

## Lekce 03: Skills — Novy kresici nastroj

**Cas**: 35 min | **Obtiznost**: Intermediate

### Co se naucite

- **Claude Code**: SKILL.md format s frontmatter, auto-invocation, `$ARGUMENTS`, sablony, `context: fork`
- **CAD**: Vytvorite propracovany `cad-tool-generator` skill a pouzijete ho ke generovani kompletniho EllipseTool

### Predpoklady

- Dokoncena Lekce 02 (CLAUDE.md a rules existuji)
- Otevren Solvina projekt: `cd ~/solvina && claude`

### Teorie: Claude Code

**Skills** jsou znovupouzitelne schopnosti balene jako `SKILL.md` soubory. Na rozdil od CLAUDE.md (ktera se nacita vzdy) se skills nactou az kdyz je Claude potrebuje — to setri kontext.

**Tri urovne nacteni:**

| Uroven | Kdy | Co |
|--------|-----|----|
| **Metadata** | Vzdy pri startu | Jen `name` a `description` z frontmatter (~100 tokenu/skill) |
| **Instrukce** | Kdyz se skill spusti | Telo SKILL.md (max ~5000 tokenu) |
| **Zdroje** | Dle potreby | Dalsi soubory ve slozce skillu |

**Dulezite frontmatter pole:**

```yaml
---
name: muj-skill           # -> /muj-skill
description: Co dela a kdy ho pouzit
argument-hint: <arg>       # Napoveda v autocomplete
allowed-tools: Read, Write # Povolene nastroje
context: fork              # Spusti v izolovanem subagentovi
disable-model-invocation: true  # Jen uzivatel muze vyvolat
---
```

**`context: fork`** spusti skill v samostatnem subagentovi s vlastnim kontextovym oknem. Hlavni konverzace se nezanasi. Vysledek se vrati jako shrnuti.

**`$ARGUMENTS`** se nahradi argumenty za `/nazev-skillu`. Napr. `/fix-issue 123` → `$ARGUMENTS` = "123".

**`!`command``** spusti shell prikaz a vlozi vysledek do skill obsahu pred odeslanim Claudovi.

> Podrobnosti viz [claude-howto/03-skills/README.md](03-skills/README.md)

### Teorie: CAD

**Elipsa** (ellipse) je zakladni tvar, ktery v Solvine chybi. Na rozdil od kruhu (definovaneho stredem a polomerem) je elipsa definovana:

- **Stred** (center): Vec2
- **Hlavni poloosa** (semi-major axis): cislo + uhel natoceni
- **Vedlejsi poloosa** (semi-minor axis): cislo

Nastroj `EllipseTool` bude mit tri stavy:
1. `idle` — ceka na kliknuti
2. `center-placed` — uzivatel klikl stred, tahne hlavni poloosu
3. `major-placed` — hlavni poloosa je dana, tahne vedlejsi poloosu

Toto je slozitejsi nez `LineTool` (2 stavy) a ukazuje silu state machine patternu.

### Prakticka cast

#### Krok 1: Vytvorte skill s sablonou

Nejdrive vytvorte strukturu:

```
Vytvor nasledujici adresare a soubory:
- .claude/skills/cad-tool-generator/SKILL.md
- .claude/skills/cad-tool-generator/templates/tool-template.ts.md
```

#### Krok 2: Napiste SKILL.md

Obsah `.claude/skills/cad-tool-generator/SKILL.md`:

```yaml
---
name: cad-tool-generator
description: Generate a complete new CAD drawing tool for the Solvina engine. Handles entity type definition, tool implementation with state machine, engine registration, toolbar button, snap support, and renderer. Use when adding new shape tools like ellipse, spline, polygon, etc.
argument-hint: <tool-name> [entity-description]
allowed-tools: Read, Write, Edit, Grep, Glob, Bash(npm run lint *)
context: fork
---

# Generate CAD Tool: $ARGUMENTS

## Step 0: Understand existing patterns

Read these files to understand the architecture:

1. @packages/cad-engine/src/tools/tool-interface.ts
2. @packages/cad-engine/src/tools/line-tool.ts
3. @packages/cad-engine/src/core/types.ts
4. @packages/cad-engine/src/core/document.ts

Also read the tool template for structure guidance:

5. Read the template at ${CLAUDE_SKILL_DIR}/templates/tool-template.ts.md

## Step 1: Define entity type in types.ts

In `packages/cad-engine/src/core/types.ts`:
- Add a new interface for the entity (e.g., `EllipseEntity extends EntityBase`)
- Add it to the `Entity` union type
- Add the entity type string to `EntityType`

## Step 2: Implement the tool

Create `packages/cad-engine/src/tools/$ARGUMENTS[0]-tool.ts`:
- Follow the state machine pattern from the template
- Define states as a union type
- Implement all Tool interface methods
- Use createEntity() for entity construction
- Use addEntity() for committing to document
- Support Escape to cancel and return to idle

## Step 3: Add rendering support

In `packages/cad-engine/src/render/renderer.ts`:
- Add a case for the new entity type in the render switch
- Use ctx.beginPath(), appropriate drawing calls, ctx.stroke()
- Respect entity color via resolveEntityColor()
- Handle both fill and stroke as appropriate

## Step 4: Add snap support

In `packages/cad-engine/src/snap/snap-engine.ts`:
- Add snap points for the new entity (center, endpoints, quadrants, etc.)
- Follow the existing pattern for other entity types

## Step 5: Register in engine

In `packages/cad-engine/src/core/engine.ts`:
- Import the new tool class
- Add to the tools Map in the constructor

## Step 6: Add toolbar button

In `src/modules/cad/components/Toolbar.tsx`:
- Add a button with appropriate label/icon
- Wire to setActiveTool()

## Step 7: Verify

Run `npm run lint` to check for type errors.
Print a summary of all created/modified files.
```

#### Krok 3: Napiste sablonu nastroje

Obsah `.claude/skills/cad-tool-generator/templates/tool-template.ts.md`:

````markdown
# Tool Template

Use this structure as the basis for every new CAD tool.
Replace `__Name__` with the tool name (PascalCase) and `__name__` with lowercase.

```typescript
import { Tool, ToolContext, ToolResult, createEntity } from './tool-interface';
import { addEntity } from '../core/document';
import { Vec2 } from '../core/types';

// Define all possible states for this tool
type __Name__State = 'idle' | 'placing';
// Add more states as needed, e.g.: | 'center-placed' | 'major-placed'

export class __Name__Tool implements Tool {
  readonly type = '__name__' as const;
  private state: __Name__State = 'idle';

  // Store intermediate points as needed
  private startPoint: Vec2 | null = null;

  activate(): void {
    this.state = 'idle';
    this.startPoint = null;
  }

  deactivate(): void {
    this.cancel();
  }

  onPointerDown(ctx: ToolContext, worldPos: Vec2, _screenPos: Vec2): ToolResult {
    switch (this.state) {
      case 'idle': {
        this.startPoint = worldPos;
        this.state = 'placing';
        return { needsRedraw: true };
      }
      case 'placing': {
        // Create the entity with final parameters
        const entity = createEntity('__name__', {
          // ... entity-specific properties using this.startPoint and worldPos
        }, ctx.activeLayerId);

        const doc = addEntity(ctx.doc, entity);
        this.state = 'idle';
        this.startPoint = null;

        return { doc, needsRedraw: true };
      }
      default:
        return { needsRedraw: false };
    }
  }

  onPointerMove(ctx: ToolContext, worldPos: Vec2, _screenPos: Vec2): ToolResult {
    if (this.state === 'placing' && this.startPoint) {
      // Return preview geometry — do NOT modify the document
      return {
        toolPreview: {
          type: '__name__',
          // ... preview properties computed from this.startPoint and worldPos
        },
        needsRedraw: true,
      };
    }
    return { needsRedraw: false };
  }

  onPointerUp(_ctx: ToolContext, _worldPos: Vec2, _screenPos: Vec2): ToolResult {
    return { needsRedraw: false };
  }

  onKeyDown(_ctx: ToolContext, event: KeyboardEvent): ToolResult {
    if (event.key === 'Escape') {
      return this.cancel();
    }
    return { needsRedraw: false };
  }

  cancel(): ToolResult {
    this.state = 'idle';
    this.startPoint = null;
    return { needsRedraw: true, toolPreview: undefined };
  }
}
```
````

#### Krok 4: Pouzijte skill k vygenerovani EllipseTool

Ted vyvolejte skill:

```
/cad-tool-generator ellipse
```

Claude bezi v izolovanem kontextu (diky `context: fork`). Nacte sablonu, prozkouma existujici kod a vygeneruje kompletni EllipseTool. Sledujte, jak:

1. Prida `EllipseEntity` interface do `types.ts`
2. Vytvori `ellipse-tool.ts` se tremi stavy (`idle`, `center-placed`, `major-placed`)
3. Prida rendering elipsy do `renderer.ts` (pouzije `ctx.ellipse()`)
4. Prida snap points (center, quadrant points)
5. Zaregistruje nastroj v `engine.ts`
6. Prida tlacitko do `Toolbar.tsx`

#### Krok 5: Zkontrolujte vysledek

```
Ukazmi vysledny ellipse-tool.ts a vysvetli stavovy automat
```

Ocekavany EllipseTool by mel vypadat priblizne takto (zjednoduseno):

```typescript
import { Tool, ToolContext, ToolResult, createEntity } from './tool-interface';
import { addEntity } from '../core/document';
import { Vec2 } from '../core/types';
import { distance } from '../geometry/math';

type EllipseState = 'idle' | 'center-placed' | 'major-placed';

export class EllipseTool implements Tool {
  readonly type = 'ellipse' as const;
  private state: EllipseState = 'idle';
  private center: Vec2 | null = null;
  private majorEndpoint: Vec2 | null = null;

  activate(): void {
    this.state = 'idle';
    this.center = null;
    this.majorEndpoint = null;
  }

  deactivate(): void {
    this.cancel();
  }

  onPointerDown(ctx: ToolContext, worldPos: Vec2): ToolResult {
    switch (this.state) {
      case 'idle': {
        this.center = worldPos;
        this.state = 'center-placed';
        return { needsRedraw: true };
      }
      case 'center-placed': {
        this.majorEndpoint = worldPos;
        this.state = 'major-placed';
        return { needsRedraw: true };
      }
      case 'major-placed': {
        const semiMajor = distance(this.center!, this.majorEndpoint!);
        const angle = Math.atan2(
          this.majorEndpoint!.y - this.center!.y,
          this.majorEndpoint!.x - this.center!.x
        );
        const semiMinor = /* compute from mouse position */ 0;

        const entity = createEntity('ellipse', {
          center: this.center!,
          semiMajorAxis: semiMajor,
          semiMinorAxis: semiMinor,
          rotation: angle,
        }, ctx.activeLayerId);

        const doc = addEntity(ctx.doc, entity);
        this.state = 'idle';
        this.center = null;
        this.majorEndpoint = null;
        return { doc, needsRedraw: true };
      }
    }
  }

  onPointerMove(ctx: ToolContext, worldPos: Vec2): ToolResult {
    if (this.state !== 'idle' && this.center) {
      return {
        toolPreview: this.computePreview(worldPos),
        needsRedraw: true,
      };
    }
    return { needsRedraw: false };
  }

  // ... onPointerUp, onKeyDown, cancel, computePreview
}
```

#### Krok 6: Commitnete

```
/diff
/commit
```

### Cviceni

1. **Vygenerujte dalsi nastroj**: Pouzijte `/cad-tool-generator spline` a sledujte, jak skill zvladne uplne jiny typ entity (krivka s kontrolnimi body vs. elipsa se stredy a osami).

2. **Vytvorte skill bez `context: fork`**: Zkopirujte `cad-tool-generator` jako `cad-tool-reviewer` (bez `context: fork`), ktery misto generovani KONTROLUJE existujici nastroj na dodrzovani patternu. Nastavte `disable-model-invocation: true`, aby ho Claude nespoustel automaticky.

### Overeni

- [ ] `ls .claude/skills/cad-tool-generator/` — obsahuje `SKILL.md` a `templates/tool-template.ts.md`
- [ ] `/cad-tool-generator ellipse` vygeneruje kompletni EllipseTool
- [ ] `packages/cad-engine/src/tools/ellipse-tool.ts` existuje a ma 3 stavy
- [ ] `packages/cad-engine/src/core/types.ts` obsahuje `EllipseEntity`
- [ ] `npm run lint` projde bez chyb (nebo s minimalnimi warningy)

### Co jste pridali do Solvina

| Soubor | Zmena |
|--------|-------|
| `.claude/skills/cad-tool-generator/SKILL.md` | **Novy** — skill pro generovani CAD nastroju (propracovana verze) |
| `.claude/skills/cad-tool-generator/templates/tool-template.ts.md` | **Novy** — sablona nastroje |
| `packages/cad-engine/src/core/types.ts` | **Upraven** — `EllipseEntity` + rozsireni unionu |
| `packages/cad-engine/src/tools/ellipse-tool.ts` | **Novy** — kompletni EllipseTool se 3 stavy |
| `packages/cad-engine/src/render/renderer.ts` | **Upraven** — rendering elipsy |
| `packages/cad-engine/src/snap/snap-engine.ts` | **Upraven** — snap pro elipsu |
| `packages/cad-engine/src/core/engine.ts` | **Upraven** — registrace EllipseTool |
| `src/modules/cad/components/Toolbar.tsx` | **Upraven** — tlacitko Ellipse |

---

## Lekce 04: Subagents — Paralelni vyvoj nove feature

**Cas**: 40 min | **Obtiznost**: Advanced

### Co se naucite

- **Claude Code**: `.claude/agents/` konfigurace, paralelni delegace na subagenty, omezeni nastroju (tool restrictions), worktree izolace
- **CAD**: Pridani entity grouping — moznost seskupovat entity do skupin, s navrhem rozlozenim do 3 paralelnich subagenty

### Predpoklady

- Dokoncena Lekce 03
- Otevren Solvina projekt: `cd ~/solvina && claude`
- Git repozitar cist (`git status` ukazuje no changes) — subagent s worktree izolaci to vyzaduje

### Teorie: Claude Code

**Subagenti** jsou specializovani AI asistenti, kterym Claude deleguje ulohy. Kazdy subagent:

- Ma **vlastni kontextove okno** (nezanasi hlavni konverzaci)
- Dostane **specificky system prompt** (jeho `.md` soubor)
- Muze mit **omezene nastroje** (napr. jen Read a Grep — zadne zapisy)
- Vraci **vysledek** zpet hlavnimu agentovi, ktery ho syntetizuje

**Konfigurace subagenta** (soubor v `.claude/agents/`):

```yaml
---
name: muj-agent
description: Popis, kdy a k cemu se ma pouzit
tools: Read, Grep, Glob          # Povolene nastroje
model: sonnet                     # Volitelne: sonnet, opus, haiku
isolation: worktree               # Volitelne: vlastni git worktree
maxTurns: 20                      # Volitelne: limit poctu kroku
---

System prompt — co agent dela, jak ma pristupovat k uloham,
jake soubory ma cist, jake vzory dodrzovat.
```

**Klicove koncepty:**

| Koncept | Popis |
|---------|-------|
| **Paralelni delegace** | Claude muze spustit vice subagenty naraz |
| **Tool restrictions** | `tools: Read, Grep` = agent muze jen cist, ne zapisovat |
| **Worktree isolation** | `isolation: worktree` = agent pracuje ve vlastni git kopii |
| **`Agent(name)` syntax** | V `tools:` pole omezi, jake dalsi subagenty muze agent volat |
| **Resumable agents** | Subagent lze pozdeji obnovit a pokracovat v praci |

**Paralelni delegace v praxi**: Kdyz reknete Claude "Pridej entity grouping", muze rozdelit praci:

1. Agent A (architekt) — navrh rozhrani (read-only)
2. Agent B (geometrie) — implementace bounding box a hit-testing (read+write)
3. Agent C (UI) — tlacitka group/ungroup v toolbaru (read+write)

Vsichni bezí soucasne. Hlavni agent pak slouci vysledky.

> Podrobnosti viz [claude-howto/04-subagents/README.md](04-subagents/README.md)

### Teorie: CAD

**Entity grouping** je zakladni CAD feature — uzivatel vybere entity a seskupi je. Skupina se chova jako jedna entita (presun, kopie, mazani). Implementace vyzaduje:

1. **Datovy model**: `GroupEntity` s polem `childIds: string[]` referencing dalsich entit v dokumentu. Group muze obsahovat dalsi group (vnoreni).

2. **Bounding box**: Pro skupinu se spocita jako obalkovy obdelnik vsech potomku. Pouziva se pro select-tool hit-testing.

3. **Hit-testing**: Klik do bounding boxu skupiny vybere celou skupinu. Double-click vstoupi do skupiny (vybere potomka).

4. **UI**: Tlacitka "Group" a "Ungroup" v toolbaru. Group je aktivni kdyz je vybrano 2+ entit. Ungroup kdyz je vybrana skupina.

### Prakticka cast

#### Krok 1: Vytvorte tri subagenty

Vytvorte slozku a tri soubory:

```
Vytvor slozku .claude/agents/ a v ni tri soubory:
cad-architect.md, geometry-specialist.md, ui-implementer.md
```

#### Krok 2: cad-architect.md (read-only navrh)

Obsah `.claude/agents/cad-architect.md`:

```yaml
---
name: cad-architect
description: Designs data model changes and interfaces for the Solvina CAD engine. Use when planning new entity types, document model extensions, or architectural changes. Read-only — produces design documents, does not modify code.
tools: Read, Grep, Glob
model: sonnet
maxTurns: 15
---

# CAD Architect Agent

You are the architecture designer for the Solvina CAD engine. Your role is to
analyze the existing codebase and produce detailed interface designs.

## Your constraints

- You can ONLY READ code — you cannot create or modify files
- Your output is a DESIGN DOCUMENT — a structured markdown specification
- Other agents will implement your design, so be precise about:
  - Exact interface definitions (TypeScript types)
  - Which files need changes and what kind of changes
  - Edge cases and constraints

## Solvina architecture

- Entity types are in `packages/cad-engine/src/core/types.ts`
- Document operations in `packages/cad-engine/src/core/document.ts` (immutable)
- All entities extend `EntityBase` which has: id, type, layerId, color, lineWeight
- Entity is a discriminated union — every new type must be added to it

## Your workflow

1. Read the existing types.ts to understand EntityBase and the Entity union
2. Read document.ts to understand immutable operations
3. Read any entity types relevant to the task
4. Produce a design document with:
   - New/modified TypeScript interfaces (exact code)
   - List of files that need changes
   - Migration notes (what existing code must adapt)
   - Edge cases to handle
```

#### Krok 3: geometry-specialist.md (geometricke operace)

Obsah `.claude/agents/geometry-specialist.md`:

```yaml
---
name: geometry-specialist
description: Implements geometric algorithms for the Solvina CAD engine — bounding boxes, hit-testing, intersections, transformations. Use when adding geometry computations for new entity types or features.
tools: Read, Write, Edit, Grep, Glob
model: sonnet
maxTurns: 25
---

# Geometry Specialist Agent

You are the geometry implementation specialist for the Solvina CAD engine.
You implement mathematical algorithms for shapes, hit-testing, and spatial
operations.

## Your constraints

- You work in `packages/cad-engine/src/geometry/` and `packages/cad-engine/src/snap/`
- You may also modify `packages/cad-engine/src/core/document.ts` for new query functions
- Always use Vec2 from core/types.ts
- All functions must be pure (no side effects)
- Use EPSILON tolerance (1e-10) for floating point comparisons
- Add JSDoc with @param and @returns to every function

## Solvina geometry context

- `geometry/math.ts` — existing utility functions (distance, midpoint, etc.)
- `snap/snap-engine.ts` — snap point calculation for all entity types
- Vec2 = { x: number; y: number }

## Your workflow

1. Read the design document or task description carefully
2. Read existing geometry code to understand patterns
3. Implement new functions in the appropriate files
4. If a new file is needed, create it in geometry/ with proper imports
5. Ensure all functions handle edge cases (degenerate shapes, zero-size, etc.)
```

#### Krok 4: ui-implementer.md (React UI)

Obsah `.claude/agents/ui-implementer.md`:

```yaml
---
name: ui-implementer
description: Implements React UI components for the Solvina CAD application — toolbar buttons, panels, dialogs, keyboard shortcuts. Use when adding UI controls for new CAD features.
tools: Read, Write, Edit, Grep, Glob
model: sonnet
maxTurns: 20
---

# UI Implementer Agent

You are the React UI specialist for the Solvina CAD application.
You implement toolbar buttons, panels, and user interactions.

## Your constraints

- You work in `src/modules/cad/components/` and `src/modules/cad/index.tsx`
- You may read (but not modify) `packages/cad-engine/src/` to understand APIs
- Use React 19 patterns — functional components, hooks
- Follow existing component style in the codebase
- No external UI libraries — use the project's existing component patterns

## Solvina UI context

- `src/modules/cad/index.tsx` — CadPage, main state management
- `src/modules/cad/components/Toolbar.tsx` — tool buttons and action buttons
- `src/modules/cad/components/PropertyPanel.tsx` — entity property editing
- CadEngine is accessed via useRef + imperative handles in CadCanvas

## Your workflow

1. Read the design document or task description
2. Read existing UI components to understand patterns (Toolbar.tsx especially)
3. Implement new UI elements following existing patterns
4. Wire up events to CadEngine methods
5. Handle disabled/enabled states based on selection
```

#### Krok 5: Spustte paralelni vyvoj

Nyni pozadejte Claude, aby pouzil vsechny tri agenty:

```
Pridej do Solviny entity grouping feature. Pouzij tri agenty paralelne:

1. @"cad-architect (agent)" — Navrhni GroupEntity interface, rozsir types.ts
   a document.ts. Specifikuj presne TypeScript typy a vsechny soubory,
   ktere se musi zmenit.

2. @"geometry-specialist (agent)" — Implementuj:
   - groupBoundingBox(doc, groupEntity) — bounding box skupiny
   - groupHitTest(doc, groupEntity, point) — je bod uvnitr skupiny?
   - Pridej snap points pro skupiny do snap-engine.ts

3. @"ui-implementer (agent)" — Pridej do Toolbar.tsx:
   - Tlacitko "Group" (aktivni kdyz vybrano 2+ entit)
   - Tlacitko "Ungroup" (aktivni kdyz vybrana skupina)
   - Keyboard shortcut Ctrl+G pro group, Ctrl+Shift+G pro ungroup
```

Claude spusti tri subagenty. Kazdy bezi ve vlastnim kontextu:

- **cad-architect** cte soubory a vytvori navrh (nemeni kod)
- **geometry-specialist** implementuje geometricke funkce
- **ui-implementer** prida UI prvky

Hlavni agent pak syntetizuje vysledky. Pokud napr. geometry-specialist potrebuje typy, ktere navrhl cad-architect, hlavni agent predava informace.

#### Krok 6: Sledujte praci agenty

Behem behu muzete:
- Stisknout `Ctrl+B` pro presunuti beziciho agentu do pozadi
- Pouzit `/tasks` pro zobrazeni stavu vsech agenty
- Pockat na dokonceni — hlavni agent vas informuje o vysledcich

#### Krok 7: Integrujte vysledky

Po dokonceni vsech agenty hlavni agent:

1. Zkontroluje konzistenci — typy z cad-architect odpovidaji implementaci
2. Prida chybejici propojeni (napr. import GroupEntity v geometry souboru)
3. Shrne vsechny zmeny

Pokud neco chybi, pozadejte o doplneni:

```
Zkontroluj, ze GroupEntity je spravne pridana do Entity union v types.ts
a ze renderer.ts umi skupinu vykreslit (vykresli potomky rekurzivne).
```

#### Krok 8: Overeni a commit

```
/diff
```

Prohlednete si zmeny — meli byste videt:
- Novy `GroupEntity` interface v `types.ts`
- Nove funkce v `geometry/` (bounding box, hit-test)
- Upraveny `snap-engine.ts`
- Upraveny `Toolbar.tsx` s novymi tlacitky
- Pripadne upravy v `document.ts` (groupEntities, ungroupEntity)

```
/commit
```

### Cviceni

1. **Pridejte worktree izolaci**: Upravte `geometry-specialist.md` a pridejte `isolation: worktree`. Zkuste znovu vyvolat agenta — vsimnte si, ze pracuje ve vlastni git kopii a zmeny se musi explicitne mergovat.

2. **Vytvorte koordinacniho agenta**: Vytvorte `.claude/agents/cad-coordinator.md` s `tools: Agent(cad-architect, geometry-specialist, ui-implementer), Read, Grep`. Tento agent sam nic neimplementuje, ale koordinuje ostatni tri. Vyzkousejte ho na pridani dalsi feature (napr. entity locking — zamykani entit proti editaci).

### Overeni

- [ ] `ls .claude/agents/` — obsahuje `cad-architect.md`, `geometry-specialist.md`, `ui-implementer.md`
- [ ] `claude agents` (z prikazove radky) — zobrazuje vsechny tri agenty
- [ ] `cad-architect` ma `tools: Read, Grep, Glob` (read-only)
- [ ] `geometry-specialist` ma `tools: Read, Write, Edit, Grep, Glob` (read+write)
- [ ] V `types.ts` existuje `GroupEntity` s `childIds: string[]`
- [ ] V `geometry/` existuje funkce pro bounding box skupiny
- [ ] V `Toolbar.tsx` existuji tlacitka Group a Ungroup
- [ ] `npm run lint` projde (nebo s minimalnimi warningy)

### Co jste pridali do Solvina

| Soubor | Zmena |
|--------|-------|
| `.claude/agents/cad-architect.md` | **Novy** — read-only architekturni agent |
| `.claude/agents/geometry-specialist.md` | **Novy** — geometricky agent (read+write) |
| `.claude/agents/ui-implementer.md` | **Novy** — UI agent (read+write) |
| `packages/cad-engine/src/core/types.ts` | **Upraven** — GroupEntity interface + Entity union |
| `packages/cad-engine/src/core/document.ts` | **Upraven** — groupEntities(), ungroupEntity() |
| `packages/cad-engine/src/geometry/group-math.ts` | **Novy** — groupBoundingBox(), groupHitTest() |
| `packages/cad-engine/src/snap/snap-engine.ts` | **Upraven** — snap points pro skupiny |
| `src/modules/cad/components/Toolbar.tsx` | **Upraven** — Group/Ungroup tlacitka + Ctrl+G shortcut |

---

## Lekce 05: MCP — Napojeni na externi nastroje
**Cas**: 30 min | **Obtiznost**: Intermediate

### Co se naucite
- **Claude Code**: Konfigurace MCP serveru (`.mcp.json`, `claude mcp add`), scopes (local/project/user), MCP prompts jako slash commands, pouziti env vars
- **CAD**: Napojeni Solvina na souborovy system pro cteni vzorkovych DXF souboru a na GitHub pro sledovani issues projektu

### Predpoklady
- Dokoncena lekce 04
- Otevren Solvina projekt: `cd ~/solvina && claude`
- Nastaveny `GITHUB_TOKEN` (pro GitHub MCP): `export GITHUB_TOKEN="ghp_..."`

### Teorie: Claude Code

MCP (Model Context Protocol) je standardizovany zpusob, jakym Claude pristupuje k externim nastrojum, API a zdrojum dat v realnem case. Na rozdil od CLAUDE.md (staticka pamet) poskytuje MCP zivy pristup k menicimu se obsahu.

**Tri klicove koncepty:**

1. **MCP Server** — externi sluzba, ktera nabizi nastroje (tools), zdroje (resources) a prompty
2. **Transport** — komunikacni protokol: `http`, `stdio` (lokalni), `sse` (deprecated), `ws`
3. **Scope** — kde se konfigurace uklada a kdo ji vidi

**Scopes MCP konfigurace:**

| Scope | Soubor | Sdileny s | Poznamka |
|-------|--------|-----------|----------|
| **Local** (default) | `~/.claude.json` (pod cestou projektu) | Pouze vy | Soukrome, jen pro vas |
| **Project** | `.mcp.json` v root projektu | Cely tym | Commituje se do gitu, vyzaduje souhlas pri prvnim pouziti |
| **User** | `~/.claude.json` | Pouze vy | Dostupne ve vsech projektech |

**Pridani MCP serveru pres CLI:**

```bash
claude mcp add --transport http github https://api.github.com/mcp  # HTTP (vzdalene)
claude mcp add --transport stdio fs -- npx @anthropic/mcp-fs       # Stdio (lokalni)
claude mcp add --transport stdio srv --env KEY=val -- npx @org/srv # S env vars
claude mcp list                                                     # Vypis serveru
claude mcp remove github                                            # Odebrani
```

**MCP prompts jako slash commands** se volaji ve formatu `/mcp__<server>__<prompt>` — napr. `/mcp__github__review`.

**Tool Search** se aktivuje automaticky, kdyz pocet MCP nastroju prekroci 10% kontextoveho okna.

> Podrobnosti viz [claude-howto/05-mcp/README.md](05-mcp/README.md)

### Teorie: CAD

V realnem CAD workflow potrebujete:

1. **Pristup k vzorkovym souborum** — testovaci DXF soubory v adresari `~/solvina/samples/` pro overovani importu
2. **Sledovani issues** — CAD-specificke bugy (spatny import vrstev, chybne snap body) evidovane na GitHubu
3. **Analyza externich formatu** — Claude muze s pomoci MCP precist raw DXF soubor a analyzovat jeho strukturu jeste pred implementaci parsovani

Solvina ma nasledujici IO moduly v `packages/cad-engine/src/io/`:
- `dxf-import.ts` — import z DXF formatu (pouziva `dxf-parser`)
- `dxf-export.ts` — export do DXF
- `svg-export.ts` — export do SVG

### Prakticka cast

#### Krok 1: Vytvorte adresar se vzorkovymi DXF soubory

Nejprve pripravte testovaci data. Vytvorte jednoduchy DXF soubor pro testovani:

```
Vytvor adresar ~/solvina/samples/ a do nej soubor test-rectangle.dxf
s minimalnim platnym DXF obsahem — obdelnik 100x50 na vrstve "Walls".
```

#### Krok 2: Konfigurace Filesystem MCP (project scope)

Vytvorte `.mcp.json` v root projektu Solvina. Tento soubor se commituje do gitu, takze ho uvidite cely tym:

```
Vytvor .mcp.json v rootu projektu s touto konfiguraci:
```

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@anthropic/mcp-fs",
        "${HOME}/solvina/samples"
      ]
    }
  }
}
```

> **Poznamka**: Pouzivame `${HOME}` misto hardcoded cesty — funguje na vsech platformach. Na Windows se `${HOME}` rozvine na `%USERPROFILE%`.

#### Krok 3: Pridani GitHub MCP pres CLI

GitHub MCP pridame pres prikazovou radku s tokenem jako env promennou:

```bash
claude mcp add --transport stdio github \
  --env GITHUB_TOKEN="${GITHUB_TOKEN}" \
  -- npx -y @modelcontextprotocol/server-github
```

Overeni, ze je server pridany:

```bash
claude mcp list
```

Vystup by mel zobrazit oba servery — `filesystem` (z `.mcp.json`) i `github` (z lokalni konfigurace).

#### Krok 4: Pouzijte Filesystem MCP k analyze DXF

Spustte Claude a pouzijte MCP k precteni a analyze vzorkoveho DXF:

```
Precti soubor test-rectangle.dxf z MCP filesystem serveru
a analyzuj jeho strukturu. Kolik entit obsahuje?
Na jakych vrstvach lezi? Jaky je bounding box?
```

Claude pouzije MCP nastroj `read_file` k precteni souboru a pak analyzuje DXF strukturu — hlavicku, sekce, entity. Vysledek pouzijte k overeni, ze vas DXF import v `dxf-import.ts` spravne zpracovava vsechny typy entit.

#### Krok 5: Pouzijte GitHub MCP pro sledovani CAD issues

```
Vytvor GitHub issue v repozitari solvina s nazvem
"DXF Import: Podpora pro LWPOLYLINE entity s oblouky"
a popisem: Aktualni DXF import nepodporuje segmenty s bulge
hodnotou v LWPOLYLINE. Pri importu souboru s oblouky
se segmenty importuji jako prime usecky.
Pridej label "cad-engine" a "enhancement".
```

Pro zobrazeni existujicich issues:

```
Zobraz vsechny otevrene issues s labelem "cad-engine"
v repozitari solvina.
```

#### Krok 6: Kombinovany workflow — MCP + analyza

Tady je sila MCP: muzete kombinovat data z ruznych zdroju v jednom dotazu:

```
1. Precti DXF soubor test-rectangle.dxf pres filesystem MCP
2. Podivej se na otevrene issues s labelem "cad-engine" na GitHubu
3. Navrh, ktere issues by slo vyresit na zaklade toho,
   co vidis v DXF souboru a aktualnim kodu dxf-import.ts
```

### Cviceni

1. **Vlastni DXF vzorek**: Vytvorte slozitejsi DXF soubor (kruznice, texty, vice vrstev) a nechte Claude analyzovat, ktere typy entit Solvina jiz podporuje a ktere chybi.

2. **MCP scope experiment**: Pridejte filesystem MCP v user scope (`claude mcp add --scope user ...`) a overete, ze je dostupny i z jineho projektu. Pak ho odeberte: `claude mcp remove --scope user filesystem`.

3. **GitHub workflow**: Vytvorte issue, nechte Claude napsat opravu kodu, a pak nechte Claude vytvorit PR — vse pomoci MCP nastroju v jedne konverzaci.

### Overeni

- [ ] `.mcp.json` existuje v root projektu a obsahuje `filesystem` server
- [ ] `claude mcp list` zobrazuje oba servery (filesystem + github)
- [ ] Claude uspesne cte DXF soubory pres Filesystem MCP
- [ ] Claude uspesne vytvari/cte issues pres GitHub MCP
- [ ] Env promenne (`${HOME}`, `${GITHUB_TOKEN}`) se spravne rozvijeji

### Co jste pridali do Solvina

| Soubor | Ucel |
|--------|------|
| `.mcp.json` | Projektova MCP konfigurace (filesystem server pro vzorky) |
| `samples/test-rectangle.dxf` | Vzorkovy DXF soubor pro testovani importu |

---

## Lekce 06: Hooks — Automaticke testy a kvalita kodu
**Cas**: 40 min | **Obtiznost**: Intermediate

### Co se naucite
- **Claude Code**: Hook eventy (PreToolUse, PostToolUse, Stop, SessionStart), matcher patterny, typy hooku (command, prompt, agent), JSON I/O
- **CAD**: Automaticke spousteni geometry testu po zmenach kodu, validace matematickych operaci v geometrickem kodu (zadne porovnavani floatu pomoci `===`)

### Predpoklady
- Dokoncena lekce 05
- Otevren Solvina projekt: `cd ~/solvina && claude`
- Nainstalovany `vitest` (Solvina ho pouziva pro testy)

### Teorie: Claude Code

Hooks jsou automatizovane akce, ktere se spousteji pri specifickych udalostech v Claude Code session. Prijimajidata pres JSON stdin a komunikuji vysledky pres exit kody a JSON stdout.

**Hook eventy (vybrane):**

| Event | Kdy se spusti | Matcher input | Muze blokovat |
|-------|---------------|---------------|---------------|
| **SessionStart** | Start/resume session | startup/resume/clear/compact | Ne |
| **PreToolUse** | Pred spustenim nastroje | Nazev nastroje | Ano (allow/deny/ask) |
| **PostToolUse** | Po uspesnem dokonceni nastroje | Nazev nastroje | Ne (ale muze pridat kontext) |
| **Stop** | Claude dokonci odpoved | (zadny) | Ano |

**Typy hooku:**

| Typ | Popis | Pouziti |
|-----|-------|---------|
| `command` | Shell prikaz (Node.js skript) | Validace, testy, linting |
| `prompt` | LLM vyhodnoceni promptu | Inteligentni kontrola dokonceni ukolu |
| `agent` | Spusti subagenta s nastroji | Slozite vicerokove overeni |
| `http` | Webhook na externi URL | Notifikace, externi systemy |

**Konfigurace** v `.claude/settings.json` (projektovy) nebo `~/.claude/settings.json` (uzivatelsky). Priklad:

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "node $CLAUDE_PROJECT_DIR/.claude/hooks/run-tests.js",
        "timeout": 120
      }]
    }]
  }
}
```

**Matchery:** presny retezec (`"Write"`), regex (`"Edit|Write"`), wildcard (`"*"`), MCP (`"mcp__github__.*"`).

**Exit kody:** `0` = uspech (parsuje JSON stdout), `2` = blokovaci chyba, ostatni = neblokovaci.

**Env promenne:** `CLAUDE_PROJECT_DIR` (root projektu), `CLAUDE_ENV_FILE` (pro SessionStart).

> Podrobnosti viz [claude-howto/06-hooks/README.md](06-hooks/README.md)

### Teorie: CAD

Geometricky kod ma dve specificka rizika:

1. **Regrese v geometrickych vypoctech** — zmena v `intersect()` nebo `snap-engine.ts` muze rozbit desitky navaznych operaci. Kazda zmena musi okamzite projit testy.

2. **Porovnavani floatu** — v geometrii nesmite nikdy pouzit `===` pro porovnani cisel s plovouci radovou carkou. Spravne je `Math.abs(a - b) < EPSILON`. Tento pattern je tak castou chybou, ze ho chceme zachytit automaticky.

Solvina ma testy v `packages/cad-engine/src/__tests__/` a pouzivaVitest. Klicove testovaci soubory:
- `geometry.test.ts` — testy geometrickych operaci
- `snap.test.ts` — testy snap systemu
- `document.test.ts` — testy dokumentoveho modelu

### Prakticka cast

#### Krok 1: Vytvorte hook pro automaticke spousteni geometry testu

Kdyz Claude zmeni jakykoliv soubor v CAD engine pomoci `Write` nebo `Edit`, chceme automaticky spustit geometricke testy. Hook je napsany v TypeScript a spousten pres `npx tsx`:

```
Vytvor soubor .claude/hooks/run-geometry-tests.ts s timto obsahem:
```

```typescript
import { execSync } from "child_process";
import { readFileSync } from "fs";

// Precteme JSON input ze stdin
const chunks: Buffer[] = [];
for await (const chunk of process.stdin) {
  chunks.push(chunk);
}
const input = JSON.parse(Buffer.concat(chunks).toString());

const toolInput = input.tool_input || {};
const filePath: string = toolInput.file_path || "";

// Spustime testy jen pro zmeny v CAD engine
const cadEnginePath = "packages/cad-engine/src/";
if (!filePath.includes(cadEnginePath)) {
  // Zmena mimo CAD engine — neni treba spoustet testy
  const output = { continue: true };
  process.stdout.write(JSON.stringify(output));
  process.exit(0);
}

try {
  // Spustime geometricke testy pres vitest
  const result = execSync(
    "npx vitest run --reporter=json packages/cad-engine/src/__tests__/geometry.test.ts",
    {
      cwd: input.cwd || process.cwd(),
      timeout: 60_000,
      encoding: "utf-8",
      stdio: ["pipe", "pipe", "pipe"],
    }
  );

  const output = {
    continue: true,
    hookSpecificOutput: {
      hookEventName: "PostToolUse",
      additionalContext: "Geometry testy prosly uspesne.",
    },
  };
  process.stdout.write(JSON.stringify(output));
  process.exit(0);
} catch (err: any) {
  // Testy selhaly — informujeme Claude, ale neblokujeme
  const stderr = err.stderr?.toString() || err.message;
  const output = {
    continue: true,
    hookSpecificOutput: {
      hookEventName: "PostToolUse",
      additionalContext: `VAROVANI: Geometry testy selhaly!\n${stderr}\nOprav selhavajici testy pred pokracovanim.`,
    },
  };
  process.stdout.write(JSON.stringify(output));
  process.exit(0);
}
```

> **Poznamka k multiplatformite**: Pouzivame `npx tsx` pro spousteni TypeScript a `node` pro JavaScript. Oba prikazy funguji na Windows, macOS i Linuxu bez uprav. Nepouzivame bash skripty, ktere by na Windows vyzadovaly WSL.

#### Krok 2: Vytvorte hook pro validaci matematickych operaci

Tento PreToolUse hook kontroluje, ze Claude nepise kod s primym porovnanim floatu (`=== ` nebo `!==` na ciselne promenne):

```
Vytvor soubor .claude/hooks/validate-math.ts s timto obsahem:
```

```typescript
// validate-math.ts — PreToolUse hook
// Kontroluje, ze se v geometrickem kodu nepouziva === pro porovnani floatu

const chunks: Buffer[] = [];
for await (const chunk of process.stdin) {
  chunks.push(chunk);
}
const input = JSON.parse(Buffer.concat(chunks).toString());

const toolName: string = input.tool_name || "";
const toolInput = input.tool_input || {};

// Zajima nas jen Write a Edit v geometrickem kodu
if (toolName !== "Write" && toolName !== "Edit") {
  process.stdout.write(JSON.stringify({ continue: true }));
  process.exit(0);
}

const filePath: string = toolInput.file_path || "";
const geometryPaths = [
  "packages/cad-engine/src/core/",
  "packages/cad-engine/src/snap/",
  "packages/cad-engine/src/tools/",
];
if (!geometryPaths.some((p) => filePath.includes(p))) {
  process.stdout.write(JSON.stringify({ continue: true }));
  process.exit(0);
}

// Zkontrolujeme obsah — hledame patterny jako: someFloat === otherFloat
const content: string = toolInput.content || toolInput.new_string || "";
const floatCompareRegex =
  /(?<!\btypeof\b\s*\w+\s*)(\w+)\s*[!=]==\s*(\w+)/g;
const safeValues = new Set(["null", "undefined", "true", "false", "0", "NaN"]);
const issues: string[] = [];
let match: RegExpExecArray | null;

while ((match = floatCompareRegex.exec(content)) !== null) {
  const [fullMatch, left, right] = match;
  if (safeValues.has(right) || safeValues.has(left)) continue;
  if (/['"`]/.test(right) || /['"`]/.test(left)) continue;
  issues.push(fullMatch);
}

if (issues.length > 0) {
  const output = {
    continue: true,
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason:
        `VAROVANI: Prime float porovnani: ${issues.join(", ")}. ` +
        `Pouzijte Math.abs(a - b) < EPSILON (viz core/types.ts).`,
    },
  };
  process.stdout.write(JSON.stringify(output));
} else {
  process.stdout.write(JSON.stringify({
    continue: true,
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "allow",
      permissionDecisionReason: "Zadne problematicke float porovnani.",
    },
  }));
}
process.exit(0);
```

#### Krok 3: Konfigurace hooku v settings.json

```
Vytvor nebo uprav .claude/settings.json s konfiguraci hooku:
```

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "npx tsx $CLAUDE_PROJECT_DIR/.claude/hooks/validate-math.ts",
        "timeout": 10
      }]
    }],
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "npx tsx $CLAUDE_PROJECT_DIR/.claude/hooks/run-geometry-tests.ts",
        "timeout": 120
      }]
    }],
    "SessionStart": [{
      "matcher": "startup",
      "hooks": [{
        "type": "command",
        "command": "node -e \"const o={continue:true,hookSpecificOutput:{hookEventName:'SessionStart',additionalContext:'CAD hooks aktivni.'}};process.stdout.write(JSON.stringify(o))\"",
        "timeout": 5
      }]
    }]
  }
}
```

#### Krok 4: Otestujte hooks v praxi

Spustte novou Claude session a zkuste zmenu v geometrickem kodu:

```
Pridej do packages/cad-engine/src/core/types.ts novou
utility funkci nearlyEqual(a: number, b: number): boolean
ktera porovnava dve cisla s toleranci EPSILON.
```

Pak zaructe, ze se hooks spusti:

```
Zmen snap-engine.ts tak, aby pouzival nearlyEqual()
misto primeho porovnani vzdalenosti.
```

Pri zmene byste v terminalu meli videt:
1. **PreToolUse** hook overi, ze nove napsany kod nepouziva `===` na floatech
2. **PostToolUse** hook automaticky spusti geometry testy a zobrazi vysledek

#### Krok 5: Prompt hook pro overeni dokonceni

Pro pokrocilejsi pouziti — pridejte `Stop` hook s prompt typem, ktery overi, ze Claude nezapomnel spustit testy:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Zkontrolu, zda Claude spustil geometricke testy po vsech zmenach v CAD engine kodu. Pokud ne, pozadej o jejich spusteni.",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

Tento hook pouziva LLM k inteligentnimu rozhodnuti, zda je uloha skutecne dokoncena.

### Cviceni

1. **Snap test hook**: Rozsirte `run-geometry-tests.ts` tak, aby pri zmenach v `snap/` spoustel i `snap.test.ts`, ne jen `geometry.test.ts`.

2. **Import validace**: Vytvorte PostToolUse hook, ktery pri zmene souboru v `io/` automaticky spusti test importu/exportu DXF.

3. **Agent hook**: Nahradte prompt hook v `Stop` agentem (`"type": "agent"`), ktery muze pouzivat nastroje k overeni, ze vsechny zmenene soubory maji odpovidajici testy.

### Overeni

- [ ] `.claude/settings.json` obsahuje konfiguraci PreToolUse, PostToolUse, SessionStart
- [ ] Oba hook skripty jsou v TypeScript/Node.js (zadny bash, zadny Python)
- [ ] Pri zmene v `packages/cad-engine/src/core/` se automaticky spusti testy
- [ ] Pri pokusu napsat `a === b` v geometrickem kodu hook to odmitne

### Co jste pridali do Solvina

| Soubor | Ucel |
|--------|------|
| `.claude/settings.json` | Konfigurace hooku (PreToolUse, PostToolUse, SessionStart) |
| `.claude/hooks/run-geometry-tests.ts` | PostToolUse hook — spousti geometricke testy po zmenach |
| `.claude/hooks/validate-math.ts` | PreToolUse hook — blokuje prime float porovnani v geom. kodu |

---

## Lekce 07: Plugins — Zabaleni CAD-specifickych Claude nastroju
**Cas**: 35 min | **Obtiznost**: Advanced

### Co se naucite
- **Claude Code**: Struktura pluginu (`.claude-plugin/plugin.json`), baleni skills + agents + hooks, distribuce, uzivatelska konfigurace (userConfig), `/reload-plugins`
- **CAD**: Zabaleni vsech artefaktu z lekci 02-06 do jednoho distribuovatelneho `cad-dev-plugin`, ktery muze pouzit kazdy clen tymu

### Predpoklady
- Dokonceny lekce 02-06 (skills, subagents, MCP, hooks)
- Otevren Solvina projekt: `cd ~/solvina && claude`

### Teorie: Claude Code

Plugin je balicek, ktery sdruzuje vice Claude Code artefaktu do jednoho instalovatelneho celku:

- **Skills** (slash commands) — markdown soubory v `commands/`
- **Agents** (subagenti) — markdown soubory v `agents/`
- **Hooks** — konfigurace v `hooks/hooks.json`
- **MCP servery** — konfigurace v `.mcp.json`
- **Settings** — vychozi nastaveni v `settings.json`

**Manifest** je soubor `.claude-plugin/plugin.json` (name, description, version, author, license).

**Struktura:** `commands/` (skills), `agents/` (subagenti), `hooks/hooks.json` (hook konfigurace), `.mcp.json` (MCP servery), `settings.json` (vychozi nastaveni), `scripts/` (pomocne skripty).

**Standalone vs plugin:** Standalone (`/my-command`) je pro osobni pouziti s manualni distribuci. Plugin (`/plugin-name:my-command`) se instaluje jednim prikazem a je vhodny pro tymy.

**Uzivatelska konfigurace (userConfig):** Pluginy mohou deklarovat parametry v manifestu. Hodnoty s `"sensitive": true` se ukladaji do systemove klicenky (ne do plaintextu). Ukazka viz Krok 2 nize.

**Persistentni data:** `${CLAUDE_PLUGIN_DATA}` je adresar pro cache a stav, ktery prezije mezi sessions.

**Vyvoj:** Behem uprav pouzijte `/reload-plugins` pro nacetni zmen bez restartu.

> Podrobnosti viz [claude-howto/07-plugins/README.md](07-plugins/README.md)

### Teorie: CAD

V predchozich lekcich jste vytvorili:

| Lekce | Artefakt | Typ |
|-------|----------|-----|
| 02 | CLAUDE.md s CAD konvencemi | Memory |
| 03 | `/dxf-analyze`, `/layer-report` skills | Skills |
| 04 | `geometry-reviewer`, `dxf-specialist` subagenti | Agents |
| 05 | `.mcp.json` s filesystem a GitHub MCP | MCP |
| 06 | `validate-math.ts`, `run-geometry-tests.ts` hooks | Hooks |

Tyto artefakty jsou rozprosteny po projektu. Kdyz novy clen tymu zacne pracovat na Solvina, musi rucne kopirovat a nastavovat kazdy kousek. Plugin vse zabali do jednoho celku.

### Prakticka cast

#### Krok 1: Vytvorte adresarovou strukturu pluginu

```
Vytvor nasledujici adresarovou strukturu v ~/solvina/cad-dev-plugin/:

cad-dev-plugin/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   ├── dxf-analyze.md
│   └── layer-report.md
├── agents/
│   ├── geometry-reviewer.md
│   └── dxf-specialist.md
├── hooks/
│   ├── hooks.json
│   ├── run-geometry-tests.ts
│   └── validate-math.ts
├── .mcp.json
└── settings.json
```

#### Krok 2: Vytvorte manifest pluginu

```
Vytvor soubor cad-dev-plugin/.claude-plugin/plugin.json:
```

```json
{
  "name": "cad-dev-plugin",
  "description": "Claude Code plugin pro vyvoj Solvina CAD engine. Obsahuje skills pro analyzu DXF, subagenty pro code review geometrie, hooks pro automaticke testy a MCP napojeni na vzorkove soubory.",
  "version": "1.0.0",
  "author": {
    "name": "Solvina Team"
  },
  "repository": "https://github.com/solvina-team/cad-dev-plugin",
  "license": "MIT",
  "userConfig": {
    "githubToken": {
      "description": "GitHub Personal Access Token pro MCP pristup k issues a PR",
      "sensitive": true
    },
    "samplesDir": {
      "description": "Cesta k adresari se vzorkovymi DXF soubory",
      "default": "${HOME}/solvina/samples"
    },
    "autoRunTests": {
      "description": "Automaticky spoustet geometricke testy po zmenach (true/false)",
      "default": "true"
    }
  }
}
```

#### Krok 3: Presunte skills do pluginu

Prekopirujte skills z lekce 03 do `commands/`:

```
Vytvor soubor cad-dev-plugin/commands/dxf-analyze.md:
```

```markdown
---
name: DXF Analyze
description: Analyzuje DXF soubor a zobrazi strukturu entit, vrstev a metadat
---

# DXF Analyze

Analyzuj zadany DXF soubor a vytvor strukturovany report:
1. **Hlavicka**: Verze DXF, jednotky, rozsah vykresu
2. **Vrstvy**: Seznam vsech vrstev s poctem entit na kazde
3. **Entity**: Rozdeleni podle typu (LINE, CIRCLE, ARC, LWPOLYLINE, TEXT, ...)
4. **Bounding box**: Celkovy rozsah vykresu (min/max souradnice)
5. **Potencialni problemy**: Duplicitni entity, prazdne vrstvy, entity mimo rozsah

Pokud je k dispozici Filesystem MCP, precti soubor pres nej.
Vystup formatuj jako prehlednou tabulku.
```

Obdobne vytvorte `layer-report.md` — skill z lekce 03, ktery analyzuje vrstvy v CadDocument (pocty entit, barvy, styl cary, prazdne/neviditelne vrstvy).

#### Krok 4: Presunte agenty do pluginu

```
Vytvor soubor cad-dev-plugin/agents/geometry-reviewer.md:
```

```markdown
---
name: geometry-reviewer
description: Specializovany subagent pro review geometrickych operaci v CAD engine
tools: read, grep, glob
---

# Geometry Reviewer

Jsi expert na 2D vypocetni geometrii. Kontroluj kod v packages/cad-engine/src/:

1. **Numericka stabilita**: EPSILON pro porovnani floatu, nikdy ===
2. **Hranicni pripady**: Usecka nulove delky, kolinerani body, rovnobezne primky
3. **Konzistence s Vec2**: Zadne volne {x, y} objekty
4. **Imutabilita**: Funkce v document.ts nesmi mutovat vstupni entity
5. **Testovaci pokryti**: Kazda geometricka funkce ma test v __tests__/

Pro kazdy problem: soubor + radek, popis, navrh opravy.
```

Obdobne vytvorte `dxf-specialist.md` — obsah prevezmente z lekce 04.
Klicove body agenta: znalost DXF formatu (skupinove kody, sekce HEADER/TABLES/BLOCKS/ENTITIES), pouziti knihovny `dxf-parser`, respektovani typu v `core/types.ts`, round-trip kompatibilita DXF importu/exportu.

#### Krok 5: Konfigurace hooku v pluginu

Hooks v pluginu se konfigruji v `hooks/hooks.json` (ne v `.claude/settings.json`):

```
Vytvor soubor cad-dev-plugin/hooks/hooks.json:
```

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "npx tsx ${CLAUDE_PLUGIN_ROOT}/hooks/validate-math.ts",
            "timeout": 10
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "npx tsx ${CLAUDE_PLUGIN_ROOT}/hooks/run-geometry-tests.ts",
            "timeout": 120
          }
        ]
      }
    ]
  }
}
```

> **Dulezite**: V plugin hooks pouzivame `${CLAUDE_PLUGIN_ROOT}` misto `$CLAUDE_PROJECT_DIR`. Tato promenna ukazuje na root adresar pluginu, takze skripty funguji nezavisle na tom, kde je plugin nainstalovany.

Pak prekopirujte hook skripty z lekce 06:

```
Zkopiruj .claude/hooks/run-geometry-tests.ts
do cad-dev-plugin/hooks/run-geometry-tests.ts

Zkopiruj .claude/hooks/validate-math.ts
do cad-dev-plugin/hooks/validate-math.ts
```

#### Krok 6: MCP a settings

Vytvorte `.mcp.json` s filesystem MCP serverem (stejny jako v lekci 05, ale s nazvem `cad-samples`) a `settings.json` s vychozim agentem:

```json
// cad-dev-plugin/.mcp.json
{
  "mcpServers": {
    "cad-samples": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-fs", "${HOME}/solvina/samples"]
    }
  }
}
```

```json
// cad-dev-plugin/settings.json
{ "agent": "agents/geometry-reviewer.md" }
```

#### Krok 8: Instalace a testovani

Pro lokalni vyvoj a testovani:

```bash
# Instalace pluginu z lokalniho adresare
/plugin install ~/solvina/cad-dev-plugin

# Overeni instalace
/plugin list
```

Po instalaci budou dostupne:
- `/cad-dev-plugin:dxf-analyze` — slash command pro analyzu DXF
- `/cad-dev-plugin:layer-report` — slash command pro report vrstev
- `geometry-reviewer` a `dxf-specialist` subagenti
- Hooks pro validaci matematiky a automaticke testy
- MCP pristup k vzorkovym souborum

#### Krok 9: Vyvojovy cyklus

Behem uprav pluginu pouzijte `/reload-plugins` pro nacetni zmen bez restartu session.

#### Krok 10: Priprava pro distribuci

Commitujte plugin do git repositare. Clenove tymu pak nainstalluji jednim prikazem:

```bash
/plugin install https://github.com/solvina-team/cad-dev-plugin
```

Pro enterprise tymy je mozne vytvorit privatni plugin marketplace — viz `pluginMarketplaces` a `strictKnownMarketplaces` v nastaveni.

### Cviceni

1. **Novy skill**: Pridejte do pluginu skill `/cad-dev-plugin:snap-debug`, ktery analyzuje snap-engine.ts a zobrazi vsechny registrovane snap typy s jejich prioritami.

2. **LSP integrace**: Pridejte do pluginu `.lsp.json` s konfiguraci TypeScript language serveru pro lepsipodporu pri editaci CAD engine kodu.

3. **Persistent state**: Vyuzijte `${CLAUDE_PLUGIN_DATA}` k ukladani statistik — kolikrat se spustily testy, kolik float porovnani se zachytilo, jaka je prumerna doba behu testu.

4. **userConfig v praxi**: Upravte `run-geometry-tests.ts` tak, aby respektoval nastaveni `autoRunTests` z userConfig — pokud je `"false"`, hook se preskoci.

### Overeni

- [ ] `plugin.json` obsahuje validni manifest s userConfig
- [ ] `commands/` a `agents/` obsahuji skills a subagenty z lekci 03-04
- [ ] `hooks/hooks.json` pouziva `${CLAUDE_PLUGIN_ROOT}` (ne `$CLAUDE_PROJECT_DIR`)
- [ ] `/plugin install ~/solvina/cad-dev-plugin` probehne bez chyb
- [ ] `/cad-dev-plugin:dxf-analyze` je dostupny jako slash command

### Co jste pridali do Solvina

| Soubor | Ucel |
|--------|------|
| `cad-dev-plugin/.claude-plugin/plugin.json` | Manifest pluginu s metadaty a userConfig |
| `cad-dev-plugin/commands/dxf-analyze.md` | Skill: analyza DXF souboru |
| `cad-dev-plugin/commands/layer-report.md` | Skill: report o vrstvach |
| `cad-dev-plugin/agents/geometry-reviewer.md` | Subagent: review geometrickeho kodu |
| `cad-dev-plugin/agents/dxf-specialist.md` | Subagent: DXF format specialista |
| `cad-dev-plugin/hooks/hooks.json` | Konfigurace hooku pluginu |
| `cad-dev-plugin/hooks/run-geometry-tests.ts` | Hook: automaticke spousteni geometry testu |
| `cad-dev-plugin/hooks/validate-math.ts` | Hook: validace float porovnani |
| `cad-dev-plugin/.mcp.json` | MCP konfigurace: pristup k vzorkovym DXF |
| `cad-dev-plugin/settings.json` | Vychozi nastaveni pluginu |

---

## Lekce 08: Checkpoints — Bezpecne experimentovani

| Parametr | Hodnota |
|---|---|
| Cas | 35 minut |
| Obtiznost | Stredni |
| Predpoklady | Lekce 01-07, funkcni Solvina s CAD renderem |

### Co se naucite

- Jak funguji automaticke checkpointy v Claude Code
- Pouziti `/rewind` a `Esc+Esc` pro navrat k predchozimu stavu
- Pet moznosti obnoveni pri rewindu
- Branching pattern pro bezpecne porovnavani pristupu
- Experimentovani s architekturou rendereru bez strachu ze ztraty kodu

### Predpoklady

- Solvina projekt s funkcnim Canvas 2D rendererem (`packages/cad-engine/src/render/`)
- Git repozitar s cistyma commitama z predchozich lekci
- Claude Code s aktivnimi checkpointy (vychozi nastaveni)

---

### Teorie

#### Claude Code: Checkpoint system

Kazdy prompt, ktery odesleme do Claude Code, automaticky vytvori checkpoint — snimek
stavu konverzace i souboru. Checkpointy se uchovavaji 30 dni a nevyzaduji zadnou
manualni akci.

**Pristup k checkpointum:**

| Metoda | Popis |
|---|---|
| `Esc + Esc` | Otevre prohlizec checkpointu primo v terminalu |
| `/rewind` | Slash command se stejnou funkci |
| `/checkpoint` | Alias pro `/rewind` |

**Pet moznosti pri rewindu:**

1. **Restore code and conversation** — Vrati soubory i konverzaci na zvoleny bod.
   Pouzijte, kdyz chcete uplne zahodit vsechny zmeny od checkpointu.
2. **Restore conversation** — Vrati konverzaci, ale necha aktualni soubory beze zmen.
   Uzitecne, kdyz jste rucne upravili kod a chcete jen resetovat kontext.
3. **Restore code** — Vrati soubory, ale zachova celou historii konverzace.
   Pouzijte, kdyz chcete videt, co Claude udelal, ale chcete puvodni kod.
4. **Summarize from here** — Komprimuje konverzaci od tohoto bodu do AI souhrnu.
   Setri kontext pri dlouhych sezenich. Puvodni zpravy zustanou v transkriptu.
5. **Never mind** — Zrusi akci a vrati se do aktualniho stavu.

**Dulezite omezeni:**

- Checkpointy nezachycuji bash operace (`rm`, `mv`, `cp`)
- Externi zmeny (editor, terminal) nejsou sledovany
- Checkpointy NEnahrazuji git — pouzivejte oba spolecne

#### CAD kontext: Strategie renderovani

Nas Canvas 2D renderer v `packages/cad-engine/src/render/` je funkcni, ale pro velke
vykresy (tisice entit) muze byt pomaly. Existuji alternativni pristupy:

- **OffscreenCanvas** — renderovani ve Web Workeru pro uvolneni hlavniho vlakna
- **WebGL** — hardwarova akcelerace, ale slozitejsi implementace pro 2D CAD
- **Dirty-region rendering** — prekreslovani pouze zmenenych oblasti platna

Dnes vyuzijeme checkpointy k bezpecnemu experimentovani s ruznymi pristupy.

---

### Prakticka cast

#### Krok 1: Overeni vychoziho stavu

Pred experimentovanim udelejte git commit, aby byl vas aktualni stav bezpecne ulozen:

```
> git add -A && git commit -m "checkpoint: stable Canvas 2D renderer before experiments"
```

Pak otevrete Claude Code v adresari Solvina:

```bash
cd ~/solvina
claude
```

Ovezte, ze renderer funguje:

```
> Spust testy pro render/ modul a ukazat mi aktualni strukturu rendereru.
```

Claude zobrazi strukturu souboru v `packages/cad-engine/src/render/` a vysledky testu.
Toto je vas vychozi checkpoint A.

#### Krok 2: Experiment 1 — OffscreenCanvas

Pozadejte Claude o refaktoring na OffscreenCanvas:

```
> Refaktoruj renderer v packages/cad-engine/src/render/ tak, aby pouzival
> OffscreenCanvas ve Web Workeru. Entity se budou serializovat a posilat
> pres postMessage do workeru, ktery provede renderovani. Zachovej
> puvodni API (renderFrame, renderGrid, renderEntities).
```

Claude vytvori:
- `render/worker/render-worker.ts` — Web Worker s OffscreenCanvas logikou
- `render/offscreen-renderer.ts` — wrapper zachovavajici puvodni API
- Upravy v `render/renderer.ts` pro delegovani na worker

**Otestujte vysledek:**

```
> Spust testy a zkontroluj, jestli OffscreenCanvas renderer funguje spravne.
```

Pravdepodobne narazite na problemy:
- `OffscreenCanvas` neni dostupny v JSDOM/testovacim prostredi
- Serializace `Path2D` objektu pres `postMessage` neni mozna
- Viewport transformace (Y-flip) se musi duplikovat v workeru

Toto je checkpoint B — nefunkcni experiment.

#### Krok 3: Rewind na vychozi stav

Stisknete `Esc + Esc` (dvakrat rychle za sebou). Zobrazi se seznam checkpointu
s casovymi znackami. Vyberte checkpoint A (pred OffscreenCanvas zmenami).

Zvolte moznost **1: Restore code and conversation**.

Claude vrati vsechny soubory do puvodniho stavu a konverzace se vrati na bod pred
experimentem. Novy worker soubor zmizi, renderer je zpet v puvodnim stavu.

```
> Potvrd, ze jsme zpet na puvodnim Canvas 2D rendereru.
```

#### Krok 4: Experiment 2 — Dirty-region rendering

Ted zkuste jiny pristup:

```
> Implementuj dirty-region rendering optimalizaci. Pridej do rendereru system,
> ktery sleduje, ktere entity se zmenily (dirty flag), a prekresli pouze
> postizenou oblast platna misto celeho canvasu. Pridej DirtyRegionTracker
> do packages/cad-engine/src/render/.
```

Claude vytvori:
- `render/dirty-tracker.ts` — sledovani zmenenych regionu
- Upravy v `render/renderer.ts` — podminene prekreslovani
- Bounding box kalkulace pro kazdu entitu

**Otestujte:**

```
> Spust testy a zmer, o kolik se snizi pocet renderovacich operaci
> pri zmene jedne entity v dokumentu se 100 entitami.
```

Tento pristup by mel fungovat lepe, protoze nevyzaduje Web Worker serializaci
a pracuje primo s existujicim Canvas 2D API.

Toto je checkpoint C — funkcni optimalizace.

#### Krok 5: Porovnani pristupu

Pouzijte `/rewind` a zvolte moznost **4: Summarize from here**, abyste ziskali
souhrn obou experimentu. Claude vytvoril AI souhrn:

```
> Shrnul jsi oba experimenty. Ktery pristup doporucujes pro Solvina projekt
> a proc?
```

Claude porovna:
- OffscreenCanvas: slozita serializace, problemy s testovanim, ale potencialne lepsi
  pro velmi velke vykresy
- Dirty-region: jednodussi implementace, funguje s existujici architekturou,
  okamzite zlepseni vykonu

#### Krok 6: Finalizace

Pokud jste spokojeni s dirty-region pristupem, udelejte git commit:

```
> Commitni dirty-region renderer s hlaskou
> "perf: add dirty-region rendering for partial canvas updates"
```

---

### Cviceni

1. **Rewind s volbou 2 (Restore conversation):** Udelejte zmenu v rendereru,
   pak pouzijte `/rewind` s volbou "Restore conversation". Overite, ze soubory
   zustavaji zmenene, ale konverzace se vratila.

2. **Rewind s volbou 3 (Restore code):** Udelejte dalsi zmenu, pouzijte
   Esc+Esc s volbou "Restore code". Overite, ze soubory se vratily, ale
   konverzace o zmenach zustava.

3. **Trojity branching:** Zkuste tri ruzne optimalizace rendereru (napr.
   layer caching, spatial indexing, batch rendering). Pouzijte checkpointy
   k porovnani vsech tri.

### Overeni

- [ ] Umite pouzit `Esc+Esc` i `/rewind` pro pristup k checkpointum
- [ ] Rozumite rozdilum mezi vsemi peti moznostmi obnoveni
- [ ] Udelali jste alespon dva experimenty s rendererem a porovnali je
- [ ] Funkcni dirty-region tracker je commitnuty v gitu

### Co jste pridali do Solvina

| Soubor | Popis |
|---|---|
| `packages/cad-engine/src/render/dirty-tracker.ts` | DirtyRegionTracker — sledovani zmenenych oblasti |
| `packages/cad-engine/src/render/renderer.ts` | Rozsireni o podminene prekreslovani dirty regionu |

---

## Lekce 09: Advanced Features — Slozita feature s planovanim

| Parametr | Hodnota |
|---|---|
| Cas | 60 minut |
| Obtiznost | Vysoka |
| Predpoklady | Lekce 01-08, pokrocila znalost Solvina architektury |

### Co se naucite

- Planning mode (`/plan`) pro navrh architektury pred implementaci
- Extended thinking (`/effort max`) pro slozite vypocty
- Background tasks pro paralelni praci
- Permission modes a jejich vyuziti v ruznych fazich vyvoje
- Session management (`/resume`) pro praci pres vice dni

### Predpoklady

- Solvina projekt se vsemi predchozimi lekcemi
- Pochopeni CAD engine architektury (core/types.ts, core/engine.ts)
- Existujici UCS (User Coordinate System) v `core/types.ts` — typ `UcsState`

---

### Teorie

#### Claude Code: Advanced features

**Planning mode** je dvourazovy pristup k implementaci:
1. **Planovaci faze:** Claude analyzuje ukol, navrhe architekturu, identifikuje
   soubory k uprave a navrhne poststup. Nedela zadne zmeny v kodu.
2. **Implementacni faze:** Po vasi revizi a schvaleni Claude provede plan.

Aktivace:
```
/plan Implementuj constraint system pro parametricke vazby mezi entitami
```

Nebo nastavte permission mode:
```bash
claude --permission-mode plan
```

**Extended thinking** umoznuje Claude hlubsi uvahu u slozitych problemu:
```
/effort max
```

Effort `max` je dostupny pouze pro model Opus 4.6 a aktivuje
nejhlubsi uroven uvazovani. Pouzivejte pro architekturni rozhodnuti,
slozite algoritmy a geometricke vypocty.

**Background tasks** umoznuji Claude pracovat na ukolu na pozadi, zatimco
vy pokracujete v jine praci. Claude vas upozorni, az bude hotov.

**Permission modes:**

| Mod | Chovani |
|---|---|
| `default` | Pta se pred kazdou zmenou |
| `plan` | Pouze cte a planuje, zadne zmeny |
| `acceptEdits` | Editace bez ptani, bash vyzaduje souhlas |
| `auto` | Vsechny akce povoleny s AI bezpecnostni kontrolou |
| `bypassPermissions` | Vse bez ptani (pouze pro automatizaci) |

**Session management:**
```
/resume              # Seznam poslednich sezeni
claude -c            # Pokracovani v poslednim sezeni
claude -r "nazev"    # Obnoveni konkretniho sezeni podle jmena
```

#### CAD kontext: Constraint/parametricke systemy

Profesionalni CAD systemy pouzivaji constraint solvery pro udrzeni geometrickych
vztahu:

- **Coincident** — dva body na stejnem miste
- **Parallel/Perpendicular** — uhel mezi useckami
- **Tangent** — dotyk krivek
- **Distance/Radius** — fixni vzdalenost nebo polomer
- **Horizontal/Vertical** — orientace usecek

Implementace constraint solveru je architekturne narocna — vyzaduje:
1. Datovy model pro constrainty (typy, reference na entity)
2. Solver algorimus (napr. Newton-Raphson pro numericke reseni)
3. Integraci s undo/redo systemem
4. UI pro pridavani a vizualizaci constraintu

Presne typ ukolu, kde planning mode a extended thinking pomohou.

---

### Prakticka cast

#### Krok 1: Planovaci faze — navrh architektury

Spustte Claude Code v plan modu:

```bash
cd ~/solvina
claude --permission-mode plan
```

Zadejte komplexni ukol:

```
/plan Navrhni a implementuj constraint/parametricky system pro Solvina CAD engine.

Pozadavky:
1. Datovy model pro constrainty v core/types.ts (Coincident, Parallel,
   Perpendicular, Distance, Horizontal, Vertical, Tangent)
2. ConstraintSolver trida pouzivajici iterativni numericke reseni
3. Integrace s CadEngine — constrainty se museji ukladat v CadDocument
   a fungovat s undo/redo
4. Vizualizace constraintu v rendereru (ikony/symboly u entit)
5. ConstraintTool pro interaktivni pridavani constraintu

Zohledni existujici architekturu:
- core/types.ts (Vec2, Entity, CadDocument)
- core/engine.ts (CadEngine, undoStack, tools map)
- render/renderer.ts (Canvas 2D rendering pipeline)
- tools/ (state machine pattern pro nastroje)
```

Claude v plan modu analyzuje codebase, precte relevantni soubory a vytvori
detailni plan BEZ jakychkoli uprav. Vystupem bude neco jako:

```
## Plan: Constraint/Parametric System

### Faze 1: Datovy model (core/types.ts)
- Novy typ ConstraintType (enum)
- Novy typ Constraint (id, type, entityRefs, parameters)
- Rozsireni CadDocument o constraints: Constraint[]
- Rozsireni Entity o constraintIds: string[]

### Faze 2: Solver (constraints/solver.ts)
- ConstraintSolver trida
- Metoda solve(document): dokument s upravenymi pozicemi
- Newton-Raphson iterace pro numericke reseni
- Residual funkce pro kazdy typ constraintu

### Faze 3: Integrace s CadEngine
- addConstraint/removeConstraint metody
- solve() volani po kazde zmene entity
- Undo/redo podpora (constrainty jsou soucasti CadDocument)

### Faze 4: Vizualizace
- renderConstraints() v rendereru
- Ikony: kolecko pro Coincident, sipky pro Parallel, atd.

### Faze 5: Interaktivni tool
- ConstraintTool ve tools/
- Vyber dvou entit -> nabidka dostupnych constraintu
```

Prectete si plan a rozhodnete se, co chcete upravit.

#### Krok 2: Revize planu

Stale v plan modu diskutujte o architekture:

```
> Plan vypada dobre, ale mam pripominky:
> 1. Solver by mel byt lazy — resit az pri renderovani, ne po kazde zmene
> 2. Chci mit constrainty jako separatni vrstvu, ne jako soucasti Entity
> 3. Pridej constraint highlighting kdyz je constraint naruseny (cervene)
> Uprav plan podle techto pripominek.
```

Claude upravi plan. Opakujte, dokud nebudete spokojeni.

#### Krok 3: Aktivace extended thinking

Pred implementaci nastavte maximalni uroven uvazovani:

```
/effort max
```

Toto je dulezite pro nasledujici krok — constraint solver vyzaduje spravny
matematicky zaklad (Jacobian matice, konvergence iteraci).

#### Krok 4: Prepnuti do implementacniho modu

Prepnete permission mode pro efektivni implementaci:

```
/permissions acceptEdits
```

V `acceptEdits` modu Claude muze editovat soubory bez ptani (usetri cas),
ale bash prikazy stale vyzaduji vase schvaleni.

Spustte implementaci:

```
> Implementuj Fazi 1 z planu — datovy model constraintu v core/types.ts
> a rozsireni CadDocument.
```

Claude prida do `core/types.ts`:

```typescript
// --- Constraint types ---
export type ConstraintType =
  | 'coincident'
  | 'parallel'
  | 'perpendicular'
  | 'distance'
  | 'horizontal'
  | 'vertical'
  | 'tangent';

export interface Constraint {
  id: string;
  type: ConstraintType;
  entityIds: [string, string];
  parameters?: { value: number };   // e.g. distance value
  satisfied: boolean;
}

// Rozsireni CadDocument:
export interface CadDocument {
  // ... existujici pole ...
  constraints: Constraint[];
}
```

#### Krok 5: Background task pro solver

Zatimco Claude implementuje solver, muzete pracovat na necem jinem.
Pokud pouzivate vice terminalu, muzete spustit druhy Claude Code sezeni:

```bash
# Terminal 1: Claude implementuje solver
> Implementuj ConstraintSolver v packages/cad-engine/src/constraints/solver.ts
> podle planu. Pouzij Newton-Raphson iteraci s maximalnim poctem 50 iteraci
> a toleranci 1e-6.

# Terminal 2: Muzete delat jinou praci
cd ~/solvina
claude -n "constraint-ui"
> Vytvor ikony pro constraint vizualizaci jako SVG konstanty v
> packages/cad-engine/src/render/constraint-icons.ts
```

Alternativne v jednom sezeni muzete zadat Claude dlouhy ukol a pockat na
notifikaci o dokonceni.

#### Krok 6: Session management — pokracovani dalsi den

Po implementaci Faze 1-3 muzete ukoncit sezeni. Dalsi den pokracujte:

```bash
# Zobrazeni poslednich sezeni
claude /resume

# Pokracovani v konkretnim sezeni podle jmena
claude -r "constraint-solver" "Pokracuj s implementaci Faze 4 — vizualizace constraintu"
```

Nebo pokracujte v poslednim sezeni:

```bash
claude -c
> Kde jsme skoncili? Pokracuj s implementaci.
```

Claude nacte kontext predchoziho sezeni vcetne planu a dosavadniho postupu.

#### Krok 7: Finalizace constraint systemu

Po implementaci vsech fazi ovezte celek:

```
> Spust vsechny testy pro constraint system. Pak vytvor jednoduchy
> integracni test: nakresli dva body, pridej coincident constraint,
> pohni jednim bodem a over, ze druhy nasleduje.
```

Commitnete vysledek:

```
> Commitni vsechny zmeny s hlaskou
> "feat: add constraint/parametric system with Newton-Raphson solver"
```

---

### Cviceni

1. **Plan mode pro dalsi feature:** Pouzijte `/plan` k navrhu systemu
   parametrickych kotovcich (dimension annotations). Zkontrolujte plan,
   ale neimplementujte — cvicite planovani.

2. **Effort levels:** Zeptejte se Claude na stejny geometricky problem
   (napr. "jak vypocitat prusecik dvou elips") s `/effort low` a pak
   `/effort max`. Porovnejte hloubku odpovedi.

3. **Session fork:** Pouzijte `claude --resume <session> --fork-session`
   k vytvoreni alternativni verze constraint solveru (napr. Gauss-Seidel
   misto Newton-Raphson). Porovnejte oba pristupy.

### Overeni

- [ ] Pouzili jste `/plan` k navrhu architektury pred implementaci
- [ ] Zmenili jste effort level pomoci `/effort max`
- [ ] Vyuzili jste session management (`/resume` nebo `claude -c`)
- [ ] Prepnuli jste permission mode behem prace (`plan` -> `acceptEdits`)
- [ ] Constraint system funguje: pridani, reseni, vizualizace, undo/redo

### Co jste pridali do Solvina

| Soubor | Popis |
|---|---|
| `packages/cad-engine/src/core/types.ts` | Typy ConstraintType, Constraint; rozsireni CadDocument |
| `packages/cad-engine/src/constraints/solver.ts` | ConstraintSolver — Newton-Raphson iterativni resic |
| `packages/cad-engine/src/constraints/residuals.ts` | Residual funkce pro kazdy typ constraintu |
| `packages/cad-engine/src/constraints/index.ts` | Public API constraint modulu |
| `packages/cad-engine/src/render/constraint-icons.ts` | SVG ikony pro vizualizaci constraintu |
| `packages/cad-engine/src/render/renderer.ts` | Rozsireni o renderConstraints() |
| `packages/cad-engine/src/tools/constraint-tool.ts` | Interaktivni nastroj pro pridavani constraintu |

---

## Lekce 10: CLI Reference — CI/CD a automatizace

| Parametr | Hodnota |
|---|---|
| Cas | 50 minut |
| Obtiznost | Vysoka |
| Predpoklady | Lekce 01-09, zakladni znalost GitHub Actions |

### Co se naucite

- Print mode (`claude -p`) pro neinteraktivni pouziti
- Piping vstupu do Claude Code
- JSON vystupni format (`--output-format json`) a jeho parsovani
- Permission modes pro automatizaci (`--permission-mode`, `--dangerously-skip-permissions`)
- Vytvoreni CI/CD pipeline pro Solvina s GitHub Actions
- Automatizovany code review na pull requestech

### Predpoklady

- Solvina projekt na GitHubu (nebo jinem Git hosting)
- Zakladni znalost YAML a GitHub Actions
- `ANTHROPIC_API_KEY` pro CI prostredi

---

### Teorie

#### Claude Code: CLI pro automatizaci

V predchozich lekcich jsme pouzivali Claude Code interaktivne. Ale `claude` je
take plnohodnotny CLI nastroj pro skripty a CI/CD.

**Print mode (`-p`)** — jednoucelovy dotaz bez interaktivniho rozhrani:

```bash
claude -p "kolik radku kodu ma tento projekt?"
```

Odpoved se vypise na stdout a program skonci. Zadna konverzace, zadne interaktivni
prvky.

**Piping** — vstup z jineho prikazu:

```bash
cat packages/cad-engine/src/core/types.ts | claude -p "zkontroluj typy"
git diff HEAD~1 | claude -p "reviewuj tento diff"
```

**JSON vystup** — strojove zpracovatelna odpoved:

```bash
claude -p --output-format json "najdi vsechny TODO komentare"
```

Vystup obsahuje strukturovany JSON, ktery lze parsovat pomoci `jq` nebo v Node.js.

**Structured output se schematem:**

```bash
claude -p --json-schema '{"type":"object","properties":{"issues":{"type":"array","items":{"type":"object","properties":{"file":{"type":"string"},"line":{"type":"number"},"severity":{"type":"string"},"message":{"type":"string"}}}}}}' \
  "najdi potencialni bugy v src/"
```

**Omezeni poctu kroku:**

```bash
claude -p --max-turns 3 "analyzuj tento kod"
```

Uzitecne v CI, kde chcete omezit cas a naklady.

**Rozpoctove omezeni:**

```bash
claude -p --max-budget-usd 2.00 "komplexni analyza"
```

#### CAD kontext: CI/CD pro CAD engine

CAD engine potrebuje specificke CI kontroly:

1. **Typova kontrola** — TypeScript strict mode pro geometricke typy
2. **Unit testy** — snap engine, viewport transformace, nastroje
3. **DXF round-trip** — export do DXF, reimport, porovnani s originalem
4. **Vizualni regrese** — screenshot testy rendereru (volitelne)
5. **Code review** — automaticka kontrola kvality pri pull requestech

---

### Prakticka cast

#### Krok 1: Lokalni automatizacni skripty

Nejprve vytvorte lokalni skripty, ktere pak pouzijete v CI.

Pozadejte Claude o vytvoreni DXF round-trip validacniho skriptu:

```
> Vytvor Node.js skript scripts/validate-dxf-roundtrip.ts, ktery:
> 1. Nacte testovaci DXF soubor z test-fixtures/sample.dxf
> 2. Importuje ho pomoci naseho DXF parseru z packages/cad-engine/src/io/
> 3. Exportuje zpet do DXF
> 4. Reimportuje exportovany DXF
> 5. Porovna entity (typ, souradnice s toleranci 1e-10)
> 6. Vypise vysledek jako JSON: { passed: boolean, entityCount: number,
>    mismatches: [...] }
> 7. Exit code 0 pri uspechu, 1 pri selhani
>
> Pouzij cross-platform Node.js prikazy (zadne bash-specificke operace).
```

Otestujte lokalne:

```
> Spust validate-dxf-roundtrip.ts a ukazat mi vysledek.
```

#### Krok 2: Automatizovany code review skript

Vytvorte skript pro automaticky code review pomoci `claude -p`:

```
> Vytvor scripts/ai-review.ts (Node.js skript), ktery:
> 1. Prijme argument --diff (cesta k diff souboru) nebo precte stdin
> 2. Zavola `claude -p --output-format json --max-turns 1` s promptem
>    pro code review zamereny na CAD-specificke problemy:
>    - Spravnost geometrickych vypoctu (floating point, epsilon porovnani)
>    - Immutabilita CadDocument (zadne mutace stavu)
>    - Spravne viewport transformace (Y-flip, world/screen souradnice)
>    - Osetreni edge cases v snap engine
> 3. Parsuje JSON vystup a vypise formatovany report
> 4. Exit code 0 pri "OK", 1 pri nalezenych problemech
>
> Pouzij cross-platform Node.js (child_process.execSync pro volani claude).
```

Otestujte lokalne:

```bash
git diff HEAD~1 | npx tsx scripts/ai-review.ts
```

#### Krok 3: GitHub Actions workflow

Pozadejte Claude o vytvoreni kompletniho CI/CD workflow:

```
> Vytvor .github/workflows/ci.yml pro Solvina projekt:
>
> Job 1 — build-and-test:
>   - Checkout, setup Node 20, install dependencies
>   - TypeScript type check (npx tsc --noEmit)
>   - Unit testy (npx vitest run)
>   - DXF round-trip validace (npx tsx scripts/validate-dxf-roundtrip.ts)
>
> Job 2 — ai-review (pouze na pull_request):
>   - Checkout s fetch-depth 0
>   - Ziskani diffu PR zmenovych souboru
>   - Spusteni claude -p s review promptem
>   - Postovani vysledku jako PR komentar pomoci gh
>   - Pouzij ANTHROPIC_API_KEY z secrets
>   - Nastav --max-turns 1 a --max-budget-usd 1.00 pro kontrolu nakladu
>
> Oba joby pouzivaji ubuntu-latest.
```

Claude vytvori soubor `.github/workflows/ci.yml`:

```yaml
name: Solvina CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npx tsc --noEmit

      - name: Unit tests
        run: npx vitest run

      - name: DXF round-trip validation
        run: npx tsx scripts/validate-dxf-roundtrip.ts

  ai-review:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Get PR diff
        run: |
          git diff origin/${{ github.base_ref }}...HEAD > pr-diff.txt

      - name: AI Code Review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          cat pr-diff.txt | claude -p \
            --output-format json \
            --max-turns 1 \
            --max-budget-usd 1.00 \
            "Review this CAD engine diff. Focus on:
            1. Geometric calculation correctness (floating point, epsilon)
            2. CadDocument immutability (no state mutations)
            3. Viewport transform correctness (Y-flip, world/screen coords)
            4. Snap engine edge cases
            5. TypeScript type safety

            Return JSON: {
              summary: string,
              issues: [{file: string, line: number, severity: 'error'|'warning'|'info', message: string}],
              approved: boolean
            }" > review-result.json

      - name: Post review comment
        if: always()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          node -e "
            const fs = require('fs');
            const result = JSON.parse(fs.readFileSync('review-result.json', 'utf8'));
            const body = [
              '## AI Code Review',
              '',
              result.summary,
              '',
              '### Issues',
              '',
              ...result.issues.map(i =>
                '- **' + i.severity.toUpperCase() + '** ' + i.file +
                (i.line ? ':' + i.line : '') + ' — ' + i.message
              ),
              '',
              result.approved ? '> Approved' : '> Changes requested',
            ].join('\n');
            fs.writeFileSync('review-body.txt', body);
          "
          gh pr comment ${{ github.event.pull_request.number }} \
            --body-file review-body.txt
```

#### Krok 4: Pokrocile CLI vzory

Procvicte dalsi CLI vzory uzitecne pro CAD vyvoj:

**Analyza geometrickeho modulu:**

```bash
cat packages/cad-engine/src/geometry/*.ts | \
  claude -p "Zkontroluj vsechny geometricke funkce na numericke problemy: \
  deleni nulou, preteceni float, chybejici epsilon porovnani."
```

**Batch generovani testu:**

```bash
# Node.js skript pro generovani testu pro kazdy tool
node -e "
const fs = require('fs');
const tools = fs.readdirSync('packages/cad-engine/src/tools')
  .filter(f => f.endsWith('.ts') && f !== 'index.ts');
for (const tool of tools) {
  console.log(tool);
}
" | while read tool; do
  claude -p --max-turns 1 \
    "Generuj unit test pro packages/cad-engine/src/tools/$tool. \
    Otestuj vsechny stavy state machine." \
    > "packages/cad-engine/src/tools/__tests__/${tool%.ts}.test.ts"
done
```

**Structured output pro metriky:**

```bash
claude -p --output-format json \
  --json-schema '{
    "type": "object",
    "properties": {
      "totalFiles": {"type": "number"},
      "totalLines": {"type": "number"},
      "modules": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "name": {"type": "string"},
            "files": {"type": "number"},
            "lines": {"type": "number"}
          }
        }
      }
    }
  }' \
  "Analyzuj packages/cad-engine/src/ a vrat metriky pro kazdy modul" \
  | node -e "
    const data = JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));
    console.log('Solvina CAD Engine Metrics');
    console.log('='.repeat(40));
    console.log('Total: ' + data.totalFiles + ' files, ' + data.totalLines + ' lines');
    data.modules.forEach(m =>
      console.log('  ' + m.name + ': ' + m.files + ' files, ' + m.lines + ' lines')
    );
  "
```

**Pokracovani v print mode:**

```bash
# Prvni dotaz
claude -p "Analyzuj vykon rendereru v packages/cad-engine/src/render/"

# Pokracovani v kontextu predchoziho
claude -c -p "Navrhni tri konkretni optimalizace na zaklade predchozi analyzy"
```

#### Krok 5: Lokalni pre-push hook s Claude

Pridejte do `.claude/settings.json` hook, ktery pred push spusti rychlou kontrolu:

```
> Pridej do .claude/settings.json hook, ktery pred git push spusti
> claude -p s rychlou kontrolou posledniho commitu. Pouzij Node.js
> pro cross-platform kompatibilitu.
```

Hook v `.claude/settings.json`:

```json
{
  "hooks": {
    "PrePush": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "node -e \"const {execSync} = require('child_process'); const diff = execSync('git diff HEAD~1').toString(); if (diff.length > 0) { const result = execSync('claude -p --max-turns 1 --output-format json \\\"Quick check: are there obvious bugs in this diff?\\\" < /dev/stdin', {input: diff}).toString(); const parsed = JSON.parse(result); if (parsed.hasBugs) { console.error('Potential issues found:', parsed.issues); process.exit(1); } }\""
          }
        ]
      }
    ]
  }
}
```

---

### Cviceni

1. **Piping experiment:** Pouzijte `cat` a pipe k odeslani ruznych souboru
   do `claude -p`. Zkuste analyzovat `core/types.ts`, `snap/` modul
   a `geometry/` modul. Porovnejte uzitecnost odpovedi.

2. **JSON schema:** Vytvorte vlastni `--json-schema` pro analyzu snap enginu.
   Schema by mela vyzadovat pole `snapTypes`, `edgeCases` a `performance`.
   Parsujte vysledek v Node.js skriptu.

3. **Multi-step pipeline:** Vytvorte Node.js skript, ktery:
   (a) spusti `claude -p` pro nalezeni problemu,
   (b) parsuje JSON vystup,
   (c) pro kazdy problem spusti dalsi `claude -p` s detailni analyzou,
   (d) vygeneruje HTML report.

### Overeni

- [ ] Umite pouzit `claude -p` pro jednoucelove dotazy
- [ ] Umite pipovat vstup do Claude Code
- [ ] Parsujete JSON vystup pomoci `--output-format json`
- [ ] GitHub Actions workflow je funkcni (build, test, DXF validace)
- [ ] AI review na pull requestech funguje a pise komentare
- [ ] DXF round-trip validacni skript vraci spravne vysledky

### Co jste pridali do Solvina

| Soubor | Popis |
|---|---|
| `scripts/validate-dxf-roundtrip.ts` | DXF export/reimport validace s JSON vystupem |
| `scripts/ai-review.ts` | Automatizovany CAD-specificky code review |
| `.github/workflows/ci.yml` | CI/CD pipeline: build, test, DXF validace, AI review |
| `.claude/settings.json` | Rozsireni o pre-push hook s rychlou kontrolou |

---

## Zaverecny prehled

### Co jsme postavili za 10 lekci

Behem tohoto tutorialu jsme rozsirili Solvina CAD engine o radu novych funkci
a zaroven se naucili vsechny klicove features Claude Code.

#### Prehled lekci

| Lekce | Claude Code feature | Co jsme pridali do Solvina |
|---|---|---|
| 01 — Slash Commands | `/commit`, `/pr`, `/review`, vlastni skills | Vlastni `/dxf-check` skill, commit workflow |
| 02 — Memory | CLAUDE.md, projektova pamet, architekturni pravidla | `.claude/CLAUDE.md` s CAD konvencemi, snap pravidla |
| 03 — Skills | Vlastni slash commands, prompt soubory, parametry | `/snap-test` skill, `/entity-scaffold` generator |
| 04 — Subagents | Delegovani ukolu, paralelni agenti, orchestrace | Multi-agent refaktoring rendereru, code review agent |
| 05 — MCP | Externi nastroje, servery, DXF/SVG integrace | MCP server pro DXF preview, GitHub integrace |
| 06 — Hooks | Event-driven automatizace, pre/post akce | Pre-commit typova kontrola, post-save lint, auto-adapt |
| 07 — Plugins | Bundled rozsireni, sdileni nastroju | CAD plugin s nastroji, sdileny team workflow |
| 08 — Checkpoints | `/rewind`, Esc+Esc, 5 moznosti obnoveni, branching | Dirty-region rendering optimalizace |
| 09 — Advanced Features | `/plan`, `/effort max`, session management, permissions | Constraint/parametricky system se solverem |
| 10 — CLI Reference | `claude -p`, piping, JSON output, CI/CD | GitHub Actions pipeline, DXF validace, AI review |

#### Finalni stromova struktura pridanych souboru

```
solvina/
├── .claude/
│   ├── CLAUDE.md                          # Lekce 02: Projektova pamet
│   └── settings.json                      # Lekce 06+10: Hooks konfigurace
│
├── .claude/skills/
│   ├── dxf-check.md                       # Lekce 01: DXF validacni skill
│   ├── snap-test.md                       # Lekce 03: Snap testovaci skill
│   └── entity-scaffold.md                 # Lekce 03: Entity generator skill
│
├── .github/
│   └── workflows/
│       └── ci.yml                         # Lekce 10: CI/CD pipeline
│
├── scripts/
│   ├── validate-dxf-roundtrip.ts          # Lekce 10: DXF round-trip validace
│   └── ai-review.ts                       # Lekce 10: Automatizovany code review
│
├── packages/cad-engine/src/
│   ├── core/
│   │   ├── types.ts                       # Lekce 09: + ConstraintType, Constraint
│   │   ├── engine.ts                      # Lekce 09: + constraint integrace
│   │   └── document.ts                    # Existujici, immutable updates
│   │
│   ├── constraints/                       # Lekce 09: Novy modul
│   │   ├── index.ts                       # Public API
│   │   ├── solver.ts                      # Newton-Raphson constraint solver
│   │   └── residuals.ts                   # Residual funkce pro constrainty
│   │
│   ├── render/
│   │   ├── renderer.ts                    # Lekce 08+09: + dirty regions, constraints
│   │   ├── dirty-tracker.ts               # Lekce 08: Dirty-region sledovani
│   │   └── constraint-icons.ts            # Lekce 09: SVG ikony constraintu
│   │
│   ├── tools/
│   │   └── constraint-tool.ts             # Lekce 09: Interaktivni constraint nastroj
│   │
│   ├── snap/                              # Existujici, snap engine
│   ├── geometry/                          # Existujici, math utilities
│   └── io/                                # Existujici, DXF/SVG import/export
│
└── src/modules/cad/                       # Existujici React UI
    ├── CadPage.tsx
    ├── CadCanvas.tsx
    ├── Toolbar.tsx
    ├── LayerPanel.tsx
    └── PropertyPanel.tsx
```

### Co dal?

Po absolvovani vsech 10 lekci mate solidni zaklad pro efektivni praci s Claude Code
na realnych projektech. Zde je nekolik smeru, kterymi muzete pokracovat:

**Prohloubeni CAD engine:**
- Implementace dalsich constraint typu (symmetry, equal, fix)
- Parametricke koty (dimension-driven design)
- Bloky a reference (INSERT entity, block definitions)
- Hatch/srafovani a regions
- 3D zaklady — rozsireni Vec2 na Vec3, izometricke pohledy

**Pokrocile Claude Code workflow:**
- Nastaveni agent teams pro vice-clennou spolupraci
- Vytvoreni custom MCP serveru pro vasi domenovou logiku
- Automaticke generovani dokumentace z kodu
- Integrace s dalsimu nastroji pres MCP (Figma, JIRA, databaze)
- Pouziti `claude --remote` pro web-based sezeni

**CI/CD rozsireni:**
- Vizualni regresni testy rendereru (screenshot porovnani)
- Performance benchmarky v CI (cas renderovani, pamet)
- Automaticke release notes generovane z git logu
- Deployment preview pro kazdou pull request vetev

**Sdileni znalosti:**
- Exportujte sve `.claude/` konfigurace jako plugin pro kolegy
- Zdokumentujte projektove konvence v CLAUDE.md pro nove cleny tymu
- Vytvorte skills specificke pro vas workflow

---

*Toto je treti a posledni cast tutorialu. Vsechny tri casti dohromady pokryvaji
kompletni sadu Claude Code features aplikovanou na realny CAD engineering projekt.*
