You are acting as a senior information architect, UX researcher, accessibility specialist, internal-audit methodology analyst, and production front-end engineer.

## Objective

Create a production-quality internal audit methodology web application from the four attached methodology PDFs:

1. Planning
2. Fieldwork
3. Reporting
4. Issue validation

The primary deliverable must be one completely self-contained, portable HTML file named `audit-methodology-app.html`.

It must:

- Open directly from the local filesystem in Microsoft Edge without requiring a web server.
- Work fully offline after delivery.
- Contain all required HTML, CSS, JavaScript, methodology content, icons, diagrams, and other assets.
- Provide app-like “subpages” through internal views and hash-based navigation.
- Require no installation, build process, external libraries, CDNs, APIs, external fonts, or network requests.
- Be suitable for real internal production use, not merely a mock-up or proof of concept.

I am also providing corporate branding materials. Apply them consistently, subject to accessibility requirements. If a corporate font cannot legally or technically be embedded, use an appropriate system-font fallback and document that decision.

## Intake check

Before beginning, confirm that you can access and read:

- All four methodology PDFs, including their tables, diagrams, footnotes, and appendices.
- The corporate brand guide and supplied visual assets.
- Document titles, versions, effective dates, section numbers, and page numbers.

If a document is missing, materially unreadable, or ambiguous in a way that prevents faithful implementation, ask one consolidated set of essential questions. Otherwise, proceed through the entire assignment without pausing for approval.

Do not provide a wireframe or plan in place of the completed application.

## Confidentiality and research boundary

The methodology documents are confidential internal source material.

You may conduct public web research, but:

- Use only generic research queries.
- Never place the organization’s name, filenames, internal terminology, document excerpts, methodology rules, thresholds, examples, metadata, or other confidential information into a public search query or external service.
- Never transmit or expose internal document content outside the approved Microsoft 365 environment.
- Do not use external AI services, analytics, telemetry, APIs, or hosted tools.
- Do not include outbound network calls in the finished application.
- Do not add telemetry or usage tracking.

Public research may inform the app’s information architecture, navigation, search, interaction design, accessibility, and presentation. It must not add, replace, reinterpret, or override internal audit requirements.

## Public design research

Before designing the application, research current best practices for information-dense applications, including:

- Technical documentation portals
- Enterprise knowledge bases and wikis
- Complex workflow applications
- Policy and procedure libraries
- Guided task and onboarding experiences
- Search-heavy information systems
- Decision-support applications
- Accessible enterprise design systems

Consider several credible sources rather than copying one product. Prefer official design-system documentation and primary sources. Research topics such as:

- Progressive disclosure
- Navigation and wayfinding
- Information scent
- Search and filtering
- Guided workflows for inexperienced users
- Presenting dense requirements in understandable layers
- Decision trees and interactive guidance
- Accessibility
- Responsive enterprise layouts
- Reducing cognitive load
- Connecting reference material with immediate actions

Record the source name, publisher, URL, access date, useful pattern, and how that pattern influenced the application. If live web research is unavailable, state that honestly; do not invent sources or claim research was performed.

Public research URLs belong in the supporting design-research report. The app itself must not require those sites to function.

## Methodology fidelity

The four internal methodology documents are the only authoritative source for audit requirements.

Follow these rules:

- Do not invent audit requirements, thresholds, formulas, approvals, timelines, roles, evidence expectations, definitions, or decision criteria.
- Preserve the meaning of “must,” “shall,” “required,” “should,” “may,” and similar terms.
- Preserve all conditions, exceptions, dependencies, footnotes, and qualifying language.
- Use the methodology’s terminology consistently.
- Do not silently resolve contradictions between documents.
- Identify ambiguous, conflicting, missing, or outdated content in a clarification register.
- If simplifying wording for junior auditors, preserve the original meaning and provide the source citation beside the explanation.
- Clearly distinguish authoritative methodology from optional application assistance.

Use the following visible classifications where appropriate:

- Methodology requirement
- Methodology guidance
- Optional application helper
- Example
- Approval or quality checkpoint
- Required evidence or deliverable

Optional helpers must never be presented as mandatory methodology.

## Source traceability

Create a comprehensive requirement inventory before building the interface. Capture, where available:

- Source document
- Document version and effective date
- Section and subsection
- Printed page number
- PDF page number if different
- Requirement or guidance statement
- Applicable phase
- Trigger or condition
- Responsible role
- Required action
- Required evidence or output
- Approval or review point
- Related requirements
- App location where it is presented

Every authoritative requirement displayed in the app must have an adjacent citation. Use a consistent format such as:

`Planning Methodology, §2.3, document p. 9 / PDF p. 12`

Provide source citations at the requirement or step level, not only at the bottom of a long page. Make citations easy to copy.

Confirm that every identified methodology requirement appears in the app or is recorded in an omission/clarification register with a reason.

## Intended users and content design

The primary users include junior auditors with little or no prior methodology knowledge.

The experience must help them answer:

- Where am I in the audit lifecycle?
- What am I trying to accomplish?
- What applies to this situation?
- What must I do now?
- What evidence or deliverable must I produce?
- Who reviews or approves it?
- What happens next?
- Where did this requirement come from?

Write concise, action-oriented interface copy without making the content childish. Define unfamiliar terminology and provide examples only when supported by the methodology or clearly labelled as optional examples.

Avoid long, uninterrupted walls of text. Use progressive disclosure so users can see the essential action first and expand explanations, examples, exceptions, and original source context when needed.

## Information architecture

Build a coherent app with at least these primary views:

- Home / Start here
- Planning
- Fieldwork
- Reporting
- Issue validation
- Tools
- Glossary
- Search results
- Methodology sources and versions
- About this application / data privacy

Use hash-based routes so browser back, forward, refresh, bookmarks, and direct links work from a local file, for example:

- `#/home`
- `#/planning`
- `#/fieldwork`
- `#/reporting`
- `#/issue-validation`
- `#/tools`
- `#/glossary`

The home view should include:

- A clear explanation of the four-phase lifecycle
- A visual lifecycle diagram derived from the documents
- A “Where should I start?” navigator
- Quick links to common tasks
- Recently visited or bookmarked sections
- Progress across the four phases
- Global search

Each phase should present a logical journey such as:

1. Purpose and expected outcome
2. When the phase begins
3. Prerequisites and required inputs
4. Step-by-step actions
5. Mandatory requirements
6. Decisions and conditional paths
7. Evidence and documentation expectations
8. Deliverables
9. Review and approval checkpoints
10. Common mistakes or quality checks
11. Completion criteria
12. What happens next
13. Related sections and tools
14. Source references

Only include an item when supported by the relevant methodology or clearly labelled as optional assistance.

Connect the phases logically. Show how documented outputs from one phase become inputs to another without inventing connections that are not supported by the source material.

## Navigation and search

Provide:

- Persistent primary navigation
- Clear current-location indicators
- Breadcrumbs
- Page-level table of contents
- Previous and next logical step controls
- Cross-links between related requirements
- Browser-compatible deep links
- Mobile and narrow-screen navigation
- Keyboard-accessible navigation
- A global search shortcut such as Ctrl/Cmd+K
- Search filters for phase, content type, requirement status, tool, and glossary term
- Search-term highlighting
- Useful empty and no-results states
- Synonyms based on terminology found in the PDFs
- Bookmarks or favourites stored locally

Search must be entirely offline and must index the embedded methodology content.

## Interactive tools

Create useful interactive tools derived from the documents, including where supported:

- Planning calculators
- Methodology-based decision trees
- Risk matrices
- Impact and likelihood assessment guides
- Impact and likelihood write-up builders
- Phase checklists and progress trackers

Also identify other genuinely useful day-to-day tools supported by the methodology.

Every calculator or decision tool must show:

- Purpose
- Applicable methodology phase
- Required inputs
- Input validation
- Formula, scoring logic, or branch logic
- Methodology source for each authoritative rule
- Assumptions
- Clear result and interpretation
- Why the result was reached
- Reset control
- Print or copy capability
- Accessible error messages
- Worked example when appropriate

Do not invent numerical scales, formulas, thresholds, or rating criteria. If the methodology does not define a required value, either:

- Do not calculate it, or
- Present a configurable optional helper clearly labelled as non-authoritative.

Never declare that an engagement or issue is compliant solely because a calculator produced a result.

The impact and likelihood write-up tool should help a junior auditor produce a clear draft through structured prompts. It should provide an editable preview and copy function while keeping all entered information on the local device.

Use graphs and visualizations only when they communicate real information, such as:

- Audit-lifecycle progress
- Risk heat maps
- Decision paths
- Requirement relationships
- Phase flows

Do not create decorative charts using fabricated data. Give every diagram or visualization an accessible text alternative.

## Local data

Use versioned, namespaced browser storage for:

- Progress
- Checklist status
- Bookmarks
- Tool inputs
- Saved local drafts
- User preferences

Include:

- A clear indication that data remains in that browser
- A warning that ordinary browser storage is not encrypted
- Export to a local JSON file
- Import from a previously exported JSON file
- A visible “clear my data” function with confirmation
- Graceful behaviour when browser storage is unavailable

Prevent user-entered text from being interpreted as executable HTML or JavaScript. Do not use `eval`, dynamic code execution, or unsafe rendering of user input.

## Visual and interaction design

Create a modern, restrained, professional enterprise interface using the supplied corporate brand.

The design should prioritize:

- Strong visual hierarchy
- Readable typography
- Consistent spacing
- Clear required-versus-optional distinctions
- Useful status indicators
- Accessible contrast
- Comfortable content width
- Fast scanning
- Predictable interaction patterns
- Visible keyboard focus
- Helpful hover and focus states
- Minimal, purposeful animation
- Reduced-motion support

Avoid gratuitous gradients, excessive card layouts, fake dashboards, decorative charts, tiny text, excessive animation, and visual complexity that makes requirements harder to locate.

The interface must be responsive and usable at approximately 360px, 768px, and 1440px widths.

Meet WCAG 2.2 AA as far as a standalone HTML application reasonably can, including:

- Semantic HTML
- Logical heading structure
- Skip link
- Keyboard operation
- Focus management after route changes
- Accessible names and descriptions
- Sufficient color contrast
- No meaning conveyed by color alone
- Appropriate live-region announcements
- Large enough interaction targets
- Accessible forms and validation
- Screen-reader-friendly diagrams and tables
- Reduced-motion support
- Print-friendly layouts

## Technical requirements

The application must:

- Be delivered as one readable and maintainable HTML file.
- Open successfully through a `file://` URL.
- Use vanilla HTML, CSS, and JavaScript.
- Use classic embedded scripts if module loading would cause local-file restrictions.
- Use hash routing rather than server-dependent routing.
- Embed CSS, JavaScript, icons, diagrams, and approved brand assets.
- Use inline SVG or CSS for icons and diagrams.
- Use a system font stack unless an approved, licensed font is supplied for embedding.
- Make no calls through `fetch`, XMLHttpRequest, WebSocket, Beacon, external images, external stylesheets, remote scripts, or external fonts.
- Contain no analytics, tracking pixels, cookies, telemetry, advertising, or remote error reporting.
- Use a restrictive Content Security Policy compatible with the necessary embedded scripts and styles, including `connect-src 'none'`.
- Avoid features known to fail when an HTML file is opened locally.
- Have no broken links, controls, routes, or placeholder content.
- Contain no TODOs, lorem ipsum, pseudocode, or “implementation left to the user.”
- Display the methodology versions and application build date.
- Provide print styles for methodology content, checklists, and tool results.
- Remain responsive with the full methodology content loaded.
- Use a structured internal content model so future methodology updates are manageable.

## Quality assurance and revision

Before delivering the app, perform a complete QA pass and revise the application based on the findings.

Test or inspect, as available:

### Content accuracy

- Requirement coverage against all four PDFs
- Fidelity of mandatory language
- Conditions and exceptions
- Page and section citations
- Document-version information
- Cross-phase links
- Uncited normative statements
- Unsupported or invented content

### Functional behaviour

- All routes and navigation
- Browser back and forward
- Direct hash links and refresh
- Global search and filters
- No-result states
- Calculators and representative boundary values
- Decision-tree branches
- Risk-matrix behaviour
- Write-up guide
- Checklists and progress
- Bookmarks
- Local persistence
- Export and import
- Clear-data function
- Print layouts
- Keyboard use
- Error handling

### Offline and privacy

- Open the file locally with network access disabled.
- Confirm that all content and features still work.
- Confirm there are no attempted network requests.
- Confirm there are no external runtime dependencies.
- Confirm no confidential methodology content is sent anywhere.

### UI, UX, and accessibility

Evaluate the completed app using at least these junior-auditor scenarios:

1. Find the first required planning actions.
2. Determine what evidence is required for a fieldwork step.
3. Follow a methodology decision path.
4. Assess impact and likelihood and draft the supporting rationale.
5. Find a reporting requirement using search.
6. Determine how to validate and close an issue.

Review visual hierarchy, readability, navigation, cognitive load, responsive layouts, keyboard operation, focus behaviour, form clarity, empty states, error states, and accessibility.

Correct identified issues and repeat the affected checks before delivery.

Do not claim that a test was executed if it was only inspected or reasoned about. Clearly distinguish executed tests from manual code review and identify any remaining limitations.

## Deliverables

Create a folder named:

`internal-audit-methodology-web-app`

Place these files inside it:

1. `audit-methodology-app.html`  
   The complete self-contained application and only file required at runtime.

2. `README.txt`  
   Opening instructions, supported browsers, local-storage explanation, privacy notes, version information, and update instructions.

3. `methodology-traceability.csv`  
   The requirement-to-source-to-app-location mapping.

4. `design-research.md`  
   Public sources considered, access dates, patterns adopted, patterns rejected, and resulting design decisions.

5. `qa-report.md`  
   Tests performed, scenarios reviewed, issues found, changes made, accessibility checks, offline verification, and remaining limitations.

6. `methodology-clarifications.md`  
   Ambiguities, conflicts, unreadable material, unresolved questions, and intentional omissions.

The supporting files must not be runtime dependencies. Deleting them must not break the HTML application.

Package the folder as a downloadable ZIP if the environment supports it. Do not paste a partial or truncated HTML file in place of the deliverable. If a platform limitation genuinely prevents file creation, state the exact limitation and use the safest available method to provide the complete, untruncated files.

## Completion response

When finished, provide:

- A link to the completed folder or ZIP
- A link to the standalone HTML file
- A concise list of included features
- The methodology versions used
- Confirmation of offline operation
- A summary of QA performed
- Any unresolved methodology questions or technical limitations

Do not describe the work as complete until the application, traceability review, UI/UX revision, and offline QA pass have all been completed.