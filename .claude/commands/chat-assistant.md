# Chat Assistant Builder

You are a specialist in building branded AI chat assistant widgets. You follow Wiil's proven architecture for conversational AI assistants embedded as chat widgets on client websites.

IMPORTANT: The reference implementation lives at the karcher-demo-assistant project. Before scaffolding, read these files from that project to use as your working template:
- `src/App.jsx` — complete widget code (components, streaming, views, state machine)
- `src/App.css` — complete styles (all views, responsive, animations)
- `server/index.js` — Express + Anthropic streaming
- `server/prompts.js` — system prompt builder

If the karcher project is not available in the workspace, ask the user to clone it or point you to an existing assistant project to use as reference.

---

## Phase 1: Brand Discovery

Ask the user these questions using the AskUserQuestion tool:

**Question 1 — Company Details:**
- Company name
- Company website URL (REQUIRED — you will fetch this to analyze brand colors, fonts, visual identity)
- What the company does (1 sentence)

**Question 2 — Assistant Persona:**
- Assistant name (the AI persona — like "Nino" for Karcher)
- Assistant role (e.g., "sales consultant", "support agent", "booking assistant")
- Language(s) to support and which is primary

**Question 3 — Features needed (multi-select):**
- Product catalog with categories
- Store locator with Google Maps
- Service/repair requests
- Deals and promotions
- Booking/scheduling
- FAQ/knowledge base
- Contact/escalation

**Question 4 — Key details:**
- Logo file/URL (SVG preferred)
- Specific brand colors (or extract from website)
- Number of physical store locations (if any)

---

## Phase 2: Website Analysis

After getting the website URL, use WebFetch to:
1. Fetch homepage — extract primary color, secondary color, accent color, font family, logo URL, visual style
2. Analyze product/service offerings for knowledge base structure
3. Find store locations, contact info, product categories

Report findings to the user and confirm brand palette before proceeding.

---

## Phase 3: Project Scaffold

### Project Structure
```
{company}-demo-assistant/
├── index.html
├── package.json
├── vite.config.js
├── Dockerfile
├── .env
├── public/
│   ├── logo.svg
│   └── widget.js
├── embed.html
├── src/
│   ├── main.jsx
│   ├── App.jsx          <- Single-file widget (ALL components)
│   └── App.css           <- ALL styles
└── server/
    ├── index.js           <- Express + Anthropic streaming + production hardening
    ├── prompts.js         <- System prompt builder
    └── knowledge-base.txt <- Company-specific data
```

### Architecture Rules (NEVER break these):
1. **Single-file widget**: ALL React components in `App.jsx`. No splitting.
2. **Single CSS file**: ALL styles in `App.css` with CSS custom properties for theming.
3. **No extra packages**: Only react, react-dom, express, @anthropic-ai/sdk, concurrently.
4. **SSE streaming**: Server-Sent Events for real-time AI responses. Never WebSockets.
5. **View-based navigation**: State machine (`"home" | "chat" | "stores" | "catalog"`). No router.

---

## Template Files

### package.json
```json
{
  "name": "{BRAND_SLUG}-demo-assistant",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "concurrently \"vite\" \"node --env-file=.env server/index.js\"",
    "dev:client": "vite",
    "dev:server": "node --env-file=.env server/index.js",
    "build": "vite build",
    "start": "NODE_ENV=production node --env-file=.env server/index.js",
    "lint": "eslint ."
  },
  "dependencies": {
    "@anthropic-ai/sdk": "^0.78.0",
    "concurrently": "^9.2.1",
    "express": "^5.2.1",
    "react": "^19.2.0",
    "react-dom": "^19.2.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^5.1.1",
    "vite": "^7.3.1"
  }
}
```

### vite.config.js
```js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: { "/api": "http://localhost:3001" },
  },
});
```

### index.html
```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
    <link rel="icon" type="image/svg+xml" href="/logo.svg" />
    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet" />
    <title>{BRAND_NAME} AI Assistant</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

### src/main.jsx
```jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import App from "./App.jsx";
createRoot(document.getElementById("root")).render(<StrictMode><App /></StrictMode>);
```

### .env
```
ANTHROPIC_API_KEY=your-key-here
VITE_GOOGLE_MAPS_API_KEY=your-maps-key-here
```

---

## server/index.js — Production Hardened

```js
import express from "express";
import Anthropic from "@anthropic-ai/sdk";
import { join, dirname } from "path";
import { fileURLToPath } from "url";
import { buildSystemPrompt } from "./prompts.js";

const __dirname = dirname(fileURLToPath(import.meta.url));
const PORT = process.env.PORT || 3001;
const client = new Anthropic();
const app = express();

// --- Production Hardening ---

// Body size limit
app.use(express.json({ limit: "50kb" }));

// CORS for embed/cross-origin
const ALLOWED_ORIGINS = process.env.ALLOWED_ORIGINS
  ? process.env.ALLOWED_ORIGINS.split(",")
  : ["*"];

app.use((req, res, next) => {
  const origin = req.headers.origin;
  if (ALLOWED_ORIGINS.includes("*") || ALLOWED_ORIGINS.includes(origin)) {
    res.set("Access-Control-Allow-Origin", origin || "*");
    res.set("Access-Control-Allow-Methods", "POST, OPTIONS");
    res.set("Access-Control-Allow-Headers", "Content-Type");
  }
  if (req.method === "OPTIONS") return res.status(204).end();
  next();
});

// Simple in-memory rate limiter (no extra packages)
const rateLimitMap = new Map();
const RATE_LIMIT = 20; // requests per minute per IP
const RATE_WINDOW = 60000; // 1 minute

function checkRateLimit(ip) {
  const now = Date.now();
  const entry = rateLimitMap.get(ip);
  if (!entry) {
    rateLimitMap.set(ip, { count: 1, start: now });
    return true;
  }
  if (now - entry.start > RATE_WINDOW) {
    rateLimitMap.set(ip, { count: 1, start: now });
    return true;
  }
  entry.count++;
  return entry.count <= RATE_LIMIT;
}

// Clean up rate limit map every 5 minutes
setInterval(() => {
  const now = Date.now();
  for (const [ip, entry] of rateLimitMap) {
    if (now - entry.start > RATE_WINDOW) rateLimitMap.delete(ip);
  }
}, 300000);

// --- Chat Endpoint ---

const MAX_MESSAGE_LENGTH = 2000;
const MAX_MESSAGES = 50;
const CONTEXT_WINDOW = 20; // Only send last N messages to Claude

app.post("/api/chat", async (req, res) => {
  // Rate limiting
  const ip = req.ip || req.connection?.remoteAddress || "unknown";
  if (!checkRateLimit(ip)) {
    return res.status(429).json({ error: "Too many requests. Please wait a moment." });
  }

  const { messages, language } = req.body;

  // Validate messages
  if (!Array.isArray(messages) || messages.length === 0) {
    return res.status(400).json({ error: "messages array is required" });
  }
  if (messages.length > MAX_MESSAGES) {
    return res.status(400).json({ error: "Too many messages" });
  }

  // Validate individual message lengths
  for (const msg of messages) {
    if (typeof msg.content !== "string" || msg.content.length > MAX_MESSAGE_LENGTH) {
      return res.status(400).json({ error: "Message too long (max 2000 characters)" });
    }
  }

  // Trim to context window
  const trimmedMessages = messages.slice(-CONTEXT_WINDOW);

  res.set({
    "Content-Type": "text/event-stream",
    "Cache-Control": "no-cache",
    Connection: "keep-alive",
  });
  res.flushHeaders();

  let aborted = false;
  res.on("close", () => { aborted = true; });

  try {
    const response = await client.messages.create({
      model: "claude-sonnet-4-5-20250929",
      max_tokens: 2048,
      system: buildSystemPrompt(language || "en"),
      messages: trimmedMessages,
      stream: true,
    });

    for await (const event of response) {
      if (aborted) break;
      if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
        res.write(`data: ${JSON.stringify({ type: "text_delta", text: event.delta.text })}\n\n`);
      }
    }

    if (!aborted) {
      res.write(`data: ${JSON.stringify({ type: "message_stop" })}\n\n`);
    }
  } catch (err) {
    console.error("Anthropic API error:", err.status, err.message);
    if (!aborted) {
      let message = "An error occurred. Please try again.";
      if (err.status === 429) message = "Service is busy. Please wait a moment and try again.";
      else if (err.status === 401) message = "Configuration error. Please contact support.";
      else if (err.status === 529) message = "Service is temporarily unavailable. Please try again shortly.";
      res.write(`data: ${JSON.stringify({ type: "error", message })}\n\n`);
    }
  }

  if (!res.writableEnded) res.end();
});

// Serve static build in production
if (process.env.NODE_ENV === "production") {
  const distPath = join(__dirname, "..", "dist");
  app.use(express.static(distPath));
  app.get("*", (_req, res) => { res.sendFile(join(distPath, "index.html")); });
}

app.listen(PORT, () => { console.log(`Server running on http://localhost:${PORT}`); });
```

---

## server/prompts.js

```js
import { readFileSync } from "fs";
import { join, dirname } from "path";
import { fileURLToPath } from "url";

const __dirname = dirname(fileURLToPath(import.meta.url));
const knowledgeBase = readFileSync(join(__dirname, "knowledge-base.txt"), "utf-8");

export function buildSystemPrompt(language) {
  const langInstruction = language === "en"
    ? "Respond in English."
    : "Respond in {BRAND_PRIMARY_LANG_NAME}. The customer prefers {BRAND_PRIMARY_LANG_NAME}.";

  return `You are {ASSISTANT_NAME}, a {ASSISTANT_ROLE} for {BRAND_NAME}.

## Your Personality
- Warm, helpful, and knowledgeable about {BRAND_NAME}'s products/services
- Keep responses concise. Never dump long lists — summarize by category, then drill into details when asked
- Proactively suggest relevant products/services
- Celebrate good choices and show enthusiasm

## Qualifying Questions
Ask ONE short qualifying question when helping customers choose. Format as scannable options:

Example:
What will you use it for?
(home icon) Home use
(business icon) Professional/business

Keep it natural, easy to answer in one reply. Never ask more than 1 question before recommending.

## Language
${langInstruction}

## Knowledge Base
Use ONLY this information. Never make up products, prices, or details.

${knowledgeBase}

## Formatting Rules
- **bold** for product names and key terms
- Bullet lists (- item) for features
- (pin icon) **Store Name** - address for locations
- Prices with currency symbol (e.g., **Model Name** - 595 GEL renders as product card)
- Keep responses concise — helpful, not overwhelming

## Sales Behavior
- Recommend 2-3 options at different price points (good / better / best)
- After recommending, suggest relevant accessories
- If on sale, LEAD with discounted price
- For deals questions: show category-level summary first, then details when asked

## Escalation
When you cannot fully resolve something:
1. Empathize: "I want to make sure you get the best help."
2. Connect: provide specific contact (store phone, service email)
3. General issues: provide hotline number

## Guard Rails
- NEVER share internal emails or processes
- NEVER mention competitors
- NEVER make up information not in the knowledge base
- If unsure, say so and direct to the appropriate team`;
}
```

---

## src/App.jsx — Complete Reference Template

IMPORTANT: Copy the COMPLETE App.jsx from the karcher-demo-assistant reference project, then adapt it:

### What to customize for each brand:
1. **LANGUAGES array** — Change language codes, labels, and flag emojis
2. **T translations** — Replace all Georgian/English strings with the client's languages
3. **GREETINGS** — Write brand-specific greeting per language
4. **SERVICES array** — Define 3-4 home screen cards based on features needed (id, icon, titleKey, descKey, prompt per language)
5. **STORES array** — Populate with actual store data (name, address, phone, lat/lng, manager — all bilingual)
6. **PRODUCT_CATEGORIES array** — Define catalog categories with consultative prompts (not imperative "show me all X" but customer-like "I'm interested in X, help me choose")
7. **ServiceIcon component** — Keep base icons (browse, service, deals, store). Add custom SVG icons for each product category
8. **inlineFormat regex** — Replace the Karcher model regex (K3, WD5, SC2, etc.) with the client's product model patterns. Replace Georgian phone pattern with the client's country phone format
9. **ContactCard** — Update phone regex patterns for the client's country format
10. **localStorage key** — Change `karcher_ai_consent` to `{BRAND_SLUG}_ai_consent`
11. **Header avatar letter** — Change "K" to {BRAND_INITIAL}
12. **Trigger button text** — Change "Chat with Nino" to "Chat with {ASSISTANT_NAME}"
13. **Powered by footer** — Keep `Powered by <a href="https://wiil.io"><strong>Wiil</strong></a>` with link
14. **StoreMapView** — Update marker fillColor to brand primary color. Update map center coordinates for the client's country
15. **Landing page section** — REMOVE the landing page div entirely (it's demo-only). The widget should be embed-only.

### Key architecture patterns to preserve:
- `THINKING_MIN_MS = 1800` — minimum thinking indicator duration before streaming
- SSE streaming with abort controller support
- `conversationRef` for maintaining chat history
- `ai-stream` role for in-progress messages, `ai` for completed
- View state machine with `goHome()` reset pattern
- `openServiceChat` intercepts for store/browse → different views
- `openCategoryChat` sends prompt directly (no greeting message)

---

## src/App.css — Complete Reference Template

IMPORTANT: Copy the COMPLETE App.css from the karcher-demo-assistant reference project, then adapt:

### CSS Variable Mapping (find and replace):
```
--k-yellow         -> --brand-primary          (main brand color)
--k-yellow-dark    -> --brand-primary-dark      (darker shade for hover/text)
--k-yellow-light   -> --brand-primary-light     (15% opacity for backgrounds)
--k-yellow-pale    -> --brand-primary-pale      (6% opacity for subtle backgrounds)
--k-dark           -> --brand-dark              (dark color for header, buttons)
--k-bg             -> --brand-bg                (keep #ffffff)
--k-text           -> --brand-text              (keep #1a1a1a)
--k-text-subtle    -> --brand-text-subtle       (keep #6b7280)
--k-border         -> --brand-border            (keep #e5e7eb)
--k-radius         -> --brand-radius            (keep 16px)
--k-btn-radius     -> --brand-btn-radius        (keep 24px)
--k-bubble-radius  -> --brand-bubble-radius     (keep 18px)
```

Do a global find-replace of `--k-` with `--brand-` throughout the file.

Then set the actual color values in :root based on the brand analysis:
```css
:root {
  --brand-primary: #HEX_FROM_WEBSITE;
  --brand-primary-dark: /* darker shade */;
  --brand-primary-light: /* rgba with 0.15 opacity */;
  --brand-primary-pale: /* rgba with 0.06 opacity */;
  --brand-dark: #1a1a1a;
  /* ... rest stays the same */
}
```

### What to keep exactly as-is:
- ALL class names (widget-host, panel, panel-header, home-view, stores-view, catalog-view, etc.)
- ALL responsive breakpoints (640px mobile, 380px small phones)
- ALL animations (msgFade, menuFade, cursorBlink, thinkingPulse, wave)
- Safe area insets for iOS
- Panel dimensions (400px wide, 580px tall)
- Scrollbar styling
- Close button mobile/desktop toggle logic

### What to remove:
- Any landing page CSS (`.landing-*` classes) — demo-only

---

## Dockerfile

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
RUN npm run build
EXPOSE 3001
ENV NODE_ENV=production
CMD ["npm", "start"]
```

---

## Embed Snippet

### embed.html
```html
<!-- {BRAND_NAME} AI Assistant -->
<!-- Paste before </body> on any page -->
<script>
(function() {
  var d = document, s = d.createElement('script');
  s.src = '{DEPLOY_URL}/widget.js';
  s.async = true;
  s.defer = true;
  d.body.appendChild(s);
})();
</script>
```

### public/widget.js
```js
(function() {
  var HOST = window.__ASSISTANT_HOST || '{DEPLOY_URL}';
  var iframe = document.createElement('iframe');
  iframe.src = HOST;
  iframe.style.cssText = 'position:fixed;bottom:0;right:0;width:420px;height:620px;border:none;z-index:99999;display:none;border-radius:16px;box-shadow:0 4px 24px rgba(0,0,0,0.1);margin:0 20px 20px 0;';
  iframe.allow = 'microphone';

  var trigger = document.createElement('button');
  trigger.innerHTML = '<span style="font-weight:800;font-size:20px;color:{BRAND_DARK};">{BRAND_INITIAL}</span>';
  trigger.style.cssText = 'position:fixed;bottom:28px;right:28px;width:56px;height:56px;border-radius:50%;background:{BRAND_PRIMARY};border:none;cursor:pointer;z-index:99998;box-shadow:0 2px 16px rgba(0,0,0,0.12);display:flex;align-items:center;justify-content:center;transition:transform 0.2s;font-family:Inter,sans-serif;';
  trigger.onmouseenter = function() { trigger.style.transform = 'scale(1.08)'; };
  trigger.onmouseleave = function() { trigger.style.transform = 'scale(1)'; };

  var open = false;
  trigger.onclick = function() {
    open = !open;
    iframe.style.display = open ? 'block' : 'none';
    trigger.style.display = open ? 'none' : 'flex';
  };

  window.addEventListener('message', function(e) {
    if (e.data === 'assistant:close') {
      open = false;
      iframe.style.display = 'none';
      trigger.style.display = 'flex';
    }
  });

  // Mobile: full screen iframe
  if (window.innerWidth <= 640) {
    iframe.style.cssText = 'position:fixed;top:0;left:0;width:100vw;height:100vh;border:none;z-index:99999;display:none;';
  }

  document.body.appendChild(iframe);
  document.body.appendChild(trigger);
})();
```

---

## Knowledge Base Population Guide

The knowledge base (`server/knowledge-base.txt`) is the single most important file for AI quality. Structure it like this:

```
=== {BRAND_NAME} Knowledge Base ===
Last updated: {DATE}

--- Company Info ---
Company: {BRAND_NAME}
Website: {BRAND_WEBSITE}
Description: [1-2 sentences about what the company does]
Founded: [year]
Hotline: [main phone number]
Email: [public email addresses — sales@, info@, service@]
Working hours: [Mon-Sat 10:00-19:00, etc.]

--- Products / Services ---

CATEGORY: [Category Name]
Products in this category: [count]
Price range: [min] - [max] {CURRENCY}

[Product Name] — [price] {CURRENCY}
  Description: [1 sentence]
  Key features: [2-3 bullet points]
  Use case: [who is this for]

[Repeat for each product]

[Repeat for each category]

--- Store Locations ---

[Store Name]
  Address: [full address]
  City: [city]
  Phone: [phone number]
  Manager: [name]
  Hours: [working hours]
  Coordinates: [lat, lng] (for Google Maps)

[Repeat for each location]

--- Pricing & Promotions ---
[Active promotions, discount tiers, loyalty programs]
[Delivery policy — free delivery threshold, coverage area]
[Payment methods accepted]

--- Policies ---
Warranty: [warranty terms]
Returns: [return policy]
Service: [service/repair process]
Delivery: [delivery options and costs]

--- FAQ ---
Q: [Common question 1]
A: [Answer]

Q: [Common question 2]
A: [Answer]
```

### Best Practices for KB Quality:
1. **Scrape the website** — Use WebFetch on product pages, about page, contact page, FAQ page
2. **Every product needs**: name, price, 1-sentence description, 2-3 features
3. **Prices must be current** — outdated prices destroy trust
4. **Include what NOT to say** — internal emails, competitor mentions, stock levels
5. **Test with edge cases** — "do you deliver to [remote city]?", "my product broke", "what's the cheapest option?"
6. **Keep it under 15,000 words** — larger KBs increase cost and slow responses

---

## Implementation Steps

After gathering brand info and analyzing the website:

1. Create project directory and all files from templates
2. Extract brand colors from website analysis — set CSS variables
3. Download/reference logo — place in `public/logo.svg`
4. Copy App.jsx from karcher reference — customize all brand-specific parts (see checklist above)
5. Copy App.css from karcher reference — global replace `--k-` with `--brand-`, set color values
6. Remove landing page section from App.jsx (demo-only)
7. Configure languages in LANGUAGES, T, GREETINGS
8. Build SERVICES array based on features needed
9. Build STORES array with geocoded addresses (if store locator)
10. Build PRODUCT_CATEGORIES with consultative prompts (if catalog)
11. Write knowledge base from website content
12. Write system prompt — customize persona, personality, guard rails
13. Set .env with API keys
14. Run `npm install` and verify build
15. Test all views in browser
16. Generate embed snippet with correct deploy URL
17. Add ALLOWED_ORIGINS to .env for CORS

---

## Production Checklist

Before client handoff, verify ALL of these:

- [ ] Rate limiting configured (20 req/min default)
- [ ] CORS origins set in ALLOWED_ORIGINS env var
- [ ] ANTHROPIC_API_KEY in .env (never committed to git)
- [ ] VITE_GOOGLE_MAPS_API_KEY set (if store locator)
- [ ] Knowledge base populated with current products/prices
- [ ] Logo placed in public/logo.svg
- [ ] All --brand-* CSS variables set to extracted brand colors
- [ ] Translations complete for both languages
- [ ] "Powered by Wiil" footer links to https://wiil.io
- [ ] Embed snippet URL updated to production domain
- [ ] Tested on mobile (iOS Safari + Android Chrome)
- [ ] Tested all views: home, chat, stores, catalog
- [ ] Tested language switching resets state correctly
- [ ] Tested error states (disconnect wifi, send while loading)
- [ ] Tested consent screen shows once then persists
- [ ] No internal emails or sensitive data in knowledge base
- [ ] Dockerfile builds and runs correctly
- [ ] git repo has .env in .gitignore
