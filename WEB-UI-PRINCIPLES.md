# Web UI, UX, and Data Principles

> A living document. `ENG-PRINCIPLES.md` §4 states the general web doctrine — functional, neat-looking, minimalistic; this file expands it into UI, UX, and data rules for our web applications, document-style local apps in particular. The house maxim — **local-first, offline-first, CRDT-first** — is enforced here, not merely admired (§5, §7). Each principle is stated as an imperative so it can be checked, enforced, and argued about. Product features appear only as examples; packdiff's review player is the running one. Section references here (§N) name this file's own sections; references into the sibling are spelled `ENG-PRINCIPLES §N`.

---

## 1. The interface is a document

- **The app is an offline-first document, not a smaller imitation of a hosted site.** Present content as a human-readable sequence of named sections (packdiff: Description, Commits, Files changed, Diff), readable top to bottom without interaction.

- **ALWAYS put a plain-language summary above the navigation.** The summary scrolls away; the single navigation row stays pinned.

- **Navigation names exact destinations.** It scrolls and moves focus to the relevant item, not merely to a nearby section — a link that lands "nearby" makes the reader search again.

- **Explicit navigation creates browser history; passive scrolling does not.** Scrolling updates the URL with `replaceState`, so copied links stay useful without polluting Back.

- **Back and Forward restore view changes, never data mutations.** Meaningful view changes — including selection ranges — are restored; mutations are never replayed.

## 2. Stability is a feature

- **Nothing jumps.** A layout shift steals the reader's place — and sometimes their click. Pinned chrome stays one row and controls reserve stable space; menus, messages, and disclosures overlay or scroll away.

- **Preserve DOM identity across updates.** Mutate an existing element in place; recreate it only when its identity truly changes or its implementation cannot accept the required update. Replacement destroys browser-owned state — selection, focus, fullscreen, playback, and scroll position — and makes stable content jump.

- **Context populates reserved space without changing geometry.** Pinned chrome may fill as the user scrolls, but its dimensions never move.

- **Stabilize the point of a direct, local interaction** when opening or closing inline UI. Do not fight ordinary scrolling with global compensation.

- **A disabled action stays in place**, grayed out with an explanation, rather than vanishing (packdiff: Side-by-side when horizontal space is insufficient). A control that disappears reads as a feature that never existed.

- **Success feedback NEVER resizes controls.** Respect reduced motion.

## 3. Prefer direct, obvious interaction

- **The whole meaningful region is the interaction target.** A small icon or gutter may reinforce the affordance but must never be the only doorway — a gutter-sized target turns interaction into a precision exercise (packdiff: the whole valid diff line accepts a comment; the `+` gutter only reinforces).

- **Direct manipulation produces the result directly** — no redundant Apply step (packdiff: dragging across commits selects and renders that range on release).

- **An interactive status performs its most useful action** rather than opening intermediary UI (packdiff: a file's comment count jumps to its first comment).

- **Every programmatic landing gets a subtle, temporary arrival cue.** A jump the user cannot see landing reads as a jump that did nothing.

- **Paths are navigable breadcrumbs.** Ancestors are actionable, stay on one line, and reveal with horizontal touch or trackpad scrolling. NEVER replace navigable information with a destructive ellipsis.

## 4. Progressive disclosure, not clutter

- **Every persistent control must justify its space and attention.** Prefer strong defaults, direct manipulation, and contextual controls over permanent chrome.

- **Secondary metadata appears at its relevant location** through restrained disclosure — persisted and discoverable, not permanently on screen.

- **Do not fill empty states with redundant labels.** Reserve layout space when stability requires it, but an empty state needs no visible zero-count label (packdiff: zero comments shows no `0 comments`).

- **Keep rich models beneath minimal interfaces.** Full functionality does not mean showing everything at once.

- **Avoid duplicate navigation.** When a content section already indexes the document, a permanent sidebar restating it is clutter (packdiff: Files changed is the file index).

## 5. Local-first, offline-first, CRDT-first

- **Local-first, offline-first, CRDT-first.** The app is fully functional with no server and no network; the local model is the one source of truth; and anything that later synchronizes merges *into* that model deterministically (§7), never replaces it. Every bullet in this section and §6–§7 exists to keep this maxim true.

- **ALWAYS identify a document by a hash of its canonical content** — never by filename, timestamp, title, or presentation. Names and rendering change; content identity must not.

- **Identical content restores identical local state** after rename, move, regeneration, or restart. Different content starts with clean state; state is never fuzzily moved between documents.

- **Durable state survives restart:** user-authored data and deliberate presentation choices alike. Local storage is today's mechanism, not the product contract.

- **Exports are today's safe push paths** (packdiff: `Copy as Markdown`, `Copy as JSON`). Future upstreams extend the local model rather than replace it.

## 6. Visible data is an integrity invariant

- **User-authored data may be collapsed but NEVER made invisible.** A collapsed container shows counts of what it holds; expanding reveals the data in place (packdiff: a collapsed file shows its comment and draft counts).

- **Drafts reopen visibly after restart.** A view that excludes user data reports how much is outside it and offers a direct route to the full view.

- **Warnings and errors NEVER reflow the document.** They may collapse to a visible status indicator, but never disappear without acknowledgement; unrelated work stays available whenever safe.

- **Sensitive mutations confirm inline and are reversible.** Deletion in particular requires deliberate confirmation and belongs to a persistent per-document undo history.

## 7. Two tiers of durable state

- **Material changes are auditable, undoable, and future-pushable:** the data the user came to produce (packdiff: comments, viewed status, deletions/restorations, and future shared events).

- **Local state is durable for recovery and comfort but not upstream material:** drafts, collapse state, theme, wrapping, and similar preferences.

- **Defaults start expanded; user state overrides defaults.** Deliberate view state is user data. Externally authored defaults may change the starting point; they never override the user's explicit choice.

- **Both tiers are preserved and inspectable.** Any pinned change count reports material changes only.

- **Model material changes as stable-ID journal operations, not whole-document replacement.** This is the CRDT-first leg of §5's maxim: journal operations merge deterministically when synchronization arrives, without implementing it prematurely.

## 8. Fast and accessible by construction

- **Local filtering, sorting, navigation, and view changes feel instant** and execute client-side (JavaScript/WASM), never round-tripping to a server.

- **Pointer, keyboard, touch, and assistive technology expose the same actions.** Shortcuts accelerate functionality; they never own it exclusively.

- **Navigation moves both viewport and focus.** Focus treatment is visible and layout-neutral.

- **Unmodified single-key shortcuts stand down in every typing context.** Teach important shortcuts where their actions live, summarize them in one discoverable place, and show platform-appropriate Command/Control notation.

- **Meaning NEVER depends on color alone.** Not every reader perceives color; pair it with shape, text, or position.
