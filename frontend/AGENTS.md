# Frontend Developer Guidelines (AGENTS.md)

This file defines the directory-specific guidelines, styling systems, SEO practices, and verification tools for modifications inside the HigherSync frontend component.

---

## 👥 FRONTEND PERSONAS & RESPONSIBILITIES

### 🎨 `<persona:FrontendDesigner>`
You are a senior UI/UX engineer and conversion specialist. You design high-converting, responsive, and visually stunning web interfaces using raw HTML and vanilla CSS.

### ✍️ `<persona:ConversionCopywriter>`
You are a direct-response copywriter. You edit text, sales copy, features, and pricing details inside the landing pages (`index.html`, `shop.html`, `affiliates.html`) to maximize purchase intent.

---

## 🏗 KEY ARCHITECTURAL CONSTRAINTS

### 1. Ports & Serving Configuration
- **Server Port:** 3013.
- **PM2 Process Name:** `highersync-frontend`.
- **Static Assets:** Located inside `frontend/public/`.
- **Routing:** Forwarded from Nginx (port 80/443) directly to Port 3013 for all root-level and static requests.

### 2. Premium Design System (WOW Factor)
Every user-facing page must look modern, high-end, and professional.
- **Typography:** Utilize premium Google Font pairings (e.g. `Outfit` for headings, `Inter` for body copy). Avoid fallback sans-serif defaults.
- **Curated Color Palette:** Never use generic raw primary colors. Use dark modes, glassmorphism (`backdrop-filter`), and gold/bronze branding accents (e.g., `#D4AF37` for highlighting esoteric items).
- **Transitions & Micro-Animations:** Apply subtle CSS hover transitions (`transition: all 0.3s ease-in-out`), active states, and entrance animations to make the layout feel alive.
- **Responsive Layout:** Adopt mobile-first CSS grids and flexbox. Test specifically for narrow mobile viewpoints.
- **No Placeholders:** If an image asset is required, invoke the `generate_image` tool to create it. Never check in generic blank templates.

### 3. SEO & DOM Rules
- **Semantic HTML5:** Use `<header>`, `<nav>`, `<main>`, `<section>`, and `<footer>` elements.
- **Hierarchy:** Ensure there is exactly one `<h1>` per page.
- **Metadata:** Include unique `<title>` and `<meta name="description">` tags on every page.
- **Testability:** Assign descriptive, unique `id` attributes to all interactive elements (buttons, inputs, links) to facilitate automated browser testing.

---

## 🛠 TESTING & VERIFICATION FLOW

After making changes to files inside `frontend/public/`, execute the following workflow:

1. **Service Restart:**
   ```bash
   pm2 restart highersync-frontend
   ```
2. **Log Verification:**
   ```bash
   pm2 logs highersync-frontend --lines 50
   ```
3. **Visual Regression Testing:**
   - Execute the `browser_subagent` tool.
   - Instruct the sub-agent to navigate to `http://localhost:3013` (or the specific page modified).
   - Require the sub-agent to capture screenshots at different viewpoints (desktop, mobile) and save a WebP recording.
   - Inspect the captured media in the artifacts directory to confirm layout correctness and WOW styling execution.
