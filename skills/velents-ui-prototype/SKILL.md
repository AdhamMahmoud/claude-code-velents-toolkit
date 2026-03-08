---
name: velents-ui-prototype
description: Protocol for reading prototypes (Figma, screenshots, design files) and implementing pixel-perfect UI from them. Covers prototype understanding, question protocol, implementation mapping, and Chrome validation.
---

# Velents UI Prototype Protocol

When a PM provides a prototype (Figma link, screenshots, video, or any design artifact), this protocol is **MANDATORY** before writing a single line of frontend code.

> **The most expensive UI bug is one discovered after implementation. The cheapest fix is a question asked before coding starts.**

---

## Phase 1: Prototype Ingestion (READ EVERYTHING FIRST)

Do NOT write any code in this phase. Only read, observe, and document.

### 1a. Open and Screenshot the Prototype

Using browser automation (`mcp__claude-in-chrome__navigate`):

1. Navigate to the prototype URL (Figma, InVision, Framer, Zeplin, Loom, etc.)
2. Take a full-page screenshot of EVERY screen/state in the prototype
3. If it's a Figma file — inspect each frame; do NOT just look at one artboard

For each screen, document in a `prototype-analysis.md` file:

```markdown
## Screen: [Screen Name]

### Layout
- Page structure: [sidebar + main? full-width? split panel?]
- Grid: [columns, gaps, max-width]
- Sticky elements: [header? sidebar? footer?]

### Typography
- Headings: [size, weight, color token]
- Body text: [size, weight, color token]
- Labels: [size, weight, color token]
- Any special text treatment: [caps, underline, truncation]

### Colors (map to semantic tokens)
- Primary actions: [button color → bg-primary?]
- Backgrounds: [page bg, card bg, sidebar bg]
- Borders: [card borders, dividers, input borders]
- Status colors: [success, warning, error, info]

### Spacing
- Page padding: [horizontal padding at different breakpoints]
- Card padding: [internal padding]
- Gap between cards/rows: [gap-4? gap-6?]
- Section spacing: [vertical rhythm between sections]

### Components used
- [List every visible component: Table, Card, Badge, Button, Input, Select, Dialog, Tabs, etc.]

### States shown
- [ ] Default/loaded state
- [ ] Loading state (skeletons shown?)
- [ ] Empty state (what is shown when no data?)
- [ ] Error state (what is shown on API error?)
- [ ] Hover states (visible on any interactive elements?)
- [ ] Active/selected states
- [ ] Disabled states

### Interactions
- [What happens on row click?]
- [What triggers a modal/sheet?]
- [What filters/sorts are interactive?]
- [Any animations or transitions?]

### Responsive behavior
- [Does the layout change at mobile/tablet?]
- [Does the sidebar collapse?]
- [Are any columns hidden at small screens?]
```

### 1b. Extract All Edge Cases Shown

Look specifically for:
- What does the prototype show when the list is empty?
- What does it show when there's an error?
- What does a loading state look like (skeleton placeholders? spinner? nothing)?
- Are there any tooltips, popovers, or help text that only appear on hover?
- Are there any confirmation dialogs before destructive actions?
- Are there breadcrumbs, page titles, or any navigation context shown?

Document EVERY state you see. If a state is NOT shown in the prototype, that is a question to ask.

---

## Phase 2: Gap Analysis — What the Prototype Does NOT Show

Before asking questions, identify all the gaps yourself:

```markdown
## Missing from Prototype

### States not shown:
- Loading state for [X] — assumed [skeleton/spinner/none]?
- Empty state for [X] — what message? what CTA?
- Error state for [X] — toast? inline error? full error page?

### Interactions not shown:
- What happens after [action]? Does the page refresh? Show toast? Navigate?
- What happens when [user does X]?

### Responsive behavior not shown:
- Mobile layout for [X]?
- Does [sidebar/table/card grid] stack on mobile?

### Copy not finalized:
- Button label for [action]: "[Draft text]" — final?
- Empty state message: "[Draft text]" — final?
- Confirmation dialog text for delete: — not shown, need copy

### Data not shown:
- Maximum length for [field] — how does truncation look?
- What if [number] is very large (e.g., 1,000,000 calls)?
- What if the name is 200 characters long?
```

---

## Phase 3: Prototype Understanding Questions (MANDATORY STOP)

After completing Phases 1 and 2, **STOP** and ask the PM/designer the following questions before writing any code.

Format questions as a numbered list. Group by category. Be specific — quote from the prototype.

```
## Prototype Questions — [Feature Name]

I've read the prototype fully. Before implementing, I need answers to [N] questions:

### States
1. The prototype doesn't show a loading state for the agent list. Should I use:
   - **A)** Row-by-row skeleton (matching the table structure)
   - **B)** Full-page spinner centered in the content area
   - **C)** Existing `<TableSkeleton>` component from the UI inventory
   **Recommend: C** — matches existing patterns.

2. The empty state for the agent list is not shown. Should I:
   - **A)** Show "No agents yet" with a "Create your first agent" CTA button
   - **B)** Show the existing `<EmptyState>` component from `/components/shared/empty-state.tsx`
   - **Recommend: B** — reuses existing component, consistent.

### Interactions
3. When a row is clicked, does it:
   - **A)** Navigate to `/agents/[id]` detail page
   - **B)** Open a sheet panel sliding in from the right
   - **C)** Open a dialog modal
   **Recommend: A** — matches how other entity lists work.

### Copy
4. The delete confirmation text is not shown. Is it:
   - **A)** "Are you sure you want to delete [agent name]? This action cannot be undone."
   - **B)** [Provide exact copy]

### Responsive
5. At mobile width, does the sidebar:
   - **A)** Collapse to a hamburger menu (existing behavior — no change needed)
   - **B)** Hide completely and show bottom navigation
```

**DO NOT START CODING until all questions are answered.**

---

## Phase 4: Pre-Implementation Mapping

After questions are answered, create a component map before writing any code:

```markdown
## Component Map — [Feature Name]

### Existing components to REUSE (from velents-ui-inventory):
| Prototype Element | Existing Component | File Path | Props to pass |
|---|---|---|---|
| Page header with title + action button | `PageHeader` | `components/shared/page-header.tsx` | `title`, `action` |
| Data table with sorting | `DataTable` | `components/ui/data-table.tsx` | `columns`, `data` |
| Status badge | `Badge` | `components/ui/badge.tsx` | `variant="success"` |
| Delete confirmation | `AlertDialog` | `components/ui/alert-dialog.tsx` | standard pattern |

### New components needed (justify each):
| Component | Reason existing ones can't be reused | File path |
|---|---|---|
| `AgentStatusToggle` | No existing toggle shows text + icon + animation together | `components/agents/agent-status-toggle.tsx` |

### Pixel values → Tailwind mapping:
| Prototype value | Tailwind class |
|---|---|
| 16px horizontal padding | `px-4` |
| 24px gap between cards | `gap-6` |
| 14px body text | `text-sm` |
| 12px label text | `text-xs` |
| #6B7280 secondary text | `text-muted-foreground` |
```

---

## Phase 5: Chrome Validation Protocol (MANDATORY after implementation)

After all frontend files are written and the dev server is running:

### 5a. Side-by-Side Comparison

Using `mcp__claude-in-chrome`:

1. **Navigate to the implemented page**: `mcp__claude-in-chrome__navigate` to `http://localhost:3000/[route]`
2. **Screenshot the implementation**: `mcp__claude-in-chrome__computer` → take screenshot
3. **Open the prototype**: `mcp__claude-in-chrome__navigate` to the Figma/prototype URL
4. **Screenshot the prototype**: take screenshot

For each screen, compare:

```markdown
## Chrome Validation Report: [Screen Name]

### Layout
- [ ] Page structure matches (sidebar position, content area width)
- [ ] Max-width matches
- [ ] Page padding matches

### Typography
- [ ] Heading size and weight match
- [ ] Body text size matches
- [ ] Color matches (use DevTools to inspect exact CSS values)

### Spacing
- [ ] Card padding matches
- [ ] Gap between cards matches
- [ ] Section spacing matches

### Components
- [ ] Table columns and widths match
- [ ] Badge colors and text match
- [ ] Button sizes and variants match

### States (test each one)
- [ ] Loading state: navigate to page with slow network (Chrome DevTools throttle)
- [ ] Empty state: test with empty data fixture
- [ ] Error state: test with API endpoint returning 500

### Interactions
- [ ] Row click → correct navigation/action
- [ ] Delete → confirmation dialog appears → delete executes
- [ ] Toast notifications appear after actions

### RTL (if Arabic locale enabled)
- [ ] Switch locale to Arabic, reload page
- [ ] Layout mirrors correctly (sidebar on right, text right-aligned)
- [ ] No layout breaks or overflow

### Discrepancies found:
| Element | Prototype | Implementation | Fix |
|---|---|---|---|
| [e.g., Card padding] | 24px | 16px | Change `p-4` to `p-6` |
```

### 5b. Fix All Discrepancies Before Marking Complete

Every discrepancy found in 5a MUST be fixed before the implementation is marked complete. Do not file them as "minor" or "nice to have" — pixel-perfect means pixel-perfect.

After fixing, re-screenshot and verify the fix closed the gap.

### 5c. DevTools Inspection for Exact Values

For any spacing/color discrepancy use Chrome DevTools:

```javascript
// In mcp__claude-in-chrome__javascript_tool:
// Inspect computed styles of a specific element
const el = document.querySelector('[data-testid="agent-card"]');
const styles = window.getComputedStyle(el);
console.log('padding:', styles.padding);
console.log('background:', styles.backgroundColor);
console.log('font-size:', styles.fontSize);
```

---

## Prototype Formats Supported

| Format | How to Access |
|---|---|
| Figma URL | Navigate in Chrome, inspect each frame. Use "Present" mode for full preview |
| Screenshot/image | Read with Read tool (Claude is multimodal) |
| Loom video | Navigate in Chrome, pause at each key state, screenshot |
| InVision | Navigate in Chrome |
| Framer | Navigate in Chrome |
| PDF | Read with Read tool (PDF support) |
| Hand-drawn sketch (photo) | Read with Read tool (multimodal) |

---

## When to Escalate to the PM

Escalate (do not guess) when:
- The prototype shows a state that is technically impossible given the data model
- Two screens in the prototype contradict each other
- A required interaction is not shown anywhere in the prototype
- The color/token in the prototype doesn't exist in the Velents design system

Never make a design decision silently. Always surface it.
