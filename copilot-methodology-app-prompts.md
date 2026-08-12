# Copilot Prompt Pack — Internal Audit Methodology Web App

A staged sequence of prompts to feed Microsoft Copilot, in order, in **one continuous chat session** (Copilot needs the earlier stages in context). A fallback single-prompt version is at the end.

**How to use:**

1. Start a fresh Copilot chat. Upload the four methodology documents (planning, fieldwork, reporting, issue validation) and the supporting documents for fieldwork and reporting.
2. Paste **Stage 0** first, then paste each stage after the previous one finishes. Review each stage's output before moving on — Stages 2 and 3 are your checkpoints to correct anything.
3. If Copilot cuts off mid-output at any point, reply exactly: `Continue from where you stopped, in a code block, without repeating anything.`
4. If your Copilot cannot upload all documents at once due to size limits, upload only the relevant document at each build stage (Stages 5–8) and note that in your message.

---

## Stage 0 — Mission briefing (paste first, with documents attached)

```
You are an expert product designer and front-end engineer. Over this conversation you will build a single self-contained HTML file: an interactive internal audit methodology web app for my audit team.

THE MISSION
The app turns our four methodology documents (Planning, Fieldwork, Reporting, Issue Validation — attached, along with supporting documents for Fieldwork and Reporting) into a modern, searchable, interactive reference that a brand-new junior auditor with zero prior knowledge can use to: (1) navigate to the right section immediately, (2) understand exactly what is required of them, and (3) know what action to take next.

NON-NEGOTIABLE REQUIREMENTS — treat these as fixed for the entire conversation:
1. COMPLETENESS: Every requirement, mandatory step, deliverable, approval, threshold, and rule in the source documents must appear in the app. You may restructure, reorder, and reformat content into digestible interactive forms, but you may never drop, weaken, or paraphrase away a requirement. Juniors will treat this app as the source of truth.
2. SINGLE FILE: The final product is exactly one .html file. All CSS, JavaScript, data, and icons inline. No external libraries, no CDN links, no internet dependency. It must work when opened directly from a file share or SharePoint.
3. NAVIGATION: A landing page routes to sub-pages for Planning, Fieldwork, Reporting, and Issue Validation, using hash-based routing (#/planning etc.) so browser back/forward and bookmarking work. Persistent navigation is visible on every page.
4. SEARCH: Instant client-side search across all content, reachable from everywhere (including a keyboard shortcut), with results that deep-link to the exact section.
5. INTERACTIVITY: The app must include, where the source content supports them: clickable process flow diagrams (audit lifecycle and per-phase flows), interactive checklists with progress saved in localStorage, calculators for any formula/threshold/scoring the methodology defines (e.g. sampling, risk rating), and decision-tree wizards for judgment calls (e.g. issue rating, escalation, validation scope). Propose additional tools if the content suggests them.
6. AUDIENCE: Junior auditors, no assumed knowledge. Every page answers: What is this? When does it apply? What must I do? What do I produce? Who approves it? What comes next?
7. QUALITY: Modern, clean, professional UI. Fast. Works in Edge and Chrome. Readable information hierarchy — progressive disclosure, not walls of text.

We will work in stages: research, content analysis, design, then building the app section by section. Do not build anything yet.

For now: confirm you have read all attached documents, list each document you can see with a one-line description of its contents, and flag anything you cannot open or that appears truncated.
```

> **Checkpoint:** Make sure Copilot lists every document. If any are missing or truncated, fix that before continuing.

---

## Stage 1 — Best-practice research

```
Stage 1: Research. Before designing anything, research how the best information-dense reference applications solve the problems we face. Do not rely only on your general knowledge — search the web for current best practices and real examples. Study patterns from products like: developer documentation sites (Stripe, MDN), government service manuals (GOV.UK Service Manual and design system), internal wikis and knowledge bases (Notion, Confluence), interactive learning tools, and any well-regarded policy/procedure portals you find.

For each of the following, report what the best examples do, and the concrete pattern we should adopt:
1. Navigation for deep content hierarchies (sidebars, breadcrumbs, in-page tables of contents, previous/next flows)
2. Search UX (instant results, keyboard shortcuts like "/" or Ctrl+K, result previews, deep-linking)
3. Progressive disclosure of dense content (accordions, expandable detail, summary-first layouts, "at a glance" boxes)
4. Orienting first-time users (landing pages, "start here" paths, role- or task-based entry points)
5. Making requirements actionable (checklists, callout boxes for mandatory vs. recommended, step-by-step patterns)
6. Interactive tools embedded in reference content (calculators, wizards, decision trees)
7. Visual design conventions that make dense content scannable (typography scale, spacing, color coding by section)

Output: a design principles brief of 15–25 numbered, specific, actionable principles, each citing which product/pattern inspired it. These principles will govern everything we build. If you cannot search the web, say so explicitly and produce the brief from your knowledge instead — do not pretend to have searched.
```

---

## Stage 2 — Content inventory

```
Stage 2: Content inventory. Go through all attached documents systematically and produce a complete structured content map. Nothing gets built until this inventory is right, and the finished app will be checked against it.

For EACH of the four methodology areas (Planning, Fieldwork, Reporting, Issue Validation), extract:
1. STRUCTURE: The full outline of topics/sections in logical order.
2. REQUIREMENTS: Every mandatory item — anything phrased as must/shall/required/mandatory — quoted or tightly paraphrased, with a reference to its source document and section.
3. PROCESS FLOWS: Every sequence of steps, with inputs, outputs, decision points, and who is responsible.
4. DELIVERABLES & APPROVALS: Every document, template, or artifact that must be produced, and every sign-off or approval gate.
5. FORMULAS & THRESHOLDS: Anything numeric — sample sizes, rating scales, scoring criteria, timelines/deadlines, materiality levels. These become calculators.
6. DECISION POINTS: Every "if X then Y" judgment call. These become decision-tree wizards.
7. SUPPORTING DOCUMENTS: For Fieldwork and Reporting, how each supporting document relates to the main methodology and where its content should surface in the app.
8. DEFINITIONS: Terms a junior auditor would not know. These become a glossary and hover-tooltips.

Also produce a PROPOSED INTERACTIVE TOOLS LIST: for each calculator, checklist, flow diagram, and decision tree you found support for, state which content drives it and what it does.

Present the inventory area by area. If it is too long for one response, do Planning and Fieldwork first and I will say "continue" for the rest.
```

> **Checkpoint:** Read this carefully — it's your best chance to catch missing or misread content. Correct anything wrong before Stage 3.

---

## Stage 3 — Information architecture & UX spec

```
Stage 3: Design. Using the Stage 1 design principles and the Stage 2 content inventory (with my corrections), produce the complete design specification for the app. No code yet.

1. SITEMAP: Every page and sub-section, with hash routes (e.g. #/fieldwork/testing). Show the hierarchy.
2. LANDING PAGE: What a first-time junior auditor sees — include an audit lifecycle diagram as the primary navigation device ("where am I in the audit? click your phase"), a task-based quick-start ("I need to..." shortcuts), and search front and center.
3. SECTION PAGE TEMPLATE: The standard layout every methodology page follows, answering in order: What is this? When does it apply? What must I do? What do I produce? Who approves? What's next? Show where requirements callouts, checklists, and tools sit in the layout.
4. NAVIGATION MODEL: Sidebar behavior, breadcrumbs, previous/next between logically adjacent sections, and cross-links (e.g. from a Fieldwork step to the Issue Validation criteria it triggers).
5. SEARCH DESIGN: How the index is built from content, keyboard shortcut, results UI, deep-linking behavior.
6. INTERACTIVE COMPONENTS: Final list of every diagram, checklist, calculator, and decision tree, each specced with inputs, logic, and outputs based strictly on the source content.
7. VISUAL DESIGN SYSTEM: Color palette (with a distinct accent color per methodology area), typography scale, spacing, and the styling of callouts distinguishing MANDATORY requirements from guidance.
8. STATE: What persists in localStorage (checklist progress, last visited page) and what resets.

End with any open questions where the source documents were ambiguous and you had to make an assumption — list each assumption so I can confirm or correct it.
```

> **Checkpoint:** Approve the design, answer its questions, request changes. When satisfied, say: `Approved with the changes above. Proceed to Stage 4.`

---

## Stage 4 — Build the shell

```
Stage 4: Build the application shell. Produce the single HTML file's skeleton in a code block:

1. Full document structure with the inline CSS design system from Stage 3 (CSS custom properties for the palette, typography, spacing, and callout styles).
2. Hash-based router in vanilla JavaScript rendering pages from a JavaScript content object, with browser back/forward support and a scroll-to-top on navigation.
3. App chrome: header with app title and search, sidebar navigation with the full sitemap (collapsible sections, active-state highlighting), breadcrumbs, and previous/next footer navigation.
4. Search: client-side index built from the content object, opened with Ctrl+K or "/", instant-filtering results with section context, Enter navigates and highlights the target section.
5. The complete LANDING PAGE, including the interactive audit lifecycle diagram (inline SVG, clickable phases) and the "I need to..." quick-start links.
6. Placeholder route entries for the four methodology sections — clearly marked SECTION CONTENT INSERTED IN LATER STAGES.
7. localStorage helpers for checklist state and last-visited page.

Constraints: vanilla JS only, no external resources, semantic HTML, keyboard-navigable. Write the content object so each section's content is a self-contained block that later stages can slot in without touching the shell. Output the entire file in one code block; if you run out of space, stop at a clean line and I will say "continue".
```

---

## Stages 5–8 — Build each section (one prompt per section)

Paste this four times, replacing `{SECTION}` with **Planning**, then **Fieldwork**, then **Reporting**, then **Issue Validation**. For Fieldwork and Reporting add: `Incorporate the supporting documents for this area as designed in Stage 3.`

```
Stage {N}: Build the {SECTION} section. Produce ONLY the JavaScript content block and any section-specific components for {SECTION}, exactly in the structure the Stage 4 shell expects, in one code block, with a comment at the top stating exactly where it slots into the shell file.

It must contain:
1. Every sub-page from the Stage 3 sitemap for this section, each following the standard section page template (What is this? / When does it apply? / What must I do? / What do I produce? / Who approves? / What's next?).
2. EVERY requirement from the Stage 2 inventory for this area, in mandatory-requirement callouts. Do not drop or dilute any.
3. This section's interactive components as specced in Stage 3: process flow diagram(s) as clickable inline SVG, interactive checklists wired to localStorage, calculators, and decision-tree wizards — fully functional, not placeholders.
4. Cross-links to related sections and glossary tooltips for terms defined in Stage 2.
5. Search-index entries for all of this section's content.

Before the code block, give a 5-line summary of what the section contains so I can sanity-check coverage. If the output is too long for one response, output sub-page by sub-page and I will say "continue".
```

---

## Stage 9 — Assembly & verification

```
Stage 9: Assemble and verify.

1. ASSEMBLE: Produce the complete final HTML file — shell plus all four section content blocks integrated — as one code block. If it exceeds one response, output it in clearly numbered parts that I can concatenate, each part ending at a clean boundary, and tell me exactly how to join them.

2. VERIFY COVERAGE: Then, as a separate output, go back to the Stage 2 content inventory and produce a requirements traceability table: every mandatory requirement from the inventory, and where it appears in the app (route + section). Flag any requirement that did not make it in, and provide the patch to add it.

3. QA CHECKLIST: Finally, list 10 things I should manually test when I open the file (routing, search, each calculator with a sample input and expected output, checklist persistence after refresh, back/forward behavior).
```

---

## Fallback — single comprehensive prompt

Use only if the staged approach isn't practical. Expect to iterate afterwards; verify requirement coverage yourself.

```
You are an expert product designer and front-end engineer. Using the attached internal audit methodology documents (Planning, Fieldwork, Reporting, Issue Validation, plus supporting documents for Fieldwork and Reporting), build a single self-contained HTML file: an interactive methodology web app.

First, briefly research (web search if available) how the best information-dense reference apps work — developer docs like Stripe and MDN, the GOV.UK Service Manual, wikis like Notion and Confluence — and apply their navigation, search, and progressive-disclosure patterns.

The app: a landing page with a clickable audit-lifecycle diagram and "I need to..." quick-start links, routing (hash-based, #/planning etc.) to sub-pages for each of the four areas. Every sub-page follows a standard template answering: What is this? When does it apply? What must I do? What do I produce? Who approves? What's next? Include: persistent sidebar navigation with breadcrumbs and previous/next links; instant client-side search (Ctrl+K) deep-linking to sections; clickable SVG process-flow diagrams; interactive checklists persisted in localStorage; calculators for every formula/threshold in the documents; decision-tree wizards for every judgment call; a glossary with tooltips; and mandatory-requirement callouts visually distinct from guidance.

Hard rules: (1) Every requirement, mandatory step, deliverable, approval, threshold, and rule in the source documents must appear — restructure freely, but never drop or weaken a requirement; juniors will treat this as the source of truth. (2) One .html file, all CSS/JS/data/icons inline, vanilla JavaScript, no external resources, works offline from a file share in Edge/Chrome. (3) Audience is junior auditors with zero knowledge — modern, clean, scannable UI with progressive disclosure.

Output the complete file in a code block. If it is too long for one response, stop at a clean boundary and I will say "continue". After the file, list any source requirements you could not fit, and 10 things I should manually test.
```

---

## Tips

- **Session hygiene:** Do the whole sequence in one chat. If the chat degrades (Copilot forgets earlier stages), start fresh: paste Stage 0, then paste Copilot's own Stage 2 inventory and Stage 3 spec back in as context, and resume from the build stages.
- **The traceability table (Stage 9.2) is the most important output** — it's your evidence that no requirement was lost. Spot-check it against the source documents.
- **Iterate per section:** After each of Stages 5–8, skim the 5-line summary against the source doc's table of contents before continuing.
- **File size:** If the assembled file is very large, Copilot's numbered-parts output in Stage 9 is normal — join the parts in Notepad/VS Code and save as `.html`.
