# Chat Assistant Skill for Claude Code

A Claude Code skill that scaffolds branded AI chat assistant widgets — the same architecture used by [Wiil](https://wiil.io) to build conversational AI assistants for companies.

## What It Does

When you run `/chat-assistant`, Claude will:

1. **Ask about your brand** — company name, website, assistant persona, features needed
2. **Analyze your website** — fetches your site to extract brand colors, fonts, visual style, and logo
3. **Scaffold the entire project** — generates a complete Vite + React + Express app with:
   - Branded chat widget (single-file architecture)
   - SSE streaming with Claude API
   - Product catalog with category grid
   - Store locator with Google Maps
   - Bilingual support
   - Consultative AI persona (asks qualifying questions, recommends products)
   - Rich message rendering (product cards, contact cards, price highlights)
   - Mobile-responsive design
   - Consent screen & language switcher

## Installation

Copy the skill file into your Claude Code commands directory:

```bash
# Global (available everywhere)
cp .claude/commands/chat-assistant.md ~/.claude/commands/chat-assistant.md

# Or project-level (available in one project)
mkdir -p .claude/commands
cp .claude/commands/chat-assistant.md .claude/commands/chat-assistant.md
```

## Usage

In any Claude Code session:

```
/chat-assistant
```

Claude will walk you through brand discovery, then generate all files.

## Architecture

Every assistant follows the same proven structure:

```
{company}-demo-assistant/
├── src/
│   ├── App.jsx          ← Single-file widget (all components)
│   └── App.css          ← All styles with CSS custom properties
├── server/
│   ├── index.js         ← Express + Anthropic SSE streaming
│   ├── prompts.js       ← System prompt builder with persona
│   └── knowledge-base.txt ← Company-specific data
├── public/
│   └── logo.svg
├── package.json
├── vite.config.js
└── .env
```

### Key Design Decisions

- **Single-file widget** — All React components in one file for easy embedding
- **No extra packages** — Only React, Express, Anthropic SDK
- **SSE streaming** — Real-time AI responses, no WebSockets
- **View state machine** — `home | chat | stores | catalog` — no router
- **CSS custom properties** — Swap brand colors by changing 4-5 variables
- **Consultative AI** — Asks qualifying questions instead of dumping product lists
- **Rich rendering** — Product cards, store cards, price badges, contact cards all parsed from markdown

## Customization

The skill generates everything brand-specific:
- Colors extracted from your website
- Logo placed in the widget header
- AI persona with your company's tone
- Product categories with consultative prompts
- Store locations with Google Maps
- Bilingual translations

## Built by Wiil

This skill encapsulates [Wiil's](https://wiil.io) approach to building AI chat assistants. Contributions welcome.
