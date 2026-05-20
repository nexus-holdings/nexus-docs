# Prompt for Claude Design — Nexus Docs design system

> Paste the section between the `---` lines into Claude Design (or any AI design assistant). Adjust references if your taste shifts.

---

I'm designing a developer documentation site called **Nexus Docs**. I want a coherent design system before I commit to component styling.

## What Nexus is

Nexus is a **platform / substrate** for self-improving agentic systems — a control plane plus memory plus execution stack that vertical AI products run on top of. The audience for the docs is: (1) the platform's primary builder, (2) future technical collaborators, (3) eventually external developers and partners. The docs are technical, not marketing.

## The vibe I want

- **Modern, stylish, minimal, functional.**
- Should feel like the docs of a serious infra company that hires excellent designers (Linear, Resend, Stripe, Vercel, Modal, Inngest, Plain, Cursor) — not a hobby project.
- Reading it should make a developer think "these are cool docs" — quietly. Not flashy.
- **Not gaudy, not gauche.** No rainbow gradients, no chunky drop shadows, no emoji-heavy callouts, no marketing hero sections, no Bootstrap-era card stacks.

## Design constraints

- **Dark theme is primary** (light theme can be a secondary deliverable but it's not the main use case)
- The base palette is navy/charcoal — slate-900 body, slate-800 surfaces, single accent
- One accent colour, used surgically. I'm currently using a soft blue (~#60a5fa) but open to alternatives (sage, dusty coral, warm amber, muted teal — anything that feels considered, not corporate)
- **Typography matters.** Body should be a clean variable sans (Inter, Geist, Söhne, GT America, IBM Plex Sans — I'm flexible). Code should be a thoughtful mono (JetBrains Mono, Söhne Mono, Berkeley Mono, Geist Mono).
- Generous whitespace and line-height. Reading-first.
- Subtle borders (1px hairlines), not shadows.

## Components I need designed (in priority order)

1. **Landing page hero** — Title + short description + 6 entry cards. Should set the tone immediately. Avoid the typical "abstract gradient mesh + giant CTA button" look.
2. **Sidebar navigation** — left rail, collapsible sections, an active-state indicator that's elegant (not a heavy filled bar).
3. **Page header** — section breadcrumb + title + optional "Updated: ..." or "Status: living document" badge.
4. **In-page TOC** — right rail, anchor links, current-section indicator.
5. **Code blocks** — line numbers, copy button, syntax highlighting, optional filename/lang label. Comments inside the code should be **lighter** than code so they recede.
6. **Inline code** — distinct from prose but not jarring. Should feel like a typed key, not a button.
7. **Callouts / admonitions** — note, warning, danger, tip, info, deprecated. Each should be visually distinct but use a shared shape language. **No coloured backgrounds — use border-left + a small icon.**
8. **Tables** — alternating row hairlines, header in a slightly different weight, code-in-cells handled cleanly.
9. **Diagrams** — I use Mermaid for flowcharts. I want them to feel native, not embedded — same line weight + colour palette as the rest of the site. Possibly with a "hand-drawn" feel, possibly very crisp — let's see what fits.
10. **Cards** — used for landing page entries + occasionally inline. Single-border, subtle hover state (no scale transform, just border colour shift).
11. **API / Reference blocks** — endpoint signature, parameter list, example request/response. Should feel like Stripe's API reference but darker.
12. **404 / not-found page** — opportunity for a moment of personality. Subtle.

## What to avoid

- Multiple competing accent colours
- Filled coloured callout backgrounds (red boxes, yellow boxes, etc.)
- Drop shadows, especially diffuse ones
- Justified text
- Slick animations / micro-interactions that draw attention to themselves
- Marketing-style hero sections with massive headlines
- Decorative background patterns
- Gradients across whole sections
- Emoji-heavy iconography (icons OK; emojis as section anchors not)
- "Friendly" anthropomorphic illustrations
- Sticky elements that shrink/animate on scroll
- Coloured logo blob on the landing page

## Reference sites that hit the brief

Strongly aligned:

- **linear.app/docs** — clean typography, restrained palette, beautifully composed
- **resend.com/docs** — dark charcoal, single accent, perfect mono-headers
- **plain.com/docs** — minimal but warm, considered colour
- **modal.com/docs** — modern, clean grids, dark theme done well
- **inngest.com/docs** — dense info, clean visual rhythm

Useful but lean further into marketing than I want:

- vercel.com/docs
- stripe.com/docs
- cursor.com/docs

## What I want from you

1. A **palette** — 1 base, 1 surface, 1-2 accent. Hex values. Reasoned.
2. A **type stack** — body, headings, mono. Specific font choices with fallbacks.
3. A **spacing scale** — modular, ~6 steps.
4. **Component sketches** for the components listed above (in priority order; even ASCII or rough wireframes are fine — I'll translate to CSS).
5. A short **rationale** for each major choice — why this colour, why this font, why this border weight. So I can extend the system as new components appear.
6. **3 things you'd say no to that I might be tempted by** — design discipline I should hold to.

Deliver the design system as a single document I can hand to an implementer. Markdown is fine.
