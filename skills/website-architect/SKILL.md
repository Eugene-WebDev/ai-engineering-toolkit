---
name: website-architect
description: Design and audit websites to be agent-friendly. Use when building, reviewing, or refactoring web UI for compatibility with AI agents that parse pages via screenshots, raw HTML, and the accessibility tree. Covers semantic HTML, stable layouts, ARIA, and cross-modality signals.
---

# website-architect — Build Agent-Friendly Websites

## How agents perceive websites

Agents can view a website in 3 primary ways: **screenshots**, **raw HTML**, and the **accessibility tree**.

### Screenshots

The agent takes a snapshot of the rendered page and uses a vision model to identify elements. Based on the screenshot, the agent can recognize that a search bar at the top-right is a global search, while a box in the middle is likely a form field. Visual cues can be helpful, as agents can use color, size, and proximity to determine importance. A big **Delete** button will likely be interpreted with more caution than a small "Help" link. However analyzing screenshots can be slow and expensive (in terms of used tokens), making it better as a backup when the structure is confusing.

### HTML

The agent analyzes the DOM and reads the HTML. It understands how elements are nested, the logical hierarchy of the DOM tree, attributes like IDs and classes that define structure, and raw data strings that form the site's informational backbone. This helps the agent understand the relationship between elements. If a "Buy Now" button is inside a product container, the agent assumes that button belongs to _that_ specific product.

### Accessibility tree

The accessibility tree is a browser-native API distills the DOM into what's most important: roles, names, and states of interactive elements. It's the page's semantic summary, used by assistive technology. For an AI agent, it functions as a high-fidelity map that ignores the visual "noise" of CSS to focus on pure utility. By interpreting this tree, an agent can learn the functional intent of every toggle, slider, and input field.

### Combined modalities

Relying on a single input creates a semantic gap. For example, in the DOM, an agent might see a `<div>` without knowing you've actually configured this as a functional button with CSS and JavaScript. With screenshots, it's possible an agent may identify where that button sits on the screen, but it's still unaware of the button's intended destination or action that it's designed to trigger.

Modern agents, therefore, combine multiple modalities. They use the DOM and accessibility tree to get a clean, structured list of interactive elements, and then cross-reference that with a visual rendering to understand layout, grouping, and visual cues.

Our job is to provide clean signals across all these channels.

## Guidelines

To help agents navigate your website, consider the following:

- All necessary actions, taken by a human or agent, should be clearly reflected in the interface.
- Ensure stable layout. Agents that take screenshots will likely be confused if your website layout is constantly shifting, for example when an **Add to cart** button on product page is in different location for each product category.
- Avoid "ghost" elements or transparent overlays that might hide interactive elements. The visual analysis done by the agent might discard nodes that are covered, even if the node appears transparent.
- Design actionable elements with semantic HTML. Prefer `<button>` and `<a>` tags over modified `<div>` and `<span>` elements. Agents recognize these as interactive.
  - If you cannot use semantic HTML, always provide the element the appropriate `role` and `tabindex`. For example, `<div role="button">`.
- Set `cursor: pointer` in CSS, which is a strong signal for actionability.
- Add the `for` attribute on `<label>` tags to link them to inputs. This helps the AI agent understand the purpose of a field by indicating the label text that is directly attached to the action string.
- Make sure any interactive elements required to continue the user journey have a visible area larger than 8 square pixels, to avoid being filtered out by visual analysis.

## Auditing agent-readiness (Chrome Lighthouse)

Chrome ships an experimental **"Agentic Browsing"** Lighthouse category that scores how well AI agents can use your site — treat it as the objective test for everything above.

- Enable: Chrome 146+ (Beta channel may be required) → visit `chrome://flags/#enable-webmcp-testing` and enable → relaunch → DevTools → Lighthouse tab → check **Agentic Browsing** → run.
- What it audits: `llms.txt` presence, **WebMCP** compatibility, accessibility labeling/roles, and **Cumulative Layout Shift (CLS)** (layout stability — the same stability agents-with-screenshots need).
- Reality check: most sites currently score low on WebMCP support, but accessibility and layout-stability fixes help humans and agents immediately. Framing that unlocks budget — a11y work "suddenly has a second customer."

## Emit modern platform code, not JS state machines

Agents (and the devs/AI tools building your UI) tend to reach for heavy JavaScript when a native browser primitive would produce cleaner, more parseable markup. Prefer declarative HTML — it exposes intent to the DOM and accessibility tree for free. The open-source **Modern Web Guidance** project (Chrome + Edge) codifies this; for Claude Code: `/plugin marketplace add GoogleChrome/modern-web-guidance` then `/plugin install modern-web-guidance@googlechrome`.

- **Tooltips/popovers:** use the native `popover` attribute (`popover="hint"` for tooltips) + CSS Anchor Positioning (`anchor-name` / `position-anchor` / `anchor()`). You get light-dismiss, focus management, and Escape-to-dismiss natively — no `js-hidden` classes or event handlers for an agent to misread.
- **Toasts/dialogs:** `popover="manual"` + declarative `popovertarget` / `popovertargetaction` for close buttons; the browser Top Layer handles stacking.
- **Enter/exit animations:** `@starting-style` + `transition: ... allow-discrete` (plus `overlay`) instead of JS timeout choreography.
- Caveats: `interestfor` (Interest Invokers) still needs a polyfill; CSS Anchor Positioning is Baseline 2025. And moving component styles into global stylesheets to satisfy a tool can break encapsulation and leave orphaned CSS — keep ownership clear.

The win is the same as the rest of this skill: fewer bespoke `<div>` widgets, more semantic elements the agent already understands.

## Don't wall agents (and disabled users) out

- **CAPTCHAs** are the anti-pattern of agent-readiness: visual/audio/time-based challenges block legitimate users and assistive tech alike. Prefer risk-based auth, device-trust signals, and human-friendly verification over challenge-gates on critical paths.

## Active tier: WebMCP (optional, early — added 2026-10-08)

Beyond being readable, a site can expose **tools** agents call directly (W3C Community Group draft "WebMCP"; source: freeCodeCamp, 2026-05-28).
- Client-side: `navigator.modelContext` (`registerTool()` in the spec; the `@mcp-b/global` polyfill uses `provideContext()` and degrades silently elsewhere). Server-side: declarative manifest at `/.well-known/webmcp` (name, tools with `inputSchema`, `endpoint`, `method`) — location not canonical yet.
- Status: partial in Chrome Canary, production support ~2027+, ~0% adoption, no production agent calls it yet. Treat as cheap future-proofing, never a substitute for semantic HTML and a clean accessibility tree.
- **Read-only tools only** to start (search, structured content) — the spec defines no auth. Tool descriptions are untrusted input to agents (see `ai-agent-security`).

## Next steps

Everything we suggest to make a site "agent-ready" also makes sites better for humans.

Making websites AI agents-friendly is an incentive to recommit to its foundational principles of building well-structured, accessible, and semantic websites.

_to be continued..._
