---
name: product-showcase
description: "Generate a comprehensive marketing website for a web app — multi-page with real screenshots, animated GIF walkthroughs, feature deep-dives, and workflow demonstrations. Browses the running app, captures screens and sequences, and produces a deployable site that actually teaches people what the product does. Especially useful for complex or agentic apps that are hard to explain. Triggers: 'showcase site', 'product page', 'show off the app', 'marketing site', 'demo site', 'product showcase', 'explain the app', 'how do I market this'."
compatibility: claude-code-only
---

# Product Showcase Generator

Generate a marketing website that actually teaches people what a web app does. Not just a hero and feature grid — a multi-page site with real screenshots, animated GIF walkthroughs of workflows, feature deep-dives, and progressive depth from "what is this" to "here's exactly how it works."

Especially valuable for complex apps, agentic AI tools, and anything where a static screenshot doesn't convey the value.

## Depth Levels

| Depth | Output | Duration |
|-------|--------|----------|
| **quick** | Single page — hero, features, CTA. Same as before. | 15-20 min |
| **standard** | Multi-page site — home, features page, how-it-works with screenshots. | 1-2 hours |
| **thorough** | Comprehensive site — home, per-feature pages, animated GIF walkthroughs, use cases, comparison page, docs-style demo. | 3-6 hours |

Default: **standard**

## Browser Tool Detection

Before starting, detect available browser tools:

1. **Chrome MCP** (`mcp__claude-in-chrome__*`) — preferred for authenticated apps
2. **Playwright MCP** (`mcp__plugin_playwright_playwright__*`) — for public apps
3. **playwright-cli** — for scripted flows

## Workflow

### 1. Gather Input

| Field | Required | Example |
|-------|----------|---------|
| App URL | Yes | `https://app.example.com` or `http://localhost:5173` |
| App name | Yes | "Acme CRM" |
| Tagline | No | "The CRM that gets out of your way" |
| Target audience | No | "Small business owners" |
| Pricing info | No | Free tier, $29/mo pro |
| CTA text + URL | No | "Start Free Trial" → signup page |
| Testimonials | No | User provides or skip section |

### 2. Capture Screenshots
Use `capture-screenshots` (shipped in `bin/`) for faster, consistent app captures: desktop (1280x720), mobile (375px), and dark mode variants in one run.
#### Quick capture (all key pages at once):
```bash
capture-screenshots http://localhost:5173 \
  --pages /,/dashboard,/contacts,/settings \
  --output showcase/screenshots \
  --prefix screen \
  --mobile --dark
```
#### For authenticated apps:
Use `--auth user:password` for HTTP Basic Auth.
```bash
capture-screenshots https://app.example.com \
  --pages /,/dashboard,/settings \
  --auth user:password \
  --output showcase/screenshots \
  --mobile --dark
```
#### Authenticated multi-tenant apps
- **Session**: Sign in once; retain one isolated context through A, B, then the signed-in default state. It lives for this run, without a persistent disk profile.
- **Run**: Save as `capture-tenants.mjs` in a project with Playwright installed. Set `APP_USER`/`APP_PASSWORD` in its environment; run `node capture-tenants.mjs`. Never log credentials, auth cookies or storage state.
- **Adapt**: Replace illustrative app IDs and app URL/login path, login fields/button, authenticated and tenant/route readiness selectors, cookie name/domain/path and tenant values, page paths/labels, unique prefixes/slugs and output directory.

```javascript
import { chromium } from 'playwright';

const appUrl = 'https://app.example.com';
const loginPath = '/login';
const outputDir = 'showcase/screenshots';
const selectors = { user: '#email', password: '#password', submit: '#login', authenticated: '#signed-in', tenant: '#tenant', route: '#title' };
const tenantCookie = { name: 'org_cookie', domain: 'app.example.com', path: '/' };
const tenants = [
  { value: 'tenant-a', prefix: 'tenant-a', label: 'Tenant A' },
  { value: 'tenant-b', prefix: 'tenant-b', label: 'Tenant B' }
];
const pages = [
  { path: '/dashboard', slug: 'dashboard', label: 'Dashboard' },
  { path: '/settings', slug: 'settings', label: 'Settings' }
];
const defaultState = { prefix: 'default', label: 'No tenant selected' };
const username = process.env.APP_USER;
const password = process.env.APP_PASSWORD;
if (!username || !password) throw new Error('Set APP_USER and APP_PASSWORD');

async function capturePages(page, state) {
  for (const route of pages) {
    const response = await page.goto(new URL(route.path, appUrl).href);
    if (!response.ok()) throw new Error('Capture page returned HTTP ' + response.status());
    await page.locator(selectors.authenticated).waitFor();
    await page.locator(selectors.tenant).filter({ hasText: state.label }).waitFor();
    await page.locator(selectors.route).filter({ hasText: route.label }).waitFor();
    await page.screenshot({ path: outputDir + '/' + state.prefix + '-' + route.slug + '.png' });
  }
}

const browser = await chromium.launch({ headless: true });
try {
  const context = await browser.newContext({ viewport: { width: 1280, height: 720 } });
  try {
    const page = await context.newPage();
    await page.goto(new URL(loginPath, appUrl).href);
    await page.locator(selectors.user).fill(username);
    await page.locator(selectors.password).fill(password);
    await page.locator(selectors.submit).click();
    await page.locator(selectors.authenticated).waitFor();
    for (const tenant of tenants) {
      await context.addCookies([{ ...tenantCookie, value: tenant.value }]);
      await capturePages(page, tenant);
    }
    await context.clearCookies(tenantCookie);
    await capturePages(page, defaultState);
  } finally {
    await context.close();
  }
} catch {
  throw new Error('Capture failed; check login, selectors and expected page content');
} finally {
  await browser.close();
}
```
B replaces A at the same cookie tuple; clear only that name/domain/path. Filtered `clearCookies` requires [Playwright 1.43+](https://playwright.dev/docs/api/class-browsercontext#browser-context-clear-cookies); unfiltered `clearCookies()` removes login cookies too.
#### What to capture:

- **a. First Impression** — the main page/dashboard becomes the hero image. Note the immediate value proposition.
- **b. Features** — each major section. Use `--pages` with all nav paths. Capture 6-10 key screens that tell the product story.
- **c. "How It Works" flow** — the main workflow in sequence. Run `capture-screenshots` multiple times with `--prefix workflow-step` as you navigate through the flow steps.
- **d. Detail shots** — zoom into specific UI elements. Use `--full-page` for scrollable content.
- **e. Both modes** — `--dark` flag captures light and dark variants automatically. Use the best-looking mode for the hero.
#### Post-capture optimisation:
```bash
img-process batch showcase/screenshots --action optimise --max-width 1920 -o showcase/screenshots-opt
```
#### f. Extract the Value Propositions

- Ask **why does the user care?**, not just list features. BAD: "Contact management page". GOOD: "See every client, their history, and what needs attention — in one view".
- BAD: "Search functionality". GOOD: "Find anything in seconds — semantic search understands what you mean, not just what you type".
### 3. Generate the Site
#### Quick Mode: Single Page (same as before)
One HTML file: hero + feature grid + CTA. Use for MVPs and quick marketing.
#### Standard Mode: Multi-Page Site
```
showcase/
├── index.html              # Home — hero, overview, feature highlights, CTA
├── features.html           # All features with screenshots and descriptions
├── how-it-works.html       # Step-by-step workflow walkthrough with screenshots
├── screenshots/            # All captured images
│   ├── hero.png
│   ├── feature-*.png
│   ├── workflow-step-*.png
│   └── *.gif               # Animated walkthroughs
└── styles.css              # Shared styles (or inline Tailwind CDN)
```

- **Home page**: Hero GIF or key screenshot, the best 3-4 feature highlights, a "How It Works" summary (3 steps), and CTA.
- **Features page**: Give every feature a real screenshot, benefit-focused description and enough space to explain it; group by category at 6+.
- **How It Works page**: Show the primary workflow step-by-step. Each step needs a screenshot or animated GIF, heading and 2-3 sentences; answer "ok but what does using it actually look like?"
#### Thorough Mode: Comprehensive Site
```
showcase/
├── index.html              # Home — hero, overview, value proposition
├── features/
│   ├── index.html          # Feature overview grid
│   ├── [feature-1].html    # Deep-dive: one page per major feature
│   ├── [feature-2].html    # Each with screenshots, GIFs, use cases
│   └── [feature-n].html
├── how-it-works.html       # Full workflow walkthrough
├── use-cases/
│   ├── [use-case-1].html   # Scenario: "A day in the life of..."
│   └── [use-case-2].html   # Scenario: "When a new client calls..."
├── compare.html            # "Why [app] vs alternatives" (optional)
├── screenshots/
│   ├── hero.png
│   ├── feature-*/           # Per-feature screenshot sets
│   └── workflows/           # Animated GIFs
└── styles.css
```

- **Per-feature deep dives**: Give each major feature a page with a hero screenshot in action, "What it does" (1-2 paragraphs on value), and "How it works" (step-by-step screenshots or GIF).
- **Per-feature value**: "Why it matters" explains the problem solved. Edge cases or power-user tips help advanced users. Link to next feature for flow between pages.
- **Use case pages**: Tell real scenarios: "It's Monday morning. You open the dashboard and see..." Walk through a realistic workflow with screenshots at each step and show the outcome. These persuade best when apps are hard to explain.
- **Comparison page** (optional): "Why [app] vs [alternatives]". Use honest comparison, not marketing fluff: Feature table, key differentiators, who it's best for.
### 4. Animated GIF Walkthroughs
Show actual interactions for key features; static screenshots miss workflow. Using Playwright or Chrome MCP, capture sequential screenshots, combine into GIFs, then display in HTML:

1. Navigate to the starting state
2. Start recording screenshots at ~2fps
3. Perform the workflow (click, type, navigate)
4. Stop recording
5. Combine frames into a GIF

```bash
# Capture each step with a sequential prefix
capture-screenshots http://localhost:5173/clients \
  --prefix workflow-01 --output .jez/screenshots
# ... navigate to next state ...
capture-screenshots http://localhost:5173/clients/new \
  --prefix workflow-02 --output .jez/screenshots

# Combine frames into GIF (Python one-liner using Pillow)
python3 -c "
from PIL import Image; import glob
frames = [Image.open(f) for f in sorted(glob.glob('.jez/screenshots/workflow-*.png'))]
frames[0].save('showcase/screenshots/workflows/create-client.gif',
    save_all=True, append_images=frames[1:], duration=500, loop=0)
"
```

- **Create**: Animate the primary "create something" flow (2-4 seconds).
- **Interactions**: Show search/filter results appearing, drag-and-drop or reorder, and the satisfying Dark mode toggle.
- **Magic moments**: Show any impressive "magic moment", such as AI classification, instant search, real-time update.
- **Length and size**: Max 10 seconds / 20 frames; shorter is better. Capture at 1280x720, display at 640x360 (half size for file size).
- **Playback**: Pause briefly (3 frames) on the final state so viewers see the result. Loop continuously — no "click to play".
- **Large files**: If >5MB, use fewer frames or crop to the relevant area.

```html
<div class="browser-frame">
  <div class="browser-frame-bar">
    <span class="browser-frame-dot"></span>
    <span class="browser-frame-dot"></span>
    <span class="browser-frame-dot"></span>
  </div>
  <img src="screenshots/workflows/create-client.gif"
       alt="Creating a new client in 3 clicks"
       loading="lazy" width="640" height="360">
</div>
```
### 5. Explaining Agentic / AI Apps
Agentic AI work is often invisible; chat-only screenshots are not compelling. Use these patterns to show the work users never see:

| Pattern | What it shows | Example |
|---------|-------------|---------|
| **Before/after** | What the user used to do manually vs what the agent does | "Used to: copy-paste from 3 systems. Now: agent does it in background." |
| **Timeline** | What happens over time — show the agent working across hours/days | "8am: agent checks inbox. 9am: classifies 47 emails. 10am: flags 3 urgent." |
| **Result showcase** | Skip the process, show the output | "Agent mined 1,200 emails → 89 clients, 340 contacts, 2,100 knowledge facts" |
| **Side-by-side** | Show the agent's work next to what a human would have done | Split screen: left is the raw email, right is the extracted structured data |
| **Magic moment GIF** | One animation of the most impressive thing | User asks a question → agent searches knowledge → returns answer with sources |

- **Outcome**: Lead with the outcome, not technology ("Know every client's history" not "AI-powered CRM").
- **Evidence**: Show volume ("Processed 1,200 emails" rather than "Processes your emails") and time comparisons ("What took 2 hours now takes 30 seconds").
- **Plain language**: Avoid jargon ("Finds connections in your data" not "Semantic vector search with RAG").

### 5. Screenshot Presentation

Screenshots are shown in browser-frame mockups using CSS:

```css
.browser-frame {
  border-radius: 8px;
  box-shadow: 0 25px 50px -12px rgba(0,0,0,0.25);
  overflow: hidden;
  border: 1px solid rgba(0,0,0,0.1);
}
.browser-frame-bar {
  background: #f1f5f9;
  padding: 8px 12px;
  display: flex;
  gap: 6px;
}
.browser-frame-dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background: #e2e8f0;
}
```

This gives screenshots a polished "app in a browser" look without needing to edit the images.

### 6. Site Navigation (multi-page)

Multi-page sites need consistent navigation:

```html
<nav>
  <a href="/">Home</a>
  <a href="/features/">Features</a>
  <a href="/how-it-works.html">How It Works</a>
  <a href="/use-cases/">Use Cases</a>  <!-- thorough only -->
  <a href="#pricing">Pricing</a>
  <button>Get Started</button>
</nav>
```

- Sticky nav on all pages
- Current page highlighted
- Mobile hamburger menu
- CTA button always visible in nav

### 7. Output

After generating, tell the user:
- Preview: `python3 -m http.server -d showcase` then open `http://localhost:8000`
- Deploy: drag the `showcase/` folder to Cloudflare Pages, Netlify, or any static host
- List any placeholder content that needs replacing
- Note which GIFs were generated and their file sizes

## Design Patterns

**Colour**: If the app has a clear brand colour, extract it and use it as the primary. Otherwise default to a neutral palette (slate/blue).

**Typography**: System font stack (no external requests). Same approach as `landing-page`.

**Responsive**: Mobile-first, screenshots scale down gracefully. On mobile, screenshots stack vertically instead of grid.

**Dark mode**: Three-state toggle (light/dark/system) with CSS custom properties.

**Performance**: Lazy-load screenshot images. Tailwind CDN for styling. No build step.

## Making It Look Premium

### Screenshot Enhancement
- **Clean data**: Before screenshotting, make sure the app has realistic data — not "Test Client" and "asdf@example.com"
- **Consistent window size**: All screenshots at the same viewport width (1280x720)
- **No browser chrome in screenshots**: Use the CSS browser-frame mockup instead of capturing the actual Chrome toolbar
- **Highlight the action**: If a screenshot shows a feature, have that feature active/open (modal open, filter applied, item selected)

### Visual Storytelling
- **Lead with the dashboard**: The hero screenshot should show the app populated with data, looking alive and useful
- **Show the app doing work**: Screenshots of results are better than screenshots of empty forms
- **Progressive reveal**: Hero shows the big picture, how-it-works shows the flow, features show the details
- **End with the payoff**: The final section before the CTA should show the result — a report generated, a task completed, a client managed

### Mix Screenshots with Mockups
Real screenshots show the product is real. Mockups show the product is polished. Use both:
- **Hero**: Real screenshot in a browser-frame mockup — proves the app exists and looks good
- **How It Works**: Can use simplified mockup illustrations (CSS shapes, icons, arrows) for the flow steps, with real screenshots for the detail
- **Feature grid**: Real screenshots — users want to see the actual UI
- **Mobile showcase**: CSS device frame mockup (phone outline) around a mobile screenshot
- **Dashboard/overview**: Real screenshot — this is the money shot

**Phone mockup frame** (CSS):
```css
.phone-frame {
  border: 8px solid #1f2937;
  border-radius: 32px;
  overflow: hidden;
  max-width: 280px;
  box-shadow: 0 25px 50px -12px rgba(0,0,0,0.3);
}
.phone-frame-notch {
  background: #1f2937;
  height: 24px;
  border-radius: 0 0 16px 16px;
  width: 120px;
  margin: 0 auto;
}
```

### Animation (CSS-only, no JS required)
```css
/* Fade in sections as user scrolls (intersection observer via CSS) */
@keyframes fadeInUp {
  from { opacity: 0; transform: translateY(20px); }
  to { opacity: 1; transform: translateY(0); }
}
.section { animation: fadeInUp 0.6s ease-out both; }

/* Subtle float on hero screenshot */
@keyframes float {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-8px); }
}
.hero-screenshot { animation: float 6s ease-in-out infinite; }
```

### Gradient Accents
Extract the app's primary colour and use it for subtle gradient backgrounds on sections:
```css
.hero { background: linear-gradient(135deg, var(--primary) 0%, transparent 60%); }
.cta-banner { background: linear-gradient(135deg, var(--primary), var(--primary-dark)); color: white; }
```

## Deployment

The showcase site needs to be hosted somewhere real — not just opened from a file. Choose based on what's available:

### Option 1: Cloudflare Workers with Static Assets (recommended)

If the app being showcased is already on Cloudflare, deploy the showcase as a Worker too:

```bash
# In the showcase/ directory, create a minimal wrangler.jsonc:
{
  "name": "myapp-showcase",
  "main": "src/index.ts",  // or use static-only pattern
  "compatibility_date": "2026-03-01",
  "assets": { "directory": "./" }
}

npx wrangler deploy
```

Or attach a custom domain: `showcase.myapp.com` or `myapp.com` (if the app itself is at `app.myapp.com`).

### Option 2: Cloudflare Pages (drag-and-drop)

Dashboard → Pages → Create → Upload assets → drag the `showcase/` folder. Gets a `.pages.dev` URL instantly.

### Option 3: Static host (Netlify, Vercel, GitHub Pages)

Any static host works — it's just HTML + CSS + images. No server-side code.

### Custom Domain

For a professional showcase, always use a custom domain:
- `myapp.com` — showcase is the main site, app is at `app.myapp.com`
- `www.myapp.com` → showcase, `app.myapp.com` → the actual app

## Contact / Enquiry Forms

Every showcase needs a way for interested people to get in touch. Don't just put "email us" — add a real form.

### Pattern 1: Formspree / Formsubmit (no backend needed)

```html
<form action="https://formspree.io/f/YOUR_FORM_ID" method="POST">
  <input type="text" name="name" placeholder="Your name" required>
  <input type="email" name="email" placeholder="Your email" required>
  <textarea name="message" placeholder="Tell us about your needs"></textarea>
  <button type="submit">Get in Touch</button>
</form>
```

Free tier handles most showcase volumes. Submissions go to your email.

### Pattern 2: Cloudflare Worker endpoint

If the showcase is on Workers, add a `/api/enquiry` route:

```typescript
app.post('/api/enquiry', async (c) => {
  const { name, email, message } = await c.req.json();
  // Send via SMTP2Go, Resend, or store in D1
  await sendEmail({ to: 'hello@company.com', subject: `Enquiry from ${name}`, body: message });
  return c.json({ success: true });
});
```

### Pattern 3: Turnstile + email (spam protection)

Add Cloudflare Turnstile to prevent bot submissions:
```html
<div class="cf-turnstile" data-sitekey="YOUR_SITE_KEY"></div>
```

Verify the token server-side before processing the submission.

### What the form should capture

| Field | Required | Why |
|-------|----------|-----|
| Name | Yes | Personalise follow-up |
| Email | Yes | Reply channel |
| Company/Role | No but helpful | Qualify the lead |
| Message / "What are you looking for?" | No but helpful | Context for response |
| Phone | No | Some people prefer a call |

**After submission**: redirect to a thank-you page or show an inline confirmation. Don't just reset the form silently.

## Additional Pages Worth Generating

| Page | When to include | What it does |
|------|----------------|-------------|
| **Pricing** | If pricing is defined | Tier cards, feature comparison, FAQ about billing |
| **About** | For products with a story | Who built it, why, the journey (pairs with jez-voice) |
| **Changelog** | For shipped products | Recent updates, shows the product is actively developed |
| **Docs link** | If app-docs exist | Link to the user guide generated by `/app-docs` |
| **Blog / Articles** | If content exists | Link to newsletter archive or blog posts about the product |
| **Privacy / Terms** | Always | Even placeholder — shows legitimacy |
| **Status page** | For SaaS | Link to uptime monitoring (Cloudflare health checks) |

## SEO Basics

Every showcase page should have:
```html
<title>[App Name] — [Tagline]</title>
<meta name="description" content="[One sentence value proposition]">
<meta property="og:title" content="[App Name]">
<meta property="og:description" content="[Value proposition]">
<meta property="og:image" content="screenshots/hero.png">
<link rel="canonical" href="https://myapp.com/">
```

For multi-page: each page gets its own title and description. Don't use the same meta across all pages.

Use the `seo-local-business` skill for JSON-LD if the product serves a specific location.

## Quality Rules

1. **Only screenshot real features** — don't fabricate capabilities the app doesn't have
2. **Choose the best screenshots** — not every page is showcase-worthy. Pick the ones that tell the product story
3. **Write benefit-focused copy** — "See all your contacts in one place" not "Contact list page"
4. **No fake testimonials** — leave the section out if none provided
5. **No invented statistics** — leave stats blocks empty if no real data
6. **Deploy to a real URL** — don't leave it as local files. A showcase needs to be shareable
