# Steal React Component

A Claude Code skill suite that extracts React components, design systems, and whole sites from live production websites, using browser automation.

**Status:** working skill suite, published for use. The repo is prompt and skill files plus project templates. It has no test suite and records no benchmark or accuracy results.

## What it does

| Skill | Command | What it does |
|-------|---------|--------------|
| `steal-react-component` | `/steal-react-component https://example.com` | Extracts individual React components by reading React Fiber internals (`__reactFiber$*` keys): props, hooks, rendered HTML, and minified source. Claude then reconstructs readable code. |
| `css-extractor` | (see `CSS-EXTRACTOR.md`) | Extracts a design system: CSS variables, typography, colors, spacing. Can emit a Tailwind config. |
| `copy-site` | `/copy-site https://example.com` | Clones a whole site: screenshots it, extracts styles and components, de-minifies them with parallel subagents, scaffolds a Next.js project, and checks the result against the original. |

The technique comes from [fant.io/react](https://fant.io/react/), "How to Steal Any React Component".

## Requirements

- Claude Code started with `claude --chrome`
- Chrome with the [Claude-in-Chrome](https://github.com/anthropics/claude-in-chrome) extension
- A target site. React sites work best; style extraction works on any site.

## Install

Each skill is split into a short dispatcher (the skill) and a longer agent file. The agent runs in a separate subagent context, so the main conversation does not fill up with browser-automation output.

```bash
# Agents (required). Use .claude/agents/ instead for project-level install.
cp AGENT.md ~/.claude/agents/steal-react-component.md
cp COPY-SITE-AGENT.md ~/.claude/agents/copy-site.md

# Skills. Use .claude/skills/ instead for project-level install.
mkdir -p ~/.claude/skills/steal-react-component
cp SKILL.md CSS-EXTRACTOR.md COPY-SITE.md ~/.claude/skills/steal-react-component/

# Optional: also expose copy-site as a slash command
mkdir -p ~/.claude/commands
cp COPY-SITE.md ~/.claude/commands/copy-site.md
```

Then run `/steal-react-component https://example.com` or `/copy-site https://example.com`.

## Repo map

| File | Purpose |
|------|---------|
| `SKILL.md`, `AGENT.md` | Component extraction dispatcher and agent (contains the `ReactStealer` script) |
| `CSS-EXTRACTOR.md` | Design system extraction (the `StyleStealer` script) |
| `COPY-SITE.md`, `COPY-SITE-AGENT.md` | Full-site cloning dispatcher and agent |
| `templates/` | Next.js, TypeScript and Tailwind scaffolding used for cloned sites (see `templates/README.md`) |

## Limitations

- Animated states may not be captured correctly.
- Interactive state such as dropdowns and modals may not be captured.
- Some component names are minified (for example `Hc`, `qv`).
- React Server Components may have no client-side Fiber data.
- Pages behind authentication need a manual login first.
- On non-React sites, only style extraction works.

## More detail

### How it works

1. React keeps a Fiber tree parallel to the DOM and attaches Fiber nodes to DOM elements under `__reactFiber$*` keys.
2. The agent injects `ReactStealer` into the page and reads each component's type, props, hooks, and rendered HTML.
3. `StyleStealer` pulls CSS variables, computed styles, and typography.
4. Several prop-to-HTML examples per component, plus the minified source, are given to the LLM to reconstruct the component.
5. For `copy-site`, the result is scaffolded into a Next.js project and compared with the original until it matches.

### Subagent architecture

Approximate token sizes. These are the author's estimates, not measurements recorded in the repo.

| Skill | Dispatcher | Agent |
|-------|------------|-------|
| steal-react-component | `SKILL.md` (~300 tokens) | `AGENT.md` (~4k tokens) |
| copy-site | `COPY-SITE.md` (~400 tokens) | `COPY-SITE-AGENT.md` (~6k tokens) |

The author's estimate for a typical site clone is about 6k tokens in the main agent and about 50k inside the subagent.

### Console API

```javascript
ReactStealer.summary()                 // list components and counts
ReactStealer.getForLLM('Button')       // reconstruction prompt for one component
ReactStealer.getBySelector('button.primary') // component behind a DOM element

StyleStealer.extractAll()              // full design system
StyleStealer.toTailwindConfig()        // Tailwind config
StyleStealer.toCSSVariables()          // CSS variables file
```

### Example output of `/copy-site https://x.com`

```
sites/
└── x.com/
    ├── README.md           # Site documentation
    ├── style-guide.md      # Design system reference
    └── app/
        ├── package.json
        ├── tailwind.config.ts
        ├── app/            # layout.tsx, page.tsx, globals.css
        └── components/     # Sidebar.tsx, Tweet.tsx, TweetComposer.tsx, ...
```

## License

MIT. See `LICENSE`.
