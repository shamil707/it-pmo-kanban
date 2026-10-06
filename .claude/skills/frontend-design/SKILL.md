---
name: frontend-design
description: Visual design guidance for the IT PMO Kanban board (index.html). Use when adding or restyling UI, such as cards, columns, the Add Task panel, filters, toasts or the summary strip. It keeps changes within the project's corporate blue tokens, system fonts and single-file vanilla constraints while avoiding templated defaults.
license: Complete terms in LICENSE.txt
---

> **Modified for the IT PMO Kanban project.** The "Project overrides" section and the description above were added on 2026-10-06. The original is `anthropics/skills` → `skills/frontend-design`, under Apache-2.0 (see LICENSE.txt). Everything below the overrides is the upstream text.

## Project overrides: IT PMO Kanban (these win over the generic guidance below)

The brief is fixed. Don't re-derive it or propose a new identity.

- **Subject:** an internal IT PMO Kanban board for a *fictitious* bank, used for demos and training.
- **Audience:** project managers and engineers.
- **Primary job:** see the state of work at a glance, move cards, and spot what's **Blocked** and **Overdue**.
- **Character:** a dense, calm, functional work tool, not a marketing page.
  - There's no hero section: the board and the header summary strip *are* the first impression.
  - Spend any boldness on making status, priority and overdue signals faster to read, not on decoration.

**Hard constraints** (see `CLAUDE.md`; a design idea that breaks one of these is rejected):
- **One file:** `index.html`, vanilla HTML/CSS/JS, and it must still open by double-click.
- **No external resources:** no web fonts, CDNs, icon libraries or image files.
  - **Typography:** use the existing system stack (`--font`, `--font-mono`). The upstream advice to "choose distinctive typefaces" doesn't apply here. Create hierarchy with size, weight, spacing and the type scale instead.
  - **Icons:** inline SVG or Unicode glyphs only. Glyphs need `aria-hidden="true"`, and icon-only buttons need an `aria-label`.
- **Colour:** use the corporate blue palette from the `:root` tokens only (`--blue-900` to `--blue-50`, the neutrals, `--prio-*`, `--pill-*`, `--status-*`, the feedback tokens).
  - Add new tokens instead of raw hex values in rules.
  - Keep the neutral "IT PMO" text wordmark, and never imitate a real bank's logo, colours or systems.
- **Never colour alone:** priority is always written out as text in its pill, and Overdue is a text badge.
- **CSS:** no `!important`. Keep specificity flat (one class per rule where possible); the card, pill and column classes rely on this.
- **Responsive:** columns sit side by side on desktop, stack below 768px, and there's no horizontal scrolling at 375px.
- **Accessibility floor:** visible `:focus-visible` rings, and `prefers-reduced-motion` respected.

**Code patterns to preserve:**
- **Rendering:** card markup is produced only by `renderCard()` and inserted only by `renderBoard()`. Don't change card contents directly in the DOM.
  - Per-card UI state, such as an open menu or a pending delete, lives in `state.ui`.
  - After a re-render, put focus back with `focusInCard()` or `focusColumnHeading()`.
- **Escaping:** every user-supplied value passes through `escapeHtml()`.
- **Motion** is only for feedback: the drop-target highlight and the toast entrance. Don't add decorative entrances or hover animation on every card.

**Words in this UI** (follows the upstream "writing" guidance):
- An action keeps the same name throughout the flow: the **Add task** button produces "`<ID>` added to `<column>`".
- Errors say what to fix ("Due date cannot be in the past.").
- Empty columns say "No tasks".
- Use plain banking-IT vocabulary: workstream, assignee, UAT, CAB, vendor.

**Process for this repo:**
1. **Plan:** state any token or layout changes as a short list. Check it against the constraints above *before* writing code.
2. **Build:** edit `index.html` only.
3. **Check:**
   - Run the `CLAUDE.md` constraint search and `node --check`.
   - Capture screenshots with the Playwright MCP server at 1440×900 and 390×844 and look at them.
   - Refresh `docs/screenshots/` when the change is visible in the README.

# Frontend Design

Approach this as the design lead at a design studio known for giving every client a distinct visual identity that is not mistaken for anyone else's. This client has already rejected proposals that felt cliché or templated, and is paying for a distinctive point of view: make deliberate, opinionated choices about palette, typography, and layout that are specific to this brief, and take aesthetic risk if justified.

## Ground your designs in the subject matter

If the brief does not identify what the product or subject matter is, identify it yourself before designing, and confirm with the client. You can come up with one concrete subject, the design's audience, and the design's primary job, as a proposal. If there's any information in your memory about the client's preferences or context about what they're building, use that as a hint. The subject's industry, subject matter, materials, and vernacular are where distinctive visual choices come from — a design for a toy for girls aged 8–11 will be very aesthetically different from a dashboard for financial analysts. Build with the brief's real content and subject matter throughout.

## Design principles

For web designs, the hero is the first thing viewers will see. Open with the most characteristic thing in the subject's world, in the form that is most appropriate: a headline, an image, an animation, a live demo, an interactive moment, or other treatments. Be deliberate with your choice: a big number with a small label, supporting stats, and a gradient accent is the default treatment, so only use it if that's truly the best option.

Typography carries the personality of the page. You don't need a different typeface for display or headline text and body content: use one family or two, and if two, make them clearly distinct.

Choose your typefaces deliberately, not the default families you would reach for on any other project, and set a clear type scale following the default guidance of The Elements of Typographic Style with intentional weights, widths, and spacing. When type is used as a headline or visual element, use the type treatment itself as an active part of the design, not a neutral delivery vehicle for the content.

Default to line lengths of less than 80 characters. Serif typefaces can have slightly longer line lengths; give serif body text slightly more line-height than a sans-serif.

Avoid these default typographic treatments; they are the commonest tells of a generated page:
- Accenting just a single word or phrase in a headline, like putting one word in italic/bold or a different color.
- Using all caps for labels.
- Adding unnecessary typographic labels above content.

Visual structure is information. Structural devices like outlines, borders, numbering, eyebrows, dividers, labels, etc., encode useful information about the content rather than decorate it. Many generic designs use numbered markers (01 / 02 / 03), but that's only appropriate if the content actually is a sequence — like a stepped process or a timeline. Before adding numbered markers, check the content really is a sequence.

Use non-user-triggered motion sparingly and deliberately, only to draw attention. A single orchestrated moment — one page-load sequence or one reveal — lands better than scattered effects; fade-and-slide-up entrances on each section and hover transitions on every card are the generic default and read as AI-generated. Motion that answers a person's action (opening, expanding, confirming) is welcome when it shows what changed.

Consider written content carefully. Often a design brief may not contain real content, and it's up to you to come up with copy and placeholder content. Copy can make a design feel as templated as the design itself. See the below section on writing for more guidance.

## Process: plan, review against the brief, build, critique

For calibration, AI-generated design right now clusters around some traits:
1. a warm cream background (near #F4F1EA) with a high-contrast serif display and a terracotta or warm-clay accent (often near #D97757 — Anthropic's own Claude-interaction accent, so on a user's brief it reads as a tell);
2. a near-black background with a single bright acid-green or vermilion accent;
3. a broadsheet-style layout with hairline rules, zero border-radius, and dense newspaper-like columns;
4. the SaaS-card kit: content chopped into identical rounded cards, one border-radius on everything regardless of hierarchy, the same soft grey shadow (rgba(0,0,0,.1)) under each, and gradient washes as decoration;
5. template chrome that appears whatever the subject: a tracked-out ALL-CAPS eyebrow label above every heading; meta strings joined with middle dots ('A · B · C'); labels built as 'WORD — fragment' with a spaced em dash; tinted near-black (#0B0B0B, #111) standing in for black; a monospace face for small data labels; a '→' appended to link and button text.

All traits are legitimate for some briefs, but they are defaults rather than choices, and they appear regardless of subject. Where the brief pins down a visual direction, follow it exactly — the brief's own words always win, including when it asks for one of these looks. Where it leaves an axis free, don't spend that freedom on one of these defaults. As with a hired human designer, there's often a careful balance between doing what you're good at and taking each project as a chance to experiment and learn.

Work in two passes. First, brainstorm a short design plan based on the client's design brief: create a compact token system with color, type, layout, and principles.
- Color: describe the core base palette as 4–6 named hex values.
- Type: the typefaces and their roles.
- Layout: a layout concept, using one-sentence prose descriptions and ASCII wireframes to ideate and compare. Include alignment guidance; should the content be left aligned, center aligned, justified?
- Principles: the high-level guidance for what makes this page unique.

Then review that plan against the brief before building: if any part of it reads like the generic default you would produce for any similar page (work through a similar prompt to see if you arrive somewhere similar) rather than a choice made for this specific brief — revise that part, say what you changed and why. Only after you've confirmed the relative uniqueness of your design plan should you start to write the code, following the revised plan.

When writing the code, be careful of structuring your CSS selector specificities. It's easy to generate CSS classes that cancel each other out (especially with a type-based selector like .section and an element-based selector like .cta). This can happen often with padding/margin between sections.

## Restraint and self-critique

Spend your boldness in one place. Let one element be the memorable thing, keep everything around it quiet and disciplined, and cut any decoration that does not serve the brief. Build to a quality floor without announcing it: responsive down to mobile, visible keyboard focus, reduced motion respected, visually accessible, harmonious color palettes. Critique your own work as you build, taking screenshots to review if your environment supports it — a picture is worth 1000 tokens. Consider Chanel's advice: before leaving the house, take a look in the mirror and remove one accessory. Human creatives have memory and always try to do something new, so if you have a space to quickly jot down notes about what you've tried, it can help you in future passes.

## More on writing in design

Words appear in a design for one reason: to make it easier to understand and use. They are design content, not decoration. Bring the same intentionality and minimalism to copywriting that you would bring to spacing and color. Before writing anything, ask what the design needs to say, and how it can best be said to help the person navigate the experience.

Write from the end user's perspective. Name things by what users will understand in simple language, not by how the system is built. A user manages notifications, not webhook config. Describe what something is or does in plain terms rather than selling it. Being specific and legible to new users is always better than being clever.

Use active voice as default. A CTA says exactly what happens when it is used: "Save changes," not "Submit." An action keeps the same name through the whole flow, so the button that says "Publish" produces a toast that says "Published." The vocabulary of an interface is the signposting for someone navigating the product. Cohesion and consistency are how people learn their way around.

Treat failure and emptiness as moments for direction, not mood. Explain what went wrong and how to fix it, in the interface's voice rather than a person's. Errors don't apologize, and they are never vague about what happened. An empty screen is an invitation to act.

Keep the tone conversational: plain verbs, sentence case, no filler, with tone matched to the brand and the audience. Let each written element do exactly one job.
